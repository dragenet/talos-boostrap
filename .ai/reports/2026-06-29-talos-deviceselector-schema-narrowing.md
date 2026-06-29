# Report: Talos deviceSelector migration + custom-schema narrowing (ADR-016 closeout)

**Date:** 2026-06-29
**Plan:** `.ai/plans/2026-06-29-talos-deviceselector-schema-narrowing.md`
**ADR closed:** `docs/cluster-bootstrap/adrs/016-talos-config-boundary.md` — this closes the final two ADR-016 technical-debt items
**Type:** Config + render cleanup — no live cluster operations, no behavior change other than the VIP interface binding (selector vs kernel name)

---

## What changed

Two coordinated changes in one slice (shared file set, deliberately implemented as a single ansible-implementor task):

### Debt 1 — deviceSelector migration

Replaced the unstable kernel-name pin `talos.network.defaultInterface.interface: eth1` with a hardware-identity match `talos.network.defaultInterface.deviceSelector: { hardwareAddr: '86:00:00:*' }` — the Hetzner private-network OUI glob, cluster-wide, one expression covering all 3 nodes. The Live Talos probe (`talosctl --talosconfig infra/talos/secrets/talosconfig get links` on a kube1 node) already confirmed the private NIC `eth1` carries HW addr `86:00:00:4d:a8:b8`; the public NIC's MAC is **not** `86:00:00:*` (Step-0 PASS → use the agreed glob). The controlplane `00-network.yaml.j2` template already supported `deviceSelector` (the deviceSelector branch on L20–28 falls back to `interface:` on L29–33) — **no template change needed**, this is config-only.

### Debt 2 — schema narrowing

Removed the now-dead transitional custom-schema keys `talos.install.{disk,diskSelector}` and `talos.patches.{all,controlplane}` from the schema, validate asserts, render pipeline, default config, and the kube1 cluster config. KEEP set untouched: `talos.network.defaultInterface` (active VIP↔interface coupling — the computed `15-network.yaml.j2` fragment sources the VIP from `cluster.vip` and the interface from this key, so removing it would duplicate the VIP source of truth, which ADR-016 deliberately avoided), `talos.version`, `talos.extensions`, `storage.volumes`, the VIP single-source assert (`validate.yml` L209–217), and the controlplane merge-key hazard assert (`validate.yml` L374–415).

The removed keys are now **rejected** at validate time (was previously: silently ignored as unknown schema keys). Five new `assert` tasks in `validate.yml` explicitly fail the render if any of `talos.install`, `talos.install.disk`, `talos.install.diskSelector`, `talos.patches.all`, or `talos.patches.controlplane` is present in the merged config. The error messages point users at the per-cluster Talos overlay (the replacement path) so the migration story is self-explanatory.

---

## Files changed (commitable)

### Config (4 files)
- `config/clusters/kube1/cluster.yaml` — `talos.install` and `talos.patches` blocks removed; `talos.network.defaultInterface.interface: eth1` → `deviceSelector: { hardwareAddr: '86:00:00:*' }`; comment refreshed.
- `config/clusters/kube1/talos/controlplane.yaml` — comment updated to reflect that the deviceSelector now lives in `cluster.yaml` and the computed `15-network.yaml.j2` fragment still owns the VIP.
- `config/defaults/cluster.yaml` — `talos.install` and `talos.patches` blocks removed; `talos.network.defaultInterface` defaults (interface/deviceSelector both null — generic for other clusters) kept.
- `config/schemas/cluster-input.schema.json` — `talos.install` and `talos.patches` definitions removed; `talos.network` (with `defaultInterface.{interface,deviceSelector}`) kept; trailing comma fixed (was invalid JSON).

### Ansible / render (4 files)
- `infra/ansible/roles/config_render/tasks/validate.yml` — old install asserts removed; 5 new rejection asserts added for the dead keys.
- `infra/ansible/roles/config_render/tasks/render_talos.yml` — both `90-user-raw` render + remove-stale tasks removed (all-scope and controlplane-scope); order comments at L56 and L270 updated to drop the `90-user-raw` reference.
- `infra/ansible/roles/config_render/templates/talos_patches/all/00-install.yaml.j2` — comment updated.
- `infra/ansible/roles/config_render/templates/talos_patches/all/30-raw-volumes.yaml.j2` — comment updated.
- `infra/ansible/roles/talos_config/tasks/main.yml` — comments updated.

### Files deleted (2)
- `infra/ansible/roles/config_render/templates/talos_patches/all/90-user-raw.yaml.j2`
- `infra/ansible/roles/config_render/templates/talos_patches/controlplane/90-user-raw.yaml.j2`

### Docs / ADR (2 files)
- `CLAUDE.md` — Implementation status: deferred `talosctl` deviceSelector probe and schema-narrowing follow-up removed; selector change noted; "live-unverified until applied" honesty caveat preserved.
- `docs/cluster-bootstrap/adrs/016-talos-config-boundary.md` — **Implementation (2026-06-29) addendum** section added: deviceSelector migrated to the `86:00:00:*` OUI glob, transitional `talos.install.*` / `talos.patches.*` schema keys removed, with the rationale for KEEPING `talos.network.defaultInterface` (VIP coupling).

---

## What was validated

| Gate | Result |
|---|---|
| Render: `playbooks/render-config.yml -e config_cluster=kube1` | First run: 20 changed (expected). |
| Render: `config_render_check=true` (idempotent) | "No render drift detected. Committed generated files match render output." |
| `ansible-lint` (all changed task files) | 0 failures, 0 warnings (61 files). |
| JSON Schema validity | `python3 -m json.tool config/schemas/cluster-input.schema.json` clean (trailing-comma fix verified). |
| Rendered controlplane network fragment (`generated/controlplane/15-network.yaml.j2`) | Emits `deviceSelector: { hardwareAddr: 86:00:00:* }` — no `interface: eth1` line. |
| Stale `90-user-raw*` artifacts in generated tree | None — `generated/all/` shows only `10-external-cloud-provider`, `20-install`, `25-kernel-modules`, `30-raw-volumes`; `generated/controlplane/` shows only `10-cilium`, `15-network`. |
| 5 new rejection asserts (`talos.install`, `talos.install.disk`, `talos.install.diskSelector`, `talos.patches.all`, `talos.patches.controlplane`) | Pass (ok) on normal + check-mode render — confirmed not to trip on the narrowed config. |
| `talos.network.defaultInterface` / `talos.version` / `talos.extensions` / `storage.volumes` | Unchanged in schema, defaults, and merged config — KEEP set honored. |
| VIP single-source assert + controlplane merge-key hazard assert | Preserved. |

### Reviewer findings and fixes

The ansible-reviewer found three issues, all fixed by the implementor before handoff:

| # | Severity | Finding | Fix |
|---|---|---|---|
| 1 | HIGH | Trailing comma in `cluster-input.schema.json` (invalid JSON) | Removed trailing comma. |
| 2 | HIGH | Removed transitional keys (`talos.install.disk`, `talos.install.diskSelector`, `talos.install`, `talos.patches.all`, `talos.patches.controlplane`) silently ignored at validate time instead of rejected — schema and config could drift. | Added 5 new rejection asserts in `validate.yml` (one per removed key). |
| 3 | LOW | Stale comments referencing removed `90-user-raw` / `talos.patches.*`. | Updated 5 comments across 4 files. |

---

## What was NOT done (deliberately deferred outside this slice)

- **Live cluster apply.** A wrong selector isolates a control-plane node. Per the plan, the user must run staged `--mode=no-reboot` one node at a time against the rendered controlplane config and verify Ready + VIP reachability + interface address after each node (hb1 → hb2 → hb3) before proceeding. Revert the config to roll back. Gated on the Step-0 probe (already PASS). **Highest-risk step — does not run from agents.**
- `flux/apps/` still empty — not part of this work.
- Legacy `config/overrides/` is still on disk as a fallback (separate cleanup slice, per the prior report's follow-up).

---

## Follow-ups / risks

- **The `86:00:00:*` glob depends on Hetzner NOT changing the private-network OUI.** Hetzner owns that OUI; if they reassign it, the selector stops matching and the VIP loses its interface binding. Migration to a per-node `permanentAddr` overlay (the agreed Step-0 fallback) is a config-only change — the template already supports it.
- **The new rejection asserts now make the transitional keys a hard error rather than a silent ignore.** Any user still carrying the old keys in a `config/clusters/<name>/cluster.yaml` will fail the render with a clear pointer at the per-cluster Talos overlay as the replacement path. The error message text is the only migration aid — no codemod / no auto-rewrite. Expected: nobody outside the original migration carries these.
- **`talos.network.defaultInterface` survives as an active schema key.** It owns the interface selector for the VIP binding in the computed `15-network.yaml.j2` fragment. ADR-016 deliberately did not push this into a `talos.network.interfaces` raw fragment because that would duplicate the VIP source of truth (cluster.vip feeds the VIP, the fragment sources the interface from `talos.network.defaultInterface`). Documented in the ADR-016 addendum.
- **Live cluster state is unverified.** As with prior slices, "Working" in CLAUDE.md means the code is built, reviewed, and passes validation — not that `bootstrap-flux.yml` or `talosctl apply-config` has been observed running on a live cluster with the new selector. The same live-validation commands from the cert-manager/Envoy-Gateway and pre-Flux reports apply: Cilium `daemonset/cilium` Ready, no `uninitialized` taint, every `providerID` starts with `hcloud://`, `flux-system` pods Ready, all HelmReleases Ready without install/uninstall churn — and additionally, the VIP (`https://10.0.0.10:6443`) is reachable from a client and `talosctl get addresses -n <node>` shows `10.0.0.x` on the interface matched by `hardwareAddr: '86:00:00:*'`.
- **Reporter's note — out-of-scope finding flagged to the user:** the ansible-reviewer noticed `infra/ansible/infra/talos/secrets/talosconfig` exists as an untracked file outside the existing gitignore path (`.gitignore` only covers `infra/talos/secrets/`, not the nested `infra/ansible/infra/...` path). The file is presumably left over from a working-out-of-tree experiment and should be either moved into the proper gitignored path or deleted. **Not fixed as part of this slice** — flagging for user decision.

---

## Artifacts

- Plan: `.ai/plans/2026-06-29-talos-deviceselector-schema-narrowing.md`
- Report: `.ai/reports/2026-06-29-talos-deviceselector-schema-narrowing.md` (this file)
- ADR addendum: `docs/cluster-bootstrap/adrs/016-talos-config-boundary.md` (Implementation (2026-06-29) section)
- Updated: `CLAUDE.md` (Implementation status — deviceSelector and schema-narrowing deferred follow-ups closed)
- Deleted: `infra/ansible/roles/config_render/templates/talos_patches/{all,controlplane}/90-user-raw.yaml.j2`
