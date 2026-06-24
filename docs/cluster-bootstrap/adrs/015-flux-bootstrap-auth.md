# ADR-015: Flux bootstrap authentication modes and secret persistence

**Status:** Accepted

## Context

`flux bootstrap` must work across the Git providers we actually use or anticipate: GitHub, Bitbucket Server / Data Center (including non-standard SSH or HTTP ports), and generic Git servers that expose no provider-specific API. The bootstrap phase also establishes how Flux will authenticate for ongoing reconciliation, and where each credential lives. This ADR bounds the supported auth modes, their secret storage, and the provider-specific bootstrap behavior (see ADR-006 for the GitOps choice, ADR-010 for the pre-Flux component install sequence, and ADR-011 for the config compiler that renders these settings from `cluster.yaml`).

## Decision

Flux bootstrap supports **three auth modes** selected in `cluster.yaml`. The renderer emits the matching bootstrap variables and secret handling.

| Mode | Bootstrap credential | Reconciliation credential | Provider APIs used |
|---|---|---|---|
| `provider-deploy-key` | Provider API token (GitHub PAT / Bitbucket app password) | SSH private key in `flux-system/flux-git-deploy` Secret | Once: create/deploy the deploy/access key via provider API |
| `https-token` | Same token passed to `flux bootstrap` | Same token stored in `flux-system/flux-git-deploy` Secret | None during bootstrap (token already has repo access) |
| `ssh-private-key` | User-supplied SSH private key | Same SSH private key in `flux-system/flux-git-deploy` Secret | None (repository must already trust the public key) |

### Config surface

The compiler renders the non-secret fields below from `config/overrides/cluster.yaml` into generated Ansible group vars. Secrets (`flux_auth_token` and optional `flux_auth_private_key_passphrase`) live in SOPS-encrypted inventory group vars and are never rendered from `config/` (ADR-011).

```yaml
flux:
  provider: generic                 # generic | github | bitbucket-server
  repoUrl: ""                       # required for generic git flows; source of truth for all modes
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
    tokenAuthBearer: false          # bearer token instead of HTTPS basic password for generic git
    sshHostname: ""                 # e.g. bitbucket.example.com:7999 when SSH host/port differs
    privateKeyFile: ""              # local control-machine path; do not commit the key
    caFile: ""
    allowInsecureHttp: false
    silent: false                   # assume deploy/access key already set up
```

SOPS-backed variables (e.g. `inventories/common/group_vars/all/flux.sops.yaml`):

```yaml
flux_auth_token: ""                 # GITHUB_TOKEN, BITBUCKET_TOKEN, or HTTPS token/password
flux_auth_private_key_passphrase: "" # optional, only for encrypted SSH keys
```

### Provider/mode behavior matrix

| Provider | Mode | Bootstrap command | Credential source | Notes |
|---|---|---|---|---|
| `github` | `provider-deploy-key` | `flux bootstrap github --token-auth=false` | `flux_auth_token` as `GITHUB_TOKEN` env | PAT creates deploy key; PAT is not stored in cluster. |
| `github` | `https-token` | `flux bootstrap github --token-auth=true` if provider fields are present; otherwise generic `flux bootstrap git` with HTTPS URL | `flux_auth_token` | Stores token-based auth in Flux secret. |
| `github` | `ssh-private-key` | `flux bootstrap git` | local `privateKeyFile` + optional passphrase | Repo must already accept the public key. |
| `bitbucket-server` | `provider-deploy-key` | `flux bootstrap bitbucket-server --token-auth=false` | `flux_auth_token` as `BITBUCKET_TOKEN` env | Set `--ssh-hostname=host:7999` for non-standard SSH port. |
| `bitbucket-server` | `https-token` | `flux bootstrap bitbucket-server --token-auth=true` if provider fields are present; otherwise generic `flux bootstrap git` with HTTPS URL | `flux_auth_token` | Server/DC only; Bitbucket Cloud uses generic Git. |
| `bitbucket-server` | `ssh-private-key` | `flux bootstrap git` | local `privateKeyFile` + optional passphrase | Put non-standard port in `repoUrl` and/or `sshHostname`. |
| `generic` | `provider-deploy-key` | unsupported; fail fast | n/a | Generic Git cannot create deploy keys via provider API. |
| `generic` | `https-token` | `flux bootstrap git` | `flux_auth_token` | Requires HTTPS URL; supports basic or bearer token mode. |
| `generic` | `ssh-private-key` | `flux bootstrap git` | local `privateKeyFile` + optional passphrase | Requires SSH URL. |

### Default: `provider-deploy-key` for supported providers

For GitHub and Bitbucket Server / Data Center, default to `provider-deploy-key` when a supported provider is declared. Flux creates a read-only deploy key (GitHub) or access key (Bitbucket) through the provider API, then stores only the SSH private key in the cluster. The provider API token is **not retained** in the cluster and is not used for reconciliation. The template default is `provider: generic` and `auth.mode: ssh-private-key`, which works without provider API access after the repository trusts the supplied public key.

### Bitbucket Server / Data Center specifics

- Support non-standard SSH ports (commonly `7999`) via `flux.auth.sshHostname` (`bitbucket.example.com:7999`). The generated `flux bootstrap` command passes `--ssh-hostname` and constructs the port-qualified SSH URL (`ssh://git@host:port/...`).
- The provider API endpoint is derived from `flux.auth.hostname` and uses HTTP Basic Auth with the API token.

### Generic Git fallback

For provider-less Git servers, only `https-token` and `ssh-private-key` are valid. The bootstrap command uses `flux bootstrap git` and relies solely on the repository URL and supplied credential. The repository must already trust an SSH key or accept the HTTPS token; no keys are created automatically.

### Secret persistence boundaries

| Secret | Storage | Rationale |
|---|---|---|
| Provider API token / HTTPS password (`flux_auth_token`) | SOPS-encrypted Ansible group_vars, passed to `flux bootstrap` at install time | In `provider-deploy-key` mode the token never reaches the cluster; in `https-token` mode it is also stored in the `flux-system` Secret for reconciliation |
| SSH private key passphrase (`flux_auth_private_key_passphrase`) | SOPS-encrypted Ansible group_vars only | Used only to unlock the local key file during bootstrap; the decrypted private key is what Flux stores in-cluster |
| HTTPS token/password (https-token mode) | SOPS-encrypted Ansible group_vars **and** `flux-system` Secret | Required for ongoing reconciliation; rotation must update both or the cluster desyncs |
| SSH private key (all SSH modes) | SOPS-encrypted Ansible group_vars **and** `flux-system` Secret | Same as HTTPS token; the in-cluster copy is the reconciliation credential |

The `flux_bootstrap` role passes secrets to `flux bootstrap` with `no_log: true`, so tokens and keys are never committed to `flux-system` in plain text. SOPS remains the durable source of truth for the Ansible-side copy.

### Why allow `https-token` despite its blast radius

`https-token` is supported because some organizations cannot or will not add deploy keys, and because generic Git servers lack a key-provisioning API. It has a broader blast radius (the same credential can read other repos, and a leak grants repository access), so it is **opt-in** and **not the default**. When used, the token should be scoped to the narrowest repository permission that still allows `git clone` / `git pull` and (for bootstrapping) `git push` to create the `flux-system` branch if it does not exist.

## Consequences

- `cluster.yaml` grows explicit auth keys under `flux.auth`: `mode`, `username`, `owner`, `repository`, `hostname`, `personal`, `private`, `readWriteKey`, `tokenAuthBearer`, `sshHostname`, `privateKeyFile`, `caFile`, `allowInsecureHttp`, and `silent`. `flux.provider` is `generic | github | bitbucket-server`, and `flux.repoUrl` remains the source of truth for generic Git flows.
- The renderer validates that `provider-deploy-key` is only used with `github` or `bitbucket-server`; generic Git rejects it. Generic `https-token` requires an HTTPS URL (or `allowInsecureHttp: true` for HTTP), and `ssh-private-key` requires an SSH URL plus `privateKeyFile`.
- `flux.repoUrl` is no longer the only required field — `flux.auth.mode` must also be set, with a sensible default of `provider-deploy-key` when a supported provider is declared and `ssh-private-key` for the generic default.
- The pre-Flux component install (Cilium, CCM) is split from the Flux-auth preflight so missing or incorrect repository auth does not block CNI/CCM installation. Flux auth errors surface immediately before `flux bootstrap` runs.
- Rotating a deploy key means: delete the old key in the provider UI, re-run bootstrap with a fresh provider API token so Flux creates a new key, then update SOPS-encrypted Ansible vars. Rotating an HTTPS or SSH key means: update the SOPS secret and re-run bootstrap so the `flux-system/flux-git-deploy` Secret is replaced. The existing idempotency gate skips bootstrap when Flux controllers already exist, so auth-mode migrations after initial bootstrap may require documented manual rotation or deleting/recreating Flux bootstrap resources.
- Provider API tokens for deploy-key mode are high-privilege but short-lived in terms of cluster exposure — they are used once and then discarded. This is the primary security advantage over `https-token`.
- Bitbucket non-standard ports add a small maintenance burden: URL construction and API base URL derivation must be tested against both standard and custom ports.

## Rejected alternatives

- **Single universal HTTPS token mode.** Simpler, but violates least privilege: a leaked token often grants broader repository access than a deploy key, and the credential stays in the cluster indefinitely. Kept only as an opt-in fallback.
- **Only `flux bootstrap git`.** Would work everywhere but loses provider-specific conveniences such as automatic deploy-key creation for GitHub and Bitbucket Server/DC, forcing manual key provisioning for those providers.
- **Only GitHub deploy keys.** Would leave Bitbucket Server/DC and generic Git servers unsupported, which contradicts the goal of a provider-agnostic bootstrap.
- **Plaintext repo credentials in `config/overrides`.** Rejected for security: `config/overrides` is committed; all tokens, passwords, and private-key passphrases must live in SOPS-encrypted group vars.
- **Flux-managed external secret (e.g. SOPS / External Secrets) for bootstrap credential.** Overly complex for bootstrap: the secret still needs to exist before Flux can reconcile anything, so the bootstrap phase must place a plaintext Secret first. We retain SOPS for the Ansible-side copy only.
- **Requiring all users to pre-create SSH keys and repository trust.** This is exactly `ssh-private-key` mode, which is supported, but making it the default would force every user through manual key provisioning before bootstrap. `provider-deploy-key` automates this for providers that expose an API.
- **Provider API token stored in-cluster and used for reconciliation.** Rejected outright: a token that can create repository keys is far more powerful than the read-only `git pull` access Flux needs for reconciliation.
- **Forcing repo auth before pre-Flux components.** Would block Cilium and CCM installation while the operator is still configuring Git credentials. Splitting the preflight lets the cluster reach a network-ready state before Flux auth is validated.
