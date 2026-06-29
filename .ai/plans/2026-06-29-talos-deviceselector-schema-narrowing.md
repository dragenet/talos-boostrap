# Plan: Talos deviceSelector migration + custom-schema narrowing (ADR-016 closeout)

Date: 2026-06-29
Status: Ready to execute (design AGREED — do not re-litigate)
Closes: the last two Talos-config technical-debt items from **ADR-016** (which superseded ADR-013)

---

## Goal

Complete the final two ADR-016 cleanups in one coordinated slice:

- **Debt 1 — deviceSelector migration.** Replace the unstable kernel-name pin
  `talos.network.defaultInterface.interface: eth1` (kube1 `cluster.yaml`) with a
  hardware-identity match: `deviceSelector: { hardwareAddr: '86:00:00:*' }`
  (Hetzner private-network OUI glob, cluster-wide, one expression for all nodes).
- **Debt 2 — schema narrowing.** Remove the now-dead transitional custom-schema
  keys `talos.install.{disk,diskSelector}` and `talos.patches.{all,controlplane}`
  (schema + validate asserts + render step + empty config blocks).

**Both debts touch the same set of files (kube1 `cluster.yaml`, schema, validate,
render_talos). They are implemented as ONE ansible-implementor task — do NOT split
into parallel tasks that would edit these files concurrently.**

### KEEP (explicitly out of scope — do not touch)
- `talos.network.defaultInterface` schema key (active VIP↔interface coupling; the
  computed `15-network.yaml.j2` fragment sources the VIP from `cluster.vip` and the
  interface from this key — removing it would duplicate the VIP source of truth,
  which ADR-016 deliberately avoided).
- `talos.version`, `talos.extensions`, `storage.volumes`.
- The VIP single-source assert (`validate.yml` L209–217).
- The controlplane merge-key hazard assert (`validate.yml` L374–415).
- The network template logic in `controlplane/00-network.yaml.j2` (it ALREADY
  renders the deviceSelector branch — Debt 1 is **config-only**, NO template change).

---

## Premises / AGREED DECISIONS (bake these in; do not reopen)

1. **deviceSelector value = cluster-wide OUI glob.** Set
   `talos.network.defaultInterface.deviceSelector: { hardwareAddr: '86:00:00:*' }`
   once in kube1 `cluster.yaml`; drop `interface: eth1`. One expression covers all
   three nodes. No per-node files.
2. **Schema-narrowing scope.** Remove `talos.install.{disk,diskSelector}` and
   `talos.patches.{all,controlplane}` from: the schema, the validate rejection
   asserts, the render step that consumes `talos.patches.*`, and the empty blocks in
   defaults + kube1 `cluster.yaml`. KEEP `talos.network.defaultInterface`,
   `talos.version`, `talos.extensions`, `storage.volumes`.

---

## Live facts (already gathered via hcloud — APPROVED, do NOT re-run)

3 kube1 nodes live (running, nbg1). Private-network NIC MACs:

| Node      | Private IP | Private NIC MAC     |
|-----------|------------|---------------------|
| kube1-hb1 | 10.0.0.11  | `86:00:00:4d:a8:b8` |
| kube1-hb2 | 10.0.0.12  | `86:00:00:4d:a8:b6` |
| kube1-hb3 | 10.0.0.13  | `86:00:00:4d:a8:b5` |

All share Hetzner private-network OUI `86:00:00`. **hcloud does NOT expose the
PUBLIC NIC MAC** — that is the one unknown, cross-checked live in Step 0.

**Live Talos probe already confirmed on the cluster:**
`talosctl --talosconfig infra/talos/secrets/talosconfig get links` on a kube1 node
showed `eth1` with HW addr `86:00:00:4d:a8:b8`, so the agreed cluster-wide
`hardwareAddr: '86:00:00:*'` selector matches the private NIC as expected.

---

## Repo facts (already verified 2026-06-29 — cited so the implementor does NOT re-discover)

- **Network template already supports deviceSelector — NO template change.**
  `infra/ansible/roles/config_render/templates/talos_patches/controlplane/00-network.yaml.j2`
  L20–28: when `talos.network.defaultInterface.deviceSelector` is a non-empty object
  it renders a `deviceSelector:` block (`{{ key }}: {{ value | to_yaml | trim }}`,
  so `hardwareAddr: '86:00:00:*'` comes out correctly quoted), else falls back to
  `interface:` (L29–33). Leave this file as-is.
- **Schema** `config/schemas/cluster-input.schema.json` (verified):
  - `talos.install` object — **L87–94** → REMOVE.
  - `talos.network` (with `defaultInterface.{interface,deviceSelector}`) — **L95–108** → KEEP.
  - `talos.patches` object — **L109–116** → REMOVE.
- **Validate** `infra/ansible/roles/config_render/tasks/validate.yml` (verified):
  - VIP single-source assert — **L209–217** → KEEP.
  - "Deferred structured helpers (ADR-013)" comment block — **L219–226** → REMOVE with the asserts.
  - `Reject talos.install.disk` assert — **L228–237** → REMOVE.
  - `Reject talos.install.diskSelector` assert — **L239–249** → REMOVE.
  - **No assert references `talos.patches.*`** (confirmed by full read) — nothing to
    remove there. The implementor must re-confirm after editing.
  - storage.volumes validation — **L251+** → KEEP.
  - controlplane merge-key hazard assert — **L374–415** → KEEP (its `fail_msg`
    mentions `talos.network.defaultInterface`; that key survives, so leave the text).
- **Render** `infra/ansible/roles/config_render/tasks/render_talos.yml` (verified):
  there are **TWO** `90-user-raw` render steps (the deprecated `talos.patches.*`
  consumers), each a render task + a "remove stale" task:
  - all-scope — **L252–265** (`talos.patches.all`) → REMOVE both tasks.
  - controlplane-scope — **L465–478** (`talos.patches.controlplane`) → REMOVE both tasks.
  - Order-comment references to `90-user-raw` at **L56** and **L270** → update to drop `90-user-raw`.
  - Template files to delete:
    `templates/talos_patches/all/90-user-raw.yaml.j2` and
    `templates/talos_patches/controlplane/90-user-raw.yaml.j2`.
  - **No committed generated `90-user-raw*` files exist** (only the two `.j2`
    templates) — kube1's `talos.patches` are empty, so the "remove stale" tasks
    currently render nothing. After removing the render+remove tasks, confirm the
    generated tree has no `90-user-raw` artifact left behind.
- **kube1** `config/clusters/kube1/cluster.yaml` (verified):
  - `talos.install` (disk/diskSelector null) — **L102–104** → REMOVE.
  - `talos.network.defaultInterface` (interface: eth1, deviceSelector: null) —
    **L105–113** → switch to the glob (drop `interface`, set `deviceSelector`).
  - `talos.patches` (all/controlplane empty) — **L114–116** → REMOVE.
- **defaults** `config/defaults/cluster.yaml` (verified):
  - `talos.install` (disk/diskSelector null) — **L111–116** → REMOVE.
  - `talos.network.defaultInterface` (interface: null, deviceSelector: null) —
    **L117–125** → KEEP (defaults stay generic for other clusters).
  - `talos.patches` (all/controlplane []) — **L126–130** → REMOVE.
- **Empty overlay** `config/clusters/kube1/talos/controlplane.yaml` (comment-only,
  L1–12): update the L8–12 comment that says "see the plan for the eventual
  migration of interface selection ... once deviceSelector is confirmed" → state
  that the deviceSelector now lives in `cluster.yaml` and the computed
  `15-network.yaml.j2` fragment still owns the VIP. (Comment-only; keep the file
  otherwise empty.)

---

## Dependency order (process)

```
Step 0 (USER live probe, read-only)  ─┐
                                       ├─ both can proceed in parallel
Implementation (config edits)        ─┘   (edits don't need probe; only the
        │                                  live APPLY waits on the probe)
        ▼
Review (ansible-reviewer)
        ▼
Docs/ADR (docs-writer + adr-writer, parallel)
        ▼
Report (report-writer)
        ▼
Live apply (USER, staged one node at a time)   ← gated on Step 0 PASS
```

Planning → implementation → review → docs/ADR → report → user live apply.
Pin nothing new; no new external tools. Keep the CLAUDE.md
"live-unverified until applied" honesty caveat.

---

## Step 0 — Live probe gate (USER runs, read-only) — agents do NOT run

**Purpose:** confirm the `86:00:00:*` glob is unambiguous before any live apply.
The config edits below can be authored in parallel with this probe; only the live
APPLY (final step) is gated on a PASS here.

Status: **already verified once** with
`talosctl --talosconfig infra/talos/secrets/talosconfig get links` on kube1;
the private NIC is `eth1` with HW addr `86:00:00:4d:a8:b8`.

USER runs against a kube1 node (e.g. `-n 10.0.0.11`):

```
talosctl get links -o yaml
talosctl get addresses
```

Confirm:
- **(a)** the NIC carrying `10.0.0.x` has MAC `86:00:00:*` (expected from hcloud), and
- **(b)** the PUBLIC NIC's MAC does **NOT** start `86:00:00` (so the glob selects
  exactly one interface per node).

**Decision fork:**
- **PASS** (public NIC MAC ≠ `86:00:00:*`) → proceed with the agreed glob
  `hardwareAddr: '86:00:00:*'`.
- **FAIL** (public NIC also matches `86:00:00`) → the glob is ambiguous. Fall back to
  a more specific selector — EITHER add a `driver:` match (e.g. virtio_net) to
  narrow to the private NIC, OR drop the cluster-wide glob and use per-node
  `permanentAddr` overlays under `config/clusters/kube1/talos/nodes/<host>.yaml`
  with each node's exact private MAC from the table above. Re-run implementation
  with the chosen fallback value. (The template already supports any
  `deviceSelector` key set, so this remains a config-only change.)

---

## Implementation — single ansible-implementor task (Debt 1 + Debt 2 together)

**Scope:** these edits share files; do NOT parallelize/split. One implementor,
one pass, then static checks.

### Edits

1. **kube1 `config/clusters/kube1/cluster.yaml`**
   - Remove the `talos.install` block (L102–104).
   - In `talos.network.defaultInterface` (L105–113): remove `interface: eth1` and
     set `deviceSelector: { hardwareAddr: '86:00:00:*' }` (use the Step-0 fallback
     value instead if Step 0 returned FAIL). Refresh the stale comment that says
     "Preserving interface: eth1 until a safe deviceSelector is confirmed".
   - Remove the `talos.patches` block (L114–116).

2. **`config/schemas/cluster-input.schema.json`**
   - Remove the `install` object (L87–94) and the `patches` object (L109–116).
   - KEEP the `network` object (L95–108).

3. **`infra/ansible/roles/config_render/tasks/validate.yml`**
   - Remove the "Deferred structured helpers (ADR-013)" comment block (L219–226) and
     both `Reject talos.install.disk` / `Reject talos.install.diskSelector` asserts
     (L228–249).
   - Re-confirm no remaining assert references `talos.patches.*` (none expected).
   - KEEP the VIP single-source assert (L209–217), storage.volumes validation
     (L251+), and the merge-key hazard assert (L374–415).

4. **`infra/ansible/roles/config_render/tasks/render_talos.yml`**
   - Remove the all-scope `90-user-raw` render + remove-stale tasks (L252–265).
   - Remove the controlplane-scope `90-user-raw` render + remove-stale tasks (L465–478).
   - Update the order comments at L56 and L270 to drop `90-user-raw`.
   - Delete the two template files:
     `templates/talos_patches/all/90-user-raw.yaml.j2` and
     `templates/talos_patches/controlplane/90-user-raw.yaml.j2`.

5. **`config/defaults/cluster.yaml`**
   - Remove the `talos.install` block (L111–116) and the `talos.patches` block
     (L126–130). KEEP the `talos.network` defaults (L117–125, interface/deviceSelector
     both null — generic for other clusters).

6. **`config/clusters/kube1/talos/controlplane.yaml`**
   - Update the L8–12 comment: deviceSelector now lives in `cluster.yaml`; the
     computed `15-network.yaml.j2` fragment still owns the VIP. Keep the file
     otherwise empty (comment-only).

   *(Note: `controlplane/00-network.yaml.j2` is NOT edited — it already renders the
   deviceSelector branch. Leave it untouched to honor "config-only".)*

### Verification (implementor runs locally — render is NOT a live cluster command)

```
ansible-playbook playbooks/render-config.yml -e config_cluster=kube1
ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true
ansible-lint
```

Confirm:
- **(i)** the rendered controlplane network fragment
  (`generated/controlplane/15-network.yaml.j2`) now emits
  `deviceSelector:` with `hardwareAddr: '86:00:00:*'` instead of `interface: eth1`;
- **(ii)** no `90-user-raw*` patch is emitted anywhere in the generated tree;
- **(iii)** the check-mode (idempotent) re-render shows **no drift**;
- **(iv)** `ansible-lint` is clean (FQCN preserved).

Report: files changed, the rendered network-fragment diff (eth1 → deviceSelector),
confirmation that no `90-user-raw` artifact remains, and check-mode/lint results.

---

## Review — ansible-reviewer (read-only)

Verify:
- Schema/validate/render/config correctness and idempotency; FQCN module names.
- `talos.network.defaultInterface`, `storage.volumes`, `talos.version`,
  `talos.extensions` are **untouched**.
- The VIP single-source assert (L209–217) and merge-key hazard assert (L374–415)
  are **preserved**.
- `talos.install.*` and `talos.patches.*` are fully gone from schema + validate +
  render + both config files (no orphan references; no dangling
  `config_render_merged.talos.patches`/`.install` lookups anywhere).
- Rendered network fragment uses the deviceSelector glob; no `90-user-raw` output.

Reviewer is read-only; if changes are needed, hand back to the implementor.

---

## Docs / ADR (parallel, after review passes)

**docs-writer:**
- Update `CLAUDE.md` Implementation status (around L173–174): remove the
  "`talosctl` deviceSelector probe (ADR-013/016) deferred" follow-up and any
  "schema narrowing deferred" follow-up; note that the deviceSelector now uses the
  `86:00:00:*` Hetzner OUI glob and the custom schema is narrowed
  (`talos.install.*` / `talos.patches.*` removed; `talos.network.defaultInterface`,
  `version`, `extensions`, `storage.volumes` retained). Preserve the
  "live-unverified until applied" honesty caveat. CLAUDE.md is USER-owned — propose
  the diff rather than silently rewriting if repo convention requires.
- Update any doc still referencing the literal `eth1` pin as the current state
  (check the ADR-016 migration report `.ai/reports/2026-06-25-...` L22 note about
  the deferred probe — mark it closed/superseded as appropriate).

**adr-writer (EVALUATE — recommend, do not force):**
- This completes ADR-016's stated intent rather than introducing a new decision, so
  the likely right artifact is a **short addendum to ADR-016** (or its report)
  recording: deviceSelector migrated to the `86:00:00:*` OUI glob, and the
  transitional `talos.install.*` / `talos.patches.*` schema keys removed, with the
  rationale for KEEPING `talos.network.defaultInterface` (VIP coupling). A brand-new
  ADR is probably overkill — recommend the addendum and let the user decide.

---

## Report — report-writer (after review)

Write `.ai/reports/2026-06-29-talos-deviceselector-schema-narrowing.md`: what
changed (Debt 1 + Debt 2), the render/check/lint results, the Step-0 probe outcome
and which selector value was used (glob vs fallback), the live-apply steps deferred
to the user, and any follow-ups. Keep the "live-unverified until applied" caveat.

---

## Live apply (USER runs — agents do NOT run; documented as steps)

**Highest-risk step: a wrong selector isolates a control-plane node.** Gated on
Step 0 = PASS (or the agreed fallback selector).

1. Render + commit + push the change (after the user approves committing).
2. Apply the new machine config **one node at a time**, starting with
   `--mode=no-reboot`:
   ```
   talosctl apply-config -n 10.0.0.11 --mode=no-reboot -f <rendered controlplane config>
   ```
3. **After each node, verify BEFORE proceeding to the next:**
   - the node stays `Ready` (`kubectl get nodes` / `talosctl health`);
   - the VIP / API endpoint (`https://10.0.0.10:6443`) is still reachable;
   - `talosctl get addresses -n <node>` shows `10.0.0.x` still on the
     selected interface (the one matched by `hardwareAddr: '86:00:00:*'`).
4. Only when a node is confirmed healthy, move to the next (hb1 → hb2 → hb3).
5. **Rollback:** if a node loses connectivity, revert the config change and
   re-apply the previous machine config to that node before touching the others.

Document this as the final, user-executed stage. Agents render/lint/review only.

---

## Step-sequence summary

0. **USER probe gate (read-only):** `talosctl get links/addresses` — confirm
   private NIC = `86:00:00:*` and public NIC ≠ `86:00:00:*`. PASS → glob;
   FAIL → `driver:`-narrowed selector or per-node `permanentAddr` overlays.
   (Config edits may be authored in parallel; only the live apply waits on this.)
1. **Implementation (single ansible-implementor):** swap kube1 `cluster.yaml` to
   `deviceSelector: { hardwareAddr: '86:00:00:*' }`; remove `talos.install.*` +
   `talos.patches.*` from schema, validate asserts, render (both `90-user-raw`
   steps + 2 templates), defaults, and kube1 config; refresh the overlay comment.
   Verify render emits the deviceSelector block, no `90-user-raw`, check-mode no
   drift, `ansible-lint` clean.
2. **Review (ansible-reviewer):** correctness/idempotency/FQCN; KEEP-set untouched;
   VIP + merge-key asserts preserved; no orphan `talos.install/patches` references.
3. **Docs/ADR (parallel):** docs-writer closes the deferred follow-ups in CLAUDE.md
   + updates `eth1` references; adr-writer recommends a short ADR-016 addendum.
4. **Report (report-writer):** closeout report with probe outcome + selector used.
5. **USER live apply (staged):** render+commit+push, then `--mode=no-reboot`
   one node at a time, verifying Ready + VIP + interface address after each before
   continuing; revert config to roll back. Highest-risk step.
```
```
