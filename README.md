# Solana MEV Searcher

A low-latency, cross-DEX **backrun-arbitrage** searcher for Solana — built and operated in Rust. **Backrun-only by design: no front-running, no sandwiching.**

## What it does
Maintains a real-time price graph across **Raydium AMM v4, Orca Whirlpool, Pump / PumpSwap and Meteora DLMM**, detects the transient cross-DEX dislocations that user swaps create, and submits **revert-protected backruns** to arbitrage them. Also runs **liquidation detection** (Kamino klend + Scope, MarginFi), record-only.

## Architecture

Six subsystems, from market data to confirmed P&L — the strategy core is abstracted on purpose. The full **low-level design** (ingestion, detection and execution internals) lives in **[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)**.

```mermaid
flowchart TB
  subgraph SRC["① Market Data"]
    G["Geyser gRPC<br/>accounts · slots · tx"]
    W["WebSocket RPC<br/>failover"]
    SH["Shred stream<br/>raw Turbine UDP"]
  end
  subgraph ING["② Ingestion &amp; Live State"]
    FEC["Shred decoder<br/>Reed–Solomon FEC"]
    MUX["Multiplexer<br/>dedup · per-account order · coverage"]
    PG["Price graph<br/>pools · vaults · reserves"]
    DS["Deep-state cache<br/>exact tick / bin arrays"]
  end
  subgraph DET["③ Opportunity Engine"]
    PS["Pairwise scan<br/>convex size search"]
    BF["Negative-cycle search<br/>multi-hop routes"]
    QQ["Exact quoters<br/>CLMM / DLMM, on-chain-equivalent"]
    SC{{"Strategy core<br/>(abstracted)"}}
  end
  subgraph EXE["④ Execution &amp; Landing"]
    TB["Tx builder<br/>legacy · v0+ALT · v1"]
    SG["Simulation gate<br/>CU sizing"]
    CAP["Flashloan capital<br/>Kamino · MarginFi"]
    LD["Multi-path landing<br/>Jito · Sender · bundles"]
  end
  subgraph SAF["⑤ Risk &amp; Safety"]
    CB["Circuit breakers"]
    PL["Realized-P&amp;L limits"]
    KS["Kill switch"]
  end
  subgraph RSR["⑥ Measurement · offline"]
    CEN["Market census"]
    RA["Race / landing audit"]
    LT["Latency &amp; shred-lead"]
  end

  G --> MUX
  W --> MUX
  SH --> FEC --> MUX
  MUX --> PG --> DS
  PG --> PS
  PG --> BF
  DS --> QQ
  PS --> SC
  BF --> SC
  QQ --> SC
  SC -->|verified opportunity| TB
  CAP --> TB
  TB --> SG --> LD
  SAF -. gates .-> EXE
  EXE -. outcomes .-> SAF
  EXE -. records .-> RSR
  RSR -. calibrates .-> DET

  classDef src fill:#e8eefc,stroke:#3b5bdb,color:#0b1b4d;
  classDef ing fill:#e7f5ff,stroke:#1c7ed6,color:#062a4d;
  classDef det fill:#e9f7ef,stroke:#2f9e44,color:#0b2e1a;
  classDef exe fill:#fff4e6,stroke:#e8590c,color:#3d1a05;
  classDef saf fill:#ffe9ec,stroke:#e03131,color:#4d0a12;
  classDef rsr fill:#f1f3f5,stroke:#868e96,color:#212529;
  classDef abs fill:#f3f0ff,stroke:#7048e8,color:#1f1033;
  class G,W,SH src;
  class FEC,MUX,PG,DS ing;
  class PS,BF,QQ det;
  class SC abs;
  class TB,SG,CAP,LD exe;
  class CB,PL,KS saf;
  class CEN,RA,LT rsr;
```

## On-chain correctness
- **Exact CLMM/DLMM quoters** — full ports of the Orca Whirlpool and Meteora DLMM swap loops (tick- and bin-array traversal, adaptive/variable fees, oracle-aware), equal to the programs' *own* on-chain output on real mainnet pools (verified against `orca-whirlpools-core` and the DLMM SDK).
- A leg that cannot be quoted exactly is **dropped, never approximated** — no x·y=k fallback on pools with plugins. This is how phantom profit is engineered out.
- Zero-copy, `bytemuck`-Pod decoders; byte layouts pinned to canonical IDLs and cross-checked against real mainnet account dumps and a golden-fixture corpus.

## Low-latency architecture
- **Lock-free hot path** — a sharded concurrent price graph with `Arc`-swap state updates, strict per-account ordering, bounded back-pressured channels, a single sequential update consumer feeding a work-stealing opportunity dispatcher. No allocation or locking on the critical path.
- Keyed, content-hashed deduplication at **~18.6 ns per update** (benchmarked), so the ingest firehose never stalls detection.
- **Tiered detection** — O(1) pair-scan on the changed pair, plus a periodic Bellman-Ford negative-cycle search run off-thread, with per-scan memoization and a provably-sound pruning pass.
- **Frankfurt co-location** (~1–2 ms to regional Jito/Helius endpoints), warm connection pooling, and leader- and slot-boundary-aware send timing.
- Every pipeline stage is instrumented with **p50 / p90 / p99 latency percentiles**.

## Transactions & execution
- **Legacy / v0 + ALT / hand-encoded v1 (SIMD-0385)** transactions, golden-tested byte-for-byte against real mainnet v1 transactions; self-managed address-lookup tables; a unique-key account-budget bound computed before every build.
- **Flashloan-funded** (Kamino, MarginFi) with exact on-chain fee accounting, proven by mainnet simulation.
- Compute-unit limits sized from live simulation — cached, shape-learned, or modeled; **atomic, revert-dropping bundles** guarded by an on-chain profit check.

## Robustness & safety
- **Fail-closed everywhere** — landing confirmed by on-chain signature; submission pauses if the chain can't be read; stream-lag and data-freshness gates; automatic stream reconnection with targeted repair reads; a quota governor that throttles cost but never starves discovery.
- **Production safety rails** — per-DEX and global circuit breakers, a kill switch, and persisted hourly / daily / lifetime realized-P&L loss limits restored across restarts.
- Secrets never reach logs, captures, or error text (centralized redaction); overflow-checked arithmetic; typed, classified error handling.

## Engineering discipline
- **Deterministic capture-and-replay** — the production pipeline replays a recorded mainnet stream *byte-identically per build*, so every change is A/B-verifiable offline against real data.
- Shred-level decoding with **Reed-Solomon (FEC) reconstruction** and kernel-timestamped arrival analysis for latency measurement.
- A **record-only, pre-registered-measurement** rollout — nothing goes live until a frozen, pre-registered metric passes. Correctness and measurement before capital.

_Source is private; a scoped technical walkthrough is available to partners under NDA._
