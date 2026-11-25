# Architecture — Solana MEV Searcher

High-level design (HLD) and low-level design (LLD) for a low-latency **Solana MEV /
cross-DEX backrun-arbitrage** engine in **Rust**, routed **orderflow-auction-first**. The
strategy core and all tuning parameters are **abstracted** on purpose (see
[Abstraction boundary](#abstraction-boundary)); everything shown is the systems
engineering that makes a competitive **Solana searcher** correct and fast.

- [1. High-level design](#1-high-level-design)
- [2. Low-level design](#2-low-level-design)
  - [2.1 Ingestion &amp; live state](#21-ingestion--live-state)
  - [2.2 Opportunity engine](#22-opportunity-engine)
  - [2.3 Execution &amp; landing](#23-execution--landing)
- [3. Abstraction boundary](#3-abstraction-boundary)

---

## 1. High-level design

Six subsystems on one data path. The **primary route is orderflow-auction (OFA)**: an
auction order triggers a priced, revert-protected backrun submitted as a **bid** (position
won in the auction, latency-neutral). The **shred-reactive** route — detect from the raw
feed, then **race to land** — is retained as **fallback**. Both share the same state
engine, exact quoters, risk gates and measurement.

```mermaid
flowchart TB
  subgraph ORD["① Order &amp; Market Sources"]
    OFA["Orderflow auction (OFA)<br/>batch-auction order intake"]
    SH["Shred stream + Geyser / WSS<br/>raw Turbine · account state"]
  end
  subgraph ING["② Ingestion &amp; Live State"]
    FEC["Shred decoder<br/>Reed–Solomon FEC"]
    MUX["Multiplexer<br/>dedup · per-account order · coverage"]
    PG["Price graph"]
    DS["Deep-state cache<br/>exact tick / bin arrays"]
  end
  subgraph DET["③ Opportunity Engine"]
    SRCH["Pair &amp; multi-hop search"]
    QQ["Exact quoters<br/>CLMM / DLMM"]
    SC{{"Strategy core<br/>(abstracted)"}}
    BR["Revert-protected<br/>backrun build"]
  end
  subgraph EXE["④ Execution"]
    BID["OFA bid<br/>latency-neutral · position by auction"]
    RACE["Race-land<br/>Jito · Sender · multi-region"]
  end
  CB["⑤ Risk &amp; safety<br/>breakers · P&amp;L limits · kill switch"]
  MEAS["⑥ Measurement · offline<br/>census · race audit · shred-lead"]

  SH --> FEC --> MUX --> PG --> DS
  PG --> SRCH
  DS --> QQ
  SRCH --> SC
  QQ --> SC
  SC --> BR

  OFA ==>|primary| SRCH
  BR ==>|primary| BID

  SH -.->|fallback| SRCH
  BR -.->|fallback| RACE

  CB -. gates .-> EXE
  EXE -. outcomes .-> CB
  EXE -. records .-> MEAS
  MEAS -. calibrates .-> DET

  classDef src fill:#e8eefc,stroke:#3b5bdb,color:#0b1b4d;
  classDef ing fill:#e7f5ff,stroke:#1c7ed6,color:#062a4d;
  classDef det fill:#e9f7ef,stroke:#2f9e44,color:#0b2e1a;
  classDef prim fill:#e6fcf5,stroke:#0ca678,color:#063b2e,stroke-width:2px;
  classDef fb fill:#f1f3f5,stroke:#adb5bd,color:#495057;
  classDef abs fill:#f3f0ff,stroke:#7048e8,color:#1f1033;
  classDef meta fill:#fff4e6,stroke:#e8590c,color:#3d1a05;
  class OFA,BID prim;
  class SH src;
  class FEC,MUX,PG,DS ing;
  class SRCH,QQ,BR det;
  class SC abs;
  class RACE fb;
  class CB,MEAS meta;
```

| # | Subsystem | Responsibility |
|---|---|---|
| ① | Order &amp; market sources | **Primary:** orderflow-auction (OFA) order intake. **Fallback trigger + shared state:** raw Turbine shreds, Geyser gRPC, WebSocket RPC. |
| ② | Ingestion &amp; state | FEC recovery, dedup, strict per-account ordering, coverage/repair, into a concurrent price graph and an exact deep-state cache. |
| ③ | Opportunity engine | Pairwise and multi-hop search over **exact** quotes; candidate scoring by the abstracted strategy core; revert-protected backrun build. |
| ④ | Execution | **Primary:** OFA bid — latency-neutral, position won in the auction. **Fallback:** race-land via Jito / Helius Sender / bundle proxies. |
| ⑤ | Risk &amp; safety | Pre- and post-send gates: breakers, realized-P&amp;L limits, persisted kill switch. |
| ⑥ | Measurement | Offline census, race/landing audits, and latency analysis that calibrate the engine. |

---

## 2. Low-level design

### 2.1 Ingestion &amp; live state

The correctness-critical layer. Two wire formats (decoded shreds and gRPC updates)
converge into one strictly-ordered state writer. The design rule is **never discard live
data for being late** — ordering is enforced per account, not by a global clock.

```mermaid
flowchart LR
  SH["UDP shreds"] --> FA["FEC set assembly<br/>32:32 Reed–Solomon<br/>Merkle-root verified"]
  FA --> DEC["Entry / tx decode<br/>as bytes go contiguous"]
  GR["gRPC updates<br/>account · slot · tx"] --> REL["Relevance filter<br/>dataSize + discriminator"]
  DEC --> DED
  REL --> DED["Dedup table<br/>content-hashed · bank-aware<br/>(contested-slot safe)"]
  DED --> CONS["Single-writer consumer<br/>per-account ordering"]
  CONS --> UPS["Price-graph upsert<br/>Arc swap · shard-safe"]
  UPS --> COV["Coverage tracker<br/>first-looks · repairs · backstop"]
  COV -->|gap detected| REP["Targeted re-reads<br/>lane-scheduled"]
  REP --> CONS
  UPS --> CLK["Slot &amp; leader clock<br/>from block footers"]

  classDef hot fill:#e7f5ff,stroke:#1c7ed6,color:#062a4d;
  classDef edge fill:#fff4e6,stroke:#e8590c,color:#3d1a05;
  class FA,DEC,DED,CONS,UPS hot;
  class COV,REP,CLK edge;
```

**What's hard here:** reconstructing blocks from a lossy shred feed fast enough to matter;
deduplicating across two feeds *and* across Alpenglow contested slots (where a
`(pubkey, slot)` pair no longer identifies a single state, so the key must be content and
bank); and guaranteeing that out-of-order arrivals never overwrite newer state, without a
global lock on the hot path.

### 2.2 Opportunity engine

Detection runs on every relevant state change, but the expensive work is fenced behind
cheap screens: a per-pair memo short-circuits unchanged pairs, a concavity-based prune
discards hopeless pairs before any exact quote, and only survivors reach the
**on-chain-equivalent** quoters.

```mermaid
flowchart TB
  IN["Trigger<br/>OFA order (primary) / shred event (fallback)"] --> MEMO{Pair memo<br/>unchanged?}
  MEMO -->|hit| RE["Re-emit last result"]
  MEMO -->|miss| SCR["Screen quote<br/>concave · per leg"]
  SCR --> PR{Prune<br/>hopeless?}
  PR -->|yes| DROP["Skip"]
  PR -->|no| SZ["Convex size search<br/>+ exact re-quote"]
  SZ --> EX["Exact deep quote<br/>DLMM bin-walk /<br/>Orca tick-array loop"]
  EX --> BUD{Account-key<br/>budget ≤ 64?}
  BUD -->|no| DROP
  BUD -->|yes| NET["Net-SOL model<br/>fees · tip/bid · flashloan · rent"]
  NET --> SC{{"Strategy scoring<br/>(abstracted)"}}
  SC --> OPP["Verified opportunity<br/>revert-protected backrun"]

  subgraph T2["Multi-hop · timer"]
    BFS["Layered negative-cycle search<br/>from WSOL + anchors"]
  end
  BFS --> BUD

  classDef det fill:#e9f7ef,stroke:#2f9e44,color:#0b2e1a;
  classDef abs fill:#f3f0ff,stroke:#7048e8,color:#1f1033;
  class SCR,SZ,EX,NET,BFS det;
  class SC abs;
```

**What's hard here:** the quoters are full reimplementations of the DEX programs' swap
math (tick arrays, bin arrays, adaptive fees, oracle timing), so a route that *detects*
positive actually *executes* positive. The pruning must be provably sound — it is only
valid while every screen is concave and non-decreasing — or it would silently drop real
routes. The account-key budget models the compiled transaction's unique-key bound *before*
building, so no work is wasted on a route that can't fit in one transaction.

### 2.3 Execution &amp; landing

From verified opportunity to confirmed P&amp;L. Gated twice (once before building, once
after simulation and before the irreversible send), landed by **signature**, and every
outcome classified so the breakers and limits see the truth. The send is routed
**primary-first**: an **OFA bid** (position won in the auction) with multi-region
**race-landing** as fallback.

```mermaid
flowchart TB
  OPP["Verified opportunity"] --> G1{Pre-submit gate<br/>age · breakers · pause · inflight}
  G1 -->|block| STOP["Refuse"]
  G1 -->|pass| VER["Route re-verify<br/>fresh tick / bin read"]
  VER --> BLD["Build tx<br/>floors · on-chain profit guard<br/>legacy / v0+ALT / v1"]
  BLD --> CU["CU sizing<br/>simulated · cached · shape · model"]
  CU --> SIM{Simulate<br/>before submit?}
  SIM -->|fail| CL["Error classifier<br/>economic vs on-chain fault"]
  SIM -->|clean| G2{Gate again<br/>+ inflight slot + pause}
  G2 -->|pass| RT{Route}
  RT ==>|primary| BID["OFA bid<br/>latency-neutral · position by auction"]
  RT -.->|fallback| RACE["Race-land<br/>multi-region Jito · Sender · proxies"]
  BID --> CFM["Landing by signature<br/>confirmed on chain"]
  RACE --> CFM
  CFM --> PNL["Realized P&amp;L<br/>native + WSOL delta"]
  PNL --> LIM["P&amp;L limits · kill switch"]
  CL --> BRK["Circuit breakers"]

  classDef exe fill:#fff4e6,stroke:#e8590c,color:#3d1a05;
  classDef saf fill:#ffe9ec,stroke:#e03131,color:#4d0a12;
  classDef prim fill:#e6fcf5,stroke:#0ca678,color:#063b2e,stroke-width:2px;
  classDef fb fill:#f1f3f5,stroke:#adb5bd,color:#495057;
  class VER,BLD,CU,CFM,PNL exe;
  class BID prim;
  class RACE fb;
  class CL,BRK,LIM saf;
```

**What's hard here:** encoding three transaction versions by hand (the v1 / SIMD-0385 wire
format has no SDK support) and keeping instruction indices stable so flashloan sysvar
checks still pass; sizing compute units from live simulation with cached and modelled
fallbacks; routing sends against a leader clock reconstructed from block footers; and
treating landing as **fail-closed** — an unreadable landing or P&amp;L pauses the engine
rather than guessing.

---

## 3. Abstraction boundary

Everything above is the *mechanism*. The *policy* — what makes it a strategy rather than a
plumbing demo — is intentionally not published:

- the **strategy core** (how opportunities are scored, ranked, and selected),
- **sizing, bid/tip, timing, and risk parameters**,
- the **specific market structure edges** the engine targets,
- keys, endpoints, and deployment configuration.

The **primary route is orderflow-auction (OFA)** participation via latency-neutral
**batch auctions** — position won by a bid, not by the open-market speed race — with
shred-reactive racing retained as fallback. The engine is built around this route and
**ready to integrate with an auction**; it is documented here at the architecture level
only, and runs record-only until a pre-registered metric passes.
