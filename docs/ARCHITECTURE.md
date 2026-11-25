# Architecture — Solana MEV Searcher

High-level design (HLD) and low-level design (LLD) for a low-latency **Solana MEV /
cross-DEX backrun-arbitrage** engine in **Rust**. The strategy core and all tuning parameters are
**abstracted** on purpose (see [Abstraction boundary](#abstraction-boundary)); everything
shown is the systems engineering that makes a competitive **Solana searcher** correct and
fast.

- [1. High-level design](#1-high-level-design)
- [2. Low-level design](#2-low-level-design)
  - [2.1 Ingestion &amp; live state](#21-ingestion--live-state)
  - [2.2 Opportunity engine](#22-opportunity-engine)
  - [2.3 Execution &amp; landing](#23-execution--landing)
- [3. Abstraction boundary](#3-abstraction-boundary)

---

## 1. High-level design

Six subsystems on one data path: market data in, verified opportunity out, every send
gated by risk, every outcome measured and fed back.

```mermaid
flowchart TB
  subgraph SRC["① Market Data"]
    G["Geyser gRPC"]
    W["WebSocket RPC<br/>failover"]
    SH["Shred stream<br/>raw Turbine UDP"]
  end
  subgraph ING["② Ingestion &amp; Live State"]
    FEC["Shred decoder<br/>Reed–Solomon FEC"]
    MUX["Multiplexer<br/>dedup · order · coverage"]
    PG["Price graph"]
    DS["Deep-state cache<br/>exact tick / bin arrays"]
  end
  subgraph DET["③ Opportunity Engine"]
    PS["Pairwise scan"]
    BF["Negative-cycle search"]
    QQ["Exact quoters<br/>CLMM / DLMM"]
    SC{{"Strategy core<br/>(abstracted)"}}
  end
  subgraph EXE["④ Execution &amp; Landing"]
    TB["Tx builder<br/>legacy · v0+ALT · v1"]
    SG["Simulation gate"]
    CAP["Flashloan capital"]
    LD["Multi-path landing"]
  end
  subgraph SAF["⑤ Risk &amp; Safety"]
    CB["Breakers"]
    PL["P&amp;L limits"]
    KS["Kill switch"]
  end
  subgraph RSR["⑥ Measurement · offline"]
    CEN["Census"]
    RA["Race audit"]
    LT["Shred-lead"]
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

| # | Subsystem | Responsibility |
|---|---|---|
| ① | Market data | Redundant feeds: Geyser gRPC (primary), WebSocket RPC (failover), raw shreds (earliest possible sight of a block). |
| ② | Ingestion &amp; state | FEC recovery, dedup, strict per-account ordering, coverage/repair, into a concurrent price graph and an exact deep-state cache. |
| ③ | Opportunity engine | Pairwise and multi-hop search over **exact** quotes; candidate scoring by the abstracted strategy core. |
| ④ | Execution &amp; landing | Build → simulate → land, with flashloan capital and leader-aware multi-path submission. |
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
  IN["Changed pool / pair"] --> MEMO{Pair memo<br/>unchanged?}
  MEMO -->|hit| RE["Re-emit last result"]
  MEMO -->|miss| SCR["Screen quote<br/>concave · per leg"]
  SCR --> PR{Prune<br/>hopeless?}
  PR -->|yes| DROP["Skip"]
  PR -->|no| SZ["Convex size search<br/>+ exact re-quote"]
  SZ --> EX["Exact deep quote<br/>DLMM bin-walk /<br/>Orca tick-array loop"]
  EX --> BUD{Account-key<br/>budget ≤ 64?}
  BUD -->|no| DROP
  BUD -->|yes| NET["Net-SOL model<br/>fees · tip · flashloan · rent"]
  NET --> SC{{"Strategy scoring<br/>(abstracted)"}}
  SC --> OPP["Verified opportunity"]

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
outcome classified so the breakers and limits see the truth.

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
  G2 -->|pass| FAN["Multi-region fan-out<br/>Jito · Sender · bundle proxies<br/>leader-aware routing"]
  FAN --> CFM["Landing by signature<br/>confirmed on chain"]
  CFM --> PNL["Realized P&amp;L<br/>native + WSOL delta"]
  PNL --> LIM["P&amp;L limits · kill switch"]
  CL --> BRK["Circuit breakers"]

  classDef exe fill:#fff4e6,stroke:#e8590c,color:#3d1a05;
  classDef saf fill:#ffe9ec,stroke:#e03131,color:#4d0a12;
  class VER,BLD,CU,FAN,CFM,PNL exe;
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
- **sizing, tip, timing, and risk parameters**,
- the **specific market structure edges** the engine targets,
- keys, endpoints, and deployment configuration.

The forward direction of this project is **orderflow-auction (OFA)** participation via
latency-neutral **batch auctions**, which replaces the open-market speed race with a
bid — a design choice, documented here at the architecture level only.
