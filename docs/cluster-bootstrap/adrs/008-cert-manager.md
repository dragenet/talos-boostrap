# ADR-008: cert-manager for TLS certificate lifecycle

**Status:** Accepted — **selectable feature** (`features.cert-manager`, pure-Flux; see ADR-012). This ADR records the tool choice and its wiring when enabled.

## Context

Services exposed via Envoy Gateway need TLS certificates. Certificates must be provisioned, renewed, and stored as Kubernetes Secrets automatically — manual rotation is not acceptable.

## Decision

Deploy **cert-manager** in the bootstrap/infrastructure layer (before apps, after CCM). cert-manager owns all certificate issuance and renewal via `ClusterIssuer` and `Certificate` resources. Envoy Gateway's `HTTPRoute` / `Gateway` TLS configuration references Secrets that cert-manager maintains.

cert-manager is installed as an infrastructure controller (Flux `infrastructure/controllers`) and its `ClusterIssuer` configs land in `infrastructure/configs`, which depends on controllers being ready.

## Consequences

- cert-manager CRDs must be present before any `Certificate` or `ClusterIssuer` resources are applied — the `infrastructure/controllers → infrastructure/configs` Flux dependency ordering enforces this.
- The ACME solver and DNS provider credentials are cluster-specific (see kube1 ADR-004).
- Chose cert-manager over manual certificate management and over Envoy Gateway's built-in cert provisioning (limited, not production-grade).
