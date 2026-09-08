<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 03 — API and CLI Surface Mapping

Route-by-route and command-by-command mapping from llm-vampire's surfaces to
their PAIR equivalents. Phase numbers refer to
[02-implementation-plan.md](02-implementation-plan.md).

## 1. OpenAI-compatible gateway (`/v1/*`)

Vampire serves one gateway on `:7777`. PAIR keeps its two compatibility
proxies with port takeover — OpenAI-compatible on `1234` (`lmstudio-proxy`),
Ollama-compatible on `11434` (`ollama-proxy`) — so unmodified clients gain the
cluster without any base-URL change at all.

| Vampire route | PAIR surface | Phase |
| --- | --- | --- |
| `GET /v1/models` (aggregated physical + virtual) | `:1234/v1/models` fan-out merge (exists) + provider-endpoint inventories (2) + injected `pair:*` virtual models (4); Ollama surface mirrors via `/api/tags` merge (exists) | 2, 4 |
| `POST /v1/chat/completions` | `:1234/v1/chat/completions` routed (exists); conformance + error envelope | 1 |
| `POST /v1/completions` | same, guaranteed and tested | 1 |
| `POST /v1/embeddings` | same, guaranteed and tested, embedding-capable owners only | 1 |
| `POST /v1/responses` | same, guaranteed and tested | 1 |
| catch-all `/v1/{path}` all verbs | documented passthrough to the selected node | 1 |
| streaming SSE + mid-stream error frame + `[DONE]` | both proxies | 1 |
| OpenAI error envelope (+ routing detail: rejected nodes, reasons) | both proxies; codes `no_suitable_node`, `upstream_unavailable`, `upstream_stream_interrupted`, `no_route_target`, `route_target_removed`, `unsupported_strategy` | 1, 4 |
| gateway terminates client credentials; upstream tokens attached per node | proxies + token vault | 6 |

## 2. Opt-in request extension

Vampire's opt-in contract, PAIR-named. Plain requests remain a transparent
proxy; the extension activates routing/fusion features per request.

| Vampire | PAIR | Notes |
| --- | --- | --- |
| body `vampire: {…}` object | body `pair: {…}` object | stripped before forwarding; `model` rewritten to the physical target |
| model `vampire:auto` / `vampire:<alias>` | `pair:auto` / `pair:<alias>` | virtual models listed in `/v1/models` |
| `X-Vampire-Mode: route\|fallback\|race\|parallel\|fusion\|debate\|pipeline` | `X-PAIR-Mode: …` | `route`/`fallback` in Phase 4; fan-out modes in Phase 7 |
| `X-Vampire-Strategy: <strategy>` | `X-PAIR-Strategy: <strategy>` | strategy names below |
| `X-Vampire-Route: <route-id>` | `X-PAIR-Route: <route-id>` | pins a configured route |
| response `X-Vampire-Route/-Strategy/-Node/-Model` | `X-PAIR-Route/-Strategy/-Node/-Model` | attribution metadata |
| `vampire.routing.strategy` / `.weights` | `pair.routing.strategy` / `.weights` | weights used by `best_available` |
| `vampire.fusion.{strategy,candidates,judge_model,min_agreement,include_dissent}` | `pair.fusion.{…}` | Phase 7 |
| cache controls (Phase 5 design) | `pair.cache: off` / `X-PAIR-Cache: off` | Phase 5 |
| signed results (`trust` block in response `vampire` metadata) | `pair.signed: true` → provenance envelope in response `pair` metadata | Phase 9 |

Strategy names (Phase 4 unless noted): `round_robin`, `least_busy`,
`least_latency`, `model_affinity`, `trusted_only`, `weighted_round_robin`,
`highest_tokens_per_second`, `best_available`, `context_window`,
`power_saver`, `fallback_chain`; `quality_score`, `cost_score` (Phase 8),
`privacy_policy` (Phase 6). Fusion strategies (Phase 7): `best_of_n`,
`majority_vote`, `ranked_vote`, `judge_synthesis`, `claim_merge`,
`contradiction_check`, `critic_refine`, `consensus_only`, `return_all`.

## 3. Control API (`/vampire/v1/*` → broker JSON-RPC)

PAIR's control plane is newline-delimited JSON-RPC 2.0 over the broker (stdio
or `--ipc`), not HTTP. Each Vampire control route maps to broker methods —
existing ones where PAIR already covers the capability, new namespaces where
it does not. New methods follow the existing relay pattern (worker method,
broker relay + notification fan-out, service bridge, `services-api.md` via
`npm run service-contracts:write`).

| Vampire control route | Broker method(s) | Status / Phase |
| --- | --- | --- |
| `GET /vampire/v1/status` | existing `discovery:get-nodes`, `cluster:status` + new `metrics:snapshot` summary | Phase 10 |
| `GET /vampire/v1/nodes` | existing `nodes/list`, `discovery:get-nodes` (+ provider/classification fields) | Phase 2 |
| `POST /vampire/v1/nodes` | existing `node/*` manual-node add, extended for engine-only endpoints | Phase 2 |
| `GET /vampire/v1/nodes/{id}` | existing node read, extended fields | Phase 2 |
| `PATCH /vampire/v1/nodes/{id}` | new `node/update` (name, tags, status, provider hint, token ref) | Phase 2 |
| `DELETE /vampire/v1/nodes/{id}` | existing manual-node remove | exists |
| drain / restore | new `node/drain` (`draining`/`disabled`/`maintenance`/`online`) | Phase 2 |
| `POST /vampire/v1/discover` | new `discovery:probe` (methods `local`/`static`/`lan_scan`, subnets, ports, timeout, trusted-only, base-urls) + `discovery:probe-result` notifications; approval via manual-nodes add | Phase 3 |
| `GET /vampire/v1/models` | existing `engine:models` + peer inventory (broker snapshot) extended with provider metadata; detailed physical inventory | Phase 2 |
| `GET /vampire/v1/metrics` | new `metrics:snapshot` (cluster + per-node health, requests, errors, avg latency, tokens/sec, cache/coalesce counters; per-realm after Phase 6) | Phases 4–6 |
| `GET/POST /vampire/v1/routes`, `GET/DELETE /vampire/v1/routes/{id}` | new `routes:list/create/get/update/delete/test` on `nvpair-route-manager` | Phase 4 |
| `GET/POST /vampire/v1/share` | new `share:get/set` on `nvpair-policy-vault` (modes off/local/personal/family/business/event + duration + model) | Phase 6 |
| `POST /vampire/v1/fusion` | new `fusion:run` | Phase 7 |
| `POST /vampire/v1/pipelines` | new `pipeline:run/create/list` | Phase 7 |
| `GET /vampire/v1/jobs/{id}` | new `jobs:get/list/cancel` (joins existing workloads view) | Phase 7 |
| `GET /vampire/v1/traces/{id}` | new `traces:get` | Phases 4/7 |
| aspirational `GET /vampire/v1/capabilities` | node capability records on `nodes/list` | Phase 2 |
| aspirational `POST /vampire/v1/routing/preview` | `routes:test` (dry-run explain) | Phase 4 |
| aspirational `POST /vampire/v1/tokens` | new `tokens:create/list/revoke/rotate` | Phase 6 |
| aspirational `POST /vampire/v1/events` | new `event:start/stop/extend/guests` | Phase 9 |
| aspirational `GET /vampire/v1/health` | existing errors surface (`errors:*`) + `metrics:snapshot` | Phases 6/10 |
| node agent `/agent/v1/*` | PAIR nodes already run the agent (scanner + node-info + engine-manager); owner-mode/tier publishing added to the advertisement | Phase 6 |

Environment/config parity: Vampire's `VAMPIRE_*` settings map to PAIR's
existing mechanisms — `NVPAIR_LOG_LEVEL`/`--log-level` (exists),
per-node settings via `nvpair-node-settings`, proxy ports via existing
persisted port plans. No new global env prefix is introduced.

## 4. CLI mapping (`vampire …` → `nvpair …`)

New `nvpair` scripting binary (Phase 10; earlier phases land their verbs with
their features). It attaches to a running broker or supervises its own, like
`nvpair-tui`; `--json` gives stable machine output; control verbs take
`--socket` for `--ipc` attachment.

| Vampire command | `nvpair` equivalent |
| --- | --- |
| `vampire serve` | not needed — the broker/UI/TUI already serve; `nvpair daemon start|stop|status` covers headless supervision |
| `vampire status [--watch]` | `nvpair status [--watch]` |
| `vampire discover [--method local\|static\|lan_scan] [--subnet] [--port] [--timeout-ms] [--trusted-only] [--base-url]` | `nvpair discover …` same flags; `--method mdns` lists the live mDNS directory |
| `vampire nodes list/add/get/update/drain/delete` | `nvpair nodes list/add/get/update/drain/delete` (`add ID BASE_URL [--provider] [--name] [--trusted] [--tag]…`, `drain ID [on|off]`) |
| `vampire models` | `nvpair models [--node] [--refresh]` |
| `vampire metrics` | `nvpair metrics [--node]` |
| `vampire route list/add/get/delete` | `nvpair route list/add/get/update/delete/test` (`add ROUTE_ID VIRTUAL_MODEL --target node:model… [--strategy] [--fallback]`) |
| `vampire share {on/off/local/personal/family/business/event/stop} [--duration] [--model]` | `nvpair share …` same shape |
| `vampire dashboard / ui [--open]` | `nvpair ui` (opens the desktop app or prints TUI guidance) |
| `vampire --version` | `nvpair --version` (stamped from `versions.json`) |
| planned T1: `chat`, `embed`, `complete`, `health`, `nodes ping`, `logs`, `config show/set/get`, `doctor`, `route test`, `version --full` | same names under `nvpair` (chat/embed/complete call the local gateway) |
| planned T2: `route strategy list/set-default/canary/shadow`, `failover show/test`, `nodes priority/maintenance/quarantine`, `queue status/drain`, `retry policy`, `route constraints`, `route benchmark`, `jobs cancel` | same names; `nodes priority` reads the scheduler ordering |
| planned T3: `discover --watch`, `pair [--qr]`, `nodes import/export/prune/tag/wake/capabilities/verify`, `topology` | same names; `pair --qr` fronts Phase 9 onboarding |
| planned T4: `history`, `stats models/tokens`, `errors`, `latency`, `bench run/compare`, `watch`, `export metrics`, `power report`, `leaderboard` | same names over `metrics:*`/`errors:*`/optimizer data |
| planned T5: `token create/list/revoke`, `auth enable`, `policy show/set`, `allowlist ip/model`, `quota set`, `audit log`, `cors set`, `redact rules`, `privacy local-only`, `session private`, `route privacy --trusted-only`, `tls enable` | same names over `policy:*`/`tokens:*`; `tls enable` is N/A (cluster mTLS is always on) — documented rename |
| planned T6: `cache status/clear`, `coalesce status`, `warmup`, `models evict`, `prefetch`, `schedule set`, `route cost-aware` | same names (`warmup`/`evict` via engine-manager load/unload) |
| planned T7: `fusion run [--mode]`, `race`, `vote`, `judge`, `ensemble create`, `best-of N`, `debate`, `verify` | same names over `fusion:*` |
| planned T8: `pipeline run/list/create`, `jobs list/get`, `trace get`, `batch run` | same names |
| planned T9: `rag index/query/collections` | Phase X extension |
| planned T10: `sign verify`, `trust score`, `daemon start/stop/status` | `sign verify` (Phase 9), `trust score` (classification + ladder), `daemon` (supervision) |
| aspirational: `scan`, `nodes approve/verify`, `token set/rotate/remove`, `access list/invite/revoke`, `aliases`, `realms`, `policy allow-model/deny-model`, `quotas`, `audit tail`, `event start/qr/guests/extend/stop`, `cache stats`, `logs tail`, `config export/import`, `shutdown` | same names, mapped to the corresponding phases (`scan`≡`discover`, `aliases`≡`route`, `realms`/`event`/`access` over `policy:*`/`event:*`) |

## 5. Ports

No new client-facing ports. New peer surfaces added by the plan (ledger
relay already rides workload propagation on `14320`; signed-result
verification is offline) reuse existing `143xx` listeners; if a genuinely new
peer surface becomes necessary it takes the next fixed `143xx` port and is
registered with the scanner like the others. The discovery probe (Phase 3) is
an outbound client, not a listener.
