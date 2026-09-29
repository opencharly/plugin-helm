# plugin-helm

The helm words for OpenCharly (Factory Unit 1) — the `step:helm-release` install
step and the `verb:helm` release-status check verb.

The plugin owns **no** Kubernetes client library: every operation shells out to
the venue's `helm` binary over the host's live `DeployExecutor` reverse channel,
so its `go.mod` carries no `k8s.io/client-go` dependency. It is a standalone Go
module served out-of-process over go-plugin gRPC via the plugin SDK.

## What it provides

| Capability | Surface |
|---|---|
| `step:helm-release` | the `external:helm-release` install-step kind — `helm upgrade --install` in-venue + a `helm uninstall` teardown reverse op |
| `verb:helm` | the `helm:` check verb — the declarative release-status assertion (the `verb:kube` analog) |

The `step:helm-release` step's opaque payload is the `#HelmReleaseStep` def,
carried through the InstallPlan IR and validated against this plugin's served
schema at authoring. It performs the `helm upgrade --install` in-venue against the
venue's kubeconfig (`/etc/rancher/k3s/k3s.yaml` on a k3s guest) and returns a
teardown `ReverseOp` (`helm uninstall`) the host records and replays. Its declared
`StepContract` is scope system, venue host-native, no gate, `Emits=false` — a
deploy-only step (helm installs happen at deploy, never at image build).

The `verb:helm` verb is EXEC-based (like `wl`/`dbus`): the host attaches its live
`DeployExecutor` over the E3b reverse channel and the plugin runs the venue's helm
binary; a missing broker is a hard fail.

## How to use it

Compose the plugin candy in a box or check bed's `candy:` list:

```yaml
- '@github.com/opencharly/plugin-helm/candy/plugin-helm:<tag>'
```

Then author the step and the assertion:

```yaml
- run: helm-release
  plugin_input:
    release: my-release
    chart: ./mychart
```

```yaml
- check: the release is deployed
  id: helm-release-exists
  helm: release-exists
  context: [runtime]
```

## Layout

- `candy/plugin-helm/` — the plugin module: `plugin.go` (provider + meta with the
  declared `StepContract`), `provider.go` (the step + verb dispatch),
  `helm.go`, `schema/helm.cue`, `params/cue_types_gen.go`, `cmd/serve/main.go`.
- `charly.yml` — the root project manifest (`discover: candy`).
- `.github/workflows/tag-on-merge.yml` — CalVer tag + `CHANGELOG/` on merge.

## Related

- Owning skill: `/charly-kubernetes:helm` — the helm words (`step:helm-release` +
  `verb:helm`).
- `/charly-internals:install-plan` — the executor reverse channel and the
  InstallPlan IR.
- `/charly-internals:plugin` — the plugin/provider model.
- [`opencharly/charly`](https://github.com/opencharly/charly) — the charly CLI.
