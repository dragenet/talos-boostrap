# Plan: cert-manager + Envoy Gateway controllers & issuers for kube1

**Date:** 2026-06-29
**Slug:** cert-manager-envoy-gateway
**Status:** Planned — not yet implemented
**Related ADRs:** ADR-008 (cert-manager), ADR-004 cluster-bootstrap (Envoy Gateway L7), kube1 ADR-004 (TLS Let's Encrypt + Cloudflare DNS-01 on dragenet.dev), ADR-010 (Ansible↔Flux handoff), ADR-011 (one-writer-per-file / no Flux-body templating), ADR-012 (structural-enable + overlay-tweak feature catalog), ADR-016 (Talos/selected-cluster config boundary), ADR-017 (shared overlay ownership); kube1 ADR-003 (three-plane network/ingress).
**Live state:** UNVERIFIED. This plan produces **built + reviewed + validated** code. "Done" means it renders cleanly, lints, `kustomize`/`flux`/`helm` builds, and passes review — NOT that it has been observed reconciling on a live cluster (CLAUDE.md unverified-live-cluster caveat).

---

## 1. Goal & scope (LOCKED — "controllers + issuers only" slice)

Land the **control-plane half** of TLS + L7 ingress for kube1, end-to-end through render/lint/build/review, and **STOP before the data path**.

**IN scope:**
- TEMPLATE-OWNED catalog bases under `flux/infrastructure/controllers/_components/`:
  - `cert-manager/` — controller only (Namespace `cert-manager`, jetstack HelmRepository, pinned HelmRelease with CRDs enabled via chart values). **CA/DNS-agnostic — NO issuers, NO DNS config, NO secrets.**
  - `gateway-api/standard/` and `gateway-api/experimental/` — two kustomize bases, each pinning the upstream Gateway API release manifest for its channel. Render selects exactly one.
  - `ingress/envoy-gateway/` — Namespace `envoy-gateway-system`, HelmRepository, pinned HelmRelease (chart-bundled Gateway API CRDs **DISABLED**), and the **GatewayClass** (controller-level binding).
- RENDER structural selection only (`config_render` role): one new config field `ingress.gatewayApiChannel`; feature→catalog-path mapping for cert-manager, envoy-gateway, gateway-api-`<channel>`.
- USER-OWNED kube1 ClusterIssuers (`letsencrypt-staging` + `letsencrypt-prod`, ACME, Cloudflare DNS-01) authored in the configs per-cluster overlay, referencing the Cloudflare token Secret **by name**.

**OUT of scope (deferred data-path follow-up — DOCUMENT, do not build):**
- No `Gateway`, no wildcard `Certificate` (`*.dragenet.dev`), no `HTTPRoute`, no host-network `EnvoyProxy`.
- No Cloudflare token Secret committed; **no Flux SOPS-with-age decryption wired this slice** → DNS-01 challenges will NOT solve until the operator provisions the secret (and SOPS lands). ACME account registration still reaches Ready.

**Agents must NOT run live cluster operations** (no `kubectl`, `flux reconcile`, `helm install`). They render, lint, build (`kustomize build`, `flux build`, `helm template`, `kubeconform`, `yamllint`), and review only. The live-apply sequence (§8) is for the USER.

---

## 2. Watch-outs — MUST be verified against live docs before/at implementation

Per CLAUDE.md "docs before code": training data is stale. The named implementor must dispatch **`web-fast-context`** (or read official docs directly: cert-manager.io, gateway-api.sigs.k8s.io, gateway.envoyproxy.io, fluxcd.io) and record the confirmed versions inline in the manifests' header comments before handoff.

- **W1 — Envoy Gateway chart ↔ Gateway API version coupling (CRITICAL).** Because we DISABLE the chart's bundled Gateway API CRDs, the pinned `gateway-api/<channel>` manifest version MUST match the Gateway API version the pinned Envoy Gateway release expects. Verify: (a) the latest stable Envoy Gateway Helm chart version + its image tag, (b) which Gateway API release (e.g. `vX.Y.Z`) that EG version targets, (c) the exact chart values key that disables the bundled CRDs (historically `gateway.envoyproxy.io` / `crds.gatewayAPI.enabled` or `gatewayConvertResources` — confirm the current key name for the pinned chart). Pin standard-install.yaml / experimental-install.yaml at that exact Gateway API version.
- **W2 — GatewayClass ↔ CRD ordering inside ONE Kustomization.** The GatewayClass (envoy-gateway catalog base) references the `GatewayClass` CRD that ships in the `gateway-api/<channel>` base. Both live in the single `infrastructure-controllers` Flux Kustomization. Confirm Flux's `wait: true` + ordered/retried apply (health-check + retryInterval 1m) establishes the CRD before the GatewayClass CR (Flux retries CRs whose CRD is not yet registered). If a hard ordering guarantee is needed, the fallback is splitting Gateway API CRDs into a separate Flux Kustomization with `dependsOn` — note this as residual-risk R1, prefer the single-Kustomization retry path first.
- **W3 — cert-manager ↔ Kubernetes v1.36.** kube1 runs `kubernetesVersion: v1.36.0` (config/clusters/kube1/cluster.yaml). Confirm the pinned cert-manager chart version supports k8s 1.36. Confirm the jetstack HelmRepository URL (`https://charts.jetstack.io`) and the correct values key to install CRDs from the chart (historically `installCRDs: true`, now `crds.enabled: true` / `crds.keep: true` — confirm for the pinned version).
- **W4 — Pin everything.** No `:latest`, no floating tags. Pin: cert-manager chart + (if surfaced) controller/webhook/cainjector image tags; Envoy Gateway chart + image tag; Gateway API manifest version. Record the pin + the doc source in each file's header comment.
- **W5 — Catalog pattern fidelity.** Study the established entries before writing: `_components/openebs/` (purely-Flux-managed base, `kustomization.yaml` resource ordering, HelmRepository/HelmRelease pinning, header comments) and `_components/cilium/` + `_components/providers/hcloud/ccm/` (HelmRelease pinning, namespace conventions). Match `.yamllint`/`kubeconform` cleanliness and the `---` document-start + header-comment style exactly.
- **W6 — Namespace deviation is deliberate.** cert-manager Namespace is **`cert-manager`** (NOT `cert-manager-system`) — its webhook/docs assume that name. Envoy Gateway Namespace is **`envoy-gateway-system`** (chart default). Record both deviations from the repo `<name>-system` convention in `_components/README.md`.

---

## 3. Catalog & file layout (concrete — design input for Phase B / Phase D-configs)

All paths relative to repo root `/Users/dominik/Projects/infra/kube1`.

### 3.1 TEMPLATE-OWNED controller catalog (Phase B)
```
flux/infrastructure/controllers/_components/
  cert-manager/
    kustomization.yaml          # resources: namespace, helmrepository, helmrelease
    namespace.yaml              # Namespace cert-manager (deliberate non-"-system")
    helmrepository.yaml         # source.toolkit.fluxcd.io/v1, url https://charts.jetstack.io
    helmrelease.yaml            # helm.toolkit.fluxcd.io/v2, pinned chart, CRDs enabled via values
  gateway-api/
    standard/
      kustomization.yaml        # resources: [standard-install.yaml]  (pinned upstream manifest)
      standard-install.yaml     # vendored, pinned Gateway API standard channel CRDs
    experimental/
      kustomization.yaml        # resources: [experimental-install.yaml]
      experimental-install.yaml # vendored, pinned Gateway API experimental channel CRDs
  ingress/
    envoy-gateway/
      kustomization.yaml        # resources: namespace, helmrepository, helmrelease, gatewayclass
      namespace.yaml            # Namespace envoy-gateway-system (chart default)
      helmrepository.yaml       # EG OCI/HTTP Helm source (confirm OCIRepository vs HelmRepository — W1)
      helmrelease.yaml          # pinned chart; bundled Gateway API CRDs DISABLED (W1)
      gatewayclass.yaml         # GatewayClass → controllerName gateway.envoyproxy.io/gatewayclass-controller
```
> Decision for the implementor to confirm via W1/W5: Envoy Gateway is distributed as an **OCI** chart (`oci://docker.io/envoyproxy/gateway-helm`). If so, the source is an `OCIRepository` (source.toolkit.fluxcd.io) referenced from the HelmRelease `chartRef`, OR a `HelmRepository type: oci`. Match whichever the repo already uses elsewhere; cilium/ccm/openebs use classic `HelmRepository`. Keep it consistent and pinned.

### 3.2 USER-OWNED kube1 ClusterIssuers (Phase B-configs)
```
flux/infrastructure/configs/overlays/clusters/kube1/
  kustomization.yaml                 # add the two issuer files to resources (currently only ../../common)
  clusterissuer-letsencrypt-staging.yaml
  clusterissuer-letsencrypt-prod.yaml
```
- Both: `cert-manager.io/v1` `ClusterIssuer`, ACME server (staging `https://acme-staging-v02.api.letsencrypt.org/directory`, prod `https://acme-v02.api.letsencrypt.org/directory`), `privateKeySecretRef` (e.g. `letsencrypt-staging-account-key` / `letsencrypt-prod-account-key`), `solvers[0].dns01.cloudflare` referencing the token Secret **by name** (`apiTokenSecretRef: { name: cloudflare-api-token, key: api-token }`) in the `cert-manager` namespace.
- `acme.email`: USER-OWNED placeholder value, fillable in this file (LE accepts no-contact; document this).
- NOTE: these are **user-owned overlay content** (ADR-017), NOT render-selected and NOT templated. They live in the per-cluster overlay, not in `configs/base/` (base is template/shared and stays `resources: []` this slice).
- Wiring: `infrastructure-configs` already `dependsOn: infrastructure-controllers` with `wait: true` (flux/clusters/kube1/infrastructure.yaml) → cert-manager CRDs exist before the issuers apply. No Flux Kustomization change needed.

---

## 4. Render wiring decisions (Phase A — `config_render` role, structural ONLY)

ADR-011: render must NOT template any Flux resource body. It only **selects** catalog paths and emits the one new config field.

- **Schema** (`config/schemas/cluster-input.schema.json`): add a top-level `ingress` object (sibling of `features`), `additionalProperties: false`, single property `gatewayApiChannel` enum `["standard","experimental"]`. This is the **ONLY** new config field. Do **NOT** add any `certManager.*` value fields (acmeEmail/dnsZone/secret are user-owned overlay content, not config). `features.ingress` + `features.certManager` already exist.
- **Defaults** (`config/defaults/cluster.yaml`): add `ingress:\n  gatewayApiChannel: standard`.
- **kube1 config** (`config/clusters/kube1/cluster.yaml`): set `features.certManager: true`, `features.ingress: envoy-gateway`, and add `ingress:\n  gatewayApiChannel: standard`.
- **compute.yml** (`config_render_flux_features`, lines ~157-162): extend the post-bootstrap Flux feature list to append:
  - `cert-manager` when `features.certManager`;
  - `envoy-gateway` AND `gateway-api-<channel>` (channel from `ingress.gatewayApiChannel`) when `features.ingress == 'envoy-gateway'`.
- **render_flux.yml** (`config_render_flux_selected_resources`, lines ~32-39): map the new feature tokens to catalog paths:
  - `cert-manager` → `../../_components/cert-manager`
  - `envoy-gateway` → `../../_components/ingress/envoy-gateway`
  - `gateway-api-standard` → `../../_components/gateway-api/standard`
  - `gateway-api-experimental` → `../../_components/gateway-api/experimental`
  - The `.j2` template (`templates/flux/selected/kustomization.yaml.j2`) joins the list generically → **no template change expected**, only the `set_fact` resources computation. **Ordering:** emit Gateway API CRDs **before** envoy-gateway (GatewayClass) in the resources list (W2 belt-and-suspenders even though Flux retries).
- **validate.yml**: add a pick-one check for `ingress.gatewayApiChannel ∈ {standard, experimental}` (mirror the existing `features.ingress` pick-one block, lines ~104-111). **KEEP** the existing "cert-dependent features require cert-manager" rule (lines ~113-121) — it already covers the ingress→certManager dependency. **No new cert-manager validation.**
- **NO configs-layer generated/selected pattern** this slice — issuers are user-owned overlay content, not render-selected. Render touches ONLY the controllers layer.

---

## 5. Ordered, dependency-aware steps

**Phase A (Ansible render)** and **Phase B (K8s catalog + user issuers)** are largely file-independent and run **IN PARALLEL** after Step 0. The single hard cross-dependency: the render's generated/selected output (Step A4 verify) needs the catalog paths from Phase B to exist for a clean `kustomize build` of `generated/selected`. Reviewers (Phase C) run in parallel after their lane. Docs/ADR (Phase D) then report (Phase E) last.

```
Step 0 (research, shared)
        │
   ┌────┴───────────────┐
   ▼                    ▼
Phase A (ansible)   Phase B (k8s)         ← PARALLEL
A1 schema/defaults  B1 cert-manager base
A2 kube1 config     B2 gateway-api bases
A3 compute+flux+val B3 envoy-gateway base + GatewayClass
                    B4 kube1 ClusterIssuers (configs overlay)
   └────┬───────────────┘
        ▼
A4 full render verify (needs B1–B3 paths to build generated/selected)
        │
   ┌────┴────┐
   ▼         ▼
C1 ansible  C2 k8s reviewer   ← PARALLEL
        ▼
Phase D: D1 docs-writer  +  D2 adr-writer   ← PARALLEL
        ▼
Phase E: E1 report-writer (LAST)
```

### Step 0 — Version & doc research (shared prerequisite) · `web-fast-context` (dispatched by the lead, results shared to B1/B2/B3)
- **Input:** §2 watch-outs W1–W4.
- **Output:** a short confirmed-facts note: cert-manager chart version + CRD values key + jetstack URL (W3); Envoy Gateway chart version + image tag + Gateway-API-version it targets + the bundled-CRD-disable values key + source type OCI-vs-HTTP (W1); the Gateway API release version for standard & experimental channels (W1); confirmation of GatewayClass `controllerName` string. All pins, with doc URLs.
- **Validation:** facts cite official docs (cert-manager.io, gateway.envoyproxy.io, gateway-api.sigs.k8s.io, fluxcd.io). No code yet.
- **Dependency:** none. Blocks B1/B2/B3 manifest authoring (versions) but NOT Phase A.

### Phase A — Render compiler (agent: `ansible-implementor`)

**Step A1 — Schema + defaults** · `ansible-implementor`
- **Files:** `config/schemas/cluster-input.schema.json` (add top-level `ingress` object per §4), `config/defaults/cluster.yaml` (add `ingress: { gatewayApiChannel: standard }`).
- **Output:** schema validates as JSON; defaults file carries the new key with a clarifying comment.
- **Validation:** schema is valid JSON; defaults still parse; yaml-language-server `$schema` reference resolves. No render run yet.
- **Dependency:** Step 0 not required (structural only).

**Step A2 — kube1 config inputs** · `ansible-implementor`
- **File:** `config/clusters/kube1/cluster.yaml`: `features.certManager: true`, `features.ingress: envoy-gateway`, add `ingress: { gatewayApiChannel: standard }`. Inline comment: enabling ingress requires certManager (existing validation), and the channel knob is structural-select (ADR-012).
- **Output:** kube1 config carries the three changes.
- **Validation:** deferred to A4 (full render).
- **Dependency:** A1 (schema must accept the new field first).

**Step A3 — compute + render_flux + validate** · `ansible-implementor`
- **Files:** `infra/ansible/roles/config_render/tasks/compute.yml`, `.../render_flux.yml`, `.../validate.yml` (and `templates/flux/selected/kustomization.yaml.j2` only if the join needs adjustment — expected NOT).
- **Output:** `config_render_flux_features` appends cert-manager / envoy-gateway / gateway-api-`<channel>`; `config_render_flux_selected_resources` maps them to the four catalog paths (CRDs ordered before envoy-gateway); validate.yml gains the `gatewayApiChannel` pick-one assert. Mirror existing comment/style blocks.
- **Validation:** `ansible-lint`. Functional render checked in A4.
- **Dependency:** A1 (uses the new field). Independent of Phase B file-wise.

**Step A4 — Full render verification** · `ansible-implementor` (after C1 fixes too; first pass here)
- **Action:** run `ansible-playbook playbooks/render-config.yml -e config_cluster=kube1` and `-e config_render_check=true` (check mode) and confirm:
  - `flux/infrastructure/controllers/generated/selected/kustomization.yaml` lists `../../_components/cert-manager`, `../../_components/gateway-api/standard`, `../../_components/ingress/envoy-gateway` alongside cilium / hcloud-ccm / openebs, with Gateway API CRDs ordered before envoy-gateway.
  - Toggle test: `features.certManager:false` + `features.ingress:none` removes all three entries; check-mode diffs CLEAN (no phantom drift).
  - Channel test: flipping `ingress.gatewayApiChannel: experimental` swaps `gateway-api/standard` → `gateway-api/experimental` (exactly one selected).
- **Then** re-run `kustomize build flux/infrastructure/controllers/generated/selected` to confirm the whole controllers overlay builds with the new catalog entries present.
- **Validation:** clean check-mode diff + successful `kustomize build`.
- **Dependency:** A3 + **B1, B2, B3** (catalog dirs must exist for the build to resolve the new paths). This is the one cross-phase join.

### Phase B — K8s catalog + user issuers (agent: `k8s-implementor`)

**Step B1 — cert-manager catalog base** · `k8s-implementor`
- **Dir:** `flux/infrastructure/controllers/_components/cert-manager/` (ordinary kustomize base per `_components/README.md`):
  - `namespace.yaml` — Namespace **`cert-manager`** (deliberate non-`-system`; header comment cites the webhook/docs reason — W6).
  - `helmrepository.yaml` — `source.toolkit.fluxcd.io/v1 HelmRepository`, `url: https://charts.jetstack.io` (confirm W3), `interval: 1h`, namespace `cert-manager`.
  - `helmrelease.yaml` — `helm.toolkit.fluxcd.io/v2 HelmRelease`, name/releaseName `cert-manager`, `targetNamespace: cert-manager`, `install.createNamespace: true`, chart `cert-manager` pinned (W3/W4), CRDs enabled via the confirmed values key (`crds.enabled: true` / `installCRDs: true` — W3). **NO issuers, NO DNS, NO secrets** (CA/DNS-agnostic). Pin any surfaced image tags.
  - `kustomization.yaml` — `resources: [namespace.yaml, helmrepository.yaml, helmrelease.yaml]`.
- **Output:** self-contained cert-manager controller base.
- **Validation (k8s lint gateway):** `kustomize build .../cert-manager`; `flux build`/`helm template` of the HelmRelease values; `kubeconform`; `yamllint`.
- **Dependency:** Step 0 (version pins).

**Step B2 — Gateway API channel bases** · `k8s-implementor`
- **Dirs:** `_components/gateway-api/standard/` and `_components/gateway-api/experimental/`:
  - Vendor the pinned upstream `standard-install.yaml` / `experimental-install.yaml` for the Gateway API version confirmed in W1. Header comment records the source URL + pinned version + that it MUST match the Envoy Gateway chart's expected Gateway API version.
  - Each `kustomization.yaml`: `resources: [<channel>-install.yaml]`.
- **Output:** two pinned channel bases; render selects exactly one.
- **Validation:** `kustomize build` each; `kubeconform` (CRD schemas); `yamllint`. (These are large CRD manifests — confirm `yamllint`/`kubeconform` config tolerates upstream formatting; match how openebs/cilium handle vendored manifests.)
- **Dependency:** Step 0 (Gateway API version pin from W1).

**Step B3 — Envoy Gateway catalog base + GatewayClass** · `k8s-implementor`
- **Dir:** `_components/ingress/envoy-gateway/`:
  - `namespace.yaml` — Namespace **`envoy-gateway-system`** (chart default).
  - `helmrepository.yaml` (or `ocirepository.yaml` — confirm source type W1) — pinned EG Helm source.
  - `helmrelease.yaml` — pinned chart + image tag; **bundled Gateway API CRDs DISABLED** via the confirmed values key (W1) so the `gateway-api/<channel>` base is the single source of truth.
  - `gatewayclass.yaml` — `gateway.networking.k8s.io/v1 GatewayClass`, `spec.controllerName: gateway.envoyproxy.io/gatewayclass-controller` (confirm exact string W1). Controller-level binding is IN scope; no `Gateway`/parametersRef/EnvoyProxy this slice.
  - `kustomization.yaml` — `resources: [namespace.yaml, helmrepository.yaml, helmrelease.yaml, gatewayclass.yaml]`.
- **Output:** Envoy Gateway controller base + GatewayClass.
- **Validation:** `kustomize build`; `flux build`/`helm template` (confirm bundled CRDs are actually suppressed in the rendered output — grep the template for `CustomResourceDefinition` of Gateway API kinds and confirm absent); `kubeconform`; `yamllint`.
- **Dependency:** Step 0; logically pairs with B2 (W1 version match) — same agent, sequence B2 before/with B3.

**Step B4 — kube1 ClusterIssuers (user-owned configs overlay)** · `k8s-implementor`
- **Dir:** `flux/infrastructure/configs/overlays/clusters/kube1/`:
  - `clusterissuer-letsencrypt-staging.yaml` + `clusterissuer-letsencrypt-prod.yaml` per §3.2 (ACME, Cloudflare DNS-01, token Secret referenced BY NAME, `acme.email` placeholder, on `dragenet.dev` per kube1 ADR-004).
  - Update `kustomization.yaml` `resources:` to add the two files alongside `../../common`.
- **Output:** two issuers wired into the kube1 configs overlay chain (→ `../../common` → `../../base`).
- **Validation:** `kustomize build flux/infrastructure/configs` (whole chain); `kubeconform` against cert-manager CRD schemas (use a schema location that includes cert-manager CRDs, or `--ignore-missing-schemas` with a note — match repo convention); `yamllint`. **Do NOT** add the Cloudflare Secret; **do NOT** wire SOPS (deferred — §7).
- **Dependency:** none on Phase A; conceptually depends on B1 (cert-manager CRDs define `ClusterIssuer`) only at live-apply time, enforced by the existing Flux `dependsOn` (no manifest change). Build-time `kubeconform` of issuers needs cert-manager CRD schemas available — note in validation.

### Phase C — Review (PARALLEL)

**Step C1 — Ansible review** · `ansible-reviewer`
- **Audit** A1–A3: schema correctness (top-level `ingress`, `additionalProperties:false`, no stray `certManager.*` fields), defaults/kube1 config, compute list extension, render_flux path mapping + CRD-before-controller ordering, the new `gatewayApiChannel` pick-one assert, that the existing cert-dependent rule is KEPT and no new cert-manager validation was added, no regression to openebs/cilium/ccm/flux-auth logic, idempotency, FQCN, `config_render_*` namespacing. Reason about `--check --diff` cleanliness.
- **Output:** findings; loop fixes back to `ansible-implementor`; A4 re-run after fixes.
- **Validation:** `ansible-lint`.
- **Dependency:** A3 (and ideally A4 first pass).

**Step C2 — K8s review** · `k8s-reviewer`
- **Audit** B1–B4: all versions pinned (cert-manager + EG charts + images + Gateway API manifest; no `:latest`/floating — W4); cert-manager Namespace is `cert-manager` and EG is `envoy-gateway-system` (W6); EG bundled Gateway API CRDs confirmed disabled (W1); GatewayClass `controllerName` correct; channel manifest version matches EG's expected Gateway API version (W1); CRD-before-GatewayClass ordering / Flux retry reasoning (W2); catalog-contract compliance (ordinary bases, header comments, `---` doc starts, resource ordering matching openebs/cilium); issuers reference the token Secret by name and do NOT embed secrets; configs overlay chain builds; NO out-of-scope Gateway/Certificate/HTTPRoute/EnvoyProxy present.
- **Output:** findings; loop fixes back to `k8s-implementor`.
- **Validation:** `kustomize build` (all new bases + configs chain), `flux build`, `helm template`, `kubeconform`, `yamllint`.
- **Dependency:** B1–B4.

### Phase D — Docs & ADR (PARALLEL, after reviews pass)

**Step D1 — Runbook + CLAUDE.md status/architecture proposal** · `docs-writer`
- **New runbook** under `docs/` (e.g. `docs/tls-ingress.md` or `docs/kube1/cert-manager-envoy-gateway.md`): how to enable (`features.certManager: true`, `features.ingress: envoy-gateway`, `ingress.gatewayApiChannel` → render → flux reconcile); the channel knob (standard vs experimental); how to verify controllers Ready and ACME account registration; and — prominently — the **DEFERRED GAPS**: (a) the Cloudflare token Secret is USER-PROVISIONED and NOT committed, Flux SOPS-with-age decryption is NOT wired this slice, so DNS-01 challenges will NOT solve until the operator provisions `cloudflare-api-token` (key `api-token`) in `cert-manager` (and later SOPS lands); ACME account still goes Ready; (b) no Gateway/wildcard Certificate/HTTPRoute/host-network EnvoyProxy yet (data-path follow-up). Operational only — no rationale/ADR content.
- **CLAUDE.md (USER-owned index):** propose the diff (do NOT silently rewrite): Architecture table line ~65 `Ingress | TBD` → `Ingress | Envoy Gateway`; Implementation status (lines ~168-174) move cert-manager + ingress from "Still deferred" toward "Working — built/reviewed, live-unverified", with the deferred-secret/SOPS caveat noted. Flag for the user to apply.
- **Output:** runbook file + proposed CLAUDE.md diff.
- **Validation:** internal consistency; links resolve; no invented behavior beyond built/reviewed state.
- **Dependency:** C1+C2 green.

**Step D2 — ADR / status** · `adr-writer`
- ADR-008 (cert-manager) and cluster-bootstrap ADR-004 (Envoy Gateway) and kube1 ADR-004 (LE+Cloudflare) are **already Accepted** — do NOT re-litigate. **Only add/update if a genuinely new decision was made.** The candidate new decision: **`ingress.gatewayApiChannel` as a structural-select config knob** (ADR-012 structural-enable) that owns the Gateway API CRD channel as the single source of truth (chart-bundled CRDs disabled). If the implementor/reviewer judges this a real decision with a trade-off (channel-override-as-structural-config; standard-default vs experimental for future TCPRoute/GRPCRoute), record it as a short **addendum to ADR-012** (or a brief note on ADR-004), NOT a brand-new ADR. Also note the deliberate `cert-manager` namespace deviation (W6) where ADR-008 is silent. If no new decision → record "no ADR change required" and stop.
- **Output:** ADR addendum (or an explicit no-op note).
- **Validation:** Nygard structure; no duplication of Accepted ADRs.
- **Dependency:** C1+C2 green. Independent of D1.

### Phase E — Report (LAST)

**Step E1 — Closing report** · `report-writer`
- **File:** `.ai/reports/2026-06-29-cert-manager-envoy-gateway.md`: what changed across `config/`, `infra/ansible/`, `flux/infrastructure/controllers/_components/`, `flux/infrastructure/configs/overlays/clusters/kube1/`, `docs/`; what was validated (ansible-lint, render check-mode toggle/channel tests, `kustomize build`/`flux build`/`helm template`/`kubeconform`/`yamllint`, both reviews); the residual risks (§7) and the explicit DEFERRED gaps (secret/SOPS + data path); and the live-apply steps the user must run (§8).
- **Dependency:** everything above. Call LAST.

---

## 6. Cross-cutting constraints checklist (every implementor step must honor)
- **One writer per file (ADR-011):** render selects catalog paths only; it NEVER templates a Flux resource body. Issuers are hand-authored user-owned overlay content, never rendered.
- **Structural-enable + overlay-tweak (ADR-012):** the only new config field is `ingress.gatewayApiChannel`. No `certManager.*` value fields. Acme email / DNS zone / secret name live in the user-owned issuer files.
- **CA/DNS-agnostic catalog:** the cert-manager base contains NO issuers/DNS/secrets. Cluster-specific issuance lives in the configs overlay.
- **Pin everything (W4):** cert-manager + EG charts and images, Gateway API manifest — all pinned, doc source in header comments. No `:latest`/floating tags.
- **Channel = single source of truth (W1):** EG chart's bundled Gateway API CRDs DISABLED; the `gateway-api/<channel>` base is the only CRD source; versions must match.
- **Scope stop:** NO Gateway, wildcard Certificate, HTTPRoute, or host-network EnvoyProxy. GatewayClass (controller binding) is the furthest data-path object included.
- **Lint gateways before handoff:** Ansible lane — `ansible-lint` + render `--check` diff (clean) + regenerate committed `generated/selected` and review diff. K8s lane — `kustomize build` (every new/changed kustomization) + `flux build` + `helm template` + `kubeconform` + `yamllint`.
- **No live ops by agents:** render/lint/build/review only.

---

## 7. Deferred gaps & residual risks
- **DEFERRED-1 (documented, not built):** Cloudflare token Secret is USER-PROVISIONED and NOT committed; Flux SOPS-with-age decryption is NOT wired this slice (`.sops.yaml` currently scopes only `infra/ansible/inventories/**` group_vars). → DNS-01 challenges will NOT solve until the operator creates the `cloudflare-api-token` Secret (and later Flux SOPS lands). ACME account registration still reaches Ready. Must be called out in the runbook (D1) and report (E1).
- **DEFERRED-2 (documented, not built):** no Gateway / `*.dragenet.dev` Certificate / HTTPRoute / host-network EnvoyProxy — the data-path follow-up slice.
- **R1 (W2 — verify):** GatewayClass-before-CRD ordering inside the single `infrastructure-controllers` Kustomization relies on Flux's retry of CRs whose CRD is not yet registered (+ `wait: true`, retryInterval 1m). Strongly expected to converge; if reviewer judges the race unacceptable, fallback is a separate Flux Kustomization for Gateway API CRDs with `dependsOn` (larger change — defer unless required).
- **R2 (W1 — verify at impl time):** exact EG chart version ↔ Gateway API version ↔ bundled-CRD-disable values key. A mismatch (chart expects a newer/older Gateway API than the pinned channel manifest) breaks GatewayClass/CRD compatibility silently. Reviewer must confirm the pinned versions agree.
- **R3 (W3 — verify):** cert-manager chart support for k8s v1.36 and the correct CRD-install values key for the pinned version.
- **R4 (W1 — verify):** EG Helm distribution form (OCI vs classic HTTP repo) → `OCIRepository`/`HelmRepository type: oci` vs classic `HelmRepository`. Keep consistent with repo conventions.
- **Live-unverified:** per CLAUDE.md, this plan delivers built+reviewed+validated code, not observed-running behavior.

---

## 8. Live-apply sequence (USER runs — agents must NOT)
1. `ansible-playbook playbooks/render-config.yml -e config_cluster=kube1` — regenerate `generated/selected/kustomization.yaml`; review git diff (cert-manager + gateway-api/standard + ingress/envoy-gateway added).
2. Commit + push; Flux reconciles `infrastructure-controllers` (`flux reconcile kustomization infrastructure-controllers --with-source`): Gateway API CRDs + cert-manager + Envoy Gateway controllers + GatewayClass apply. Confirm `kubectl get pods -n cert-manager`, `-n envoy-gateway-system`, and `kubectl get gatewayclass`.
3. `infrastructure-configs` reconciles after controllers (existing `dependsOn` + `wait`): the two ClusterIssuers apply. Confirm `kubectl get clusterissuer` — `letsencrypt-staging`/`letsencrypt-prod` ACME account should reach Ready.
4. **Provision the Cloudflare token Secret manually** (`cloudflare-api-token` / key `api-token` in `cert-manager`, scoped `Zone:DNS:Edit` on `dragenet.dev`) — DEFERRED-1. Until then DNS-01 challenges won't solve (no Certificate is requested this slice anyway).

---

## 9. Return summary (sequence + parallelization)
- **Step 0** shared version/doc research (`web-fast-context`) feeds the catalog manifests.
- **Phase A** (`ansible-implementor`: schema/defaults → kube1 config → compute/render_flux/validate) and **Phase B** (`k8s-implementor`: cert-manager base ∥ gateway-api bases ∥ envoy-gateway+GatewayClass ∥ kube1 ClusterIssuers) run **in parallel** — no shared files.
- **Join:** Step A4 full render verify needs Phase B catalog dirs to exist to build `generated/selected`.
- **Phase C** reviewers (`ansible-reviewer` ∥ `k8s-reviewer`) in parallel, looping fixes back.
- **Phase D** `docs-writer` (runbook + CLAUDE.md proposal) ∥ `adr-writer` (ADR-012 addendum only if the channel-knob is a real new decision; else no-op).
- **Phase E** `report-writer` last.
