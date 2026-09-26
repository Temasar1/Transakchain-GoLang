# Scaling Hydra: five mathematically grounded designs for a `hydra-node` fork

Companion to `HYDRA-LIGHT-DECENTRALIZATION.md` (which fixes the *trust* architecture: n = 2 heads,
hubs, HTLC routing, permissionless contest). This document attacks the *scaling* problem: how to
remove the resource, participant-count and on-chain-footprint limits of the current head protocol
without giving up the isomorphism or the n-of-n safety proofs, and where a deliberate compromise of
the state machine buys a large gain.

Everything is stated with explicit complexity, explicit cryptographic assumption, and the prior
work it rests on. Every place where the design touches this repository is named.

---

## 0. What "scale" means here, and what the current protocol costs

A head has n parties, a UTxO set U of size |U|, and a per-snapshot transaction delta of size |Δ|.

| Dimension | Current cost | Where it comes from |
|---|---|---|
| **S1** On-chain verification per close/contest/increment/decrement | **O(n)** Ed25519 verifications, O(n) datum bytes (`parties` list) | `verifySnapshotSignature` verifies each party's signature individually; `maximumNumberOfParties = 29` is the script-budget ceiling |
| **S2** Off-chain messages per snapshot | **O(n²)** total (`ReqSn` + n `AckSn`, each broadcast to all) | ADR 006 "network broadcasts all messages", etcd cluster per head |
| **S3** Per-party storage | **O(\|U\|)** full L2 UTxO set + full L1 chain follower | every party re-executes every tx (`applyTransactions` in `HeadLogic`) |
| **S4** Per-party compute per snapshot | **O(\|Δ\|)** ledger execution incl. Plutus scripts + n signature checks | isomorphism: every party is a full validator |
| **S5** Liveness | requires **n/n** parties online for every snapshot | `Certified = ∀ i → Signed i`; round-robin `isLeader` |
| **S6** L1 transactions per head lifecycle | ≥ 3 (init, ≥1 deposit, close, fanout) + contests | ADR 033 already removed commit/collectCom |
| **S7** Finality | 1 snapshot round (≈ one network RTT among n) | coordinated head |

S7 is the thing Hydra does well. S1, S3, S4, S5 are why "every participant runs a node" is not
scalable, and S1 is why n ≤ 29. The claim in the problem statement is correct: the protocol buys
fast finality with per-party space and compute, and the isomorphism is precisely what makes S3/S4
unavoidable for a *full* party.

### The one theorem you cannot design around

**Lemma 0.1 (unanimous safety ⇒ n/n liveness).** If a protocol guarantees that no state can be
finalized without party *i*'s signature (i's veto — this is what gives *i* trustless safety), then
no state can be finalized while *i* is offline. *Proof.* Immediate: finalization requires a
signature only *i* can produce (EUF-CMA). ∎

So S5 is not a bug to fix; it is the dual of trustless safety. Any design that "fixes" S5 for party
*i* has either (a) removed *i*'s veto (committee/threshold trust), or (b) made *i*'s participation
so cheap that being online is not a burden, or (c) made *i* not a party of the head that needs to
progress (hub-and-spoke). The right fork does (b) and (c) and refuses (a) for user funds. The five
ideas below are organised by which cost they remove; §6 combines them.

---

## 1. Idea 1 — Constant-size heads: BLS aggregate multisignatures with proofs of possession

**Removes:** S1 (O(n) → O(1) on-chain), the n ≤ 29 cap, O(n) datum size.

### Construction

- Party keys: BLS12-381, `pk_i = g₂^{sk_i}` (or G1, mirrored). Each party publishes a **proof of
  possession** `π_i = H_pop(pk_i)^{sk_i}` at head init.
- Aggregate key: `apk = ∏ pk_i` (n G2 additions, done once; store `apk` and `n` in the head datum).
- Snapshot signature: `σ_i = H(snapMsg)^{sk_i}`, aggregate `σ = ∏ σ_i`.
- On-chain check: `e(σ, g₂) = e(H(snapMsg), apk)` — **two pairings, independent of n**.

### Why it is sound

- Rogue-key attacks (an adversary choosing `pk_adv = pk*/∏ pk_i` to forge an aggregate alone) are
  blocked by PoP: Ristenpart & Yilek 2007 ("The Power of Proofs-of-Possession"); the exact scheme is
  **MSP-pop** of Boneh, Drijvers & Neven 2018 ("Compact Multi-Signatures for Smaller Blockchains"),
  proven unforgeable in the ROM under co-CDH on BLS12-381.
- The Agda security model **already abstracts the aggregate**: `aggKey`, `aggSigOf`, `msVfy`,
  and the postulate `aggSound : AggVerified → ∀ i → PartyVerified i`. MSP-pop instantiates
  `aggSound` exactly (a verifying aggregate under `apk` implies every constituent verified). No
  change to the safety proofs; only the postulate becomes a theorem of a concrete scheme.
- `Hydra.Tx.Crypto` literally says it "will change when we adopt aggregated multi-signatures
  including aggregate keys"; the BLS builtins (`bls12_381_millerLoop`, `bls12_381_finalVerify`,
  G1/G2 ops) are already used on-chain by the KZG accumulator (`Hydra.Contract.CRS`).

### Complexity after

| | before | after |
|---|---|---|
| on-chain sig verification | n · Ed25519 | 2 pairings |
| datum | O(n) `parties` | `apk` (96 B) + `n` + `hash(parties)` |
| off-chain AckSn aggregation | n messages carrying n sigs | leader aggregates; O(n) G1 adds, one 48 B σ |

The participant cap moves from L1 script budget to **liveness only** (Lemma 0.1). A head of 1 000
parties is *verifiable* on-chain; whether it is *live* depends on Idea 2/3 making participation cheap.

### Cost / caveat

Hydra keys become BLS keys (they are already separate from Cardano payment keys, so nothing on the
wallet side changes). Who is *authorised* to close/contest (`mustBeSignedByParticipant`, keyed on
participation tokens per party) must move to either permissionless contest (previous document §7.1)
or a Merkle proof of membership in `hash(parties)`.

**Fork touch points:** `hydra-tx/src/Hydra/Tx/Crypto.hs` (key type + `MultiSignature`),
`hydra-plutus/src/Hydra/Contract/Head.hs` (`verifySnapshotSignature`, datum),
`hydra-node/src/Hydra/HeadLogic.hs` (AckSn collection → aggregation), `Hydra.Chain.maximumNumberOfParties`,
`spec/src/Hydra/Protocol/Security.lagda.typ` (discharge `aggSound`).

---

## 2. Idea 2 — Recursive heads (head factories): use the isomorphism against itself

**Removes:** S6 for all but one head (L1 cost per user → O(1/b^d)), decouples S5 per subtree.

### Construction

The L2 ledger inside a head is Cardano's ledger, including Plutus. Therefore the **Hydra head
validator itself runs inside a head**. A "root" head between b hubs on L1 can host b child heads
whose "layer 1" is the root's L2 ledger; recursively, depth d gives up to `b^d` leaves, and **only
the root ever touches L1**. Opening, funding, closing and fanning out a child head are L2
transactions in the parent — instant, fee-free, with the parent's finality (S7).

This is the channel-factory construction of Burchert, Decker & Wattenhofer 2018 ("Scalable Funding
of Bitcoin Micropayment Channel Networks") and, specifically for Hydra, **Interhead Hydra**
(Jourenko, Larangeira & Tanaka 2021, "Interhead Hydra: Two Heads Are Better Than One"), which
constructs virtual heads across existing heads. The isomorphism makes it *free* here: no new
validator, the same `Head.hs` script deployed on the parent's L2.

### Soundness argument

Let `H_child` be a head whose base ledger is the L2 ledger of `H_parent`.

- **Safety** of `H_child` is the Hydra safety theorem with "L1" instantiated to `H_parent`'s ledger;
  the Agda proofs are parametric in the ledger (`applyTxs` is abstract), so they carry over.
- **Exit** requires that a close/contest/fanout of `H_child` can be *executed* on its base ledger.
  Two cases: (i) `H_parent` is live → the child's close is an ordinary parent L2 tx. (ii) `H_parent`
  closes → the child's head UTxO (script output with datum) is **fanned out to L1** and continues its
  life there, because the validator is the same script. Hence the child's guarantees reduce to the
  parent's guarantees plus the child's own.
- **Timeout nesting** (the only new inequality, same as Perun virtual channels, Dziembowski et al.
  2019): a child contestation period `CP_c` must be enforceable after a worst-case parent exit:
  `T_child_deadline ≥ T_parent_deadline + Δ_fanout + Δ_L1`. Validity intervals on L2 already derive
  from L1 time (ADR 020), so `mustBeWithinContestationPeriod` evaluates identically on both layers.

### What it buys, quantitatively

For N users organised as b-ary tree of depth d = log_b N: L1 transactions **O(1)** total, per-user
liveness dependency **b** (its leaf head), root parties **b** hubs. Combined with Idea 1 the root can
itself have hundreds of hubs.

### Caveats

- Minting inside a head is allowed but tokens minted and not burned block finalization
  (`known-issues.md`). A child head mints head/participation tokens at init and burns them at fanout —
  consistent — but an *abandoned* child (never closed) would block the parent's final fanout.
  Mitigation: parent-level rule that any party may force-close a child after its own deadline;
  partial fanout (`checkPartialFanout`) already lets unaffected outputs leave first.
- Liveness of a child's *exit* through a non-live parent depends on the parent closing; that is bounded
  by the parent's CP. This is the price of factories and is well understood.

**Fork touch points:** `hydra-node` must support running the head protocol against a *Hydra head as
chain backend* (a new `Hydra.Chain` implementation observing/posting to a parent head's API instead of
a Cardano node), plus the multi-head `NodeState` (today one `headState` per process).

---

## 3. Idea 3 — Stateless parties: commitments + proof-serving nodes (the space fix)

**Removes:** S3 (O(|U|) → O(own UTxOs)), most of S4 for light parties.

### Construction

The head already commits to the UTxO set with a **KZG polynomial accumulator** `η = g₁^{P(τ)}`,
`P(X) = ∏_{u∈U}(X − h(u))` (Kate, Zaverucha & Goldberg 2010; Nguyen 2005 bilinear accumulator),
with O(1)-size membership proofs verified by one pairing (`Hydra.Contract.CRS`). Today it is used
only at fanout. Idea 3 uses it **at every snapshot**:

1. A **full node** (hub or leader) computes `η_{s+1}` and, for each light party *i*, a membership
   proof `w_i` for *i*'s post-snapshot UTxOs (and a non-membership/spend witness for any UTxO of *i*
   consumed by a tx *i* signed).
2. Light party *i* verifies `e(w_i, g₂^{τ} · g₂^{−h(u)}) = e(η_{s+1}, g₂)` for each own UTxO: **O(k)
   pairings for k own UTxOs**, no ledger state, no Plutus execution.
3. Party *i* signs `snapMsg(…, η#_{s+1}, …)` only if every own UTxO is either present or was spent by
   a transaction *i* itself signed.

### Why the hub must serve proofs (and why that is fine)

Camacho & Hevia 2010 ("On the Impossibility of Batch Update for Cryptographic Accumulators") show
that no accumulator lets a party update its witness after m changes in less than Ω(m) work without
the full set. So a light party *cannot* maintain its own proof cheaply — but it does not need to:
the proof is **verified**, not trusted. A lying proof-server can only cause the light party to refuse
to sign (liveness), never to sign something false (safety). If updatable proofs are preferred over
served proofs, switch the set accumulator to an **aggregatable subvector commitment** with O(1)
proof updates (Tomescu et al. 2020, aSVC; Gorbunov et al. 2020, Pointproofs) — same BLS12-381
builtins, positions instead of set elements. A trusted-setup-free alternative is a hash-based
accumulator (Dryja 2019, Utreexo; Boneh, Bünz & Fisch 2019) with O(log |U|) proofs, at ~30× proof
size.

### Security statement (precise)

Let `L` be the light parties and `F` the full parties of a head.

- **Own-funds safety for i ∈ L** needs only *i*'s honesty and A1 (crypto): by step 3, every
  certified η contains *i*'s UTxOs unless *i* signed the spend; by KZG binding and fanout's
  membership check (`headIsFinalizedWith`), *i*'s UTxOs are fanned out.
- **Global ledger validity** (no inflation, scripts respected) needs **≥ 1 honest party in F**,
  because only full parties check `applyTxs`. This is the *one* new assumption Idea 3 introduces,
  and it is removed by Idea 4. Without Idea 4, deploy Idea 3 only where F contains the user's own
  hub *and* at least one independent full party (e.g. in a hub-interconnect head), or accept the
  1-of-|F| assumption explicitly.

**Fork touch points:** `Hydra.Tx.Accumulator` (per-snapshot proof generation, currently only at
fanout), `HeadLogic.onOpenNetworkReqSn` (a light-party branch that verifies proofs instead of
`applyTransactions`), a `ReqSn` extension carrying `(η_{s+1}, {w_i})`, and a new light client.

---

## 4. Idea 4 — Validity-proved snapshots: the sanctioned compromise of the state machine

**Removes:** the "≥ 1 honest full party" assumption of Idea 3; makes S4 O(1) for everyone; **and is
the only route past the 1-of-n bound for non-parties** (Theorem 2.1 of the previous document).

### Construction

The leader attaches to each `ReqSn` a succinct proof
`π_s : ∃ Δ. applyTxs(U_s, Δ) = U_{s+1} ∧ η_{s+1} = Acc(U_{s+1}) ∧ Δ ⊆ seen`, and parties (and,
optionally, the on-chain validator) verify `π_s` instead of re-executing Δ. With Groth16 (Groth 2016)
over BLS12-381 the verifier is 3 pairings + a small MSM — implementable with the existing Plutus
builtins; Plonk-style verifiers (Gabizon, Williamson & Ciobotaru 2019) cost more pairings but need
no per-circuit setup beyond a universal SRS.

### The compromise, stated honestly

Proving *arbitrary Plutus execution* inside a SNARK is today orders of magnitude too expensive for a
per-snapshot prover. The defensible compromise is a **two-tier ledger**:

- **Tier P (payment ledger):** UTxO transfers with Ed25519 witnesses, native assets, no scripts. Its
  transition function is a few thousand constraints per tx — provable in real time on commodity
  hardware. Light heads run Tier P.
- **Tier I (isomorphic ledger):** full Cardano/Plutus. Full heads run Tier I, exactly as today.

Tier P heads are *not* isomorphic; that is the trade. They are, however, exactly what "payments to
strangers cheaply and frictionlessly" needs, and a Tier P head can be a child (Idea 2) of a Tier I
head, so the isomorphic layer remains available where it is used (DApps, auctions) while the mass
payment layer is validity-proved.

### What changes in the trust model

If the **on-chain** close/contest/fanout also verifies `π` for the *chain* of snapshots since the last
on-chain checkpoint (recursive or folded proofs, e.g. Nova-style folding — Kothapalli, Setty &
Tzialla 2022), then a snapshot that verifies on-chain is *valid by construction*, not merely
*signed*. Then:

- non-parties are protected against **invalid** states even if all n parties collude;
- what remains is **data availability / censorship** (parties refuse to reveal state or to include a
  user's tx) — the standard rollup residue, mitigated by the exit game (contest with any newer proven
  state) and by the user's own signature on spends.

This is the point at which threshold or single-operator *progress* becomes safe with respect to
validity, and where Lemma 0.1's trade-off is genuinely sidestepped: safety no longer requires the
user's signature on every state, only on spends of the user's own UTxOs, which the proof enforces.

### Prover cost budget (order of magnitude, Tier P)

Groth16 proving ≈ tens of ms per transfer on a laptop-class machine; a snapshot of 100 transfers
≈ seconds. On-chain verification ≈ constant, comfortably inside Plutus budgets given the KZG
verification already fits. Universal-setup schemes add a trusted-SRS assumption identical in kind to
the one the KZG accumulator already carries (`KZGTrustedSetup.hs`).

**Fork touch points:** a `Hydra.Ledger.Payment` instance of `IsTx`/`Ledger` (Tier P), a circuit for
its transition, `ReqSn` extension carrying `π`, an optional `mustVerifyTransitionProof` conjunct in
`checkClose`/`checkContest`.

---

## 5. Idea 5 — Offline-tolerant closing with a bonded warden committee (liveness without losing the veto)

**Relaxes:** S5's *online-during-contest* requirement without touching the *online-to-sign*
requirement (which Lemma 0.1 says must stay).

### Construction

Brick (Avarikioti, Kogias, Wattenhofer & Zindros 2020) and the watchtower literature (Cerberus
channels, Outpost) separate two things Hydra currently conflates: the parties who *author* state (must
be n/n) and the parties who *attest to the latest sequence number* (can be a committee). Each
snapshot's `(headId, s, η#)` is sent to k wardens who each countersign it and post a **bond** on L1.
A close is contestable by *any* warden presenting a higher-numbered certified snapshot (needs the
permissionless-contest change). A warden who *signs* a sequence number and later fails to contest a
lower close is provably negligent — the two signatures are the fraud proof — and is slashed to the
victim.

### Soundness

- Wardens hold no keys of the parties: they cannot author state (Theorem 4.3 of the previous
  document); the veto is intact.
- Stale-close protection needs **any one** of the k wardens honest-and-live; a rational warden is
  live because the bond exceeds the largest bribe it could accept, which is bounded by the head's
  value (`mustPreserveHeadValue` makes the head value public).
- Negligence is *objectively provable on-chain* (signed sequence number s, later close at s' < s,
  no contest before deadline), so slashing does not require any subjective judgment.

This converts the *user* liveness requirement from "online once per CP" to "online only when
transacting". Combined with Idea 3 the user's device does nothing between payments.

**Fork touch points:** a warden countersignature in the snapshot flow (`AckSn` fan-out to wardens), a
bond/slash validator on L1 independent of `Head.hs`, and the permissionless-contest arm.

---

## 6. Which is "best"? A trilemma, then a recommended stack

**Proposition 6.1 (no perfect design).** No head protocol can simultaneously offer (i) trustless
per-party safety without honest-majority/committee assumptions, (ii) progress while an arbitrary
party is offline, and (iii) no validity proof of the state transition. *Proof.* (i) requires party
*i*'s veto over finalization (otherwise a coalition finalizes without *i*, i.e. trust in that
coalition). By Lemma 0.1 the veto forbids (ii). Dropping the veto while keeping (i) requires the
finalization mechanism to *verify* that *i*'s funds were not moved without *i*'s authorization —
which is a validity proof, contradicting (iii). ∎

So the choice is exactly one of:

| Give up | Resulting system | Our verdict |
|---|---|---|
| (ii) offline progress | Hydra as today, made cheap (Ideas 1, 2, 3, 5) | **Adopt for the isomorphic tier** |
| (iii) no validity proof | validity-proved head (Idea 4) | **Adopt for the payment tier** |
| (i) trustless safety | threshold/committee sidechain | **Reject for user funds** |

### Recommended fork ("Hydra Light")

1. **Idea 1** first — smallest change, largest unlock (removes n ≤ 29, O(1) on-chain), and the Agda
   model already anticipates it.
2. **Idea 2** — hubs form a root head; users' heads are children; L1 cost per user ≈ 0.
3. **Idea 3** — light parties verify KZG proofs instead of running a ledger; per-user storage
   O(own UTxOs). With user + hub in a 2-party child head, F = {hub} and the user's own-funds safety
   is unconditional; global validity for Tier I heads relies on ≥ 1 honest full party, which in a
   hub-interconnect head is the other hubs.
4. **Idea 5** — wardens with bonds so users need not watch L1.
5. **Idea 4** — Tier P validity-proved ledger for the mass-payment leaves; later, on-chain proof
   verification to lift the 1-of-n bound for non-parties.

Resulting per-user cost: sign one BLS signature and verify k pairings per own transaction; store a
few hundred bytes; be online only when paying or being paid. Resulting L1 cost: O(1) per hub tree,
O(1)-size transactions regardless of n. Finality: unchanged for Tier I (one snapshot round);
one snapshot round plus proving time for Tier P.

---

## 7. Assumptions ledger (complete)

| Tag | Assumption | Used by |
|---|---|---|
| C1 | Ed25519 / BLS12-381 co-CDH in the ROM (EUF-CMA; MSP-pop unforgeability) | all; Idea 1 |
| C2 | q-SDH on BLS12-381 and an honest KZG/SRS ceremony (τ destroyed) | Idea 3, Idea 4 (universal SRS), existing fanout |
| C3 | Knowledge soundness of the chosen SNARK (Groth16 per-circuit CRS or Plonk universal SRS) | Idea 4 |
| L1 | Cardano safety beyond the configured rollback depth | all |
| L2 | Cardano inclusion within Δ_L1 for honest submitters; CP sized accordingly, nested per Idea 2 | all exits |
| H1 | Each user's own client signs only after verifying its own proofs (`signHonest` premises) | Ideas 3, 4 |
| H2 | ≥ 1 honest full party per Tier I head with light parties | Idea 3 without Idea 4 |
| E1 | Warden bond > maximal bribe; at least one warden live | Idea 5 |

Each is either standard (C1, C3, L1, L2), already present in the codebase (C2), the user's own
discipline (H1), or explicitly scoped and removable (H2 by Idea 4; E1 by the user watching L1
themselves).

---

## 8. What remains unsolved (so nobody over-claims)

- **Data availability** in validity-proved heads: a proof shows the state is valid, not that the
  user can *learn* it. Exit games need the user (or warden) to hold the latest proven state; this is
  the same residue as every rollup.
- **Mass exit** of a large hub tree within one CP is bounded by L1 throughput; CP and tree size must
  be co-designed.
- **Prover cost for Tier I** (Plutus-in-SNARK) is not solved by anyone at production scale; Idea 4 is
  restricted to Tier P for that reason.
- **Liveness** is still not machine-checked in the Agda spec (`postulate Liveness`). Ideas 1–3 do not
  change the liveness argument; Idea 5 changes only the *contest* liveness and needs its own
  (simple) proof.
- **Formal composition** of nested heads (Idea 2) is argued parametrically over the abstract ledger
  in the Agda model; a mechanised proof of the timeout-nesting inequality is future work.

---

## 9. References (by idea)

- Hydra: Chakravarty, Coretti, Fitzi, Gaži, Kant, Kiayias, Russell — *Hydra: Fast Isomorphic State Channels*, FC 2021.
- Idea 1: Boneh, Drijvers, Neven — *Compact Multi-Signatures for Smaller Blockchains*, ASIACRYPT 2018. Ristenpart, Yilek — *The Power of Proofs-of-Possession*, EUROCRYPT 2007. Bellare, Neven — *Multi-Signatures in the Plain Public-Key Model*, CCS 2006.
- Idea 2: Burchert, Decker, Wattenhofer — *Scalable Funding of Bitcoin Micropayment Channel Networks*, 2018. Jourenko, Larangeira, Tanaka — *Interhead Hydra: Two Heads Are Better Than One*, 2021. Dziembowski, Eckey, Faust, Malinowski — *Perun: Virtual Payment Hubs over Cryptocurrencies*, S&P 2019.
- Idea 3: Kate, Zaverucha, Goldberg — *Constant-Size Commitments to Polynomials and Their Applications*, ASIACRYPT 2010. Nguyen — *Accumulators from Bilinear Pairings*, CT-RSA 2005. Camacho, Hevia — *On the Impossibility of Batch Update for Cryptographic Accumulators*, LATINCRYPT 2010. Tomescu et al. — *Aggregatable Subvector Commitments for Stateless Cryptocurrencies*, SCN 2020. Gorbunov, Reyzin, Wee, Zhang — *Pointproofs*, CCS 2020. Dryja — *Utreexo*, 2019. Boneh, Bünz, Fisch — *Batching Techniques for Accumulators*, CRYPTO 2019.
- Idea 4: Groth — *On the Size of Pairing-Based Non-Interactive Arguments*, EUROCRYPT 2016. Gabizon, Williamson, Ciobotaru — *PLONK*, 2019. Kothapalli, Setty, Tzialla — *Nova: Recursive Zero-Knowledge Arguments from Folding Schemes*, CRYPTO 2022. Kalodner et al. — *Arbitrum*, USENIX Security 2018 (fraud-proof alternative).
- Idea 5: Avarikioti, Kogias, Wattenhofer, Zindros — *Brick: Asynchronous Incentive-Compatible Payment Channels*, FC 2021. Decker, Russell, Osuntokun — *eltoo*, 2018 (symmetric update, which Hydra's monotone snapshot numbering already mirrors). Aumayr et al. — *Sleepy Channels*, CCS 2022.
