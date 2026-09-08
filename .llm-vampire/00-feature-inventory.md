<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 00 — LLM Vampire Feature Inventory

The complete feature surface of `japer-technology/llm-vampire`, compiled from
its README, DESIGN-API.md, IMPLEMENTATION-PLAN.md, ASPIRATION.md,
POSSIBILITIES.md, POSSIBLE-COMMANDS.md, docs/security-design-thesis.md,
docs/analysis/MVP.md, and the `src/vampire/` source. **This list defines "all of
the features"** — every item is mapped in
[01-architecture-mapping.md](01-architecture-mapping.md) and scheduled in
[02-implementation-plan.md](02-implementation-plan.md).

Status legend — where the feature stands *in llm-vampire today*:
**S** shipped (its Phases 0–4 scaffold) · **D** designed (its Phases 5–7,
ASPIRATION roadmap, security thesis, POSSIBILITIES).

## A. OpenAI-compatible gateway (S)

- A1. One stable OpenAI-compatible base URL; existing clients change only the
  base URL (Vampire: `http://localhost:7777/v1`).
- A2. Routes: `GET /v1/models`, `POST /v1/chat/completions`,
  `POST /v1/completions`, `POST /v1/embeddings`, `POST /v1/responses`.
- A3. Compatibility catch-all: unknown `/v1/{path}` (GET/POST/PUT/PATCH/DELETE)
  transparently forwarded.
- A4. Native streaming: SSE / chunked passthrough preserved end to end;
  mid-stream upstream failure emits an OpenAI-style error frame then
  `data: [DONE]`.
- A5. OpenAI-style error envelopes for all failures, with optional extension
  detail (`vampire_routing_error` / `no_suitable_node`, rejected-node reasons,
  `upstream_unavailable`, `upstream_stream_interrupted`).
- A6. Header hygiene: hop-by-hop, `Host`, `Content-Length`, `Authorization`,
  and `Cookie` stripped before forwarding — client credentials terminate at the
  gateway and are never forwarded to nodes.
- A7. Query strings and end-to-end headers preserved through the proxy.
- A8. `/v1/models` aggregation: de-duplicated union across every registered
  node, plus injected virtual models; single-node passthrough when no registry
  entries exist.
- A9. Long-generation friendliness: no read timeout on inference, bounded
  connect timeout, pooled upstream connections.

## B. Providers and normalization (S)

- B1. Provider adapters that probe and normalize inventories:
  OpenAI-compatible (`/v1/models`) and native Ollama (`/api/tags`).
- B2. Provider fingerprinting from response headers/`owned_by`: LM Studio,
  llama.cpp, Ollama, LocalAI, vLLM, text-generation-webui, KoboldCpp, GPT4All,
  Jan, generic OpenAI-compatible.
- B3. Known-port hints per provider family: LM Studio `1234`, Ollama `11434`,
  llama.cpp/LocalAI `8080`, vLLM `8000`, text-generation-webui `5000`,
  KoboldCpp `5001`, GPT4All `4891`, Jan `1337`; ports are hints, not limits.
- B4. One normalized model catalog retaining provider-specific metadata
  (format, quantization, params string, size, architecture, context length,
  capability flags: vision, tools, reasoning).
- B5. LM Studio depth: aware of the desktop app, headless `llmster`, `lms` CLI,
  API families (`/v1`, `/api/v0`, `/api/v1`), API-token auth, JIT
  loading/TTL/auto-evict, version landmarks (0.3.6 / 0.4.0+); a documented
  owner setup checklist (LMSTUDIO-SETUP.md).
- B6. Capability verification per endpoint: reachability, auth requirement,
  token validity, available/loaded models,
  chat/responses/completions/embeddings/streaming/tool support, context
  window, latency, throughput, health, owner sharing mode, policy labels.

## C. Discovery (S: local/static scan · D: mDNS, QR, agent)

- C1. Manual endpoint registration with any HTTP(S) base URL (S).
- C2. Localhost detection across default provider ports (S).
- C3. Bounded LAN subnet scan, explicit opt-in developer mode, with
  overridable ports/subnets/timeouts and a `trusted_only` filter (S).
- C4. Scan safety bounds: private/loopback subnets only, caps on subnets,
  ports, hosts, candidates, and probe concurrency; SSRF guard on every target
  URL (scheme http/https; IPs must be loopback/private; link-local, reserved,
  multicast rejected) (S).
- C5. Endpoint classification ladder: unknown OpenAI-compatible →
  OpenAI-compatible → owner-labelled → agent-verified → business/event
  approved (D).
- C6. mDNS/Bonjour opt-in advertisement and browsing (D).
- C7. QR-code event onboarding (D).
- C8. Optional node agent (`/agent/v1/*`): capability manifest, owner-mode
  publishing, health/load publishing, one-click contribution control (D).
- C9. LM Link-aware detection (D).

## D. Node registry and lifecycle (S)

- D1. Registry CRUD: list, register, get, patch, delete; stable node IDs.
- D2. Immediate provider interrogation on registration; periodic refresh with
  health/latency/error bookkeeping and TTL-cached inventory.
- D3. Node fields: id, name, host, base URL, provider, agent base URL, status,
  trusted flag, tags, active requests, queue depth, tokens/sec.
- D4. Manual unavailability states honored by routing: `draining`, `disabled`,
  `maintenance`; drain/restore without unregistering (`nodes drain ID [off]`).
- D5. Trust is owner-granted, never auto-assigned by reachability;
  `trusted=false` default.
- D6. Per-node health checks and status visibility.

## E. Routing (S: MVP strategies · D: extended strategies)

- E1. Virtual models: `vampire:auto` plus configured aliases (designed set:
  `vampire:fast`, `vampire:balanced`, `vampire:best`, `vampire:code`,
  `vampire:embeddings`, `vampire:event-safe`, `vampire:business-confidential`,
  `vampire:family-study`); virtual models appear in `/v1/models`.
- E2. Route policies: virtual model → ordered `node:model` targets + strategy +
  fallback; CRUD with strategy validation and collision checks against
  physical model IDs.
- E3. MVP strategies (S): `round_robin`, `least_busy`, `least_latency`,
  `model_affinity`, `trusted_only`, plus `fallback` failover.
- E4. Extended strategy set (D): `weighted_round_robin`,
  `highest_tokens_per_second`, `best_available` (weighted scoring),
  `context_window`, `power_saver`, `race`, `fallback_chain`, `quality_score`,
  `cost_score`, `privacy_policy`; preferred-node and owner-priority routing;
  hedged requests; quorum; retry/circuit breakers; cold-load penalties;
  warm-model preference.
- E5. Opt-in activation: extension request object in the body, `X-Vampire-*`
  request headers (`X-Vampire-Mode`, `X-Vampire-Strategy`, `X-Vampire-Route`),
  or a `vampire:` model ID — plain requests behave like a transparent proxy.
- E6. Response metadata: `X-Vampire-Route` / `-Strategy` / `-Node` / `-Model`
  headers; extension object stripped and model rewritten to the physical
  target before forwarding.
- E7. Routing errors are actionable: unsupported strategy, no route target,
  route target removed, with per-node rejection reasons.
- E8. Model-eligibility gate: only nodes advertising the requested model are
  candidates; empty inventory excluded until refreshed.

## F. Coalescing and cache (D — Vampire Phase 5)

- F1. Exact request fingerprinting (model, messages, system prompts, tools,
  temperature, seed, top-p, max tokens, response format, attachment hash,
  safety mode, plus realm and user/tenant).
- F2. In-flight deduplication: concurrent identical prompts collapse into one
  upstream inference.
- F3. Streaming multiplex: one upstream stream fans out to all coalesced
  waiters.
- F4. Exact result cache with TTL; realm-scoped keys; per-request and
  per-realm disable flags.
- F5. Opt-in semantic (similarity) cache, realm-scoped (D, aspiration).
- F6. Reuse-policy variations: broadcast answer, stream multiplexing,
  deterministic cache, concurrent best-of-N, consensus answer, fast-then-good,
  creative divergence (dedup disabled), policy-sensitive non-reuse — "reuse is
  a policy decision, not just a cache decision".

## G. Fusion, pipelines, jobs, traces (D — Vampire Phase 7)

- G1. Fan-out modes: `route` (S), `race`, `parallel`, `fusion`, `debate`,
  `pipeline`.
- G2. Fusion strategies: `best_of_n`, `majority_vote`, `ranked_vote`,
  `judge_synthesis`, `claim_merge`, `contradiction_check`, `critic_refine`,
  `consensus_only`, `return_all`; judge model and `min_agreement` /
  `include_dissent` options.
- G3. Dedicated fusion endpoint (`POST /vampire/v1/fusion`) with per-candidate
  roles, contributor attribution, and fused-result metrics.
- G4. Multi-stage pipelines (planner → executor → critic → refiner, map-reduce)
  via `/vampire/v1/pipelines`.
- G5. Async jobs: `GET /vampire/v1/jobs/{id}` status/result; job cancellation.
- G6. Execution traces: `GET /vampire/v1/traces/{id}` — routing decision,
  candidates, contributors, timings.

## H. Policy, governance, and access (D — Vampire Phase 6 + security thesis)

- H1. Owner share modes: Off, Local only, Personal remote, Family share,
  Business contribution, Location/event share, Free local share (never
  default). Shipped today as stored state (`off|local|personal|family|
  business|event` + enabled/duration/model) with get/set surface (S); enforced
  later (D).
- H2. Realms as trust boundaries: `personal`, `family`, `business`,
  `classroom`, `event`, `community`, `lab`; each defines who may use it,
  eligible nodes, allowed models, caching allowed, semantic reuse allowed,
  logging allowed, data egress allowed, guest expiry.
- H3. Policy engine: inputs (identity, realm, prompt sensitivity, requested
  model, endpoint owner mode/classification, time window, token budget,
  retention rules) → outputs (allow, deny, route-to-node-class,
  safe-model-only, require owner approval, require stronger auth, strip logs,
  disable cache, disable semantic reuse).
- H4. Client-facing gateway auth: optional bearer token, constant-time compare,
  OpenAI-style 401, unauthenticated drop-in mode when unset (S, single token;
  D, multi-token/identity).
- H5. Token vault: per-node upstream tokens, per-realm tokens, event tokens,
  short-lived tokens, expiry, rotation, permission labels, secure local
  storage, tokens never exposed to guests; upstream bearer forwarded per node.
- H6. Consent and contribution tiers (`vampire<NN>` token tags): unset = no
  opt-in; bare tag = 100% idle-utilisation ceiling; `vampire50` = 50%, etc.
  Canonical `vampire<NN>`, NN ∈ 0–100. The tag is a label, not a credential.
- H7. Consent invariants: C1 affirmative consent (reachable ≠ consenting);
  C2 label-not-credential (attribution needs real auth); C3 live policy (tag
  re-read every refresh, persistent 401/403 = consent withdrawn); C4 metering
  anchored in the client layer; C5 real enforcement (scheduler caps duty
  cycle/rate/concurrency at the declared ceiling; downward changes bind
  in-flight work).
- H8. Idle gating: owner activity on a node drives eligible utilisation to 0%
  regardless of tier — tiers cap *idle* contribution only.
- H9. Metering ledger: attribute work to node owner (provider) and consumption
  to authenticated client (payer); time-segmented accounting on tier changes,
  never retroactive.
- H10. Per-realm rate limits, quotas, token budgets, audit logs; CORS
  allowlist; node/model allowlists; per-user RBAC (business audience).
- H11. Two distinct trust boundaries, never conflated: node layer (gateway →
  engine endpoint, owner consent) vs client layer (client → gateway, identity
  and accounting).
- H12. No-credential-leakage principle and privacy-boundary routing (privacy
  class constrains eligible nodes).

## I. Observability and metrics (S: basic · D: policy-aware)

- I1. Cluster status summary (version, nodes total/online).
- I2. Metrics snapshot: cluster (nodes online/offline, active requests, queue
  depth, tokens/sec) and per-node (health, requests, errors, average latency).
- I3. Per-request performance capture (latency, tokens/sec where the provider
  reports it).
- I4. Policy-aware, per-realm counters (D); usage history, per-model stats,
  error explorer, latency reports, benchmarking runs, leaderboards, power
  reports (D, command tiers).
- I5. Routing/execution traces (see G6) and explainable optimization —
  "optimization must be explainable" is a design principle.

## J. Dashboard and UX (S)

- J1. Browser dashboard served by the same process: status, nodes, discovery
  (subnet/port/timeout/trusted-only), models, routes, metrics, owner share
  mode read/set.
- J2. Prompt playground that calls the gateway's `/v1/chat/completions`.
- J3. `dashboard` / `ui` command prints or opens the dashboard URL.
- J4. Family/business/event dashboards with usage visibility and one-click
  shutdown (D).

## K. CLI (S: core · D: 100-command roadmap)

- K1. Shipped: `serve`, `status`, `discover` (methods local/static/lan_scan,
  subnet/port/timeout/trusted-only/base-url flags), `nodes list|add|get|update|
  drain|delete`, `models`, `metrics`, `route list|add|get|delete`, `share`,
  `dashboard|ui [--open]`, `--version`, desktop launcher.
- K2. Designed tiers: harden-MVP (`chat`, `route test`, `health`, `nodes
  ping`, `embed`, `complete`, `logs`, `config`, `doctor`, `status --watch`);
  routing/reliability (`route canary/shadow`, `nodes priority|maintenance|
  quarantine`, `queue`, `retry policy`, `route benchmark`, `jobs cancel`);
  discovery/lifecycle (`discover --method mdns|--watch`, `pair [--qr]`, `nodes
  import|export|prune|tag|wake|capabilities|verify`, `topology`); observability
  (`metrics --node`, `history`, `stats`, `errors`, `latency`, `bench`,
  `watch`, `export metrics`, `power report`, `leaderboard`); security/policy
  (`token create|list|revoke`, `auth enable`, `policy`, `allowlist`, `quota`,
  `audit log`, `cors`, `redact rules`, `privacy local-only`, `session
  private`, `tls enable`); cache (`cache status|clear`, `coalesce status`,
  `warmup`, `models evict`, `prefetch`, `schedule set`); fusion (`fusion run`,
  `race`, `vote`, `judge`, `ensemble create`, `best-of`, `debate`, `verify`);
  pipelines/jobs (`pipeline run|list|create`, `jobs list|get`, `trace get`,
  `batch run`); RAG (`rag index|query|collections`); fabric (`sign verify`,
  `trust score`, `daemon start|stop|status`).

## L. Model optimizer (D — ASPIRATION Phase 7)

- L1. Model catalogue: name, alias, host nodes, quantization, context length,
  tool/embedding support, measured latency and tokens/sec, quality scores,
  task suitability, memory footprint, warm/cold state, restrictions.
- L2. Benchmarks feeding the catalogue.
- L3. Task classifier and task-to-model mapping.
- L4. Optimization profiles: fastest-acceptable, highest-quality,
  private-only, local-only, cheapest-energy, low-latency chat, long-context,
  code-focused, embedding-focused, creative, event-safe,
  business-confidential.
- L5. Warm-model preference and cold-load penalties in routing.
- L6. Per-realm approved model lists; prompt compression (exploratory).

## M. Event mode (D — ASPIRATION Phase 8)

- M1. QR onboarding for guests.
- M2. Temporary guest tokens with auto-expiry.
- M3. Local event web UI.
- M4. Safe model profile; per-event rate limits; no persistent history.
- M5. Owner stop button (instant withdrawal).

## N. Verifiable inference fabric (D — JAPER envelope)

- N1. Signed result envelope: trace ID, selected node/model, node trust level,
  model hash, response hash, signature.
- N2. Outcome schema with validation status (`japer.outcome.v1`).
- N3. Trust levels ladder: `untrusted`, `local`, `trusted`, `verified`,
  `japer-secured`; trusted-node registry, provenance, tamper-evident logs.

## O. Explorations (POSSIBILITIES.md, 33 areas)

Beyond the concrete roadmap above, POSSIBILITIES.md enumerates exploration
areas whose *distinct* capabilities are: speculative/draft-refine inference and
request racing (§4), distributed agent workflows (§8), local RAG and knowledge
collections (§9), conversation memory/state (§10), structured output and
validation (§11), code/dev workflows (§12), multimodal routing (§13), tool/MCP
integration (§23), workflow automation (§24), research/experimentation harness
(§27), and deployment patterns (§28). Its own "strongest near-term
architecture" (§33) is the orchestrator + optional node agents + browser
dashboard + routing engine + fusion engine + security layer — the shape the
concrete roadmap already follows. These are tracked as **post-plan
extensions** in [02-implementation-plan.md](02-implementation-plan.md) §Phase
X; everything else in §§1–33 reduces to the A–N items above.

## P. Non-goals (adopted unchanged)

No auth bypass; no public-IP scanning; no GPU freeloading (no use without
consent); not an engine replacement; not a training platform; not a public GPU
marketplace; not enterprise DLP. PAIR's own non-goals (SECURITY.md) remain in
force.
