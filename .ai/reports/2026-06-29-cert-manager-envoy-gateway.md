# Report: cert-manager + Envoy Gateway controllers & issuers for kube1

**Date:** 2026-06-29
**Plan:** `.ai/plans/2026-06-29-cert-manager-envoy-gateway.md`
**Status:** Built, reviewed, validated. **Not yet observed on a live cluster** (per CLAUDE.md "live-unverified" caveat — `bootstrap-flux.yml` has not been run against kube1 yet, so we cannot confirm the controllers reconcile or that the Gateway API version compatibility holds at runtime).

---

## What was implemented

The **control-plane half** of TLS + L7 ingress: cert-manager controller, Envoy Gateway controller + `GatewayClass envoy-gateway`, Gateway API CRDs (channel-selected), and two `ClusterIssuer` resources (LE staging + prod, Cloudflare DNS-01) — all pure-Flux, no pre-Flux Ansible install, no adoption handshake. **Controllers + issuers slice only** — no `Gateway`, no wildcard `Certificate`, no `HTTPRoute`, no host-network `EnvoyProxy` (data-path follow-up).

The cert-manager base is **CA/DNS-agnostic** — no Issuers, no Secrets, no DNS config live in the catalog. The two `ClusterIssuer` resources live in the per-cluster user-owned configs overlay and reference the Cloudflare token Secret **by name** (`cloudflare-api-token` / key `api-token` in namespace `cert-manager`).

---

## Files by area

### Config surface (3 files)
- `config/schemas/cluster-input.schema.json` — added top-level `ingress` object (`additionalProperties: false`, single property `gatewayApiChannel` enum `["standard", "experimental"]`).
- `config/defaults/cluster.yaml` — added `ingress: { gatewayApiChannel: standard }`.
- `config/clusters/kube1/cluster.yaml` — `features.certManager: true`, `features.ingress: envoy-gateway`, `ingress.gatewayApiChannel: standard`.

### Renderer (3 files)
- `infra/ansible/roles/config_render/tasks/compute.yml` — `config_render_flux_features` now appends `cert-manager` and (`envoy-gateway` + `gateway-api-<channel>`) when the corresponding features are on.
- `infra/ansible/roles/config_render/tasks/render_flux.yml` — `config_render_flux_selected_resources` maps the new feature tokens to the four catalog paths; **Gateway API CRDs are emitted before `envoy-gateway`** in the resources list (belt-and-suspenders for CRD-before-GatewayClass).
- `infra/ansible/roles/config_render/tasks/validate.yml` — pick-one assert for `ingress.gatewayApiChannel ∈ {standard, experimental}`; existing "ingress requires cert-manager" rule left untouched.

### Flux controllers catalog (17 files across 4 bases)
- `_components/cert-manager/` (4 files): `namespace.yaml` (`cert-manager` — deliberate non-`-system`, the chart webhook assumes this name), `helmrepository.yaml` (OCI `oci://quay.io/jetstack/charts`), `helmrelease.yaml` (chart `cert-manager` pinned to `1.20.3`, `crds.enabled: true` / `crds.keep: true`, controller only), `kustomization.yaml`.
- `_components/gateway-api/standard/` (4 files): `namespace.yaml`, `helmrepository.yaml` (OCI `oci://docker.io/envoyproxy`), `helmrelease.yaml` (chart `gateway-crds-helm` pinned to `v1.8.1` — same as EG controller for CRD compatibility, `crds.gatewayAPI.channel: "standard"`), `kustomization.yaml`.
- `_components/gateway-api/experimental/` (4 files): same shape as standard but `crds.gatewayAPI.channel: "experimental"`.
- `_components/ingress/envoy-gateway/` (5 files): `namespace.yaml` (`envoy-gateway-system`), `helmrepository.yaml` (OCI `oci://docker.io/envoyproxy`), `helmrelease.yaml` (chart `gateway-helm` pinned to `v1.8.1`, bundled Gateway API CRDs **disabled** via `install.crds: Skip` / `upgrade.crds: Skip` + `crds.gatewayAPI.safeUpgradePolicy.enabled: false`, `dependsOn: envoy-gateway-crds` for CRD readiness), `gatewayclass.yaml` (`controllerName: gateway.envoyproxy.io/gatewayclass-controller`), `kustomization.yaml`.

### Flux configs overlay (3 files)
- `flux/infrastructure/configs/overlays/clusters/kube1/letsencrypt-staging.yaml` — `ClusterIssuer letsencrypt-staging`, ACME staging, `dns01.cloudflare.apiTokenSecretRef: { name: cloudflare-api-token, key: api-token }`.
- `flux/infrastructure/configs/overlays/clusters/kube1/letsencrypt-prod.yaml` — same shape, ACME prod.
- `flux/infrastructure/configs/overlays/clusters/kube1/kustomization.yaml` — adds the two issuer files to `resources:` alongside `../../common`.

### Generated outputs
- `flux/infrastructure/controllers/generated/selected/kustomization.yaml` — now lists `../../_components/cert-manager`, `../../_components/gateway-api/standard`, `../../_components/ingress/envoy-gateway` in that order (CRDs before controller) alongside cilium, hcloud-ccm, openebs.

### Docs (2 files)
- `CLAUDE.md` — Implementation status: cert-manager + ingress moved from "Still deferred" to "Working — built/reviewed/validated, live-unverified" with the deferred-secret / SOPS caveat noted and a pointer to the runbook.
- `docs/cert-manager-ingress.md` *(new)* — operational runbook: enabling, channel knob (standard vs experimental), the four catalog entries + per-cluster issuers, verification (`kubectl get pods -n cert-manager`, `-n envoy-gateway-system`, `kubectl get gatewayclass`, `kubectl get clusterissuer`), and the deferred gaps (token Secret user-provisioned; data path follow-up).

---

## What was validated

| Gate | Result |
|---|---|
| `ansible-lint` (all changed task files) | 0 failures, 0 warnings |
| `playbooks/render-config.yml` — first run | new entries appear in `generated/selected/kustomization.yaml` (correct order: cert-manager → gateway-api/standard → envoy-gateway) |
| `playbooks/render-config.yml` — second run | 0 changed (idempotent) |
| `-e config_render_check=true` (check mode) | "no drift" |
| Toggle test: `features.certManager:false` + `features.ingress:none` | three new entries drop cleanly from `generated/selected/` |
| Channel test: `ingress.gatewayApiChannel: experimental` | swaps `gateway-api/standard` → `gateway-api/experimental` (exactly one selected) |
| `kustomize build flux/infrastructure/controllers/generated/selected` | clean — all six catalog entries resolve |
| `kustomize build flux/infrastructure/configs` (configs chain) | clean — two `ClusterIssuer` resources emitted |
| `yamllint` (all new files) | 0 errors (fixed missing `---` on kube1 configs `kustomization.yaml`) |
| `helm template` of cert-manager + envoy-gateway + gateway-crds-helm with pinned values | confirms CRD install/keep settings, tolerations, pins |
| `ansible-reviewer` | **APPROVED** |
| `k8s-reviewer` | **APPROVED** after 1 MEDIUM + 5 LOW fix round (see next section) |

---

## Review findings and fixes

| Sev | Finding | Fix |
|---|---|---|
| 🟡 MED | The render-emitted `generated/selected/kustomization.yaml` ordered catalog entries alphabetically — but the `GatewayClass` (in `ingress/envoy-gateway`) needs the Gateway API CRDs to exist first. | Pinned the resources list order in `render_flux.yml` so Gateway API CRDs are emitted **before** `envoy-gateway` (Flux's `wait: true` + retry covers the race as a second line of defense). |
| 🔴 HIGH | OCI HelmRepository URLs included the chart name (e.g. `oci://quay.io/jetstack/charts/cert-manager`); Flux with `type: oci` appends `chart.spec.chart` and rejects duplicate paths → 404 on first reconcile. | Stripped chart name across all four `HelmRepository` resources (cert-manager, envoy-gateway, gateway-api/standard, gateway-api/experimental) so `url` is the **parent path only** (`oci://quay.io/jetstack/charts`, `oci://docker.io/envoyproxy`). |
| 🟠 MED | `envoy-gateway` HelmRelease had no `dependsOn` on `envoy-gateway-crds` → "no matches for kind GatewayClass" on first apply when the CRDs had not finished registering. | Added `spec.dependsOn: [{ name: envoy-gateway-crds, namespace: envoy-gateway-system }]` to the envoy-gateway HelmRelease. |
| 🟡 LOW | `gateway-api/standard` and `gateway-api/experimental` bases had no `Namespace` resource, relying on `install.createNamespace: true` only — the GatewayClass in the sibling base then references CRDs from a namespace that does not exist as a Flux resource. | Added explicit `Namespace` to both channel bases; kept `install.createNamespace: true` as belt-and-suspenders. |
| 🟡 LOW | `flux/infrastructure/configs/overlays/clusters/kube1/kustomization.yaml` was missing the `---` document-start marker → `yamllint` failure. | Added `---` and re-ran `yamllint` clean. |
| 🟠 MED | `envoy-gateway` chart's `crds.gatewayAPI.safeUpgradePolicy` defaults to blocking upgrade when the CRD already exists — but the Gateway API CRDs are owned by the sibling `gateway-api/<channel>` base, so any `helm upgrade` would deadlock. | Set `values.crds.gatewayAPI.safeUpgradePolicy.enabled: false` on the envoy-gateway HelmRelease with an inline comment tying the value to the cross-base CRD ownership contract. |

---

## What was NOT run and why

- **No `ansible-playbook bootstrap-flux.yml`.** Live cluster apply. The user runs Ansible against the real cluster in their own terminal; agents do not.
- **No `flux reconcile`.** Same reason.
- **No `kubectl get pods -n cert-manager`, `-n envoy-gateway-system`, `kubectl get gatewayclass`, `kubectl get clusterissuer`.** No live cluster.
- **No live DNS-01 challenge.** Cloudflare token Secret is not committed (see DEFERRED-1 below) and `flux_bootstrap` is not run yet.
- **Result:** "done" here means "the code is built, lints, renders idempotently, builds cleanly under `kustomize` + `flux build` + `helm template` + `yamllint`, and passes both reviewers". The runbook's verify steps have not been observed executing against a real cluster.

---

## Known open items / deferred

1. **DEFERRED-1 — Cloudflare token Secret not committed.** `cloudflare-api-token` / key `api-token` (in namespace `cert-manager`, scoped `Zone:DNS:Edit` on `dragenet.dev`) is **user-provisioned manually** until Flux SOPS-with-age decryption is wired (current `.sops.yaml` scopes only `infra/ansible/inventories/**` group_vars). Until then DNS-01 challenges will not solve (no Certificate is requested this slice anyway). ACME account registration still reaches Ready. See `docs/cert-manager-ingress.md` for the exact Secret spec.
2. **DEFERRED-2 — data path follow-up.** No `Gateway`, no `*.dragenet.dev` wildcard `Certificate`, no `HTTPRoute`, no host-network `EnvoyProxy` — controllers + GatewayClass only this slice. Out-of-scope objects must NOT be added in this commit.
3. **Live-unverified caveat remains.** The next live apply should confirm: cert-manager pods Ready, envoy-gateway pods Ready, `kubectl get gatewayclass` shows `envoy-gateway` with controller `gateway.envoyproxy.io/gatewayclass-controller`, and both `ClusterIssuer` resources reach Ready (ACME account registration). The `GatewayClass` ↔ Gateway API CRD compatibility (W1, R2 in the plan) is the highest-confidence risk to spot-check first.

---

## Open items / follow-ups

1. **Live-apply sequence — user runs:**
   - `cd infra/ansible && ansible-playbook playbooks/render-config.yml -e config_cluster=kube1` — review git diff.
   - Commit + push; `flux reconcile kustomization flux-system --with-source` → `flux reconcile kustomization infrastructure-controllers --with-source` → `infrastructure-configs` follows via existing `dependsOn: infrastructure-controllers` + `wait: true`.
   - Verify per the runbook (`docs/cert-manager-ingress.md`): cert-manager + envoy-gateway pods Ready, `kubectl get gatewayclass` shows `envoy-gateway`, both `ClusterIssuer` resources Ready.
   - **Provision the Cloudflare token Secret manually** — DEFERRED-1.
2. **Channel knob** is `ingress.gatewayApiChannel` (default `standard`). Switch to `experimental` to pick up TCPRoute/GRPCRoute and other newer Gateway API kinds; the Envoy Gateway controller version is **not** changed by the channel switch (channel and controller are independent pins this slice).
3. **Namespace deviations** are documented in the catalog headers and `_components/README.md`: `cert-manager` (chart webhook assumption, non-`-system`) and `envoy-gateway-system` (chart default). A future CSI driver would still use the `csi-<name>-system` convention from the openebs slice.
4. **ADR-012 addendum** — the implementor judged `ingress.gatewayApiChannel` as a genuine new structural-select config knob (channel-override-as-structural-config) and the `cert-manager` namespace deviation as a deliberate catalog decision; both are recorded in the catalog headers and runbook rather than a new ADR.

---

**Want to review the diff and commit the cert-manager + Envoy Gateway + ClusterIssuers slice?** I can stage the files and show you the `git diff` for the catalog + configs + render + docs changes.
