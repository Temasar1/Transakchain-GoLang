# TransakChain — 3-Month Research Plan

---

## Team Roles

| Role | Background | Primary Focus |
|---|---|---|
| Protocol Lead | Advanced blockchain | Architecture calls, mentoring, ZK/consensus design |
| Protocol Dev A | TS/JS | MMR + core data structures |
| Protocol Dev B | TS/JS | SMT + nullifier set |
| Research Writer | Strong writing, some tech | Whitepaper drafting, literature review |
| Tooling Dev *(if 5th)* | TS/JS | DA mock layer, query tooling, tests |

---

## Week 0 — Prerequisites (before sprint starts)

| Topic | Resource | Owner |
|---|---|---|
| Cardano eUTxO model | Plutus Pioneer docs | Everyone |
| Hashing & Merkle trees | Any standard crypto primer | Everyone |
| MMR & SMT theory | Peter Todd's MMR writeup, Vitalik's SMT post | Devs |
| ZK basics (concept only) | Any "ZK explained" intro talk | Everyone |
| BFT consensus basics | HotStuff paper intro | Lead + 1 dev |

Lead runs **2× 1-hour knowledge transfer sessions** this week.

---

## Month 1 — Foundations & Prior Art

| Week | Goal | Output |
|---|---|---|
| 1 | Cardano eUTxO + Plutus/Aiken hands-on | Working testnet validator example |
| 2 | MMR/SMT theory deep dive | Internal notes doc |
| 3 | ZK + consensus fundamentals | Internal notes doc |
| 4 | Review prior art: Validium, Plasma, Celestia, Mina | Comparison table |

---

## Month 2 — Prototyping

| Week | Goal | Output |
|---|---|---|
| 5 | MMR implementation in TypeScript | Tested MMR library |
| 6 | SMT + nullifier set in TypeScript | Tested SMT library |
| 7 | Self-indexing datum schema + mock DA layer | Working mock node |
| 8 | eUTxO validity checks (no ZK yet) | End-to-end local demo |

---

## Month 3 — Draft Whitepaper

| Week | Goal | Output |
|---|---|---|
| 9 | Finalize architecture decisions + diagrams | Diagram set |
| 10 | Draft Sections 1–6 (motivation → ledger state) | Whitepaper v0.1 |
| 11 | Draft Sections 7–11 (validity → sharding) | Whitepaper v0.2 |
| 12 | Internal review, revise, polish | **Whitepaper v1.0** |

---

## Weekly Cadence

- **Mon** — planning/sync (30 min)
- **Wed** — Lead office hours / unblock (1 hr)
- **Fri** — demo + writeup (1 hr)

---

## End-of-Month-3 Success Criteria

- [ ] Working MMR + SMT prototypes in TypeScript
- [ ] Local demo: tx submit → batch → mock L1 commit → query
- [ ] Draft whitepaper v1.0 (all 11 sections from research doc)
- [ ] Open research questions logged for Month 4+

---

## Key Risk

Single point of knowledge bottleneck (Lead). Mitigate by pairing the Lead with a different dev each week rather than working solo — knowledge must spread, not just output.
