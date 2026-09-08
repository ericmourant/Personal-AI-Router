<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 01 — Architecture Mapping and Gap Analysis

How each llm-vampire concept lands in PAIR. Feature IDs (A1, E3, H6, …) refer to
[00-feature-inventory.md](00-feature-inventory.md).

## Architectural translation

Vampire is one Python process serving three surfaces (OpenAI gateway, HTTP
control API, browser SPA). PAIR is a supervised family of Go workers behind a
JSON-RPC broker, with Electron and a TUI as clients. The translation rules:

| Vampire surface | PAIR home |
| --- | --- |
| OpenAI-compatible gateway (`:7777/v1`) | The existing compatibility proxies: `lmstudio-proxy` (OpenAI-compatible, port `1234`) and `ollama-proxy` (Ollama-compatible, port `11434`). No third gateway port is added; port takeover already gives unmodified clients the cluster. |
| `/vampire/v1/*` HTTP control API | Broker JSON-RPC methods (stdio / `--ipc`), relayed worker surfaces, and — where peers need it — the fixed `143xx` HTTP peer ports. The control plane stays JSON-RPC; a scripting CLI provides the automation ergonomics the HTTP control API served. |
| Browser dashboard SPA | The Electron desktop app and `nvpair-tui`. |
| `vampire` CLI | A new `nvpair` scripting CLI binary (see [03-api-and-cli-mapping.md](03-api-and-cli-mapping.md)). |
| In-process registries (nodes/routes/share) | Existing stores: `nvpair-manual-nodes`, `nvpair-cluster-manager`, `nvpair-node-settings`; new stores only where none fits (routes, tokens, ledger). |
| `vampire` request object / `X-Vampire-*` headers / `vampire:` models | `pair` request object / `X-PAIR-*` headers / `pair:` virtual model IDs. |

## Concept-by-concept map

Status: ✅ PAIR has it · 🟡 partial · ❌ missing (new work).

### Gateway and proxying

| Feature | Status | PAIR reality and gap |
| --- | --- | --- |
| A1 stable base URL, drop-in clients | ✅ | Port takeover on `1234`/`11434`; engines move behind the proxy; `OLLAMA_HOST` alias honored. |
| A2 full OpenAI route set | 🟡 | `lmstudio-proxy` forwards OpenAI-compatible inference routes and merges `GET /v1/models`. Gap: guarantee and test the full set — `/v1/completions`, `/v1/embeddings`, `/v1/responses` — end to end, including cross-node routing for embeddings. |
| A3 catch-all passthrough | 🟡 | Proxies forward to the selected node; gap: explicit conformance tests for unknown `/v1/*` paths and non-POST verbs. |
| A4 streaming + mid-stream error frame | 🟡 | Streaming passthrough exists ("responses stream back along the same path"). Gap: OpenAI-style mid-SSE error frame + `data: [DONE]` on upstream interruption. |
| A5 OpenAI error envelopes | 🟡 | Proxy returns an actionable local `502` when no owner is routable. Gap: emit the OpenAI error JSON shape everywhere, add machine-readable codes (`no_suitable_node`, `upstream_unavailable`, …) and per-node rejection reasons. |
| A6 credential hygiene | 🟡 | Cluster ingress is mTLS and engines bind loopback. Gap: explicitly strip client `Authorization`/`Cookie` before forwarding once client-facing auth (H4) exists, and forward per-endpoint upstream tokens (H5) instead. |
| A7 header/query preservation | ✅ | Reverse proxy semantics already. Add conformance tests. |
| A8 `/v1/models` aggregation + virtual models | 🟡 | Fan-out merge across candidates exists on both proxies. Gap: inject virtual models (E1) and provider-endpoint inventories (B/C). |
| A9 long-generation timeouts, pooling | ✅ | Existing proxy behavior; verify with conformance tests. |

### Providers, normalization, discovery

| Feature | Status | PAIR reality and gap |
| --- | --- | --- |
| B1–B2 provider adapters + fingerprinting | ❌ | PAIR's inventory comes from peer engine-managers (`em` endpoint, `modelsByEngine`). Nothing probes a bare LM Studio/Ollama/llama.cpp/vLLM/LocalAI/Jan/GPT4All/KoboldCpp/TGW server that has no PAIR installed. New: a provider-adapter package and **engine-only endpoints** as first-class routing targets. |
| B3 known-port hints | ❌ | New scan defaults `(1234, 11434, 8080, 8000, 5000, 5001, 4891, 1337)`, overridable. |
| B4 normalized catalog with provider metadata | 🟡 | `models` / `modelsByEngine` / `loadedByEngine` exist for PAIR nodes. Gap: extend the shared wire types with provider, api format, quantization, context length, capability flags; include engine-only endpoints. |
| B5 LM Studio depth | 🟡 | `nvpair-engine-manager` manages LM Studio locally (manifest, `lms` stop, delete-model restart). Gap: consume `/api/v0`–`/api/v1` metadata (context length, quantization, capabilities, tokens/sec) during interrogation; document owner setup. |
| B6 capability verifier | 🟡 | Scanner enriches peers over HTTP; manual nodes probed every 10 s. Gap: verify chat/completions/embeddings/streaming/tools per endpoint and record auth requirements + token validity. |
| C1 manual registration of any base URL | 🟡 | `nvpair-manual-nodes` accepts user-added PAIR nodes (probes node-info). Gap: accept arbitrary engine base URLs (engine-only endpoints) with provider hint. |
| C2–C4 localhost/subnet scan with bounds + SSRF guard | ❌ | Scanner is mDNS-only today. New: opt-in bounded port-probe discovery with Vampire's exact safety bounds; never auto-register (permission before routing). |
| C5 classification ladder | ❌ | New endpoint classification: unknown → OpenAI-compatible → owner-labelled → verified (paired PAIR node) → realm-approved. |
| C6 mDNS opt-in advertisement | ✅ | `_nvpair-node._tcp` consolidated record; own responder sharing UDP 5353. Vampire's "planned" item is done here. |
| C7 QR onboarding | ❌ | PIN pairing exists (EAP-NOOB); add QR transport for the same exchange + guest tier (M1). |
| C8 node agent | ✅/🟡 | PAIR *is* the node agent (scanner + node-info + engine-manager + cluster-manager). Gap: publish owner mode + contribution ceiling + policy labels in the advertisement/enrichment (H1, H6). |
| C9 LM Link awareness | ❌ | Optional adapter nicety: recognize LM Link-backed LM Studio endpoints in classification. |

### Registry, routing, scheduling

| Feature | Status | PAIR reality and gap |
| --- | --- | --- |
| D1–D3 registry CRUD + node fields | 🟡 | Discovery snapshot + manual nodes + trust store exist with `node/*`, `nodes/list`. Gap: PATCH-style editing, tags, engine-only endpoint records, provider field. |
| D4 drain/disable/maintenance | ❌ | No drain state; proxies route to any eligible node. New: routing-visible node status honored by both proxies and the scheduler. |
| D5 owner-granted trust | ✅ | PIN pairing pins certificates; `trusted` flag in discovery snapshot. |
| D6 health checks | ✅ | Manual nodes probed every 10 s; telemetry backoff; eviction rules. |
| E1 virtual models in `/v1/models` | ❌ | New `pair:` aliases injected by the proxies. |
| E2 route policies (CRUD) | ❌ | New route-policy store + JSON-RPC surface. |
| E3 MVP strategies | 🟡 | Scheduler ranks by pending + smoothed GPU pressure (≈`least_busy`), with stable-ID fallback and per-proxy reservations; manual pin exists in TUI. Gap: named, selectable strategies — `round_robin`, `least_latency`, `model_affinity`, `trusted_only` — applied per route. |
| E4 extended strategies | ❌ | `weighted_round_robin`, `highest_tokens_per_second`, `best_available` weights, `context_window`, `power_saver`, `fallback_chain`, `quality_score`, `cost_score`, `privacy_policy`, hedging, circuit breakers, warm-model preference (documented today as a scheduler limitation). |
| E5 opt-in request extensions | ❌ | New `pair` body object + `X-PAIR-Mode` / `X-PAIR-Strategy` / `X-PAIR-Route` parsing in the proxies; plain requests unchanged. |
| E6 response metadata headers | ❌ | New `X-PAIR-Route` / `-Strategy` / `-Node` / `-Model` response headers; extension stripped before forwarding. |
| E7 actionable routing errors | 🟡 | Local `502` exists; add coded envelopes + rejected-node detail. |
| E8 model-eligibility gate | ✅ | Capability gate with `:latest` normalization, 404-failover, no-blind-retry of 4xx. |

### Coalescing, cache, fusion

| Feature | Status | PAIR reality and gap |
| --- | --- | --- |
| F1–F4 fingerprint, dedup, multiplex, TTL cache | ❌ | New shared Go package + proxy integration. Fingerprints are salted hashes; no prompt content at rest (repo logging rule). |
| F5 semantic cache | ❌ | Later, realm-gated, embeddings-based; explicitly opt-in. |
| F6 reuse-policy variations | ❌ | Policy hooks (H3) decide reuse per realm/request. |
| G1–G6 fusion, pipelines, jobs, traces | ❌ | Biggest architectural addition. PAIR's invariant is "one request goes to one node and the proxy never splits it", and peer ingress never re-routes. Fusion therefore lives in a **new worker that acts as a local client of the proxies**: it fans out N ordinary requests (each individually routed, trust-checked, and metered), then aggregates. Jobs/traces are its stores, relayed over the broker; the workload manager already gives every node cluster-wide job visibility. |

### Governance, security, observability, UX

| Feature | Status | PAIR reality and gap |
| --- | --- | --- |
| H1 owner share modes | ❌ | `nvpair-node-settings` stores typed per-node prefs; advertise mode via scanner enrichment; enforce at proxy/cluster gates. |
| H2 realms | ❌ | Layer named realms over cluster membership (cluster-manager) with per-realm policy documents. |
| H3 policy engine | ❌ | New policy evaluation consulted by proxies (and fusion) before routing. |
| H4 client-facing auth | ❌ | Today loopback plaintext is unauthenticated by design; peers use mTLS. Add optional bearer tokens on the loopback surface (default off — drop-in preserved). |
| H5 token vault | ❌ | Store upstream endpoint tokens (e.g. LM Studio API tokens) OS-keychain-backed where available, file-based otherwise; owned by a Go service, never logged, never sent to clients. |
| H6–H9 tiers, consent invariants, idle gating, metering | ❌ | Contribution ceiling (`pair<NN>` semantics) declared by each node owner, advertised, enforced by scheduler + ingress; ledger in the workload manager (already relays every transition). Idle gating from owner-activity detection (node-info). |
| H10 rate limits, quotas, audit, CORS, allowlists | ❌ | Per-realm counters + limits at the proxies; audit = metadata-only ledger. |
| H11 two trust boundaries | ✅ | Already structural: loopback client surface vs mTLS cluster surface vs engine loopback. Keep it explicit in docs. |
| H12 no credential leakage / privacy-class routing | 🟡 | mTLS + loopback engines exist; add privacy-class node filters (E4) and A6 stripping. |
| I1–I3 status/metrics/perf capture | 🟡 | Telemetry, workloads, jobs view, GPU charts exist. Gap: one aggregated metrics snapshot (cluster + per-node counters incl. requests, errors, avg latency, tokens/sec) over JSON-RPC. |
| I4 policy-aware metrics, history, bench | ❌ | Follows policy phases; benchmarking joins the optimizer. |
| I5 explainable routing (traces) | 🟡 | Jobs view shows where work ran. Gap: per-request routing trace with candidate scoring. |
| J1–J3 dashboard, playground, open-URL | 🟡 | Desktop UI + TUI cover nodes/models/health/jobs; `inference-demo` exists in Electron. Gap: discovery review, routes editor, share/realm/token panels, metrics view, playground promoted to a first-class panel. |
| J4 family/business/event dashboards | ❌ | Realm-scoped views + one-click stop (M5). |
| K1–K2 CLI | ❌ | TUI is interactive only. New `nvpair` scripting CLI over broker JSON-RPC. |
| L1–L6 optimizer | ❌ | Catalogue from engine-manager + measured throughput; warm-model preference fixes a documented scheduler limitation; profiles map to virtual models. |
| M1–M5 event mode | ❌ | Guest realm + QR pairing + expiring trust + safe-model profile + stop button. |
| N1–N3 signed results | 🟡 | Node identity + pinned certs exist — the signing key is already there. Gap: optional response provenance envelope + trust-level ladder surfaced to clients. |
| O explorations | ❌ | Post-plan extensions (RAG, agents, structured output, multimodal, tools/MCP). |
| P non-goals | ✅ | Consistent with SECURITY.md; restated in [04-security-and-governance.md](04-security-and-governance.md). |

## New components this plan introduces

| Component | Kind | Owns |
| --- | --- | --- |
| `nvpair-route-manager` | new worker (14th binary) | Route policies, virtual models, named strategies, provider-endpoint registry, endpoint classification; feeds proxies via the broker exactly like `node/set-priority` does today. |
| `nvpair-policy-vault` | new worker (15th binary) | Realms, owner modes (enforcement), client tokens, upstream token vault, contribution tiers, policy evaluation, metering ledger write path. |
| `nvpair-fusion` | new worker (16th binary) | Fan-out modes, fusion strategies, pipelines, async jobs, traces; a client of the local proxies. |
| `nvpair` CLI | new binary (17th) | Scripting surface over broker JSON-RPC; parity with Vampire's CLI. |
| `shared/` additions | packages | Provider adapters + fingerprinting; request fingerprint/coalesce/cache; policy and realm wire types; signed-result envelope. |

Everything else is an extension of an existing worker: `lmstudio-proxy` and
`ollama-proxy` (gateway conformance, opt-in extensions, coalescing hooks,
enforcement), `nvpair-node-scanner` (probe discovery, share-mode/tier
advertisement), `nvpair-manual-nodes` (engine-only endpoints, drain states),
`nvpair-engine-manager` (richer interrogation, catalogue, benchmarks),
`nvpair-job-scheduler` (strategy framework, warm preference, ceilings),
`nvpair-workload-manager` (ledger relay), `nvpair-cluster-manager` (realms,
QR/guest pairing, signing), `nvpair-node-settings` (owner modes, policy
storage), `nvpair-ui-broker` (relays for every new surface), `nvpair-tui` and
the desktop app (new panels).

Alternatives considered and rejected: extending `nvpair-job-scheduler` to own
routes (it holds no socket and is deliberately model-blind; routes need
persistence and a control surface); a single "vampire" mega-worker (violates
PAIR's one-responsibility worker layout); adding an HTTP control API cloning
`/vampire/v1/*` (duplicates the broker contract — the CLI covers automation).
