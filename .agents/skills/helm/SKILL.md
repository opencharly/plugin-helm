---
name: helm
description: The helm words — the `step:helm-release` install step (a `helm upgrade
  --install` performed IN-VENUE, returning a `helm uninstall` ReverseOp) and the `verb:helm`
  declarative release-status assertion, both served out-of-process by candy/plugin-helm;
  plus the `helm_charts:` deploy field the k8sgen emitter renders.
---

MUST be invoked before any work involving: the `helm-release` install
step, the `helm:` check verb, the `helm_charts:` deploy field, the
`--enable-helm` kustomize apply path, or `candy/plugin-helm`.

charly speaks helm through **two words and one deploy field**, and none of
them puts helm knowledge in the kernel — `candy/plugin-helm` is a
standalone Go module served out-of-process over go-plugin gRPC via the
SDK, and it owns NO Kubernetes client library (the verb is EXEC-based over
the live DeployExecutor reverse channel).

| Surface | Class | What it does |
|---|---|---|
| `helm-release:` | install step (`step:helm-release`) | `helm upgrade --install` IN-VENUE; returns a `helm uninstall` ReverseOp |
| `helm:` | check verb (`verb:helm`) | assert a release's existence / status / revision / values hash |
| `helm_charts:` | `kubernetes:` deploy field | emitted as a kustomize `helmCharts:` entry, applied with `--enable-helm` |

## `step:helm-release` — the install step

A plugin-contributed `external:helm-release` install-step KIND (F3) whose
OPAQUE payload is the plugin's `#HelmReleaseStep`, carried through the
InstallPlan IR and validated against the plugin's served schema at
authoring time. Field set:

| Field | Required | Meaning |
|---|---|---|
| `chart` | yes | chart name (bare, resolved via `repo`) or a full reference |
| `release` | yes | the release name |
| `repo` | no | chart repository URL; omit for a pre-added repo or an OCI ref |
| `version` | no | pin the chart version (`--version`) |
| `namespace` | no | target namespace (created if absent) |
| `values` | no | inline values map |
| `values_files` | no | values YAML paths, venue-relative (`-f`) |
| `wait` | no | `--wait` for readiness |
| `timeout` | no | wait timeout, e.g. `5m` |

**Kubeconfig resolution is a three-arm precedence chain**, not a fixed
path. In order, first match wins:

1. an operator-set `KUBECONFIG` — never clobbered;
2. the k3s guest default `/etc/rancher/k3s/k3s.yaml`, when present;
3. the standard `$HOME/.kube/config`, when present.

Arm 3 is the one that carries a non-k3s cluster: on any venue that is
not a k3s guest, it is what makes an unconfigured `helm-release` step
find the cluster at all. helm reads `KUBECONFIG` itself, so exporting it
is the whole resolution.

**Teardown is record-and-replay.** The step returns a `helm uninstall`
ReverseOp that the host records and replays at teardown — you never author
an uninstall step yourself.

## The authoring law this surfaced: mutating installs live in a CANDY

A bed's own plan runs **verify-only** (mutating steps skipped), and
`charly deploy add` lowers only **candy** plans' `run:` steps. So any
mutating install — the `helm-release` invocation itself, and any in-venue
`kubectl wait` it depends on — MUST live in a candy's `run:` steps, never
in the bed's own plan. Authoring the install into the bed produces a bed
that passes while installing nothing.

A second-order trap rides along: a `check:` step does **not** run during
`deploy add`, so the `k3s-server` candy's own node-ready check cannot gate
your install. An install candy waits for the node itself.

**Two independent mechanisms decide when a step runs, and confusing them
is the trap.**

First the step KEYWORD. `charly deploy add` lowers `run:` steps and
nothing else — `check:`, `agent-*:` and `include:` steps are never
lowered, whatever `context:` they carry
(`sdk/deploykit/install_build.go:694-697`, which drops non-`run:`
keywords *before* it looks at context at all). So the `check:` step in
the example below does not execute during `deploy add` **because it is a
`check:` step** — not because of its context. Giving it
`context: [deploy]` would not change that.

(This is about `deploy add` LOWERING. `context:` on a `check:` step is not
inert in general — it selects which CHECK MODE the step runs in, which is
a different walk entirely; see "Author the STEP `context: [runtime]`"
under `verb:helm` below.)

Then, for `run:` steps only, `context:` splits the work: a step scoped
runtime-ONLY is left to the check Runner, and everything else goes on
the install timeline (`install_build.go:701`). That predicate tests only
"runtime-only or not", so on a `run:` step `[deploy]`, `[build]` and
`[build, deploy]` are equivalent at execution.

That is how one candy both installs a release at deploy time and asserts
it at check time: the install is a `run:` step on the install timeline,
and the assertion is a `check:` step the install timeline never sees.

`candy/helm-chart` is the worked example — the `check-helm-vm` bed's
install leg. Its executable lines are reproduced verbatim, because the
readiness step's exact shape is the lesson; the `run:`/`check:`
descriptions and comments are shortened for reading:

```yaml
helm-chart:
  candy:
    require: [k3s-server, plugin-helm]   # plugin-helm must be in the
    plan:                                # scanned candy set to be built
      - run: wait for the k3s node to be Ready in-venue
        id: hc-wait-node-ready
        command: |
          set -euo pipefail
          export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
          # The kubeconfig lands when the k3s API server STARTS, which can
          # be seconds-to-minutes after the k3s service starts. So poll
          # /readyz until the apiserver answers, THEN let kubectl wait for
          # the node. Waiting on the node alone fails on a cold cluster.
          deadline=$(( $(date +%s) + 300 ))
          until /usr/local/bin/kubectl get --raw /readyz >/dev/null 2>&1; do
            if [ "$(date +%s)" -ge "$deadline" ]; then
              echo "k3s apiserver did not become ready within 300s" >&2
              exit 1
            fi
            sleep 1
          done
          /usr/local/bin/kubectl wait --for=condition=Ready node --all --timeout=300s
        context: [deploy]
      - run: install the chart via the external helm-release step
        id: hc-helm-release-install
        context: [deploy]
        helm-release:
          repo: https://prometheus-community.github.io/helm-charts
          chart: prometheus-pushgateway
          release: web-pushgateway
          namespace: web
          wait: true
          timeout: 5m
      - check: the release is deployed in the web namespace
        id: hc-release-exists
        context: [runtime]
        helm:
          method: release-exists
          release: web-pushgateway
          namespace: web
```

Three things about that readiness step are deliberate. **The `/readyz`
poll is bounded and condition-driven**, with an explicit deadline and a
non-zero exit — it is a synchronization primitive, not an R3
`sleep`-and-hope; a bare `sleep 60` in its place is the workaround the
rule forbids. **Waiting on the node alone is not enough**: on a cold
cluster `kubectl wait node` runs before the apiserver answers at all, so
the step must gate on the apiserver first.

And **its `export KUBECONFIG=…` is the candy's own line, for its own
`kubectl` calls in that one shell** — it is not you configuring the helm
step, and it does not contradict the three-arm chain above. Each step
runs in its own shell, so that export does not reach the
`helm-release:` step; that step resolves the kubeconfig itself, by the
chain. The candy hardcodes the k3s path because it is a k3s-guest bed
fixture; a candy for another venue would not.

Note the Agent Driven Evaluation (ADE) split (R2) — ADE is the
spec-is-the-test discipline owned by `/charly-check:check`: the candy that
installs the release also asserts it exists; the bed's own plan keeps only
the deeper status/revision assertions.

**Where the `helm` binary comes from.** The step shells `helm` IN-VENUE,
so the venue must already carry it — `candy/plugin-helm` ships the WORD,
never the binary. The binary comes from the **`kubernetes` candy** (the
Kubernetes client toolchain: `kubectl`, `helm`, `k3d`, from the distro
package with an upstream `get-helm-3` fallback). `check-helm-vm` composes
it explicitly — `add_candy: [k3s-server, kubernetes, helm-chart]` — and a
venue that omits it fails the step at execution, not at authoring. Compose
`kubernetes` into any venue where you author a `helm-release` step.

## `verb:helm` — the declarative assertion

The `verb:kube` analog. Its PRIMARY input field is `method`, so
`helm: release-exists` desugars to `{method: "release-exists"}`.

| `helm:` method | Method-exclusive modifiers | Asserts |
|---|---|---|
| `release-exists` | — | the release exists |
| `status` | `status:` (default `deployed`) | the release status |
| `revision` | `revision:` (int) | release revision ≥ N |
| `values-hash` | `values_hash:` | SHA-256 of the rendered values |

Every method takes `release:` (required) and `namespace:` (default
`default`). The verb is EXEC-based and probes a live cluster, so it only
makes sense under `charly check live` / `charly check run`.

**Author the STEP `context: [runtime]`, and understand that this is what
makes it skip under `charly check box` — not the verb's EXEC nature.**

This does not contradict the `deploy add` rule above, and the two are worth
holding apart: THERE, `context:` is irrelevant because a `check:` step is
never lowered at all; HERE, `context:` is decisive because it selects the
check mode the step is active in. Different walks, different questions.

A plugin verb with no
`context:` defaults to `[build, deploy, runtime]`
(`spec/spec/verb_context.go:148`), and box mode selects `build`
(`candy/plugin-check/plan_grammar.go:42`) — so an unscoped `helm:` step is ACTIVE at image
time and FAILS against a cluster that does not exist, rather than
skipping. With `[runtime]` the skip is reported visibly, never silently
green.

## The `kubernetes:` substrate arm — the `helm_charts:` field

A `kubernetes:` deploy does NOT shell out to helm — that would be a
boundary-law violation. It declares `helm_charts:` on the deploy, which
the k8sgen emitter translates into a kustomize `helmCharts:` transformer
entry in the OVERLAY kustomization (the base stays chart-free).

`helm_charts:` is a list of `#HelmChart` entries. Six fields — a SUBSET
of the step's, with no `values:`, `wait:` or `timeout:` (kustomize renders
the chart, so there is no release to wait on at apply time):

| Field | Required | Meaning |
|---|---|---|
| `chart` | yes | chart name → the kustomize entry's `name` |
| `release` | yes | release name → `releaseName` |
| `repo` | no | chart repository URL |
| `version` | no | chart version to pin |
| `namespace` | no | target namespace |
| `values_files` | no | **first entry only** → `valuesFile`; see below |

```yaml
my-app:
  kubernetes:
    image: my-app
    from: production
    deploy:
      helm_charts:
        - repo: https://prometheus-community.github.io/helm-charts
          chart: prometheus-pushgateway
          release: web-pushgateway
          namespace: web
```

**`values_files` is where the two arms diverge — do not assume it
carries over.** A kustomize `helmCharts` entry holds exactly one
`valuesFile`, so the emitter takes the FIRST entry and silently discards
the rest (`candy/plugin-k8sgen/k8sgen.go:204-205`). The `helm-release:`
step passes every file, one `--values` each. So
`values_files: [base.yaml, prod.yaml]` applies both files through the
step and only `base.yaml` through a `kubernetes:` deploy — no error, no
warning. If a `kubernetes:` deploy needs several values files, pre-merge
them into one.

The emitter also sets `includeCRDs: true` on every entry,
unconditionally (`k8sgen.go:207`); there is no field to turn it off.

`chart`, `release`, `repo`, `version` and `namespace` do mean the same
thing in both arms, so those transfer between venues unchanged.

Because the kustomize helmCharts transformer is gated behind
`--enable-helm`, which `kubectl apply -k` cannot pass, the apply path
renders first: `kubectl kustomize --enable-helm <overlay> | kubectl apply
-f -` (and the symmetric form for delete).

## Beds

- `check-helm-vm` — the R10 bed: a disposable `k3s-server` guest,
  `candy/helm-chart` as the install leg, `verb:helm` + `kube:` assertions.
  Its own `kind: kubernetes` profile is `check-helm-vm-ctx`, per-deploy
  scoped so it never resolves through a sibling bed's context.
- `check-k8s-deploy` — proves the `helm_charts:` emission and the
  `--enable-helm` apply on the externalized `deploy:kubernetes` substrate.

## Related skills

- `/charly-kubernetes:check-k8s` — the `kube:` verb these beds pair with.
- `/charly-kubernetes:kubernetes` — the `kubernetes:` deploy substrate and
  its Kustomize generation.
- `/charly-check:check` — the plan-step vocabulary and how the provider
  registry dispatches declarative verbs.
- `/charly-internals:plugin` — the Provider model and out-of-process
  dispatch this rides on.
- `/charly-internals:install-plan` — defines the vocabulary this page
  uses for the step: the **InstallPlan IR** the step is carried through,
  the **F3** plugin-contributed `external:<word>` step KIND and its
  **OPAQUE payload**, and the **ReverseOp** record-and-replay teardown
  (see that skill's step-kinds reference page).
- `/charly-infrastructure:k3s` — the `k3s-server` candy the beds deploy.
- `/charly-coder:kubernetes-layer` — the `kubernetes` candy that installs
  the `helm` binary the step shells in-venue.
