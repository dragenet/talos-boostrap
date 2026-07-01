---
title: Ingress and TLS
description: Day-2 verification and operational runbook for cert-manager and Envoy Gateway on this template's cluster.
type: guide
audience: [operator, user]
tags: [ingress, tls, cert-manager]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - ../adr/0004-envoy-gateway-l7.md
---

# Ingress and TLS

Operational runbook for the TLS + L7 controllers on this template's cluster. The catalog
decisions (CRD channel ownership, namespace deviations, why we disable the
chart-bundled Gateway API CRDs) are recorded in the catalog headers under
`flux/infrastructure/controllers/_components/`; this document is the
day-2 verification procedure.

## Overview

The TLS + L7 plane is split into three Flux-managed catalog entries plus a
per-cluster configs overlay:

| Catalog entry | Namespace | What it installs |
|---|---|---|
| `_components/cert-manager/` | `cert-manager` | cert-manager chart `1.20.3` from `oci://quay.io/jetstack/charts` (controller only — no Issuers, no Secrets, CA/DNS-agnostic) |
| `_components/gateway-api/standard/` (or `experimental/`) | `envoy-gateway-system` | Raw CRD YAML committed in-repo — Gateway API CRDs (channel picked) + Envoy Gateway CRDs |
| `_components/ingress/envoy-gateway/` | `envoy-gateway-system` | `gateway-helm` chart v1.8.1 from `oci://docker.io/envoyproxy` (bundled Gateway API CRDs **disabled**), plus the `GatewayClass envoy-gateway` resource |
| `flux/infrastructure/configs/overlays/clusters/<cluster>/` (user-owned) | `cert-manager` | Two `ClusterIssuer` resources (`letsencrypt-staging`, `letsencrypt-prod`, ACME + Cloudflare DNS-01) — reference the token Secret **by name** |

The render compiler selects the three catalog entries when
`features.certManager: true` and `features.ingress: envoy-gateway` are set in
`config/clusters/<cluster>/cluster.yaml`, and picks `gateway-api/standard` vs
`gateway-api/experimental` from `ingress.gatewayApiChannel` (default
`standard`). The `generated/selected/kustomization.yaml` includes
`../../_components/cert-manager`, `../../_components/gateway-api/<channel>`,
and `../../_components/ingress/envoy-gateway` in that order — the CRD YAML is
applied before the Envoy Gateway controller so the GatewayClass and routes
have their CRDs available by the time the controller reconciles.

> **Why this is pure-Flux (not pre-Flux Ansible):** the cert-manager + EG
> controllers have no Talos-coupled lifecycle (no node taints to clear, no
> Cilium-toleration race) and there is no existing Helm release to adopt on a
> first install — unlike Cilium and the hcloud CCM, which the
> `flux_bootstrap` role installs pre-Flux. Flux installs the chart from
> scratch the first time the catalog entry is included in
> `generated/selected/`. No adoption handshake, no pre-Flux component.

## Enabling

Both flags are **already enabled** in `config/clusters/<cluster>/cluster.yaml`:

```yaml
features:
  certManager: true
  ingress: envoy-gateway

ingress:
  gatewayApiChannel: standard
```

To **disable**, set `features.certManager: false` and `features.ingress: none`
and re-render — the render's paired copy/remove tasks drop the three catalog
references from `generated/selected/kustomization.yaml` cleanly.

To **switch the Gateway API channel** to experimental (e.g. for TCPRoute /
GRPCRoute features), set `ingress.gatewayApiChannel: experimental` in the
selected-cluster config and re-render. The render swaps exactly one catalog
entry: `gateway-api/standard` → `gateway-api/experimental`. The Envoy Gateway
controller version is **not** changed by the channel switch — the channel knob
only selects which CRD manifest the controller reconciles against.

> **Channel + controller version coupling:** the EG chart `v1.8.1` is pinned
> to match the Gateway API / Envoy Gateway CRDs rendered into the repo. If you
> bump the EG controller chart, you MUST refresh the committed CRD YAML from
> the matching upstream release line. A mismatch breaks GatewayClass / route
> reconciliation silently (missing fields, dropped resources).

## Post-bootstrap verification

### 1. Verify the controllers are running

```bash
# cert-manager: controller, webhook, cainjector all Ready
kubectl -n cert-manager get pods
# Expected: cert-manager-<hash>, cert-manager-webhook-<hash>,
# cert-manager-cainjector-<hash> all 1/1 Running.

# Envoy Gateway: controller pod Ready
kubectl -n envoy-gateway-system get pods
# Expected: envoy-gateway-<hash> 1/1 Running.

# Gateway API CRDs registered
kubectl get crds | grep gateway.networking.k8s.io
# Expected: gateways.gateway.networking.k8s.io,
# gatewayclasses.gateway.networking.k8s.io,
# httproutes.gateway.networking.k8s.io, ... (full standard channel set)

# GatewayClass bound to Envoy Gateway
kubectl get gatewayclass envoy-gateway
# Expected: name envoy-gateway, controllerName
# gateway.envoyproxy.io/gatewayclass-controller, Accepted=True (after
# controller reconciliation; may show blank briefly on first apply).
```

If any controller pod is not Ready, do **not** proceed — the ClusterIssuers
require the cert-manager webhook, and the GatewayClass requires the EG
controller.

### 2. Verify the ClusterIssuers

```bash
kubectl get clusterissuers -o wide
# Expected: letsencrypt-staging and letsencrypt-prod both listed.
# READY column shows True once the ACME account registers.
# The first reconciliation may take ~30s as the ACME server is contacted.
```

The ClusterIssuer is `False` on first apply because the `cert-manager`
controller has not yet registered an ACME account with Let's Encrypt. After
~30s it should reach `READY=True` even **without** the Cloudflare token
Secret — account registration uses the public ACME server and does not
require the DNS-01 solver. DNS-01 challenges only fail when a `Certificate`
is requested that needs them, and no Certificate is requested in this slice.

```bash
# Detail when something is wrong — look at the Conditions column
kubectl describe clusterissuer letsencrypt-staging
```

### 3. Provision the Cloudflare token Secret manually

Flux SOPS-with-age decryption is not yet wired for the `cert-manager`
namespace. The Cloudflare API token Secret must be created **out of band**.
The token needs `Zone:DNS:Edit` scope on the DNS zone the ClusterIssuers
challenge against (see the `apiTokenSecretRef` in the issuer manifests).

```bash
kubectl -n cert-manager create secret generic cloudflare-api-token \
  --from-literal=api-token=<your-cloudflare-api-token>
```

The Secret **must** be named `cloudflare-api-token` with the data key
`api-token` — both names are referenced by the `letsencrypt-staging` and
`letsencrypt-prod` ClusterIssuer manifests. A mismatch (renamed Secret,
wrong key) silently fails DNS-01 challenges with no obvious error.

> **Future: Flux SOPS-with-age wiring.** A follow-up slice will move this
> Secret into the Flux catalog and decrypt it from a SOPS-encrypted file in
> the repo (`.sops.yaml` currently scopes only the hcloud token). Until then,
> this `kubectl create secret` step is the only manual piece of the TLS
> pipeline. Re-run the command if the Secret is ever deleted; it is
> idempotent.

### 4. Re-verify after the Secret is created

The ClusterIssuer status does not auto-recheck; the
`cert-manager-controller` re-reads it on a short interval (default 1m). To
force a re-read, annotate the ClusterIssuer (no-op annotation triggers a
reconcile):

```bash
kubectl annotate clusterissuer letsencrypt-staging \
  cert-manager.io/test-trigger="$(date +%s)" --overwrite
kubectl get clusterissuer letsencrypt-staging -o wide
# READY should still be True; conditions should be clean.
```

A `Certificate` request against either issuer is the real end-to-end test —
until that exists, the "challenge would solve" status can only be inferred
from the issuer's `Registered` condition.

## Known limitations

- **Cloudflare token Secret is user-provisioned manually** — no Flux SOPS
  wiring yet. DNS-01 challenges do not solve until the Secret exists. ACME
  account registration still reaches `READY=True` without it.
- **No `Certificate` resources exist** — the data path is intentionally out
  of scope. No wildcard cert, no per-service certs. Adding a
  `Certificate` is the first step of the data-path follow-up.
- **No `Gateway` / `HTTPRoute` / host-network `EnvoyProxy`** — the EG
  controller is installed and the `GatewayClass` is bound, but no
  `Gateway` resource exists yet. An HTTP listener requires both a
  `Certificate` (above) and a `Gateway` referencing this `GatewayClass`.
- **No LB Service / public IP** — the Hetzner CCM ships with
  `networking.enabled: false` (see `../adr/0005-hcloud-ccm.md`), so the EG controller cannot
  provision a Hetzner LB Service automatically. Production traffic
  ingress must come through a load balancer pointing at Envoy proxy pods, or
  the pods must run host-network (with `nodeSelector` + `hostNetwork: true`
  on the `EnvoyProxy`) — both are part of the data-path follow-up.

## Troubleshooting

- **`cert-manager` webhook pods `CrashLoopBackOff`**: check
  `kubectl -n cert-manager logs cert-manager-webhook-<hash>` — the webhook
  uses an internal `cert-manager-webhook.cert-manager.svc` Service; the
  namespace MUST be `cert-manager` (not `cert-manager-system`) for the
  internal DNS to resolve. The catalog entry pins the namespace; do not
  rename it.
- **GatewayClass shows `Accepted=False` after the controller is Ready**:
  the EG controller takes ~10s to register itself as the handler. Wait
  one reconcile interval. If it stays `False`, check the EG controller
  logs for "no matches for kind GatewayClass" — that means the CRDs from
  `gateway-api/<channel>` did not apply before the controller came up;
  `flux reconcile kustomization infrastructure-controllers --with-source`
  will re-apply.
- **ClusterIssuer `READY=True` but DNS-01 still fails on Certificate
  requests**: the Cloudflare token Secret is missing or mis-keyed. Confirm:
  ```bash
  kubectl -n cert-manager get secret cloudflare-api-token \
    -o jsonpath='{.data.api-token}' | base64 -d | head -c 8 && echo
  # Expected: the first 8 chars of your token. If empty or "Error", the
  # Secret is missing or the key is wrong.
  ```
- **EG bundled Gateway API CRDs are NOT being skipped** (a `helm template`
  of the EG release shows Gateway API kinds in the rendered CRDs): the
  chart's `install.crds: Skip` / `upgrade.crds: Skip` keys are chart-version
  specific. If you bump the EG chart, verify the keys still suppress the
  bundled CRDs. A non-zero count from
  `helm template envoy-gateway oci://docker.io/envoyproxy/gateway-helm --version v<X.Y.Z> | grep -c "kind: CustomResourceDefinition"`
  means the CRDs will conflict with the `gateway-api/<channel>` catalog entry.
- **Channel switch (`standard` → `experimental`) leaves stale resources**:
  flipping the channel in `cluster.yaml` swaps the catalog entry in
  `generated/selected/`, but the previously installed CRDs (now absent from
  the new manifest) are not auto-pruned. The experimental channel may add
  new CRDs that did not exist in standard; the standard-only CRDs are
  retained by k8s garbage collection only if no other resource references
  them. To clean up before re-rendering, `kubectl delete crd <stale-crd>` —
  or simply re-render, let Flux re-apply, and accept the harmless stale
  CRDs (they sit unused, not affecting cluster behaviour).

## Related GitOps config

- **Catalog entries** under `flux/infrastructure/controllers/_components/`:
  - `cert-manager/` — Namespace, OCI `HelmRepository`, pinned `HelmRelease`
    (controller only, `crds.enabled: true` / `crds.keep: true`).
  - `gateway-api/standard/` and `gateway-api/experimental/` — Namespace,
    OCI `HelmRepository`, `HelmRelease` (`gateway-crds-helm` chart, channel
    selected via `crds.gatewayAPI.channel`).
  - `ingress/envoy-gateway/` — OCI `HelmRepository`, `HelmRelease` (bundled
    CRDs `Skip`, `dependsOn: envoy-gateway-crds`), and the
    `GatewayClass envoy-gateway`.
- **User-owned ClusterIssuers** under
  `flux/infrastructure/configs/overlays/clusters/<cluster>/`:
  `letsencrypt-staging.yaml`, `letsencrypt-prod.yaml`. Edit the
  `spec.acme.email` placeholder (LE accepts an empty value but no expiry
  notices will be sent).
- **Pins** — every Helm chart version is recorded in its
  `helmrelease.yaml` header comment with a release date. On a chart
  version bump, the CRD chart (`gateway-crds-helm`) and the controller
  chart (`gateway-helm`) MUST bump to the same release line.
- **Architectural background** — see `../adr/0004-envoy-gateway-l7.md`
  (Envoy Gateway L7) and `../examples/kube1-hetzner/adrs/0004-tls-lets-encrypt-cloudflare.md`
  (LE + Cloudflare on `dragenet.dev`).
