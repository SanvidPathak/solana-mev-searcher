# Solana MEV Searcher

A low-latency cross-DEX **backrun-arbitrage** searcher for Solana, built and operated in Rust.

**What it does.** Maintains a real-time price graph across Raydium AMM v4, Orca
Whirlpool, Pump / PumpSwap and Meteora DLMM, detects the transient cross-DEX price
dislocations that user swaps create, and submits **revert-protected backruns** to
arbitrage them. Backrun-only by design — no front-running, no sandwiching.

**Stack & infrastructure.**

- Rust workspace (core · DEX plugins · detection · executor · safety), zero-copy
  on-chain decoders, exact CLMM/DLMM quoters.
- Flashloan-funded (Kamino, MarginFi); legacy / v0+ALT / v1 (SIMD-0385) transactions.
- Frankfurt co-location; Helius LaserStream gRPC ingest; builder-aware landing.
- Deterministic capture-and-replay testing; a record-only, measurement-gated rollout
  discipline — nothing goes live until a pre-registered measurement passes.

**Engineering focus.** On-chain decode/quote correctness, latency, and rigorous
measurement before any capital is risked.

_Source is private._
