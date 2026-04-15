---
id: "DEC-REALTIME-GENRPC-PUBSUB"
type: "decision"
title: "Realtime Uses Gen_rpc Instead of Distributed Erlang for Cross-Node Messaging"
domain: "realtime"
status: "draft"
last_verified: 2026-04-15
freshness_note: "Deep read of gen_rpc adapter, application.ex, runtime.exs, and RPC module"
freshness_triggers:
  - "lib/realtime/gen_rpc/pub_sub.ex"
  - "lib/realtime/gen_rpc.ex"
  - "lib/realtime/application.ex"
  - "config/runtime.exs"
  - "mix.exs"
known_unknowns:
  - "Exact throughput benchmarks comparing gen_rpc vs Distributed Erlang PubSub in Realtime's workload"
  - "Whether the Supabase gen_rpc fork adds features beyond upstream or only patches"
  - "Historical incident or scaling event that originally motivated the switch"
tags:
  - decision
  - architecture
  - realtime
  - distributed-systems
aliases: []
relates_to:
  - type: "parent"
    target: "[[SYS-REALTIME]]"
  - type: "refines"
    target: "[[GLOSSARY-REALTIME]]"
sources:
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/gen_rpc/pub_sub.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/gen_rpc.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/application.ex"
  - category: "implementation"
    type: "code"
    ref: "config/runtime.exs"
  - category: "implementation"
    type: "code"
    ref: "mix.exs"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime/nodes.ex"
  - category: "implementation"
    type: "code"
    ref: "lib/realtime_web/channels/realtime_channel/message_dispatcher.ex"
notes: ""
---

## Context

Realtime is a multi-tenant, multi-region Elixir/Phoenix application that must fan out broadcast, presence, and CDC messages to WebSocket clients distributed across a cluster of nodes. Phoenix's default PubSub adapter uses Distributed Erlang (`pg2` / `:pg`), which establishes a full mesh of connections between all BEAM nodes. In a multi-region deployment with many nodes, full-mesh Distributed Erlang becomes a bottleneck: every node must maintain a TCP connection to every other node, and PubSub messages are sent individually to each peer. Realtime needed a cross-node messaging layer that could handle region-aware routing, connection pooling, and optional TLS without requiring a full Distributed Erlang mesh for message transport.

## Decision

Replace Phoenix's default Distributed Erlang PubSub adapter with a custom `Realtime.GenRpcPubSub` adapter backed by a Supabase-forked `gen_rpc` library. All cross-node PubSub traffic (broadcast, presence diffs, CDC events) flows over `gen_rpc` TCP/SSL connections rather than Distributed Erlang distribution ports. The adapter implements a two-tier region-aware fan-out: messages are sent directly to nodes within the originating region, and a single representative node in each remote region receives the message and re-fans it locally.

## Key Facts

- PubSub adapter is hardcoded to `Realtime.GenRpcPubSub` in `application.ex` via `defp pubsub_adapter, do: Realtime.GenRpcPubSub` -- not configurable at runtime → `lib/realtime/application.ex`
- `GenRpcPubSub` implements `Phoenix.PubSub.Adapter` behaviour with `broadcast/4` and `direct_broadcast/5` callbacks that delegate to `GenRpc.abcast/4` → `lib/realtime/gen_rpc/pub_sub.ex`
- Two-tier fan-out: `broadcast/4` sends `:ftl` (forward-to-local) to same-region nodes and `:ftr` (forward-to-region) to one node per remote region; the remote node then re-broadcasts `:ftl` within its region → `lib/realtime/gen_rpc/pub_sub.ex`
- Remote region representative is chosen deterministically via `Nodes.node_from_region/2` using `:erlang.phash2(key, member_count)` over sorted region node lists → `lib/realtime/nodes.ex`
- Worker pool: `GenRpcPubSub` starts `broadcast_pool_size` (default 10) named GenServer workers; each broadcast selects a worker via `phash2(self(), pool_size)` for load distribution → `lib/realtime/gen_rpc/pub_sub.ex`, `config/runtime.exs`
- Workers set `Process.flag(:message_queue_data, :off_heap)` and `fullsweep_after: 20` to reduce GC pauses under high message throughput → `lib/realtime/gen_rpc/pub_sub.ex`
- `gen_rpc` is a Supabase fork (`git: "https://github.com/supabase/gen_rpc.git"`, pinned to specific ref) supporting both TCP (default port 5369) and SSL (port 6369) transport, configurable per-environment → `mix.exs`, `config/runtime.exs`
- Connection multiplexing: `MAX_GEN_RPC_CLIENTS` (default 5) controls how many TCP connections are opened between any two nodes; calls are distributed across them using `phash2(key, max_clients)` for ordering guarantees when a key is provided → `lib/realtime/gen_rpc.ex`
- Key-based routing ensures messages for the same tenant/channel use the same TCP connection, preserving ordering; keyless calls use random selection → `lib/realtime/gen_rpc.ex`
- `GenRpc.call/5` wraps `:gen_rpc.call` with telemetry (`[:realtime, :rpc]` events), latency tracking, and structured error logging; all RPC failures are normalized to `{:error, :rpc_error, reason}` → `lib/realtime/gen_rpc.ex`
- Authorization checks transparently proxy to remote nodes: `Authorization.get_read_authorizations/4` and `get_write_authorizations/4` detect `node(db_conn) != node()` and auto-dispatch via `GenRpc.call` → `lib/realtime/tenants/authorization.ex`
- `MessageDispatcher` caches serialized messages per serializer within a single dispatch cycle, avoiding redundant JSON encoding when fanning out to multiple subscribers on the same node → `lib/realtime_web/channels/realtime_channel/message_dispatcher.ex`
- Distributed Erlang is still used for cluster membership (libcluster) and `:syn` registry, but not for PubSub message transport → `lib/realtime/application.ex`, `config/runtime.exs`

## Rationale

The two-tier region-aware design reduces cross-region network hops. Instead of N*M messages (N nodes in region A to M nodes in region B), only one message crosses the region boundary, then fans out locally. The `gen_rpc` library provides TCP connection pooling (avoiding the single-connection-per-node constraint of Distributed Erlang), optional TLS for inter-node traffic in production, and configurable timeouts/batching. The key-based connection selection preserves message ordering per tenant without requiring a global lock.

## Consequences

### Positive
- Cross-region message volume scales as O(regions) rather than O(nodes), significantly reducing inter-region bandwidth
- TCP connection pooling (`MAX_GEN_RPC_CLIENTS`) allows tuning parallelism between node pairs independently of Distributed Erlang
- SSL transport secures inter-node traffic in production without requiring Erlang distribution over TLS
- Key-based routing preserves message ordering per tenant/channel while still distributing load across connections
- Worker pool with off-heap message queues isolates PubSub from GC pressure on the main schedulers

### Negative
- Dependency on a Supabase-forked `gen_rpc` library creates maintenance burden and divergence risk from upstream
- Two-tier fan-out adds a hop for cross-region messages (origin -> region representative -> region peers), introducing latency
- Debugging cross-node message delivery requires understanding both gen_rpc internals and the custom adapter layer
- Distributed Erlang is still required for `:syn` and `libcluster`, so the system runs two parallel networking stacks

### Neutral
- `gen_rpc` configuration is extensive (11+ environment variables for ports, SSL certs, timeouts, batching, compression) but well-documented in `runtime.exs`
- The adapter is a drop-in replacement at the PubSub level; Phoenix Channels and Presence APIs remain unchanged

## Related

- [[SYS-REALTIME]] -- parent system describing overall architecture
- [[GLOSSARY-REALTIME]] -- defines GenRpcPubSub, Syn, Broadcast, Presence terms
