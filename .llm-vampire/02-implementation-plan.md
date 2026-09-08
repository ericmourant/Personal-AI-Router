<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 02 — Phased Implementation Plan

Ten phases, ordered by dependency, each independently shippable and each
covering a slice of the feature inventory
([00-feature-inventory.md](00-feature-inventory.md)). Every phase follows the
same discipline: Go services first; JSON-RPC contract changes ripple through
the broker relay, desktop service bridge, tests, and docs in the same change
(`npm run service-contracts:check`); affected binaries bumped in
`services/versions.json`; per-component `go test ./...` plus `services/tests`
cross-process coverage; desktop work validated with `npm run lint`,
`npm run typecheck`, `npm run test:unit`, `npm run dead-code:check`.

Feature-ID coverage is stated per phase so completeness against the inventory
is checkable at review time.

---

## Phase 1 — Gateway conformance and error envelope

**Covers:** A2–A7, A9, E7 (envelope groundwork) · **Vampire parity:** its Phase 1

Make the OpenAI-compatible surface complete and provably conformant, so every
later feature builds on a tested gateway.

- `lmstudio-proxy`: guarantee routed forwarding for `/v1/models`,
  `/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`, `/v1/responses`,
  and a documented catch-all for other `/v1/*` paths and verbs; preserve query
  strings and end-to-end headers; keep no-read-timeout inference semantics.
- Both proxies: adopt OpenAI-style error envelopes with machine-readable codes
  (`no_suitable_node`, `upstream_unavailable`, `upstream_stream_interrupted`,
  later `no_route_target`, `route_target_removed`, `unsupported_strategy`),
  including per-node rejection reasons in the error detail; emit the mid-SSE
  error frame followed by `data: [DONE]` when an upstream stream dies.
- Add a gateway conformance test suite in `services/tests` driving both proxies
  against mock OpenAI-compatible and Ollama servers (Vampire's "contract tests
  asserting `/v1/*` parity" practice), covering streaming, catch-all, errors,
  and header hygiene.
- Documentation: extend each proxy `README.md` with the guaranteed route table
  and error codes.

**Acceptance:** an unmodified OpenAI client exercises chat, completions,
embeddings, responses, and streaming through `:1234` across two mock nodes,
and observes only OpenAI-shaped errors under every induced failure.

---

## Phase 2 — Provider adapters and engine-only endpoints

**Covers:** B1–B6, C1, C9, D1–D4, D6 · **Vampire parity:** its Phase 2 (registry)

Teach PAIR to route to LLM services that do not run PAIR — Vampire's core
premise — while keeping paired PAIR nodes the preferred, verified tier.

- New `shared/` provider package: adapters for OpenAI-compatible (`/v1/models`)
  and native Ollama (`/api/tags`) interrogation; provider fingerprinting
  (LM Studio, llama.cpp, Ollama, LocalAI, vLLM, text-generation-webui,
  KoboldCpp, GPT4All, Jan, generic); normalized model records carrying
  provider metadata (format, quantization, context length, capability flags);
  LM Studio `/api/v0`–`/api/v1` enrichment (B5) with graceful degradation by
  version.
- `nvpair-manual-nodes`: accept **engine-only endpoints** — any HTTP(S) base
  URL plus optional provider hint and upstream token reference — alongside
  manual PAIR nodes; interrogate on add, re-probe on the existing 10 s cadence;
  record capability-verifier results (B6: reachability, auth required, token
  validity, per-surface support, context window, latency, throughput).
- Node lifecycle: add routing-visible statuses `draining`, `disabled`,
  `maintenance` (D4) honored by both proxies and excluded from scheduler
  rankings; add tags and PATCH-style editing (D1–D3).
- Proxies: route to engine-only endpoints over plain HTTP(S) with the
  endpoint's own upstream token attached (vault integration lands in Phase 6;
  until then tokens are stored by manual-nodes with the same never-log rules);
  eligibility gate and 404-failover apply unchanged; classification (C5
  groundwork) recorded per endpoint: paired PAIR node → verified; engine-only →
  OpenAI-compatible/owner-labelled.
- Broker + bridge + TUI + desktop: extend node objects and `node/*` methods
  with provider, classification, status, tags; Add Node flow gains
  "engine endpoint" entry; nodes list renders provider and drain state;
  `node/drain` method added.

**Acceptance:** a bare LM Studio and a bare Ollama on the LAN are added by URL,
appear in the catalog with provider metadata, serve routed inference through
both fitting proxies, drain cleanly, and never receive PAIR peer traffic
(mTLS surfaces remain PAIR-node-only).

---

## Phase 3 — Opt-in discovery scan and classification

**Covers:** C2–C5, B3 · **Vampire parity:** its Phase 2 (discovery)

- `nvpair-node-scanner`: new opt-in probe discovery alongside mDNS —
  localhost detection and bounded LAN scan over the default provider ports
  `(1234, 11434, 8080, 8000, 5000, 5001, 4891, 1337)` with overridable
  subnets/ports/timeout; enforce Vampire's exact bounds (private/loopback
  subnets only, ≤8 subnets, ≤16 ports, ≤256 hosts per subnet, ≤1024
  candidates, probe concurrency 16) and an SSRF guard on every target
  (http/https only; loopback/private IPs only; link-local, reserved, multicast
  rejected).
- Results are **candidates, never registrations**: each carries provider
  fingerprint, classification (C5), and interrogated inventory; an owner
  approves a candidate into `nvpair-manual-nodes` (permission before routing).
- Broker relay + notifications for scan progress/results; desktop "Discover
  services" review panel and TUI equivalent; scan requires an explicit user
  action every time — no background probing, no persistence of refused
  candidates.
- Advertise nothing new: probing is a client-side act; PAIR nodes keep mDNS.

**Acceptance:** a subnet scan finds mock providers on non-default ports when
asked, respects every bound, refuses public subnets, and nothing is routed to
until explicitly approved.

---

## Phase 4 — Virtual models, route policies, named strategies

**Covers:** E1–E8, A8 (virtual injection), I5 (groundwork) · **Vampire parity:** its Phase 3

- New worker **`nvpair-route-manager`**: persistent route-policy store
  (virtual model → ordered `node:model` targets, strategy, fallback chain),
  CRUD with strategy validation and collision checks against physical model
  IDs; synthesizes the default `pair:auto` policy spanning all online
  eligible targets; owns endpoint classification records from Phase 2/3.
- Strategy framework in `nvpair-job-scheduler` + proxies: named strategies
  selectable per route — `round_robin`, `least_busy` (today's pending+pressure
  ranking, now named), `least_latency`, `model_affinity`, `trusted_only`,
  `weighted_round_robin`, `highest_tokens_per_second`, `best_available`
  (weighted scoring), `context_window`, `power_saver`, `fallback_chain`,
  `quality_score`, `cost_score`, `privacy_policy` (the last three activate
  fully with Phases 6/8 inputs); per-proxy reservations and the eligibility
  gate stay authoritative; retry/circuit-breaker policy per route (E4).
- Proxies: parse the opt-in `pair` body object and `X-PAIR-Mode` /
  `X-PAIR-Strategy` / `X-PAIR-Route` headers; treat `pair:*` model IDs as
  routed virtual models; strip the extension and rewrite `model` before
  forwarding; add `X-PAIR-Route` / `-Strategy` / `-Node` / `-Model` response
  headers; inject virtual models into merged `/v1/models`; per-request routing
  trace records (decision, candidates, scores) retained as metadata only.
- Broker `routes:*` relay; desktop routes editor + TUI routes tab; docs for
  the extension surface (a new `docs/` page mirroring DESIGN-API's opt-in
  contract, PAIR-named).

**Acceptance:** `pair:auto` and a user-defined alias route correctly under
each MVP strategy with failover; plain requests behave byte-identically to
Phase 1; routing decisions are explainable from the trace record.

---

## Phase 5 — Request coalescing and exact cache

**Covers:** F1–F4, F6 (hooks) · **Vampire parity:** its Phase 5

- New `shared/` package: canonical request fingerprinting (model, messages,
  system prompts, tools, sampling params, seed, response format, attachment
  hash, plus realm/client identity once Phase 6 lands) as salted hashes —
  prompt content never stored or logged.
- Proxies: in-flight deduplication keyed by fingerprint; streaming multiplex
  (one upstream stream feeding all coalesced waiters, including late joiners
  within the window); TTL exact-result cache with size bounds; per-request
  disable (`pair.cache: off` / `X-PAIR-Cache: off`), per-node and later
  per-realm disable; deterministic-only caching rules (temperature/seed
  gating) for the F6 "deterministic cache" variation; cache/coalesce counters
  in metrics.
- Broker `cache:*` methods (status, clear); desktop/TUI cache status; the
  remaining F6 variations (best-of-N, consensus, fast-then-good, creative
  divergence) are wired when their modes arrive in Phase 7, with
  policy-sensitive non-reuse gated by Phase 6.

**Acceptance:** N identical concurrent streaming prompts produce one upstream
inference and N complete client streams; repeated identical deterministic
requests hit the cache within TTL; disable flags bypass both; no prompt text
appears in any store or log.

---

## Phase 6 — Policy, realms, owner modes, tokens, tiers, metering

**Covers:** H1–H12, I4, C8 (owner-mode publishing) · **Vampire parity:** its Phase 6 + security thesis

The governance phase; detailed design in
[04-security-and-governance.md](04-security-and-governance.md).

- New worker **`nvpair-policy-vault`**: realm definitions (`personal`,
  `family`, `business`, `classroom`, `event`, `community`, `lab`) with the
  eight per-realm rules (users, nodes, models, caching, semantic reuse,
  logging, egress, guest expiry); owner share modes with duration/model
  restriction (enforced, not just stored); policy evaluation (H3 inputs →
  outputs) consulted by proxies and fusion before routing; client token
  issuance/rotation/revocation and the upstream **token vault**
  (OS-keychain-backed where available); contribution tiers (`pair<NN>`
  semantics, H6) with the five consent invariants C1–C5; metering ledger
  write path (provider/payer attribution, time-segmented on tier changes,
  metadata only).
- Enforcement points: proxies gain optional client bearer auth (default off —
  drop-in preserved), constant-time compare, OpenAI-style 401, CORS allowlist
  for browser clients, per-realm rate limits/quotas, and `Authorization`/
  `Cookie` stripping toward upstreams (A6 completed); `nvpair-job-scheduler`
  enforces contribution ceilings and duty cycles, binding downward tier
  changes on in-flight work; idle gating from owner-activity detection via
  `nvpair-node-info`; scanner advertises owner mode/tier/policy labels
  (consumed as claims, verified via mTLS identity per H7-C2).
- `nvpair-workload-manager` relays ledger events cluster-wide;
  `nvpair-node-settings` stores per-node owner preferences; metrics become
  policy-aware (per-realm counters); audit log = the metadata ledger.
- Desktop + TUI: share-mode control, realm management, token management,
  quota/limit views, audit view. Broker `policy:*`, `tokens:*`, `share:*`,
  `ledger:*` relays.

**Acceptance:** a family realm with a model allowlist and guest expiry admits
an approved client, refuses others with 401/403 envelopes, caps a `pair50`
node at its idle ceiling, zeroes contribution while its owner is active, and
produces a provider/payer-attributed ledger — all without storing prompt
content anywhere.

---

## Phase 7 — Fusion, pipelines, jobs, traces

**Covers:** G1–G6, F6 (remaining variations), I5 · **Vampire parity:** its Phase 7

- New worker **`nvpair-fusion`**: executes fan-out modes `race`, `parallel`,
  `fusion`, `debate`, `pipeline` by issuing ordinary requests through the
  local proxies (each leg individually routed, policy-checked, coalesced, and
  metered — the one-request-one-node proxy invariant is preserved because fan
  out happens above the proxy); fusion strategies `best_of_n`,
  `majority_vote`, `ranked_vote`, `judge_synthesis`, `claim_merge`,
  `contradiction_check`, `critic_refine`, `consensus_only`, `return_all` with
  judge model, `min_agreement`, `include_dissent`; multi-stage pipelines
  (planner → executor → critic → refiner, map-reduce); async job store with
  cancellation; execution traces (routing decisions, contributors, timings)
  joined with Phase 4 trace records.
- Entry points: the `pair` extension (`mode: race|parallel|fusion|debate|
  pipeline`) on the OpenAI surface for synchronous cases, and broker
  `fusion:*` / `jobs:*` / `traces:*` methods for explicit and async use;
  results carry contributor attribution and metrics (G3).
- Desktop + TUI: fusion runner, jobs list (extending the existing Jobs view),
  trace inspector.

**Acceptance:** a race returns the first sufficient answer and cancels the
rest; judge_synthesis fuses three candidate models with dissent included; a
pipeline runs async, is cancellable, and its trace names every contributor,
node, and timing.

---

## Phase 8 — Optimizer

**Covers:** L1–L6, E4 completion (`quality_score`, `cost_score`, warm preference)

- `nvpair-engine-manager`: model catalogue (L1) built from inventory +
  interrogation metadata + measured latency/tokens-per-second; benchmark runs
  (L2) as owner-initiated engine actions against local models.
- `nvpair-job-scheduler`: warm-model preference and cold-load penalties —
  removing the documented "model load state is not considered" limitation —
  plus quality/cost scoring inputs.
- `nvpair-route-manager`: optimization profiles (L4) as prebuilt virtual
  models (`pair:fast`, `pair:balanced`, `pair:best`, `pair:code`,
  `pair:embeddings`, `pair:event-safe`, `pair:business-confidential`,
  `pair:family-study`); task classifier (L3) as an optional pre-router for
  `pair:auto`; per-realm approved model lists bind through policy (L6).
- Desktop + TUI: catalogue and benchmark views, leaderboard (I4).

**Acceptance:** with one warm and one cold owner of the same model, `pair:auto`
prefers the warm node; `pair:code` selects per its profile; benchmarks update
the catalogue and rankings observably.

---

## Phase 9 — Event mode and verifiable results

**Covers:** M1–M5, C7, N1–N3, J4

- `nvpair-cluster-manager`: QR onboarding wrapping PIN pairing (M1/C7);
  temporary guest identities with auto-expiry (M2) bound to the `event` or
  `classroom` realm; the signed-result envelope (N1–N2) using existing node
  keys — trace ID, selected node/model, model/response hashes, signature —
  attached on request (`pair.signed: true`) and verifiable offline; trust
  ladder surfaced per node (N3: untrusted / local / trusted-endpoint /
  verified-paired / realm-secured).
- Event realm defaults: safe model profile (`pair:event-safe`), rate limits,
  no persistent history, guest tokens, auto-expiry, owner stop button (M5 —
  one action drains + revokes guests + stops sharing) in desktop and TUI (J4
  realm dashboards).

**Acceptance:** a guest joins by QR, gets only the safe profile within limits,
expires automatically, is severed instantly by the stop button; a signed
response verifies against the node's pinned identity and fails verification
when tampered.

---

## Phase 10 — Scripting CLI and surface completion

**Covers:** K1–K2, J1–J3 remainder, I1–I4 remainder

- New **`nvpair` CLI** binary speaking broker JSON-RPC (spawning a broker or
  attaching via `--ipc`, exactly like `nvpair-tui`): full mapping in
  [03-api-and-cli-mapping.md](03-api-and-cli-mapping.md) — status, discover,
  nodes (list/add/get/update/drain/delete/ping/refresh), models, metrics,
  route (list/add/get/update/delete/test), share, policy, tokens, cache,
  coalesce, fusion/race/vote/judge, pipeline, jobs, trace, bench, doctor,
  `--watch` variants, `--json` output for automation.
- Promote the desktop prompt playground (existing inference demo) to a
  first-class panel targeting virtual models (J2); ensure every feature added
  in Phases 2–9 has desktop and TUI parity; aggregated `metrics:snapshot`
  broker method if not already landed (I2).
- Documentation pass: getting-started for aggregation scenarios (family,
  small business, event — the three Vampire audience walkthroughs), CLI
  reference, updated architecture/security docs.

**Acceptance:** every Vampire CLI verb has an `nvpair` equivalent or a
documented rename; `make check`, desktop checks, and the full test matrix
pass; the feature-inventory traceability table in this folder shows every
A–N item landed.

---

## Phase X — Post-plan extensions (tracked, not scheduled)

The POSSIBILITIES-only explorations (inventory §O): local RAG and knowledge
collections, distributed agent workflows, conversation memory, structured
output validation, code/dev workflows, multimodal routing, tool/MCP
integration, workflow automation, research harness, deployment patterns.
Each becomes its own proposal after Phase 10; none blocks Vampire feature
parity.

## Sequencing rationale and risks

- Phases 1–4 mirror Vampire's own shipped scaffold (proxy → registry/discovery
  → routing) because each is the next one's substrate; coalescing (5) needs
  stable routing; governance (6) needs identities to govern; fusion (7) needs
  routing + policy + coalescing to fan out safely; optimizer (8) needs
  catalogue + strategies; event mode (9) needs realms + pairing; the CLI (10)
  needs the surfaces it scripts.
- **Riskiest items:** fusion's fan-out (kept above the proxies to preserve the
  one-request-one-node invariant); tier enforcement's "utilisation" definition
  (must be precise before Phase 6 ships — see
  [04-security-and-governance.md](04-security-and-governance.md)); client
  auth on a surface that is deliberately unauthenticated today (shipped
  default-off); engine-only endpoints widening the attack surface (bounded by
  SSRF guards, classification, and policy).
- **Explicit non-changes:** no third gateway port (Vampire's `:7777` collapses
  onto PAIR's `1234`/`11434` takeover model); no HTTP clone of the control
  API; no desktop-side routing logic; existing wire contracts unchanged unless
  a phase says otherwise.
