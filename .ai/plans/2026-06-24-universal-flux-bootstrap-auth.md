# Universal Flux bootstrap auth support

Date: 2026-06-24

## Goal and scope

Add a provider-agnostic Flux bootstrap authentication layer for kube1 while preserving the existing Ansible pre-Flux / Flux handoff.

In scope:

- Auth modes:
  - `provider-deploy-key`: provider-specific bootstrap uses a provider API token to create a deploy/access key, but Flux stores only SSH material in-cluster.
  - `https-token`: Flux authenticates to the repository over HTTPS using a token/password supplied from SOPS-backed Ansible vars.
  - `ssh-private-key`: Flux authenticates to an existing repository over SSH using a user-provided private key file and optional passphrase.
- Git providers:
  - GitHub / GitHub Enterprise via `flux bootstrap github` where provider-specific behavior is needed.
  - Bitbucket Server / Data Center via `flux bootstrap bitbucket-server`, including non-standard SSH ports such as Bitbucket's default `7999`.
  - Generic Git fallback via `flux bootstrap git`.
- Split repo-auth preflight from cluster/pre-Flux preflight so Cilium / CCM install does not fail early just because Flux repository credentials are not configured yet.
- Add/update ADRs and user-facing docs for the new auth contract, secret ownership, supported provider/mode matrix, and validation commands.

Out of scope:

- Running live `bootstrap-flux.yml`, `kubectl`, `flux reconcile`, provider APIs, or mutating any real repository from automation in this planning/implementation flow.
- Managing plaintext tokens or private keys in git.
- Solving Flux credential rotation for already bootstrapped clusters beyond documenting current limitations and safe recovery.
- Adding GitLab/Gitea-specific bootstrap commands unless a later requirement expands the matrix.

## Current-state facts from the repo

Repo context was narrowed with `graphify query` / `graphify explain` before targeted reads.

- `bootstrap-flux.yml` delegates all bootstrap work to the `flux_bootstrap` Ansible role.
- `flux_bootstrap/tasks/main.yml` currently runs a single `preflight.yml` before namespaces, Cilium, hcloud CCM, then `flux.yml`.
- `flux_bootstrap/tasks/preflight.yml` currently checks `kubectl`, `helm`, and `flux`; asserts `flux_repo_url` is non-empty; asserts `hcloud_token` only when `hcloud-ccm` is active; checks kubeconfig and apiserver reachability when pre-Flux components exist.
- Because the repo URL assertion is in the first preflight, Cilium / CCM cannot be installed if `flux.repoUrl` is still empty, even though those components do not depend on repository auth.
- `flux_bootstrap/tasks/flux.yml` currently always runs `flux bootstrap git` with only `--url`, `--branch`, `--path`, `--kubeconfig`, and `--timeout`.
- The Flux bootstrap idempotency gate skips the bootstrap command whenever `deployment/helm-controller` exists in `flux-system`. That should remain a known limitation for auth migration/rotation unless implementation explicitly changes it.
- `config/defaults/cluster.yaml` currently exposes only `flux.repoUrl`, `flux.repoBranch`, and `flux.path`.
- `config/overrides/cluster.yaml` has `flux.repoUrl: ""`, `flux.repoBranch: master`, and `flux.path: ./flux/clusters/kube1`.
- `config_render/templates/group_vars/common/flux.yml.j2` renders only `flux_repo_url`, `flux_repo_branch`, and `flux_path` into generated `inventories/common/group_vars/all/flux.yml`.
- `config_render/tasks/validate.yml` validates provider/features and rejects computed keys, but has no Flux auth schema validation yet.
- SOPS group vars are already wired via `community.sops.sops` and `.sops.yaml` matches `infra/ansible/inventories/.*/group_vars/.*\.sops\.ya?ml$`, so a future encrypted `inventories/common/group_vars/all/flux.sops.yaml` or provider-specific auth secret file fits the existing secrets boundary.
- ADRs likely affected:
  - `docs/cluster-bootstrap/adrs/006-fluxcd-gitops.md` currently says FluxCD is selected and Cilium/CCM are pre-Flux.
  - `docs/cluster-bootstrap/adrs/010-ansible-flux-provider-handoff.md` documents the pre-Flux/Flux boundary and current sequence.
  - `docs/cluster-bootstrap/adrs/011-config-compiler.md` documents the config compiler and one-writer-per-file invariant.

## Official docs to consult before implementation

Implementors must consult these official references before editing code and cite any non-obvious flag behavior in comments/docs:

- Flux CLI `flux bootstrap git`: https://fluxcd.io/flux/cmd/flux_bootstrap_git/
  - SSH URL format: `ssh://git@<host>[:<port>]/<org>/<repo>`.
  - SSH private key flags: `--private-key-file`, optional `--password`, optional `--ssh-hostname`, `--ssh-hostkey-algos`.
  - HTTPS flags: `--username`, `--password`, optional `--with-bearer-token`, optional `--ca-file`, and `--allow-insecure-http` for HTTP only.
  - `--silent` assumes deploy key setup is already complete.
- Flux CLI `flux bootstrap github`: https://fluxcd.io/flux/cmd/flux_bootstrap_github/ and guide https://fluxcd.io/flux/installation/bootstrap/github/
  - Requires `GITHUB_TOKEN` for provider API operations.
  - `--token-auth=false` uses the token to create a deploy key and does not store the PAT in cluster; `--token-auth=true` stores HTTPS token auth in cluster.
  - Provider flags include `--owner`, `--repository`, `--hostname`, `--personal`, `--private`, and `--read-write-key`.
- Flux CLI `flux bootstrap bitbucket-server`: https://fluxcd.io/flux/cmd/flux_bootstrap_bitbucket-server/ and guide https://fluxcd.io/flux/installation/bootstrap/bitbucket/
  - Server/Data Center only; Bitbucket Cloud uses generic Git.
  - Requires `BITBUCKET_TOKEN` for provider API operations.
  - Provider flags include `--owner`, `--username`, `--repository`, `--hostname`, `--personal`, `--private`, `--read-write-key`, and `--ssh-hostname`.
- Atlassian Bitbucket Server/Data Center SSH docs:
  - https://confluence.atlassian.com/bitbucketserver/enable-ssh-access-to-git-repositories-776640358.html
  - https://support.atlassian.com/bitbucket-data-center/kb/which-ports-does-bitbucket-server-listen-on-and-what-are-they-used-for/
  - Default SSH Git port is commonly `7999`; `ssh://git@host:7999/PROJECT/repo.git` is the expected pattern unless admins forward port 22.
- Flux GitRepository/source secret docs: https://fluxcd.io/flux/components/source/gitrepositories/#ssh-authentication
  - Runtime secret keys: SSH uses `identity`, `identity.pub`, `known_hosts`; HTTPS basic uses `username`, `password`; bearer token uses `bearerToken`.

## Proposed design contract to agree before code

This is an architectural/security boundary, so finalize it in an ADR before implementation proceeds beyond schema work.

Recommended config surface:

```yaml
flux:
  provider: generic                 # generic | github | bitbucket-server
  repoUrl: ""                       # required for generic git flows; useful as display/source of truth for all modes
  repoBranch: main
  path: ./flux/clusters/kube1
  auth:
    mode: ssh-private-key           # provider-deploy-key | https-token | ssh-private-key
    username: git                   # HTTPS basic username, or Bitbucket API username when needed
    owner: ""                       # GitHub owner or Bitbucket project/user
    repository: ""                  # Provider-specific repository slug/name
    hostname: ""                    # GitHub Enterprise / Bitbucket Server hostname
    personal: false
    private: true
    readWriteKey: false
    tokenAuthBearer: false          # use bearer token instead of HTTPS basic password when supported by generic git
    sshHostname: ""                 # e.g. bitbucket.example.com:7999 when SSH host/port differs
    privateKeyFile: ""              # local control-machine path; do not commit the key
    caFile: ""
    allowInsecureHttp: false
    silent: false
```

Recommended SOPS-backed variables, loaded from encrypted inventory group vars and never rendered from `config/`:

```yaml
flux_auth_token: ""                 # GITHUB_TOKEN, BITBUCKET_TOKEN, HTTPS token/password depending on mode
flux_auth_private_key_passphrase: "" # optional, only for encrypted SSH keys
```

Provider/mode behavior matrix:

| Provider | Mode | Bootstrap command | Credential source | Notes |
|---|---|---|---|---|
| `github` | `provider-deploy-key` | `flux bootstrap github --token-auth=false` | `flux_auth_token` as `GITHUB_TOKEN` env | PAT creates deploy key; PAT should not be stored in cluster. |
| `github` | `https-token` | Prefer `flux bootstrap github --token-auth=true` if provider fields are present; otherwise generic `flux bootstrap git` with HTTPS URL | `flux_auth_token` | Stores token-based auth in Flux secret. |
| `github` | `ssh-private-key` | `flux bootstrap git` | local `privateKeyFile` + optional passphrase | Repo must already accept the public key. |
| `bitbucket-server` | `provider-deploy-key` | `flux bootstrap bitbucket-server --token-auth=false` | `flux_auth_token` as `BITBUCKET_TOKEN` env | Set `--ssh-hostname=host:7999` for non-standard SSH port. |
| `bitbucket-server` | `https-token` | Prefer `flux bootstrap bitbucket-server --token-auth=true` if provider fields are present; otherwise generic `flux bootstrap git` with HTTPS URL | `flux_auth_token` | Server/DC only; Bitbucket Cloud must use generic Git. |
| `bitbucket-server` | `ssh-private-key` | `flux bootstrap git` | local `privateKeyFile` + optional passphrase | Put non-standard port in the SSH URL and/or `sshHostname`. |
| `generic` | `provider-deploy-key` | unsupported; fail fast | n/a | Generic Git cannot create deploy keys via provider API. Use `ssh-private-key` or manually provision deploy key. |
| `generic` | `https-token` | `flux bootstrap git` | `flux_auth_token` | Requires HTTPS URL; supports basic or bearer token mode. |
| `generic` | `ssh-private-key` | `flux bootstrap git` | local `privateKeyFile` + optional passphrase | Requires SSH URL. |

## Ordered implementation steps

### 0. ADR decision checkpoint

- Owner: `adr-writer` after the user agrees the config surface, secret ownership, and provider/mode matrix.
- Dependencies: none; blocks durable implementation if the schema/security contract changes.
- Task boundary:
  - Draft a new provider-agnostic ADR, recommended path `docs/cluster-bootstrap/adrs/015-flux-bootstrap-auth.md`, or update ADR-006/010/011 if the owner prefers no new ADR.
  - Cross-reference ADR-006 for FluxCD, ADR-010 for pre-Flux/Flux handoff, and ADR-011 for config compiler ownership.
  - Record rejected alternatives: only `flux bootstrap git`; only GitHub deploy keys; plaintext repo credentials in `config/overrides`; forcing repo auth before pre-Flux components.
- Validation before next step: user accepts the ADR direction or explicitly chooses a modified schema/matrix.

### 1. Config schema defaults and validation

- Owner: `ansible-implementor`.
- Dependencies: Step 0 decision should be accepted, or implementor must stop and ask.
- File set:
  - `config/defaults/cluster.yaml`
  - `config/overrides/cluster.yaml` only for safe illustrative defaults/comments; do not add real secrets.
  - `infra/ansible/roles/config_render/tasks/validate.yml`
- Work:
  - Add the new `flux.provider` and `flux.auth.*` schema keys with documented defaults.
  - Preserve existing `flux.repoUrl`, `flux.repoBranch`, and `flux.path` compatibility.
  - Validate pick-one values for provider and auth mode.
  - Validate mode/provider constraints:
    - `generic + provider-deploy-key` fails with an actionable message.
    - Generic `https-token` requires HTTPS URL unless `allowInsecureHttp` is true for HTTP.
    - `ssh-private-key` requires an SSH URL and `privateKeyFile`.
    - Bitbucket Server provider-deploy-key with non-standard SSH must accept `sshHostname` including port.
  - Validate provider-specific required fields (`owner`, `repository`, `hostname`, `username` where Flux/Bitbucket require them) without requiring secrets at render time.
- Validation:
  - Run render in check mode from `infra/ansible`: `ansible-playbook playbooks/render-config.yml -e config_render_check=true`.
  - Run `ansible-lint` after the full Ansible slice lands.

### 2. Render generated Flux auth vars

- Owner: `ansible-implementor`.
- Dependencies: Step 1.
- File set:
  - `infra/ansible/roles/config_render/templates/group_vars/common/flux.yml.j2`
  - Generated `infra/ansible/inventories/common/group_vars/all/flux.yml`
  - `infra/ansible/roles/config_render/tasks/render_ansible.yml` only if needed.
- Work:
  - Render non-secret Flux auth settings into generated common group vars with `flux_` / `flux_auth_` prefixes.
  - Do not render `flux_auth_token` or `flux_auth_private_key_passphrase`; those come from SOPS-backed group vars.
  - Keep the generated file header and one-writer invariant.
  - Re-render outputs from `config/overrides`, then review the generated diff.
- Validation:
  - `ansible-playbook playbooks/render-config.yml` to update generated files.
  - `ansible-playbook playbooks/render-config.yml -e config_render_check=true` must report no drift after rendering.

### 3. Split preflight into pre-Flux and Flux-auth phases

- Owner: `ansible-implementor`.
- Dependencies: Step 2 can proceed in parallel if variable names are agreed; final merge depends on Step 2 names.
- File set:
  - `infra/ansible/roles/flux_bootstrap/tasks/main.yml`
  - Existing `preflight.yml`, or new focused files such as `preflight_pre_flux.yml` and `preflight_flux_auth.yml`
  - `infra/ansible/playbooks/bootstrap-flux.yml` comments only if needed.
- Work:
  - Move cluster/pre-Flux checks into an early preflight:
    - `kubectl` and `helm` availability when pre-Flux components are active.
    - kubeconfig existence and apiserver reachability when pre-Flux components are active.
    - `hcloud_token` assertion only when `hcloud-ccm` is active.
  - Move `flux` CLI availability, `flux_repo_url` / provider fields, auth mode validation, and secret-material checks to a late preflight immediately before `flux.yml`.
  - Ensure `flux.repoUrl` or repo auth failures do not block namespaces, Cilium, or hcloud CCM.
  - Preserve fail-fast behavior before actually running `flux bootstrap`; the play should not silently skip Flux unless an explicit skip flag is separately agreed.
- Validation:
  - `ansible-lint`.
  - `ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml --syntax-check`.
  - Do not run live `bootstrap-flux.yml` without explicit user approval.

### 4. Implement Flux bootstrap command builder per provider/mode

- Owner: `ansible-implementor`.
- Dependencies: Steps 1-3.
- File set:
  - `infra/ansible/roles/flux_bootstrap/defaults/main.yml`
  - `infra/ansible/roles/flux_bootstrap/tasks/flux.yml`
  - Optional focused task files, e.g. `flux_github.yml`, `flux_bitbucket_server.yml`, `flux_git.yml`, if that keeps command construction readable.
- Work:
  - Preserve the existing idempotency gate that skips bootstrap when `flux-system/helm-controller` already exists, but document its auth-rotation limitation.
  - Build command argv without shell interpolation.
  - Use `no_log: true` on tasks that pass tokens, passwords, passphrases, or provider token env vars.
  - For `provider-deploy-key`:
    - GitHub: run `flux bootstrap github --token-auth=false` with `GITHUB_TOKEN` from `flux_auth_token`.
    - Bitbucket Server/DC: run `flux bootstrap bitbucket-server --token-auth=false` with `BITBUCKET_TOKEN` from `flux_auth_token`; include `--ssh-hostname={{ flux_auth_ssh_hostname }}` when set.
    - Generic: fail in late preflight with an actionable message.
  - For `https-token`:
    - Use provider-specific bootstrap with `--token-auth=true` when provider fields are complete, otherwise use `flux bootstrap git` against an HTTPS URL.
    - Pass token via `--password` for generic git; add `--with-bearer-token=true` when configured.
    - Include `--username`, `--ca-file`, and `--allow-insecure-http` only when configured/applicable.
  - For `ssh-private-key`:
    - Use `flux bootstrap git` with `--url`, `--private-key-file`, optional `--password` passphrase, optional `--ssh-hostname`, optional `--ssh-hostkey-algos`, and optional `--ca-file`.
    - Support SSH URLs with embedded non-standard ports such as `ssh://git@bitbucket.example.com:7999/PROJECT/repo.git`.
  - Keep `--branch`, `--path`, `--kubeconfig`, and `--timeout` on every bootstrap command.
- Validation:
  - `ansible-lint`.
  - Syntax check both hcloud and manual inventory paths.
  - Reviewer may request a dry command-construction matrix, but must not run commands that mutate a real repo or cluster.

### 5. Secret handling and examples

- Owner: `ansible-implementor` for variable wiring; `docs-writer` for examples.
- Dependencies: Step 4 names.
- File set:
  - `infra/ansible/roles/flux_bootstrap/defaults/main.yml`
  - Optional docs-only sample under `docs/` if needed; do not create plaintext auto-loaded secret files.
  - `.sops.yaml` only if the existing regex does not cover the chosen encrypted secret path; current regex should already cover `inventories/common/group_vars/all/*.sops.yaml`.
- Work:
  - Document that real `flux_auth_token` and optional `flux_auth_private_key_passphrase` live in encrypted inventory group vars, e.g. `infra/ansible/inventories/common/group_vars/all/flux.sops.yaml`.
  - Do not commit real tokens, private keys, decrypted SOPS files, or generated Talos secrets.
  - If an example is added, keep it outside Ansible's auto-loaded group vars or make it obviously placeholder-only and non-secret.
- Validation:
  - Confirm no plaintext secret-looking values appear in `git diff`.
  - `ansible-lint` must not expose secrets in failure output.

### 6. User-facing docs and CLAUDE updates

- Owner: `docs-writer`.
- Dependencies: Steps 1-5 names and final behavior; ADR step accepted.
- File set:
  - `CLAUDE.md` for workflow/status updates.
  - A focused runbook such as `docs/flux-bootstrap-auth.md`, or an appropriate existing docs page if preferred.
  - `docs/adding-a-provider.md` if provider onboarding needs the Flux auth fields explained.
- Work:
  - Explain the three auth modes, when to use each, and the security trade-off:
    - provider deploy key: least token persistence in cluster, requires provider API token at bootstrap time.
    - HTTPS token: easiest generic path, token is stored in cluster as Flux source secret.
    - SSH private key: works everywhere after manual deploy-key setup, private key is stored in cluster secret.
  - Document GitHub, Bitbucket Server/DC including `:7999`, and generic Git examples.
  - Explain the split preflight behavior: pre-Flux components can install first; Flux auth errors surface later before `flux bootstrap`.
  - Include exact SOPS guidance and live commands the user runs, without embedding secrets.
  - Keep docs in teaching mode: briefly explain deploy keys, PATs, HTTPS token auth, SSH known_hosts, and why pre-Flux auth is delayed.
- Validation:
  - `docs-writer` self-check for consistency with implementation variable names.
  - No secrets in examples.

### 7. Review and validation pass

- Owner: `ansible-reviewer` for Ansible/config/render changes; `docs-writer` or main orchestrator for docs sanity; `adr-writer` owns ADR wording only.
- Dependencies: Steps 1-6.
- Work:
  - Review idempotency, FQCN usage, `no_log`, variable namespacing, and generated-file ownership.
  - Review provider/mode matrix and failure messages.
  - Confirm pre-Flux components are not blocked by missing repo auth.
  - Confirm no live-cluster/provider/repo mutations were run by agents without approval.
- Required validation before handoff:
  - `ansible-lint` from `infra/ansible`.
  - `ansible-playbook playbooks/render-config.yml -e config_render_check=true`.
  - `ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml --syntax-check`.
  - `ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-flux.yml --syntax-check`.
  - If the user explicitly approves live/check-mode cluster access, run the agreed `--check --diff` command and report results; otherwise record that live validation is deferred for the user.

### 8. Report after implementation

- Owner: `report-writer`.
- Dependencies: reviewer approval.
- Work:
  - Write `.ai/reports/2026-06-24-universal-flux-bootstrap-auth.md` summarizing changes, validation, live steps deferred to the user, and follow-ups.

## Small implementor task boundaries

Use separate implementor tasks instead of one bundled change:

1. `ansible-implementor`: config schema validation only (`config/defaults`, `config/overrides` comments, `config_render/tasks/validate.yml`).
2. `ansible-implementor`: render generated Flux auth vars only (`flux.yml.j2`, generated `group_vars/all/flux.yml`).
3. `ansible-implementor`: split `flux_bootstrap` preflight only; no command auth matrix yet.
4. `ansible-implementor`: implement provider/mode command builder in `flux_bootstrap`.
5. `ansible-implementor`: secret variable/no_log polish and examples wiring after names stabilize.
6. `adr-writer`: ADR draft/update after user agrees decision.
7. `docs-writer`: runbook + `CLAUDE.md` update after code behavior is known.
8. `ansible-reviewer`: read-only review and validation.
9. `report-writer`: closing report.

## Risks, side effects, and live boundaries

- `flux bootstrap github`, `flux bootstrap bitbucket-server`, and `flux bootstrap git` can mutate a real repository, create deploy/access keys, commit manifests, and create/modify cluster secrets. Agents must not run them against live targets without explicit approval.
- Provider-deploy-key mode uses provider API tokens at bootstrap time. Those tokens are sensitive, may grant repo admin rights, and must be SOPS-backed and `no_log` protected.
- HTTPS token mode stores credentials in the Flux runtime secret. This is easier but has higher blast radius if the cluster secret is exposed.
- SSH private key mode stores private key material in the Flux runtime secret. The deploy key should be repository-scoped and preferably read-only unless image automation requires write access.
- Bitbucket Server/DC often uses SSH port `7999`; failure to include the port in `repoUrl` or `sshHostname` will make bootstrap succeed locally or fail later depending on host key scan and controller clone behavior.
- Flux auto-scans SSH host keys for private-key bootstrap; local `ssh_config` can affect hostname/port behavior. Docs must warn about this and recommend explicit URLs/`sshHostname`.
- Existing idempotency gate skips bootstrap when Flux controllers exist. Auth mode changes after an initial bootstrap may require documented manual rotation or deleting/recreating Flux bootstrap resources.
- Splitting preflight intentionally allows Cilium/CCM to be installed before repo auth is valid. This is desired, but failed Flux auth then leaves a partially bootstrapped cluster with pre-Flux components installed and no Flux controllers. Re-running after fixing auth must be documented as the recovery path.
- No Hetzner cost-incurring operations are planned. No provisioning, snapshot builds, or live cluster commands should be run by agents.

## ADR needs

ADR update is required before final implementation because this changes durable GitOps bootstrap authentication, secret ownership, and the Ansible/Flux handoff.

Recommended: create `docs/cluster-bootstrap/adrs/015-flux-bootstrap-auth.md` and cross-reference ADR-006, ADR-010, and ADR-011. If the owner prefers fewer ADRs, update those three ADRs directly, but still record the provider/mode matrix, rejected alternatives, and security consequences.
