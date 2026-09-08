<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# LLM Vampire → PAIR: Feature Adoption Plan

This folder holds the plan for implementing every feature of
[`japer-technology/llm-vampire`](https://github.com/japer-technology/llm-vampire)
— the "Local LLM Aggregator and Maximizer" — as a **modification to the NVIDIA
Personal AI Router (PAIR)** rather than as a separate product.

LLM Vampire is a single-process Python/FastAPI gateway that discovers,
normalizes, and combines owner-approved local LLM services (LM Studio, Ollama,
llama.cpp, vLLM, LocalAI, Jan, GPT4All, KoboldCpp, text-generation-webui, and
other OpenAI-compatible servers) behind one governed OpenAI-compatible endpoint.
PAIR is a multi-process Go + Electron system that already discovers, pairs, and
routes across PAIR nodes. The two overlap heavily; this plan absorbs everything
Vampire does (shipped and designed) into PAIR's architecture, and skips nothing.

## Documents

| Document | Contents |
| --- | --- |
| [00-feature-inventory.md](00-feature-inventory.md) | The complete catalog of llm-vampire features — shipped (its Phases 0–4) and designed (Phases 5–7, the ASPIRATION roadmap, the security thesis). This is the traceability list the rest of the plan must cover. |
| [01-architecture-mapping.md](01-architecture-mapping.md) | Concept-by-concept mapping of Vampire onto PAIR components, with a gap analysis: what PAIR already has, what is partial, what is missing. |
| [02-implementation-plan.md](02-implementation-plan.md) | The phased implementation plan: scope, components touched, JSON-RPC contract changes, desktop/TUI surfaces, tests, and acceptance criteria per phase. |
| [03-api-and-cli-mapping.md](03-api-and-cli-mapping.md) | Route-by-route mapping of Vampire's `/v1/*` gateway, `/vampire/v1/*` control API, opt-in request extensions, and CLI onto PAIR's ports, JSON-RPC methods, and a new scripting CLI. |
| [04-security-and-governance.md](04-security-and-governance.md) | Vampire's governance model — consent invariants, `vampire<NN>` contribution tiers, realms, owner modes, token vault, metering, enforcement — expressed in PAIR's trust architecture. |

## Ground rules

The plan is written to PAIR's standing conventions, which every phase obeys:

1. **Services own behavior; the desktop renders it.** Routing, discovery,
   policy, caching, fusion, and cryptography land in Go services under
   `services/`. Nothing in this plan adds routing or policy logic to the
   Electron tree.
2. **One canonical path, no legacy fallbacks.** Where Vampire and PAIR provide
   the same capability two ways, the plan picks one and deletes nothing that
   other clients depend on. Existing wire names, ports, and Go symbols keep
   their spelling; they are external contracts.
3. **PAIR-native naming.** Vampire's opt-in extension surface is adopted with
   PAIR names: the `pair` request object, `X-PAIR-*` headers, and `pair:` virtual
   model IDs replace `vampire`, `X-Vampire-*`, and `vampire:`. Features carry
   over; branding does not. User-facing copy says *engine*, not *backend*.
4. **Contract changes travel together.** Any new or changed JSON-RPC method
   updates the producing Go service, the broker relay, every consumer, the
   desktop bridge under `desktop/src/electron/service-bridge/`, the tests, and
   the documentation in the same change, then passes
   `npm run service-contracts:check`.
5. **Privacy invariants are shared.** Vampire's "never store prompts by
   default" matches PAIR's "never log prompts, messages, response bodies,
   pairing PINs, or key material". Every cache, ledger, trace, and metric in
   this plan stores hashes and operational metadata, never content, unless an
   owner explicitly opts a realm into retention.
6. **Versioning and hygiene.** Each phase bumps the affected binaries in
   `services/versions.json`, carries two-line SPDX headers on new files, keeps
   commits small and signed off, and runs the existing checks (`make check`,
   `npm run dead-code:check`, per-component `go test ./...`, `services/tests`).

## The one-paragraph summary

PAIR already has the hard parts Vampire still lists as future work — mDNS
discovery, PIN pairing with pinned mutual TLS, dual compatibility proxies with
port takeover, model-eligibility gating, scheduler-driven failover, cluster-wide
workloads/errors/telemetry, an engine control plane, a desktop UI, and a TUI.
What PAIR lacks is exactly Vampire's center of gravity: routing to **bare
engine endpoints that don't run PAIR**, provider adapters and one normalized
catalog, opt-in port-scan discovery, virtual models and per-route routing
strategies, request coalescing and caching, client-facing auth with tokens,
realms/owner-modes/policy, contribution tiers with metering, fusion and
pipelines, an optimizer, event mode, signed results, and a scripting CLI. The
plan adds those as new Go workers and extensions to existing ones, phase by
phase, each independently shippable.
