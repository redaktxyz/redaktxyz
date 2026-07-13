<div align="center">

# ▪ REDAKT

### Mask before it leaves. Prove it on-chain.

**Every prompt you send to an AI provider leaks names, emails, API keys, medical notes, client data.**
REDAKT is a drop-in privacy proxy: swap **one baseURL**, and PII is masked before your prompt reaches the model, then re-hydrated in the reply. Every request mints a **Redaction Certificate**, Merkle-batched and anchored to Solana, publicly verifiable forever.

[![Chain](https://img.shields.io/badge/chain-Solana-14F195?style=flat-square&logo=solana&logoColor=white)](https://solana.com)
[![Launch](https://img.shields.io/badge/launch-PumpFun%20fair%20launch-1DA1F2?style=flat-square)](https://pump.fun)
[![Team allocation](https://img.shields.io/badge/team%20allocation-0%25-success?style=flat-square)](#-token-at-a-glance)
[![Relay](https://img.shields.io/badge/relay-open%20source-brightgreen?style=flat-square&logo=github)](https://github.com/redaktxyz)
[![Overhead](https://img.shields.io/badge/scrub%20overhead-p95%20%3C%2040ms-blueviolet?style=flat-square)](#-tech-stack)
[![License](https://img.shields.io/badge/license-MIT-lightgrey?style=flat-square)](LICENSE)

**[redakt.xyz](https://redakt.xyz)** · **[Playground](https://redakt.xyz)** · **[Cert Explorer](https://redakt.xyz/certs)** · **[Burn Dashboard](https://redakt.xyz/burn)** · **[GitHub](https://github.com/redaktxyz)** · **[X](https://x.com/redaktxyz)** · **[Telegram](https://t.me/redaktxyz)**

</div>

---

## What is REDAKT

Hundreds of millions of people paste sensitive data into AI chat boxes and API calls every day. Names, national IDs, bank details, medical notes, `.env` contents, client lists, unreleased specs. All of it lands on third-party model provider servers where it can be logged, retained, subpoenaed, or breached. *"We don't train on your data"* is a policy promise. It is not a guarantee, and it has never been provable.

REDAKT starts from two observations. Developers will not rewrite an application for privacy, but they will swap one `baseURL`. And a privacy claim nobody can verify is worth nothing. So privacy became a proxy rather than an SDK rewrite, and every scrubbed request mints a certificate that anyone can check against Solana.

- **One line of code.** Standard chat-completions schema in, standard chat-completions schema out. Point your client at the relay and you are done.
- **Masked before it leaves.** The Scrubber catches entities on the way out; the response is re-hydrated on the way back, so your app still gets a normal reply.
- **Proof, not policy.** Every certified request is Merkle-batched and anchored on Solana. Verify any request, in your own browser, against a root we cannot rewrite.
- **No-log by architecture.** The relay is stateless and open-source. There is no database write path for prompt content, and CI fails the build if a log statement ever names one.
- **Privacy is never for sale.** The Scrubber runs identically for the free tier and the largest holder. $REDAKT buys capacity, retention, priority and identity. It never buys better masking.

---

## Core Mechanics

| Mechanic | What it does | Why it matters |
|---|---|---|
| **The Scrubber** | Tier 1 deterministic (regex, Luhn, entropy — ~0ms) · Tier 2 statistical NER (~15ms) · Tier 3 hashed custom dictionaries | PII never reaches the model provider |
| **The Vault** | Holds `PERSON_1 → "David Miller"` in relay memory only, wiped after re-hydration | The mapping must exist to re-hydrate, and it must die |
| **Redaction Certificate** | `prompt_hash`, `masked_hash`, entity count + types, mode, relay signature | Proof a request was scrubbed, without revealing the prompt |
| **Merkle Anchor** | Certs batch for 10 min or 1,000 certs → root → Solana memo tx | One transaction per batch, so cost per cert is dust |
| **Leak Score** | `Σ entity_weight × confidence` → CLEAN / LEAKY / CRITICAL | The playground scares you with your own production data |
| **Usage → Burn** | 50% of $REDAKT fees burned at settlement · 20% of SOL revenue to buyback + burn | Supply shrinks against verifiable API usage |

---

## How It Works

```
  YOUR APP                                                    MODEL PROVIDER
     │                                                              ▲
     │  baseURL: https://redakt.xyz/api/v1   ← the only line you change
     ▼                                                              │
┌─────────────────────────────── REDAKT RELAY ──────────────────────┼────────┐
│                                (open source, stateless)           │        │
│                                                                   │        │
│   ①  THE SCRUBBER            ②  THE VAULT              ③  FORWARD │        │
│   "I'm David Miller,          PERSON_1 → David Miller    masked  ─┘        │
│    dave@corp.com,             EMAIL_1  → dave@corp.com   prompt            │
│    key sk-live-4f9…"          SECRET_1 → sk-live-4f9…    only              │
│           │                        │                                       │
│           ▼                        │  in memory. never on disk.            │
│   "I'm PERSON_1,                   │  wiped after ④.                       │
│    EMAIL_1, key SECRET_1"          │                                       │
│                                    ▼                                       │
│   ⑤  CERT EMITTER            ④  RE-HYDRATOR  ◄── normal model response     │
│   { prompt_hash, masked_hash,    PERSON_1 → "David Miller"                 │
│     entity_count: 3, types,      your app gets a normal reply,             │
│     mode, ts, relay_sig }        as if REDAKT were not there               │
│           │                                                                │
└───────────┼────────────────────────────────────────────────────────────────┘
            ▼
     ⑥  MERKLE BATCH  ──►  ⑦  SOLANA ANCHOR  ──►  ⑧  ANYONE VERIFIES
        10 min or             one memo tx              redakt.xyz/certs
        1,000 certs           per batch                recompute the path,
                                                       compare to the root.
                                                       No trust required.
```

**What a certificate proves:** at time T, a request whose original hashed to H was processed with N entities masked.
**What it does not prove:** detection perfection. The Scrubber is best-effort, and we say so everywhere. Overclaiming is the fastest way to die in this category.

---

## Repository Layout

Five repos. Each is independently useful, and the boundary between them is the trust boundary.

| Repo | Role | Contains |
|---|---|---|
| **[redakt-substrate](documents/redakt-substrate/)** | The relay | The open-source scrubbing proxy: Scrubber (3 tiers), Vault, upstream forwarder, re-hydrator, cert emitter, Leak Score. Ships the CI no-log linter that fails the build if any log statement names `prompt`/`content`/`body`. |
| **[redakt-dossier](documents/redakt-dossier/)** | The argument | `WHITEPAPER.md` and `THREAT_MODEL.md` (STRIDE, 25 numbered threats, residual-risk section stating plainly what relay mode cannot protect against). |
| **[redakt-conduit](documents/redakt-conduit/)** | The developer surface | Full API reference (`ENDPOINTS.md`) and the TypeScript SDK. `examples/basic-usage.ts` sends an identical request body down both paths so the one-line promise is provable, not asserted. |
| **[redakt-playground](documents/redakt-playground/)** | The public face | Next.js app: the landing page whose hero **is** the live playground, the Cert Explorer with in-browser Merkle verification, the burn dashboard, the developer dashboard. Detection runs client-side in WASM: pasted text never leaves the browser. |
| **[redakt-pipeline](documents/redakt-pipeline/)** | The proof machine | Everything after the relay signs a cert: batcher, Merkle tree, Solana anchoring, explorer indexer, settlement + buyback-burn jobs. Plus `RUNBOOK.md` and the metadata-only `SCHEMAS.md`. |

**Marketing collateral** lives outside `documents/`:

| Folder | Contents |
|---|---|
| **[Article/](Article/)** | Long-form X Articles: the launch piece, the mechanics deep-dive, the roadmap and community piece. |
| **[Caption/](Caption/)** | 29-caption campaign: engagement hook, pinned intro, vision, technical deep-dives, trust and traction, roadmap, and post-launch announcements. |

> **A note on the certificate format.** The canonical preimage is a pipe-delimited field list, deliberately not JSON, because JSON key order is an implementation detail and two honest verifiers could each declare the other's valid certificate a forgery. Leaves are domain-tagged `0x00`, internal nodes `0x01`, and odd nodes are **promoted, not duplicated**. All four implementations (relay, pipeline, SDK, browser) must produce identical bytes. Change one and every certificate ever anchored stops verifying.

---

## Tech Stack

| Layer | Choice | Note |
|---|---|---|
| Relay | TypeScript, stateless | Horizontally scalable by design. Open-source, because the absent code *is* the audit. |
| Detection | Deterministic pattern engine + compact in-process NER | Target p95 overhead **< 40ms** |
| Playground scan | Same detection compiled to **WASM** | Runs in-browser. Nothing pasted ever leaves. |
| Frontend | Next.js + Tailwind + Framer Motion | The hero is the product, not a mock |
| Database | Managed encrypted Postgres | **Metadata only.** No write path for content exists. |
| Chain | Solana mainnet | Anchors, burns, payments |
| Anchoring | Merkle batch → memo transaction | No custom program, no upgrade authority, nothing anyone can rewrite |
| Secrets | Per-user envelope encryption | BYOK upstream keys, used only to forward |

---

## Token at a Glance

| | |
|---|---|
| **Ticker** | `$REDAKT` |
| **Chain** | Solana |
| **Total supply** | 1,000,000,000 |
| **Launch** | PumpFun fair launch — bonding curve → AMM graduation |
| **Team allocation** | **0%** |
| **Pre-sale** | **None** |
| **Creator fee** | 100% routed to buyback + burn during launch phase |

**Holder tiers** — capacity, retention, priority, identity:

| Tier | Threshold | Unlocks |
|---|---|---|
| **Cleartext** | 0% | Playground + BYOK free tier (100 req/day), 30-day cert retention |
| **Masked** | ≥ 1% (10M) | 2x rate limits, 1-year retention, 5 dictionaries, branded Privacy Report |
| **Sealed** | ≥ 2% (20M) | 5x rate limits + priority routing, permanent retention, unlimited dictionaries, Edge mode beta |
| **Blackout** | ≥ 5% (50M) | 10x rate limits, dedicated relay lane, namespace fee waived, governance, first relay-operator slots |

> **Doctrine:** holding $REDAKT **never** buys better privacy. The Scrubber runs identically for every request, on every tier, for every user. The token buys capacity, retention, priority and identity. Privacy quality is not for sale, and that is the spine of the brand. In the relay source, the caller's tier is never passed to a scrub call — the absence is the proof.

**Supply reduction:** 50% of every fee paid in $REDAKT is burned at settlement · 20% of SOL revenue auto-buyback + burn · verified org namespace registration burned 100% · every burn is a Solana transaction linked from [redakt.xyz/burn](https://redakt.xyz/burn).

---

## Roadmap

| Phase | Window | Ships |
|---|---|---|
| **P1 — MVP Relay** | 4 weeks | Landing page with a *working* playground, relay core (Scrubber + Vault + re-hydration), BYOK free tier, cert pipeline on devnet, Cert Explorer v1, open-source repo public |
| **P2 — Token + Payments** | 4 weeks | Credits (SOL top-ups), $REDAKT fair launch on PumpFun, +20% bonus, 50% settlement burn, `/burn` dashboard, anchoring moves **devnet → mainnet**, org namespaces, custom dictionaries |
| **P3 — Edge Mode** | 8 weeks | npm SDK with on-device scrubbing (the mapping never leaves the client), browser extension that masks inside any AI chat UI, edge-mode certs, team plans, signed audit exports |
| **P4 — Relay Network** | 6-12 months | Third-party relay operators staking 25,000,000 $REDAKT, slashing for failed no-log audits, confidential-compute lanes, geo-routing, governance |

The path from *"verify the code, trust the deployment"* (relay mode) to *"trust nobody, verify everything"* (edge mode) is not a footnote. It is the product story.

---

## ⚠️ Scam Warning

- **There is no presale.** $REDAKT launches on PumpFun as a 100% fair launch. Anyone offering you an allocation, a whitelist, or an early buy is stealing from you.
- **We will never DM you first.** Not for support, not for a "verification", not for a wallet connect.
- **Verify the contract address** against [redakt.xyz](https://redakt.xyz) and the official [X account](https://x.com/redaktxyz) before you touch anything. Tokens with our name and a different CA are fake.
- **We will never ask for your seed phrase, your private key, or your upstream provider key** in a DM. The relay only ever receives an upstream key through the encrypted BYOK flow in the dashboard.
- The only official links are the ones in this README.

---

## Quick Start

The relay and the SDK are both pullable and runnable. The fastest path from zero to a masked request:

```bash
# 1. Run the open-source relay locally
git clone https://github.com/redaktxyz/redakt-substrate
cd redakt-substrate && npm install
cp .env.example .env          # set RELAY_SIGNING_KEY + upstream config
npm run dev                   # relay on :8080

# 2. Point any existing chat-completions client at it
git clone https://github.com/redaktxyz/redakt-conduit
cd redakt-conduit && npm install
npm run example:basic         # same request body, two paths, one masked
```

Already have an app? Then it is genuinely one line:

```diff
  const client = new OpenAICompatibleClient({
-   baseURL: "https://api.your-provider.com/v1",
+   baseURL: "https://redakt.xyz/api/v1",
+   defaultHeaders: { "x-redakt-key": "rk_live_…" },
    apiKey: process.env.UPSTREAM_API_KEY,
  });
```

Your request schema does not change. Your response schema does not change. What changes is what the model provider gets to see.

---

<div align="center">

**REDAKT** · [redakt.xyz](https://redakt.xyz) · [@redaktxyz](https://x.com/redaktxyz) · [t.me/redaktxyz](https://t.me/redaktxyz)

</div>

### Disclaimer

This repository is for informational purposes only and does not constitute financial, legal, or investment advice. $REDAKT is a utility token associated with the REDAKT platform and should not be considered an investment vehicle. Cryptocurrency involves substantial risk including the potential loss of all funds.

REDAKT's detection engine operates on a best-effort basis. Redaction Certificates prove that a request was processed and how many entities were masked. They do not guarantee that every sensitive element in any given input was detected.

REDAKT is not a substitute for legal compliance advice, a data processing agreement, or regulatory counsel. Users remain responsible for the data they transmit and for their agreements with upstream model providers. Always perform your own research.

Built on Solana.
