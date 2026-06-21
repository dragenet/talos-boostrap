# Plan: ADR-007 reviewer fixes

## Context

The ADR-007 provider restructuring is complete and `ansible-lint` passes. The `ansible-reviewer` found 5 issues — 1 medium, 4 low. All are stale references or minor inconsistencies, not logic changes.

## Findings and fixes

### 1. Medium — `roles_path` is hcloud-specific in global config

**File:** `infra/ansible/ansible.cfg` line 7
**Current:** `roles_path = roles:roles/providers/hcloud`
**Problem:** Hardcodes a provider-specific path in global Ansible config. Weakens ADR-007's "new provider = add `providers/<name>/` only" goal.

**Fix options (pick one):**
- **Option A (simpler):** Keep `roles:roles/providers/hcloud` but add a comment explaining this is the known exception — when a second provider is added, generalize to `roles:roles/providers` and reference roles as `hcloud/hcloud_servers`.
- **Option B (cleaner now):** Change to `roles_path = roles:roles/providers` and update the role reference in `provision-infra.yml` from `role: hcloud_servers` to `role: hcloud/hcloud_servers`. Also update the 4 utility-playbook references (now deleted, so moot) and the `--tags hcloud_servers` example in CLAUDE.md.

**Recommendation:** Option B. It's the correct ADR-007 shape and the role rename is trivial.

### 2. Low — stale `utils/` entries in CLAUDE.md tree

**File:** `CLAUDE.md` lines 98-102 (repository structure tree)
**Problem:** Still lists `utils/` with 4 playbook files that were deleted.
**Fix:** Already fixed in the current session — the tree no longer shows `utils/`. Verify after applying.

### 3. Low — stale inventory path in manual provider comment

**File:** `infra/ansible/inventories/providers/manual/hosts.yml` line 11
**Current:** `#   ansible-playbook -i inventories/common -i inventories/manual \`
**Fix:** Change to `-i inventories/providers/manual`

### 4. Low — stale hcloud inventory comment about hosts.yml merge

**File:** `infra/ansible/inventories/providers/hcloud/hcloud.yml` lines 4-5
**Current:** `# Merges by hostname with the static entries in hosts.yml (kube1-hb1/2/3),`
**Problem:** No hcloud `hosts.yml` exists — the dynamic inventory directly composes `node_public_ip` / `node_private_ip` from Hetzner hostvars.
**Fix:** Reword to: `# Composes node_public_ip and node_private_ip from Hetzner hostvars` (or similar).

### 5. Low — adding-a-provider.md SOPS step is misleading

**File:** `docs/adding-a-provider.md` lines 62-67
**Current:** Step 2 says "Update `.sops.yaml` at the repo root — add the new path pattern"
**Problem:** The `.sops.yaml` regex is already generic (`infra/ansible/inventories/.*/group_vars/.*\.sops\.ya?ml$`) and matches any new provider. No edit needed unless changing recipient policy.
**Fix:** Change step 2 to note that the existing generic regex already covers new providers — no `.sops.yaml` edit needed unless adding a new age recipient.

## Validation after fixes

```bash
cd infra/ansible && ansible-lint
ansible-doc -t role hcloud/hcloud_servers   # if Option B chosen
ansible-inventory -i inventories/common -i inventories/providers/manual --list
```

## Copy-paste prompt for new session

```
Continue the ADR-007 provider restructuring. The directory moves, utils removal, and most reference updates are done. The ansible-reviewer found 5 remaining issues. Fix them all:

1. **ansible.cfg roles_path** — change from `roles:roles/providers/hcloud` to `roles:roles/providers` (Option B). Update the role reference in `playbooks/providers/hcloud/provision-infra.yml` from `role: hcloud_servers` to `role: hcloud/hcloud_servers`. Update the `--tags hcloud_servers` example in CLAUDE.md to `--tags hcloud/hcloud_servers` if needed (check if Ansible tags support namespaced role names — if not, keep the tag as `hcloud_servers` but change the role name).

2. **CLAUDE.md tree** — verify the `utils/` entries are already removed from the repository structure tree (lines ~98-102). They should be gone.

3. **manual/hosts.yml** — line 11: change `-i inventories/manual` to `-i inventories/providers/manual`.

4. **hcloud/hcloud.yml** — lines 4-5: reword the comment about "merges with static entries in hosts.yml" to say the dynamic inventory composes node_public_ip/node_private_ip directly from Hetzner hostvars.

5. **adding-a-provider.md** — step 2 (lines 62-67): change the instruction to note that the existing generic `.sops.yaml` regex already covers new providers, so no edit is needed unless adding a new age recipient.

After all fixes, run `ansible-lint` from `infra/ansible/` and report the result.
```
