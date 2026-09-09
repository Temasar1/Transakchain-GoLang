# Decentralizing Hydra: light nodes, payments to strangers, and what can actually be proven

Status: research findings / thesis. Grounded in the protocol as implemented in this repository
(`hydra-plutus/src/Hydra/Contract/Head.hs`, `hydra-node/src/Hydra/HeadLogic.hs`,
`spec/src/Hydra/Protocol/Security.lagda.typ`). Every claim below is either (a) a theorem already
machine-checked in the Agda spec, (b) a corollary of a concrete validator check quoted by name, or
(c) explicitly marked as an assumption.

---

## 0. Honest framing before anything else

The request asks for a design that "can never be compromised". No security analysis can deliver that
statement, and one that claims to is wrong by construction. What *can* be delivered, and what this
document delivers, is stronger in practice:

1. A precise, enumerated list of assumptions.
2. Theorems of the form "**if** all assumptions hold **then** no adversary, regardless of strategy,
   can violate property P", with proofs or proof sketches that reduce to those assumptions.
3. **Tightness / impossibility results**: for each assumption, an explicit attack showing that
   dropping it makes P false. This is what tells you the design is not over-trusting anyone: each
   trust point is *necessary*, not just convenient.

Anything outside that envelope (cryptographic breaks, Cardano L1 halting, implementation bugs) is a
residual risk that is named in §8 rather than hidden.

---

## 1. The problem, stated precisely

Hydra Head today (coordinated head protocol):

| Fact | Where it is enforced |
|---|---|
| A head has a fixed party set `parties` chosen at `Init`. | `HeadParameters`, participation tokens minted at init |
| A snapshot is confirmed only when **every** party signs it (n-of-n). | Agda `Certified sys snap = ∀ i → Signed sys i snap`; on-chain `verifySnapshotSignature` requires `length parties == length sigs` and verifies **each** Ed25519 signature |
| Only a party may `close` or `contest`. | `mustBeSignedByParticipant` (signer's key hash must be a participation-token name in the inputs) |
| Each party may contest at most once; each contest pushes the deadline by one contestation period until all have contested. | `checkSignedParticipantContestOnlyOnce`, `mustPushDeadline` |
| A contest must present a strictly newer multisigned snapshot. | `mustBeNewer` (`snapshotNumber' > snapshotNumber`) |
| Max 29 parties (script budget for n Ed25519 verifications in one tx). | `Hydra.Chain.maximumNumberOfParties = 29` |
| Snapshot leader rotates round-robin. | `HeadLogic.isLeader` |
| Peer topology is static and identical on all nodes (etcd cluster). | `docs/docs/known-issues.md`, ADR 032 |
| A `hydra-node` needs a Cardano chain backend (node socket or Blockfrost) plus an embedded etcd. | `Hydra.Options` |

Consequences for the two goals in the problem statement:

- **Paying a stranger.** A stranger is, by definition, not in `parties`. A non-party has *no*
  on-chain recourse whatsoever: they cannot close, cannot contest, and the validator never looks at
  anything but the n signatures. So a non-party's funds inside a head are safe **iff at least one
  party is honest and online during the contestation period**. That is the "delegated head" trust
  model already documented in `docs/topologies/delegated-head`.
- **Light nodes.** Being a party means being needed for every snapshot (liveness) and running the
  full node stack (resources). Neither is acceptable for a phone.

So the naive reading of "decentralize the head" (make one large, open, dynamic-membership head) runs
straight into the following impossibility.

---

## 2. Impossibility results (why the obvious designs fail)

### Theorem 2.1 (Non-parties have exactly 1-of-n safety; this is tight)

*Statement.* Under the current validator, for any UTxO set U\* and any head with party set P, if all
n parties in P collude they can produce a valid `Close`/`Contest`/`Fanout` sequence that fans out
U\*. Consequently the safety of anyone who is not in P reduces exactly to "at least one member of P
is honest and contests in time". No mechanism available to a non-party can improve on this without
changing what the validator verifies.

*Proof.* `checkClose`, `checkContest` and the fanout checks (`headIsFinalizedWith`,
`checkPartialFanout`, `checkFinalPartialFanout`) verify: the multisignature over
`(headId, version, snapshotNumber, η#, δ#, κ#)`, value preservation, deadlines, the KZG
accumulator binding `mustBindAccumulatorCommitment`, and KZG membership of the fanned-out outputs in
that accumulator. None of these checks consults ledger validity
of the transactions that produced the state; that is deliberately off-chain (`Soundness` in the Agda
spec is derived from the *honest-signing discipline*, `signHonest`, not from the validator). Hence
n colluding signers can sign `snapMsg` for an arbitrary η committing to U\* and the validator
accepts it. The Agda `soundness` theorem is stated with the hypothesis `lookup (honest sys) h ≡ true`
for **some** h — the proofs are vacuous when `honest = ∅`. ∎

*Tightness.* With one honest party h, `soundness` and `completeness` hold for h's confirmed set, and
h can contest (`mustBeNewer`) with the latest certified snapshot, so 1-of-n is achieved. It cannot be
improved for non-parties: the only on-chain input is signatures. The only way past this is to have
the validator verify *state transitions* (fraud or validity proofs), i.e., to turn the head into a
rollup — which is a different protocol with a different cost profile, not a Hydra head.

### Theorem 2.2 (Threshold signing destroys per-party safety)

*Statement.* Replacing n-of-n with k-of-n (k < n) breaks `Consistency` for honest parties outside a
colluding k-set.

*Proof.* The machine-checked `agree` (L1) and `cert-nest` (L2) lemmas rest on the fact that a
certified snapshot carries *every* party's signature, hence the honest party's own, hence obeys the
honest numbering guard `sigDedup` (one signature per number). With k-of-n, k corrupt parties can
certify snapshot `s'` with the same number as the honest party's confirmed `s` but conflicting
contents; L1 fails, and `consistency` (which needs `confirmedTxs i ⊆ confirmedTxs j ∨ …`) is
false. The honest party's confirmed payments can be rolled back on L1. ∎

*Interpretation.* k-of-n buys liveness under n−k faults at the price of converting "my signature is
required" into "an honest majority of a committee protects me". That is an honest-majority sidechain.
It does not give finality "close to" the state machine; it gives a strictly weaker kind. We reject
it.

### Theorem 2.3 (No large permissionless head)

n ≤ 29 today because the close/contest transaction verifies n Ed25519 signatures on-chain. Even with
aggregate signatures (BLS is available on Plutus and the Agda spec already models `aggSound`), a
head with thousands of parties cannot make progress: n-of-n liveness requires every party online for
every snapshot, and the probability that all N phones are online in a snapshot window is
\(p^N \to 0\). So "one big head that everyone joins" is dead both on safety (2.1/2.2) and liveness.

**Conclusion of §2.** The decentralization must come from *many small heads* whose membership gives
every user their own veto, connected by a routing layer. This is exactly the structure Lightning
arrived at, and Hydra's primitives make it cleaner (full Cardano ledger in-head, KZG accumulator for
O(1) inclusion proofs, incremental commit/decommit for liquidity management).

---

## 3. The proposed architecture: Hydra Light

### 3.1 Roles

| Role | Runs | Trust required from others |
|---|---|---|
| **Hub** | Full `hydra-node` (multi-head capable, see §7), Cardano node or Blockfrost | None for safety; liveness of counterparties |
| **Light node (user)** | A thin client: Ed25519 keypair, signs/verifies snapshots, stores last certified snapshot + own KZG membership proof, watches L1 via any untrusted observer | None for safety of own funds (Thm 4.1); needs *some* guardian or itself online once per contestation period |
| **Guardian (watchtower)** | Stateless L1 watcher holding light nodes' latest `(snapshot header, σ, η)` blobs; posts `Contest` when it sees a stale close | None. Guardians hold no secrets and cannot steal (Thm 4.3). Can only fail to act; mitigated by redundancy + bounty (§5) |

### 3.2 Topology

- Every user opens a **2-party head** `{user, hub}`. Since Init directly opens a head with an empty
  UTxO set (ADR 033), the user funds it via one `Deposit`/`Increment`; the hub deposits inbound
  liquidity likewise.
- Hubs interconnect via **pairwise or small (≤29) multi-party heads**.
- A payment from Alice (hub H₁) to a stranger Bob (hub H₃) is routed
  `Alice→H₁ ⇒ H₁→H₂ ⇒ H₂→H₃ ⇒ H₃→Bob` using in-head **HTLCs**, i.e., ordinary Plutus validators
  executed on the L2 ledger. Bob needs no relationship with Alice, only with *his* hub, whom he does
  not need to trust (Thm 4.1).

### 3.3 Why n = 2 is the sweet spot, not a compromise

With n = 2 and the user honest, the *existing* Agda theorems apply verbatim with
`parties = 2`, `honest = {user}`:

- `consistency`: the hub can never get the user to confirm conflicting transactions.
- `soundness`: any snapshot whose multisignature verifies applies cleanly to U₀ and consists only of
  transactions the user has seen (`sigSeen-inv`).
- `completeness`: every transaction the user confirmed is in the finalized snapshot, provided the
  finalized one is at least as new — which `mustBeNewer` + the user's contest guarantee (Thm 4.2).

Nothing new has to be proven for safety; the proofs are inherited. **This is the central finding:
decentralization of Hydra is obtained by instantiating the already-verified protocol at n = 2 per
user and composing heads, not by weakening the protocol.**

### 3.4 Finality semantics ("close to the state machine")

For the receiver Bob, a payment is *final* the moment his 2-party snapshot including the HTLC
settlement (or the direct transfer) is certified — he holds both signatures and can enforce it on L1
against any behaviour of the hub. This is **identical** to single-head finality. The only difference
from a single big head is *latency*: k hops ⇒ k snapshot round trips plus HTLC settlement, and a
failed route costs an HTLC timeout rather than an instant rejection. There is no probabilistic
component and no honest-majority component. That is as close to state-machine finality as is
possible without being the state machine itself (Thm 2.1 shows it cannot be closer for a non-party).

---

## 4. Security theorems for Hydra Light

### Assumptions (the complete list)

- **A1 (Crypto).** Ed25519 is EUF-CMA secure (Agda `sigUnforge`); hash functions are collision
  resistant; KZG/BLS12-381 accumulator is sound under q-SDH **and** the trusted setup CRS toxic
  waste τ was destroyed (`hydra-plutus/src/Hydra/Contract/KZGTrustedSetup.hs`).
- **A2 (L1 safety).** Cardano does not roll back beyond the depth the node waits for before acting.
- **A3 (L1 liveness / censorship resistance).** A valid transaction submitted by an honest actor is
  included within time Δ_L1, where the head's contestation period CP ≥ Δ_L1 + (max tolerated offline
  window of the user or at least one of their guardians).
- **A4 (Honest signing).** The user's light client signs only snapshots that extend its own confirmed
  snapshot by transactions it has verified as applicable (the `signHonest` premises).
- **A5 (Local state).** The user or at least one guardian retains the user's latest certified
  `(header, σ, η, membership proof)`.

Nothing else is assumed. In particular: **no honesty of the hub, no honesty of any other hub, no
honesty of guardians, no honesty of the chain observer** the light client uses.

### Theorem 4.1 (Fund safety of a light node against a fully adversarial hub)

*Statement.* Under A1–A5, a light node u in head `{u, H}` with confirmed snapshot `s̄` cannot lose
funds beyond what it authorised in transactions it signed, whatever H does, including colluding with
all other hubs and guardians.

*Proof.* Any on-chain close/contest/fanout must verify a multisignature under `parties = [u, H]`
(`verifySnapshotSignature` requires exactly 2 valid signatures). By A1, a verifying signature under
u's key implies u signed (`ms-unforgeable`). By A4 (`sigApp`, `sigChain`), every snapshot u signed
is applicable to U₀ and extends u's chain; by `cert-nest` its number ≥ any earlier one u signed. The
hub's only remaining move is to close with an **older** certified snapshot `s_c < s̄`. By A5 and A3,
u or a guardian submits `Contest` with `s̄` before `contestationDeadline`; `mustBeNewer`,
`mustBeValidSnapshot`, `mustBeWithinContestationPeriod` accept it, and `mustNotChangeParameters` /
`mustPreserveHeadValue` prevent the contest from altering anything else. After the deadline,
`Fanout` is bound by `mustBindAccumulatorCommitment` to the η of the contested snapshot, so the
fanned-out UTxO is `U₀ ∘ T_{s̄}` (Agda `reflects`, with `ηEq` supplied by the honest contester who
posts the η it signed). Hence u receives exactly its balance in `s̄`. ∎

*Tightness.* Drop A3 (hub censors u for a full CP): stale close succeeds — the classical channel
attack; CP is the knob. Drop A5: u cannot construct the contest. Drop A4 (u signs blindly): u
authorises theft itself. Drop A1: forge u's signature. Each assumption is necessary.

### Theorem 4.2 (Latest-snapshot-wins)

*Statement.* Under A1–A3, the snapshot that is fanned out has the highest number among all certified
snapshots presented on-chain before the deadline.

*Proof.* Each contest requires `snapshotNumber' > snapshotNumber`; the closed datum is monotone in
snapshot number; `afterContestationDeadline` gates fanout. With a strictly monotone sequence and a
finite deadline, the last accepted is the maximum presented. Two certified snapshots with the same
number are identical (`agree`), so "highest number" is well-defined for honest parties. ∎

### Theorem 4.3 (Guardians are trust-free)

*Statement.* A guardian holding `(header, σ, η)` for the user's latest snapshot can neither steal nor
worsen the user's outcome, under any strategy.

*Proof.* The blob contains no signing key. The only transaction a guardian can build with it is a
`Contest` presenting a certified snapshot; by Theorem 4.2 such a contest can only raise the snapshot
number, never lower it, and by `mustPreserveHeadValue` cannot move value. Posting nothing is the
worst a guardian can do, and that is the no-guardian baseline. ∎

*Note.* This requires the validator change in §7.1 so that non-participants may post `Contest`.
Under the *current* validator (`mustBeSignedByParticipant`) only u itself can contest, so the
guardian would have to *be* u's key — which is not trust-free. §7.1 proves the change is safe.

### Theorem 4.4 (Atomicity of routed payments to strangers)

*Statement.* Let a payment be routed over hops `h₁…h_k` with HTLCs of hash `H(x)` and in-head
timeouts `T₁ > T₂ > … > T_k` such that `T_i − T_{i+1} ≥ CP + Δ_L1 + Δ_fanout`. Under A1–A3 and
Theorem 4.1 applied at every hop, either every hop settles (receiver reveals x, each intermediate hub
is paid) or every hop refunds after its timeout; no honest intermediary loses funds.

*Proof.* Standard HTLC induction, with the Hydra-specific ingredient: if a head closes mid-flight, the
HTLC UTxO is fanned out to L1 as the *same* script UTxO (the L2 ledger is Cardano's ledger, so the
validator is identical on L1), and it remains claimable with x or refundable after T_i on L1. The
timeout gap covers the worst case where hop i+1 settles on L1 at the last moment (T_{i+1} + CP +
Δ_fanout) and hop i must still claim with x before T_i. ∎

*Tightness.* Violating the gap inequality lets a malicious downstream hub claim late and leave the
upstream hub unable to claim — the same wormhole/griefing family as Lightning, bounded by the
inequality.

### Theorem 4.5 (Liveness under a network adversary, single head)

The Agda spec explicitly *does not* machine-check liveness (`postulate Liveness : Set`). The prose
lemmas (`lem:reqconf`, `lem:eternal`) hold for a network adversary who delivers messages and does
not close the head. For a 2-party head the leader alternates `isLeader` between u and H, so
liveness requires both online. This is *acceptable* here because liveness failure in a 2-party head
harms only its two members and is bounded: the harmed party closes and exits after CP (Thm 4.1).
There is no global liveness dependency. ∎

---

## 5. Incentive design and its security reasoning

Each incentive is chosen so that the *only* profitable strategy is the honest one, and every
deviation is either unprofitable or already covered by a theorem above.

| Actor | Reward | Cost / risk | Deviation | Why it does not pay |
|---|---|---|---|---|
| Hub | Routing fee (bps of routed value) + fee for inbound liquidity | Capital locked in each pairwise head; L1 fees for opening/closing heads | Stale close against a user | Fails by Thm 4.1/4.2; hub loses L1 fees and reputation |
| | | | Refuse to sign (liveness griefing) | User closes with `s̄`; hub's liquidity is frozen for CP; user pays one L1 fee. Cost is symmetric, gain is zero |
| | | | Steal via routing | HTLC atomicity (Thm 4.4) |
| User (light node) | Cheap, instant payments | One L1 deposit; L1 fee to exit | Refuse to sign | Hub closes; user loses nothing, hub's capital frozen for CP; hub can rate-limit/refuse future channels |
| Guardian | **On-chain bounty** (below) or subscription | Watching L1 (trivial) | Do nothing | Loses bounty to a competing guardian; redundancy makes single failure harmless (Thm 4.3) |

**Bounty without touching the head validator.** `mustPreserveHeadValue` forbids paying a contester
out of the head. Instead, the user locks a small L1 UTxO under a *bounty validator* whose spending
rule is: "this tx also spends the head UTxO of head `headId` with a `Contest` redeemer and the
resulting closed datum's `snapshotNumber` exceeds the input's". Any guardian who lands the contest
collects it in the same transaction. The bounty is unclaimable otherwise, so it is safe to post at
channel open. Racing guardians are harmless: the first valid contest wins the bounty, later ones (if
newer) still improve the user's outcome.

**Sybil/spam.** Opening a head costs L1 fees and locks capital on both sides; a hub can require a
minimum deposit. There is no free resource to spam.

**Hub concentration.** Anyone with a `hydra-node` and capital can be a hub with no permission; the
market for routing fees is the decentralization pressure. Users hold their own veto (Thm 4.1), so a
dominant hub can degrade *liveness/price*, never *safety*. That is the correct place for the
decentralization boundary to sit: the thing we cannot economically guarantee (hub count) is the
thing whose failure is recoverable.

---

## 6. Light node resource budget

What a light node must do, per the theorems (and nothing more):

| Operation | Cost |
|---|---|
| Verify counterpart's Ed25519 signature on `snapMsg` | one signature verification |
| Verify the snapshot's transaction delta is applicable to its confirmed state | ledger application over its *own* small UTxO view; for a payment channel this is a handful of UTxOs |
| Sign `snapMsg` | one Ed25519 signature |
| Store `(header, σ_u, σ_H, η, KZG membership proof of own UTxO)` | ~ hundreds of bytes; the KZG proof is a single G1 point (48 bytes) |
| Watch L1 for `Close` on its `headId` | polling any untrusted observer (`hydra-chain-observer` or Blockfrost); a lying observer can only *delay*, and guardians cover the gap |
| Be online | at least once per CP (user) **or** have ≥1 guardian |

No cardano-node, no etcd, no Haskell runtime required on the client. This fits a phone or a
sub-$10 microcontroller-class device with a network stack. The current `hydra-node` is *not* this
client; a dedicated light client implementing `ReqSn`/`AckSn`/`ReqTx` semantics over a plain
WebSocket to the hub is required (§7.3). The transport is untrusted: everything that matters is
covered by signatures the light client verifies itself.

---

## 7. Concrete changes to this repository (what must be built)

### 7.1 Validator: permissionless contest (required for trust-free guardians)

Change `checkContest` so that when the signer is **not** a participant:

- all existing checks hold (`mustBeNewer`, `mustBeValidSnapshot`, `mustBeWithinContestationPeriod`,
  `mustNotChangeParameters`, `mustPreserveHeadValue`, `mustBindAccumulatorCommitment`,
  `mustNotChangeVersion`), **and**
- the deadline is **not** pushed (`contestationDeadline' == contestationDeadline`), **and**
- `contesters` is unchanged.

*Safety argument.* The two reasons contest is participant-only today are (i) bounding the total
deadline extension to n·CP and (ii) not letting outsiders mutate `contesters`. Both are preserved by
the rule above, because a non-participant contest changes only the snapshot fields. It can only
increase `snapshotNumber` (Thm 4.2), so it is a pure improvement in fidelity to the latest certified
state. Griefing by spamming contests is impossible: each requires a *strictly newer certified*
snapshot, and only the parties can produce those; if all parties are colluding to grief, they already
control the head (Thm 2.1). Participant contests keep their existing semantics unchanged, so every
existing test and the Agda on-chain coverage remain valid for the participant path.

### 7.2 In-head HTLC validator + timeout discipline

A standard hash-time-lock Plutus validator usable both in-head and post-fanout on L1 (the code is
identical by construction). Wallet/hub software must enforce the inequality of Thm 4.4 with the
head's CP taken from `HeadParameters`.

### 7.3 Light client protocol

A minimal client speaking the coordinated-head message set (`ReqTx`, `ReqSn`, `AckSn`) to the hub
over WebSocket/HTTP, implementing exactly the `signHonest` guards from the spec
(`s = s̄ + 1`, `v = v̂`, applicability of Δ, only-seen), and producing the guardian blob. Its
correctness obligation is A4 and nothing else; this is a small, auditable surface.

### 7.4 Multi-head hub

`hydra-node` today manages one head per process (with one etcd cluster per head). A hub with
thousands of users needs one process managing many heads, or a lightweight per-head relay in place
of etcd for the 2-party case (a 2-node etcd cluster is overkill: the hub can act as the message
relay since the transport is untrusted anyway).

### 7.5 Bounty validator

The L1 script of §5. Independent of the head validator; can ship separately.

### 7.6 Formal work

- Instantiate the Agda `System` at `parties = 2`, `honest = [true, false]` and export the three
  theorems as corollaries (mechanical, no new proof content).
- Add the permissionless-contest arm to `OnChain.lagda.typ`'s contest validity bundle and extend the
  differential oracle (`Reference`/`ReferenceBridge`) so the Plutus and Agda versions are cross-checked.
- State and prove the HTLC atomicity lemma (Thm 4.4) over a k-head composition; this is new but
  standard.

---

## 8. Residual risks that no design removes (the "adversary moment" list)

These are the ways the system can still be compromised. Each corresponds to a broken assumption in
§4, and each is named so that it can be monitored rather than pretended away.

1. **Cryptographic break** of Ed25519, BLAKE2b, or BLS12-381 — total loss of guarantees. Industry
   standard; same exposure as Cardano itself.
2. **KZG trusted setup compromise.** If τ from the CRS is known, false accumulator membership proofs
   exist. Since **every** fanout path (`headIsFinalizedWith`, `checkPartialFanout`,
   `checkFinalPartialFanout`) verifies outputs by KZG membership against the closed datum's
   `accumulatorCommitment`, a τ-holder who is also a head party could fan out outputs not in the
   signed state. This is a *Hydra-specific* trust point that does not exist in Cardano L1 and must be
   stated in any threat model. It does not weaken Theorem 4.1's n-of-n argument (the commitment
   itself is still signed by the user), but it does make A1's "τ destroyed" clause load-bearing.
   Mitigation: a verifiable multi-party ceremony with many independent contributors.
3. **L1 censorship for a full CP.** Stale-close attacks succeed. Mitigation is CP sizing and guardian
   redundancy across jurisdictions/relays; it cannot be eliminated on a chain without inclusion
   guarantees.
4. **Mass exit.** A hub with N channels failing produces N closes competing for block space within
   one CP. CP must be sized with N in mind, or hubs bounded in channel count. This is a known
   channel-network limit, not a Hydra-specific one.
5. **Implementation gap.** The Agda proofs are about the *model*; the Haskell/Plutus code is linked by
   the differential oracle and test suites, not by extraction. Bugs in `HeadLogic.hs` or `Head.hs`
   are outside the proofs.
6. **Liveness is not machine-checked** (Agda `postulate Liveness`). Only safety is. In this design,
   liveness failures are local (one head) and recoverable via exit, which is why this is tolerable.
7. **Time coupling.** HTLC timeouts live in L2 slot time (`currentSlot` in `HeadLogic`), which is
   derived from L1 ticks; a hub feeding the light client a wrong clock can only delay the client's
   view, not forge signatures, but the timeout inequality must use conservative L1-anchored values.
8. **Data loss** of the guardian blob by both user and all guardians (A5). Mitigation: the blob is
   public-safe (Thm 4.3), so it can be replicated freely, including to the hub's competitors.

---

## 9. Answer to the thesis question

**Can Hydra be decentralized with light nodes, payments to strangers, and near-state-machine
finality, with incentives, without weakening its guarantees?**

Yes — but only in one shape, and the shape is forced by Theorems 2.1–2.3: *many n = 2 heads in
which the user is a party, hubs interconnected by small heads, HTLC routing, permissionless
contest, and bounty-paid guardians.* Every user's safety is then the already machine-checked n-of-n
safety with themselves as the honest party; nothing is delegated except liveness, and delegated
liveness is recoverable by exit. Finality for the receiver is identical to single-head finality;
the only cost is routing latency and HTLC timeouts on failed paths.

What it is **not**: a single large open head, a threshold-signed committee, or a rollup. The first
two are provably weaker (§2); the third is a different system with proof-generation costs that do
not fit the "very small machine" requirement today.

The claim "can never be compromised" is replaced by the correct, defensible claim: **compromise
requires violating one of A1–A5, and each of A1–A5 is shown necessary by an explicit attack.** That
is the strongest statement a security argument can honestly make, and it is one the existing Hydra
formalization is already most of the way to supporting.
