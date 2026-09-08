<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 04 — Security and Governance Mapping

How llm-vampire's governance design — consent, tiers, realms, tokens,
metering, enforcement — is realized inside PAIR's existing trust architecture.
This document governs Phase 6 (and parts of Phases 2, 3, 9) of
[02-implementation-plan.md](02-implementation-plan.md).

## 1. Two trust boundaries, kept distinct

Vampire's thesis insists on two layers that must never be conflated. PAIR
already separates them physically:

| Layer | Vampire | PAIR realization |
| --- | --- | --- |
| **Node layer** — gateway → engine endpoint; expresses the *owner's* consent and permissions | LM Studio API token on the node; `vampire<NN>` tag | Paired PAIR nodes: PIN pairing + pinned mutual TLS (exists). Engine-only endpoints (Phase 2): the endpoint's own auth (e.g. LM Studio API token) held in the token vault and presented upstream. Contribution ceilings advertised by the node, verified against its mTLS identity. |
| **Client layer** — client → gateway; identity, authorization, accounting | Realm bearer tokens on `:7777` | Loopback plaintext surface on the proxies, today unauthenticated by design. Phase 6 adds *optional* client bearer tokens (default off, preserving drop-in behavior), realm-scoped, issued/rotated/revoked by `nvpair-policy-vault`. |

Credential hygiene follows: client credentials terminate at the proxy and are
never forwarded upstream; upstream endpoint tokens are attached by the proxy
from the vault and never revealed to clients; neither appears in logs (the
repository's never-log rule already covers key material).

## 2. Consent and contribution tiers

Vampire's `vampire<NN>` tag (bare = 100%, `vampire50` = 50%, unset = no
consent) is adopted as a **PAIR contribution ceiling** with the same
semantics and the same five invariants:

- **C1 — Affirmative consent.** Reachable ≠ consenting. A PAIR node
  contributes only after pairing *and* a non-`off` owner share mode; an
  engine-only endpoint only after explicit owner registration/approval.
  Discovery (mDNS or probe scan) never creates routing eligibility by itself.
- **C2 — A label is not a credential.** The advertised ceiling/owner mode is
  treated as a claim. For paired nodes the claim binds to the node's pinned
  mTLS identity; for engine-only endpoints, possession of a tag-like token
  proves nothing about identity, so attribution and realm privileges require
  the verified tiers. Anything unverified stays in the `untrusted` /
  `local` classification and gets no accounting-based benefits.
- **C3 — Live, revocable policy.** Owner mode, ceiling, and endpoint tokens
  are re-read on the existing refresh cadences (scanner enrichment, 10 s
  manual-node probes) and never cached across change windows. Persistent
  `401`/`403` from an endpoint = consent withdrawn → the node leaves rotation
  automatically. The owner stop button (Phase 9) is the immediate form:
  drain + revoke guests + stop sharing in one action.
- **C4 — Metering anchored in the client layer.** The ledger derives from the
  proxies' server-side accounting, not node self-reports: each routed request
  records provider (node identity) and payer (authenticated client identity or
  `local-anonymous` when auth is off), engine, model, job ID, node ID, token
  counts, and timing — **metadata only, never content**. Tier changes create
  time-segmented accounting: a new ceiling applies from its effective moment,
  never retroactively. `nvpair-workload-manager` relays ledger events so every
  member sees cluster-wide attribution.
- **C5 — Real enforcement.** `nvpair-job-scheduler` enforces ceilings as
  admission control: per-node duty-cycle, rate, and concurrency caps derived
  from the declared `pair<NN>` value. A downward change binds in-flight work
  (throttle new admissions immediately; drain running legs). "Utilisation" is
  given one precise definition before Phase 6 ships — the enforced quantity is
  **admitted concurrent inference legs and their duty cycle per wall-clock
  window on that node**, with GPU-pressure telemetry as a secondary damper —
  so owners can predict what a ceiling does.

**Idle gating.** Tiers cap *idle* contribution. Owner activity on a node
(interactive session signals surfaced via `nvpair-node-info`) drives eligible
external utilisation to zero regardless of tier; local work is never gated.

## 3. Owner modes and realms

Owner modes (enforced per node, stored in `nvpair-node-settings`, advertised
via scanner enrichment): `off`, `local`, `personal`, `family`, `business`,
`event`, plus Vampire's explicit "free local share" as a deliberate,
never-default setting. Modes bound which realms may route to the node, with
optional duration and model restriction.

Realms are named trust scopes layered on cluster membership: `personal`,
`family`, `business`, `classroom`, `event`, `community`, `lab`. Each realm
defines exactly Vampire's eight rules: who can use it (identities/tokens),
eligible nodes, allowed models, caching allowed, semantic reuse allowed,
logging allowed, data egress allowed, guest expiry. Realm definitions live in
`nvpair-policy-vault`; the proxies (and fusion) evaluate policy before
routing, yielding Vampire's decision set: allow / deny / route-to-node-class /
safe-model-only / require owner approval / require stronger auth / strip logs
/ disable cache / disable semantic reuse.

Privacy boundaries are routing boundaries: a request's realm and privacy
class constrain the candidate set *before* the scheduler orders it (the
`privacy_policy` and `trusted_only` strategies), so data never reaches a node
class the policy excludes.

## 4. Token vault

One vault, two token populations, owned by `nvpair-policy-vault`:

- **Upstream endpoint tokens** (node layer): per-endpoint API tokens (LM
  Studio 0.4.0+ tokens and any OpenAI-compatible bearer), stored
  OS-keychain-backed where the platform provides one, encrypted-at-rest file
  storage otherwise; referenced by ID everywhere else (node records hold a
  reference, never the secret).
- **Client tokens** (client layer): realm-scoped bearer tokens with expiry,
  rotation, revocation, and permission labels; short-lived event/guest tokens
  (Phase 9) auto-expire and are revoked by the stop button.

Vault rules: constant-time comparison; tokens never logged, never surfaced to
guests, never exported by the CLI; `runtime-tools`-style secret scanning
habits apply to fixtures and tests.

## 5. Trust-level ladder and verifiable results

Vampire's ladder maps onto PAIR's classification (Phases 2/3/9):

| Vampire level | PAIR meaning |
| --- | --- |
| `untrusted` | Probe-discovered candidate, unapproved — never routable |
| `local` | Loopback engine on this node |
| `trusted` | Owner-approved engine-only endpoint |
| `verified` | Paired PAIR node (PIN + pinned mTLS identity) |
| `japer-secured` → `realm-secured` | Verified node inside a realm with signed-result support |

Signed results (Phase 9) reuse the node identity keys established at pairing:
on request, a response carries trace ID, selected node/model, model hash,
response hash, and a signature verifiable against the node's pinned
certificate — Vampire's "verifiable inference fabric" without new key
infrastructure. Tamper-evident, metadata-only trace/ledger records complete
the provenance story.

## 6. Discovery and scan safety

Adopted unchanged from Vampire (Phase 3): probe scanning is explicit,
owner-initiated, and bounded — private/loopback subnets only; ≤8 subnets, ≤16
ports, ≤256 hosts per subnet, ≤1024 candidates, concurrency 16; SSRF guard on
every candidate URL (http/https schemes; loopback/private IPs only;
link-local, reserved, multicast rejected); results are candidates requiring
approval, never auto-registered. No public-IP scanning, ever. PAIR's existing
posture is unchanged: telemetry stays the lowest-sensitivity plaintext
surface, and nothing in this plan moves policy or token data onto it —
policy-bearing claims ride mTLS-gated surfaces or the mDNS record only as
unverified hints.

## 7. Client-surface hardening (Phase 6)

- Optional bearer auth on the proxies' loopback personality; default off so
  existing clients stay drop-in; OpenAI-style `401` with `WWW-Authenticate:
  Bearer` when enabled.
- CORS allowlist for browser-based local clients.
- Per-realm rate limits, quotas, and token budgets enforced at the proxies;
  per-node concurrency caps already respected via scheduler admission.
- Logging controls per realm (`store_prompts` remains hard-off; realms may
  additionally suppress metadata retention); audit = the metering ledger.
- The unauthenticated plaintext personality remains loopback-only and
  `403`-guarded for non-loopback callers (existing behavior, retained).

## 8. Non-goals, restated for PAIR

No authentication bypass; no scanning beyond owner-designated private
networks; no use of any endpoint without owner consent (no GPU freeloading);
PAIR does not become an engine replacement, a training platform, a public GPU
marketplace, or an enterprise DLP product. SECURITY.md's existing threat-model
boundaries stay authoritative; this plan extends what PAIR governs, not what
it promises to defend against.
