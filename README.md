# Solana MEV Searcher

A low-latency, cross-DEX **backrun-arbitrage** searcher for Solana — built and operated in Rust. **Backrun-only by design: no front-running, no sandwiching.**

## What it does
Maintains a real-time price graph across **Raydium AMM v4, Orca Whirlpool, Pump / PumpSwap and Meteora DLMM**, detects the transient cross-DEX dislocations that user swaps create, and submits **revert-protected backruns** to arbitrage them. Also runs **liquidation detection** (Kamino klend + Scope, MarginFi), record-only.

## Engineering
- **Rust workspace** — core · DEX plugins · detection · executor · safety — with zero-copy on-chain decoders and **exact CLMM/DLMM quoters** (full swap-curve math, not x·y=k approximations).
- **Flashloan-funded** (Kamino, MarginFi); **legacy / v0 + ALT / v1 (SIMD-0385)** transactions with self-managed address-lookup tables.
- **Frankfurt co-location**; gRPC (LaserStream / Yellowstone) ingest; leader- and builder-aware landing.

## Execution & safety
A complete execution pipeline, exercised live on Solana mainnet: **detection → route verification → build → simulation → bundle submission → on-chain landing confirmation → realized-P&L accounting → loss limits.**
- **Jito-compatible and Jito-whitelisted** — multi-region bundle submission; plugs into Jito-protocol-compatible searcher networks by endpoint change.
- Atomic, **revert-dropping** bundles guarded by an on-chain profit check.
- **Kill switch, circuit breakers, and hourly / daily / lifetime P&L loss limits** in production.

## Discipline
Deterministic **capture-and-replay** testing and a **record-only, measurement-gated** rollout — nothing goes live until a pre-registered measurement passes. The focus throughout is on-chain decode/quote correctness, latency, and rigorous measurement before any capital is risked.

_Source is private; a scoped technical walkthrough is available to partners under NDA._
