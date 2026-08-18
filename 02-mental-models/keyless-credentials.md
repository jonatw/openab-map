# Keyless Credentials

How an OpenAB host can hold **zero standing credentials** — no long-lived AWS key, no standing Tailscale auth-key, no GitHub PAT sitting on disk — and still reach every resource it needs. Instead of storing secrets, the host proves *who it is* and exchanges that identity for a short-lived token at each border it crosses.

This is a deployment pattern, not an OpenAB built-in. OpenAB itself resolves its own bot tokens and API keys through the [secret providers](../04-decision-trees/secrets-strategy.md) — this doc is about the layer *underneath*: how the host or agent obtains those credentials (and AWS / network access) without a permanent key to leak. It is the same trust chain the [Trust Model](../01-core-concepts/trust-model.md) describes *inside* OpenAB, extended outward to the borders the host has to cross.

## The Problem With Stored Keys

Every long-lived credential is a liability with three costs:

- **Blast radius** — a leaked AWS access key or PAT works until someone notices and rotates it.
- **Rotation toil** — someone (or some cron) has to rotate, redistribute, and audit.
- **Storage risk** — the key has to live *somewhere*, and every somewhere is an attack surface.

The keyless model removes the standing secret entirely. The only thing on the host is a short-lived identity document that expires on its own.

## The Pattern: Identity → Federation → Short-Lived Token

Each border (AWS, tailnet, GitHub) accepts a *trusted, time-boxed credential* rather than a stored key. The host presents proof of identity; the border issues a token that self-expires.

```mermaid
flowchart LR
    X509[X.509 cert<br/>host identity] -->|IAM Roles Anywhere| AWS[AWS session<br/>short-lived STS]
    AWS -->|GetWebIdentityToken<br/>signed JWT| TS[Tailscale WIF<br/>ephemeral node auth]
    AWS -->|scoped read| VAULT[SSM SecureString<br/>per-principal KMS]
```

- **Leg A — X.509 → AWS.** [IAM Roles Anywhere](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/introduction.html) trades a certificate (issued by a trusted CA) for a short-lived AWS session. No `aws_access_key_id` on the host.
- **Leg B — AWS → network.** The AWS session calls [`sts:GetWebIdentityToken`](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetWebIdentityToken.html) ([IAM outbound identity federation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_outbound.html)) and receives a **short-lived signed JWT** asserting the caller's AWS identity — verifiable by anyone through the account's OIDC discovery endpoint. This is AWS acting as an OIDC *issuer*, not the familiar inbound `AssumeRoleWithWebIdentity` flow. [Tailscale Workload Identity Federation](https://tailscale.com/docs/features/workload-identity-federation) trusts that issuer and exchanges the JWT for a short-lived Tailscale credential — an ephemeral node registration or a scoped API token. SSH login is then gated by tailnet ACLs — no stored SSH key, no standing auth-key.
- **Leg C — scoped storage.** Anything that genuinely *must* be stored (a bootstrap secret) lives in SSM SecureString under a per-principal KMS grant, so a compromised host reads only its own secrets, and every access is logged in CloudTrail.

## Where Your Host Starts

The entry point depends on what identity the host already has:

| Scenario | Starting point | First step |
|----------|---------------|-----------|
| Off-cloud host | X.509 cert only | IAM Roles Anywhere (Leg A) |
| AWS-native (EC2 / ECS / Lambda) | Instance/task role | Use the role directly — skip Leg A |
| Already has AWS creds | Existing session | Federate outward (Leg B) |
| Non-AWS target | STS-issued JWT | Exchange at the target's federation endpoint |

## Maturity — What's Real Today

Being explicit about what ships versus what's proposed:

| Piece | Status |
|-------|--------|
| IAM Roles Anywhere → AWS session | **Today** — GA AWS feature |
| STS outbound federation → Tailscale WIF | **Today** — AWS GA 2025-11-19, Tailscale WIF GA 2026-02-19 |
| SSM SecureString + per-principal KMS | **Today** |
| Unified credential broker across providers | **Proposed** — GitHub-focused today; pluggable-provider design under discussion |

## Gotchas

*Checked against AWS and Tailscale documentation, August 2026 — verify against the linked docs before you build on them.*

- **Regional endpoint.** `GetWebIdentityToken` [is not available on the STS global endpoint](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetWebIdentityToken.html) — call a regional one (`AWS_STS_REGIONAL_ENDPOINTS=regional`).
- **Enable it on the account first.** Outbound identity federation is off by default; until it's enabled, the call returns `403 OutboundWebIdentityFederationDisabled`.
- **Pin audience and lifetime in the IAM policy, not just in the role.** `sts:GetWebIdentityToken` takes a *required* `Audience` and an *optional* `DurationSeconds` (60–3600s, default 300) — both chosen by the caller. Constrain them with the [`sts:IdentityTokenAudience`, `sts:DurationSeconds` and `sts:SigningAlgorithm` condition keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_outbound_policies.html), which also work in SCPs, RCPs and VPC endpoint policies. Note the absent-key trap: if the caller omits `DurationSeconds`, an `Allow` conditioned on it evaluates false and the call is denied — that's the condition working, not a missing key.
- **A token can't outlive the session that minted it.** Requesting a duration past the calling session's own expiry returns `403 SessionDurationEscalation`. This bites Leg A specifically: a Roles Anywhere session is itself short-lived, so late in that session a 1-hour token request fails.
- **The CA key is the new crown jewel.** You removed the standing access key, but the CA that signs the X.509 certs is now the single high-value secret. Protect it accordingly (HSM / KMS-backed, tight access, audited).

## Further Reading

- Worked reference implementation: [jonatw/flightdeck — keyless-clearance](https://github.com/jonatw/flightdeck/blob/main/docs/foreign-policy/keyless-clearance.md) (full end-to-end setup, sequence diagram, and rationale)
- [Trust Model](../01-core-concepts/trust-model.md) — how OpenAB isolates its *own* secrets from agents
- [Secrets Strategy](../04-decision-trees/secrets-strategy.md) — where to store the secrets OpenAB still needs
- AWS: [IAM Roles Anywhere](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/introduction.html) · [Outbound identity federation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_outbound.html)
- Tailscale: [Workload identity federation](https://tailscale.com/docs/features/workload-identity-federation)
