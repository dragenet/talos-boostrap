---
title: "ADR-004: TLS via Let's Encrypt + Cloudflare DNS-01 on dragenet.dev"
description: cert-manager uses Let's Encrypt with Cloudflare DNS-01 challenges for dragenet.dev, enabling wildcard certificates without port 80.
type: adr
audience: [contributor, ai]
tags: [adr, tls, cert-manager, cloudflare]
status: accepted
created: 2026-07-01
updated: 2026-07-01
---

# ADR-004: TLS via Let's Encrypt + Cloudflare DNS-01 on dragenet.dev

**Status:** Accepted

## Context

cert-manager (see cluster-bootstrap ADR-008) needs a `ClusterIssuer` configured with an ACME CA and a challenge solver. Services are exposed under the `dragenet.dev` zone, which is managed on Cloudflare.

## Decision

Use **Let's Encrypt** (production) as the ACME CA with the **DNS-01** challenge solver backed by Cloudflare. cert-manager uses a Cloudflare API token (scoped to `dragenet.dev` zone DNS edit permissions) stored as a Kubernetes Secret to fulfil DNS-01 challenges.

DNS-01 is preferred over HTTP-01 because:
- It works for wildcard certificates (`*.dragenet.dev`).
- It does not require port 80 to be reachable during issuance — compatible with the Cloudflare Tunnel zero-trust plane where no ports are open.

## Consequences

- A Cloudflare API token with `Zone:DNS:Edit` permission scoped to `dragenet.dev` must be created and stored as a SOPS-encrypted secret in the cluster repo.
- Certificates are issued for `*.dragenet.dev` and/or specific subdomains as needed by `Gateway` and `HTTPRoute` resources.
- Let's Encrypt staging issuer should be configured alongside production to avoid rate-limit issues during initial setup and testing.
- DNS propagation adds a few seconds to initial certificate issuance; renewal is automatic well before expiry (cert-manager default: 2/3 of validity period).
