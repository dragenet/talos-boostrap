# Provider portability refactor — closing report

**Date:** 2026-06-22
**Plan:** `.ai/plans/2026-06-22-provider-portability-refactor.md`
**Closes review notes:** A–D from `.ai/review-2026-06-10.md`
**ADR produced:** `docs/cluster-bootstrap/adrs/014-node-discovery-dynamic-inventory.md`

## What changed

Closed out the provider-portability refactor implied by review notes A–D. A–C were already converged in the repo before this session; D was mostly converged but undocumented. This session finished the small remaining gaps and recorded the dynamic-inventory decision as ADR-014.

### Findings on arrival

Empirical validation (ansible-reviewer + repo-fast-context):

- **A — Layering.** Provider seam already in target shape (`providers/<name>/` rule). No changes needed.
- **B — Playbook renaming.** Already done. Playbooks in target locations: `bootstrap-talos.yml`, `bootstrap-flux.yml`, `upgrade-talos.yml`, `providers/hcloud/{image-talos.yml, provision-infra.yml}`. No stale `import_playbook` references.
- **C — Vars split.** Already clean. `inventories/common/`: `cluster.yml`, `talos.yml`, `flux.yml`. `inventories/providers/hcloud/`: `hcloud.yml` + `hcloud.sops.yaml`. No provider vars misplaced in `common/`.
- **D — Dynamic inventory.** Mostly converged. `node_ips.yml` is gone; the `hetzner.hcloud.hcloud` dynamic inventory plugin composes `node_public_ip` / `node_private_ip` / `node_role`; roles consume `hostvars[node].node_public_ip`. One latent gap (see Changes §1) and no ADR.

A–C are recorded as Accepted in ADR-007 (ansible-portable-bootstrap). D was not previously recorded in any ADR — ADR-014 closes that gap.

### Design discovery: `node_private_ip` — static vs DHCP

Researched via official Talos docs and repo evidence:

- Talos officially supports DHCP for etcd (NodeAddress resources) and VIP on a DHCP interface; the Production Clusters guide uses `dhcp: true`. No guidance favors static IPs for etcd.
- `talosctl bootstrap` has no `--control-plane-nodes` flag — omitting it is the only supported workflow (control planes join via the discovery service). The rationale comment in `talos_config/tasks/main.yml:196-204` references a nonexistent flag but reaches the correct conclusion.
- On Hetzner, DHCP ≡ static: the provider assigns a fixed private IP at the API level (`server_network.ip`) and its DHCP serves that exact IP.
- **Conclusion:** `node_private_ip` is correctly a forward-looking contract field — declared by every provider, unconsumed on Hetzner (DHCP+VIP suffices), and available for providers like bare-metal manual that may need static addresses. Wiring static IPs into the Talos config now would add per-node patch complexity for zero benefit on the only currently active provider.

### Changes made this session

1. **`inventories/providers/hcloud/hcloud.yml`** — added `network: kube1` (with `# why` comment) so `hcloud_private_ipv4` / `node_private_ip` is reliably populated per the provider contract. Hardcoded (not Jinja) because the inventory plugin loads before `group_vars` resolve — matches the existing `label_selector` pattern; only `lookup()` plugins resolve at inventory-load time (per the official `hetzner.hcloud.hcloud` docs).
2. **`inventories/providers/hcloud/group_vars/all/hcloud.yml`** — fixed stale private-IP comments (previously said nodes start at `.2` / VIP `.10`; actual offset is 11, so nodes live at `.11+`). No variable values changed.
3. **`roles/providers/hcloud/hcloud_servers/tasks/main.yml`** — fixed stale IP example comment (line 35: `.2`/`.3` → `.11`/`.12`/`.13`). Comment only.
4. **`docs/cluster-bootstrap/adrs/014-node-discovery-dynamic-inventory.md`** — NEW ADR, Status: Accepted. Records the dynamic-inventory decision: `node_*` hostvar contract, `hetzner.hcloud.hcloud` replacing `node_ips.yml`, DHCP-on-Talos-side rationale, rejected alternatives, trade-offs. Cross-references ADR-007 and ADR-010.
5. **`docs/adding-a-provider.md`** — five drift fixes:
   - Forward-looking `node_private_ip` note
   - Added the `compose:` block to the dynamic-inventory example (it was missing — a new provider following the guide would get groups but no contract hostvars)
   - Load-time auth / SOPS-lookup callout
   - Chicken-and-egg ordering callout (group_vars depend on inventory, inventory plugins may depend on group_vars via `lookup()`)
   - ADR-014 reference

`docs/node-roles.md` and `README.md` needed no changes.

## What was validated

All checks were safe/local; no live cluster contact.

- **`ansible-lint`** — PASS (0 failures, 0 warnings, 29 files, production profile).
- **`ansible-playbook --syntax-check`** — PASS for:
  - `bootstrap-talos.yml` (hcloud)
  - `bootstrap-flux.yml` (hcloud)
  - `provision-infra.yml` (hcloud)
  - `image-talos.yml` (hcloud)
  - `bootstrap-talos.yml` (manual)
- **`ansible-inventory -i inventories/common -i inventories/providers/manual --list`** — PASS: 3 hybrid nodes with `node_role` / `node_public_ip` / `node_private_ip`.
- **`ansible-reviewer`** — approved the `network:` change; flagged and resolved the one stale role comment.
- **graphify** — graph refreshed via `graphify update .`.

## What was not run, and why

- **`ansible-inventory --list` / `--graph` for the hcloud path** — fails locally with `Failed to import the required Python library (requests)`. This is an environmental gap (missing `requests` in this venv), not a code defect. Could not empirically confirm `hcloud_private_ipv4` is populated with `network: kube1` set. **Should be verified on a host with the hcloud collection's Python deps installed** (or on first real use).
- **Live cluster operations** — out of scope; nothing in this session touches the running cluster.

## Open items / follow-ups

- **Verify hcloud dynamic inventory populates `node_private_ip`** with `network: kube1` on a host with the `requests` Python library installed. Environmental gap from this session; cheap to confirm on the next real `provision-infra.yml` run.
- **Optional accuracy nit:** the rationale comment in `roles/talos_config/tasks/main.yml:196-204` references `--control-plane-nodes`, a flag that does not exist in `talosctl bootstrap`. The conclusion is correct; the flag reference is imprecise. Left unchanged this session (out of the approved scope) — could be tidied in a future pass.
- **Flux bootstrap remains NOT built** — `flux_bootstrap` role is a stub, `flux_repo_url` is unset. Unchanged, out of scope.

## Artifacts

- Plan: `.ai/plans/2026-06-22-provider-portability-refactor.md`
- Report: `.ai/reports/2026-06-22-provider-portability-refactor.md` (this file)
- ADR: `docs/cluster-bootstrap/adrs/014-node-discovery-dynamic-inventory.md`
- Review notes closed: `.ai/review-2026-06-10.md` (A–D)
