# AGENTS.md — plugin-helm

Standalone plugin repo owning the helm words — the `step:helm-release` install
step and the `verb:helm` check verb. The plugin is a Go module at
`candy/plugin-helm/` (module path
`github.com/opencharly/plugin-helm/candy/plugin-helm`); the root `charly.yml`
only declares `discover: candy` so the repo is a project and its candy is
scanned.

Canonical files:

- `candy/plugin-helm/charly.yml` — the `plugin-helm:` candy entity (`plugin:`
  block, `plan:` check).
- `candy/plugin-helm/plugin.go` — the provider (`NewProvider()` + `NewMeta()`
  with the declared `StepContract`).
- `candy/plugin-helm/provider.go` — the step + verb dispatch over the E3b
  reverse channel.
- `candy/plugin-helm/helm.go` — the helm invocation and matcher evaluation.
- `candy/plugin-helm/schema/helm.cue` — the self-contained `#HelmReleaseStep` /
  `#HelmInput`.
- `candy/plugin-helm/params/cue_types_gen.go` — generated params (do not
  hand-edit).
- `.github/workflows/tag-on-merge.yml` — CalVer tag + `CHANGELOG/` on merge.
- `README.md` — user overview only; never agent guidance.

## Load these skills first (R0)

- `/charly-kubernetes:helm` — the helm words: the `step:helm-release` install
  step and the `verb:helm` release-status assertion (the plugin's owning skill).
- `/charly-internals:plugin` — the plugin authoring reference: the `plugin:`
  block, the unified Provider model (incl. the `step` + `verb` classes), the
  per-plugin CUE-schema contract, placement.
- `/charly-internals:install-plan` — the executor reverse channel and the
  InstallPlan IR that carries the opaque step payload.
- `/charly-internals:git-workflow` — before any git/PR action.

## Build / validate / test

- `go build ./...` in `candy/plugin-helm/` — compile the plugin module.
- `go test ./...` in `candy/plugin-helm/` — the plugin's Go tests.
- `charly box validate` at the repo root — the structural check (the candy +
  `plugin:` block, CUE schema).
- The merge gate is the **org-wide** `charly/pr-validator` (required check
  `validate / validate`, defined in `opencharly/.github`); this repo has **no**
  per-repo candy gate.
- The R10 witness is the `check-helm` bed (in `opencharly/charly`): a disposable
  `vm:` + `k3s-server` + `kubernetes` composition that installs a real chart via
  `step:helm-release` and asserts it deployed via `verb:helm`.

## Modify this repo

- Edit the `plugin-helm:` candy entity, the Go source, and `schema/helm.cue`
  **together** — the schema is the single source for the `params/` struct.
- Keep the plugin free of a Kubernetes client library: it drives the venue's
  `helm` binary over the host executor. A missing broker is a hard fail.
- The step is deploy-only (`Emits=false`); its declared `StepContract` is the
  contract the host carries opaquely. Keep it in step with the actual behavior.

## Landing

- PR-only. Every change lands through a pull request; the org-required
  `charly/pr-validator` validates the diff and body and arms native auto-merge on
  PASS. Direct pushes to `main` are blocked.
- History lives in `CHANGELOG/` (written by `tag-on-merge` at merge time); the PR
  body IS the changelog.
- The authoritative rulebook is the umbrella `AGENTS.md` in
  `opencharly/opencharly` and `charly/AGENTS.md` in the charly repo. Do not
  restate its rules here.
