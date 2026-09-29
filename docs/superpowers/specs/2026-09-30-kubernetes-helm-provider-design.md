# Kubernetes / Helm YAML language provider — design

Date: 2026-09-30
Status: draft, not yet reviewed by a maintainer

## Problem

`bce` indexes source code (Python, JS/TS, Java, C#, Go) into a symbol/call graph. A large
category of real repositories is not source code but Kubernetes/Helm YAML: `Chart.yaml`,
`values.yaml`, `templates/*.yaml` (Deployments, Services, ConfigMaps, Secrets, Ingresses,
HPAs...), and `_helpers.tpl`. Today none of it is indexed — `docs/languages.md` lists no
YAML provider — so a GitOps/infra repo produces an empty graph and `bce context` has
nothing to answer with. An agent asking "what breaks if I change this ConfigMap" has to
grep, the exact failure mode `bce` exists to avoid for code.

## Goals (v1)

1. Extract one `Symbol` per Kubernetes resource (`apiVersion` + `kind` present) from any
   `.yaml`/`.yml` file, Helm-templated or not.
2. Extract one `Symbol` per leaf key in `values.yaml` / `values-*.yaml` — "leaf" meaning
   a key whose value is a scalar (string/number/bool/null), not a nested map or list; a
   nested map key like `image` (holding `repository`/`tag`) is not itself a symbol, only
   its scalar descendants (`image.repository`, `image.tag`) are.
3. Extract one `Symbol` per `{{- define "name" -}}` block in `*.tpl` files.
4. Resolve `{{ .Values.x.y }}` and `{{ include "name" }}` references inside templates to
   (2) and (3) as `REFERENCES` edges.
5. Resolve name-based references between resources as `REFERENCES` edges:
   `envFrom[].configMapRef.name` / `secretRef.name`, `volumes[].configMap.name` /
   `secret.secretName` → ConfigMap/Secret; Ingress `backend.service.name` → Service;
   HPA `scaleTargetRef.name` → Deployment/StatefulSet.
6. Ship as an optional grammar under the existing `[langs]` extra, same as Java/C#/Go.

## Non-goals (v1)

- Service ↔ Deployment resolution via label selectors (set matching, not a literal name —
  meaningfully different and riskier code; candidate for a follow-up).
- Running `helm template` / actual chart rendering. `bce`'s other providers are AST-only,
  no execution, no environment dependency; this stays consistent with that and with
  determinism (same source, same commit → same graph, no cluster or values-file needed).
- Kustomize-specific semantics (`patchesStrategicMerge`, `bases`, JSON6902 patches).
- Global/subchart value overrides (Helm's `global:` values, parent→subchart value
  passing) — v1 resolves `.Values.x` only within the chart that defines the template.

## Architecture

New file `src/bce/indexing/parser/languages/kubernetes_provider.py`, registered exactly
like the other five providers (`languages: tuple[str, ...] = (".yaml", ".yml", ".tpl")`).
No changes to `extractor`, `scorer`, or MCP tools — the whole point of the
`LanguageProvider` abstraction (`src/bce/indexing/parser/base.py`) is that new languages
plug in without touching the pipeline.

**Grammar:** `tree-sitter-yaml` (PyPI `tree-sitter-yaml==0.7.2`, confirmed resolvable),
added under the `langs` extra in `pyproject.toml` next to
`tree-sitter-java`/`tree-sitter-c-sharp`/`tree-sitter-go`. `.tpl` files are not YAML
(Helm's `_helpers.tpl` is pure Go-template) and are handled by a separate, much smaller
regex-only path inside the same provider — no tree-sitter parse for them.

### The "is this actually a Kubernetes file" gate

Not every `.yaml` is a Kubernetes manifest (`docker-compose.yml`, CI workflow files,
`pre-commit-config.yaml`...). Each `---`-separated document in a `.yaml`/`.yml` file is
only turned into a `RESOURCE` symbol if it has **both** a top-level `apiVersion` and
`kind` key after parsing. A document that lacks either is skipped silently — not an
error, just "not our concern" — so this provider never produces false symbols for
unrelated YAML.

### Handling `{{ }}` (the core technical problem)

Real chart templates are not valid YAML (`tag: {{ .Values.image.tag }}` is ambiguous —
`{{` opens a YAML flow mapping). The provider:

1. Splits the file on `---` into documents, tracking each document's starting line.
2. Regex-scans each document for brace-balanced `{{ ... }}` spans and replaces each with
   a short quoted placeholder scalar (`"__bce_tpl_0__"`, `"__bce_tpl_1__"`...), recording
   the original expression text and the 1-based source line it started on.
3. Parses the neutralized document with `tree-sitter-yaml`. `kind`, `metadata.name`,
   `metadata.namespace`, and the fields listed in goal 5 are read from this real AST
   (provenance `TREESITTER`).
4. Separately, every captured `{{ ... }}` expression is matched against two small,
   fixed regexes: `\.Values\.([\w.]+)` and `include\s+"([\w.-]+)"`. Matches become
   `UnresolvedRef`s (provenance `HEURISTIC` — `Provenance` already documents this label
   as "synthesized ... trusted less in scoring", which is exactly right here: this is
   text pattern matching, not a real parse of Go template semantics).

Step 4 does not attempt to evaluate the template — `{{ if .Values.enabled }}` or
`{{ range .Values.items }}` are not modeled, only direct `.Values.x.y` / `include "name"`
references are extracted. This is a deliberate, stated limitation, not a bug: covering
control flow would mean writing a Go-template interpreter, which is a different (and much
riskier-to-determinism) project.

### Scoping cross-resource resolution: `derive_package`

The existing linker (`src/bce/indexing/linker/linker.py`) resolves `UnresolvedRef`s by
building a global `(package, name) -> symbol_id` table from every fragment's exports, then
matching each file's unresolved refs against its own `package` (same mechanism Java's
`derive_package` override already uses for `package com.example.api;`). This provider
reuses that mechanism as-is — no linker changes — but the `package` value matters a lot:

- If `derive_package` returned a value unique to *each file* (the default, path-derived
  behaviour), a Deployment and a ConfigMap in different files would never share a
  `package` and the ConfigMap reference would never resolve. This would silently make
  goal 5 do nothing.
- `KubernetesProvider.derive_package()` instead walks up from the file's directory
  looking for the nearest ancestor containing a `Chart.yaml`, and returns that directory
  (repo-relative). Every file inside one chart shares that value, so
  `envFrom.configMapRef.name: my-config` (recorded as `UnresolvedRef(qualifier="ConfigMap",
  name="my-config", kind="reference")`) resolves against the ConfigMap's export key
  `"ConfigMap.my-config"` registered under the same chart-directory package — mirroring
  exactly how Java resolves `Segment.lookup` as `qualifier="Segment", name="lookup"`
  against a same-package export.
- For a file with no `Chart.yaml` ancestor (plain manifests, no Helm), `derive_package`
  falls back to the file's own parent directory. This scopes resolution to "files placed
  together on purpose," which is also a deliberate safety property: two unrelated apps
  that each happen to define a ConfigMap named `config` in different directories must
  never cross-link.
- `values.yaml` keys and `_helpers.tpl` defines are exported under the same chart-directory
  package, as `"values.<dotted.path>"` and `"helper.<define-name>"`, so the `.Values`/
  `include` references from step 4 resolve through the identical mechanism.

### Node/edge model

- One new `SymbolKind.RESOURCE` (a Kubernetes object: Deployment, Service, ConfigMap...),
  carrying `k8s_kind`, `namespace` (empty string when unset in the manifest — never
  guessed), and `api_version` as node properties alongside the usual `name`.
  `values.yaml` leaf keys reuse `SymbolKind.CONSTANT` (they are static configuration, the
  same rationale Python uses for uppercase module constants). `_helpers.tpl` defines reuse
  `SymbolKind.FUNCTION` (callable via `include`, the closest existing fit).
- No new `NodeLabel` — resources/values/helpers are all `Symbol` nodes, consistent with
  "every provider produces File + Symbol nodes" in `docs/languages.md`. This keeps the
  change additive: no scoring, web UI legend, or REST schema touches a new label type.
- `REFERENCES` edges for both `.Values`/`include` resolution and cross-resource name
  resolution, provenance `HEURISTIC` for the regex-derived `.Values`/`include` refs,
  `TREESITTER` for the AST-derived cross-resource name refs (steps 3 vs. 4 above).

## Testing

Follows the existing golden-test pattern in `tests/test_extractor_langs.py`
(`Extractor().extract_file(repo_id=..., path=..., source=...)`, assert on symbol names /
edges from a small inline byte-string source). New file `tests/test_extractor_kubernetes.py`.
All fixtures are synthetic and generic (`example.com`, `my-app`, `my-config` — the same
convention the existing Java/Go/C# fixtures already use) — **no real chart names, values,
domains, or secrets from any private repository go into this public PR.**

Cases to cover: a Deployment referencing a ConfigMap by name (goal 5), a template with
`{{ .Values.image.tag }}` resolving to a `values.yaml` key (goal 4), an `include` call
resolving to a `_helpers.tpl` define, a non-Kubernetes `.yaml` file producing zero symbols
(the gate), and a `.yaml` file with no `Chart.yaml` ancestor still extracting its own
resource symbol (no cross-resource resolution required for the file-local extraction to
work).

## Docs / metadata to update alongside the code

- `docs/languages.md`: new row + a "Kubernetes / Helm" section following the existing
  per-language write-up pattern.
- `pyproject.toml`: `tree-sitter-yaml` under `[langs]`.
- `CHANGELOG.md`: entry under `Unreleased`.

## Process

This is a large enough change (new `SymbolKind`, a provider with real resolution logic)
that it should go through the repo's **"Language support"** issue template
(`.github/ISSUE_TEMPLATE/language_support.yml`) before the implementation PR, so a
maintainer can react to the scope — in particular the `{{ }}`-neutralization approach and
the `derive_package`-as-chart-root scoping — before more time goes into it.
