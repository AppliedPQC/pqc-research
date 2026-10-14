# Post-quantum migration for Bitcoin layer 2s

### Abstract

A Bitcoin layer 2 has a post-quantum problem that neither Bitcoin nor Ethereum
has alone. It settles to a base layer it cannot change, borrows a consensus stack
from a third ecosystem, and operates a bridge whose trust root is cryptography of
its own choosing. Its migration therefore divides into surfaces the L2 can
change, surfaces it inherits from an upstream ecosystem, and surfaces on which it
can only bound exposure until the base layer acts.

This paper gives a taxonomy of an L2's cryptographic surfaces keyed on the party
that can fix each, and a roadmap that carries every surface through the four
stages of a cryptographic migration. It reviews the state of the three base
layers an L2 draws on, analyses the migration options for each surface, and
examines one deployed L2, GOAT, as a case study. Four findings recur. Symmetric
and hash-based layers protect only what they carry, so a bridge built from
post-quantum primitives remains exposed when what it carries is an
elliptic-curve assertion. Settlement and custody outputs depend on the base
layer's output types, so the L2 can only bound their exposure until the base
layer acts. A bridge's pairing-based verifier can be replaced by a hash-based one
under garbling but not under witness encryption, and the measured cost of such a
verifier is of the order of 10^8 gates. Consensus keys are the least costly
surface, and their migration costs bandwidth and storage rather than computation.

### Contents

1. Introduction
2. Surfaces and their owners
3. A migration roadmap
4. The base layers
5. Settlement and custody outputs
6. The bridge proof system
7. Consensus keys
8. The execution layer
9. Case study: GOAT
10. Recurring pitfalls
11. Conclusion

---

## 1. Introduction

Most post-quantum analysis of blockchains treats a chain as one cryptographic
system: one signature scheme securing funds, one consensus mechanism and one
governance process to change them. A Bitcoin L2 is instead a composition of
systems designed separately, migrating on separate timelines and owned by
separate parties.

- **The L2 cannot fix its base layer.** Bitcoin has specified no post-quantum
  signature scheme:
  [BIP-360](https://github.com/bitcoin/bips/blob/master/bip-0360.mediawiki)
  states that it "does not include the introduction of post-quantum signature
  schemes", and
  [BIP-361](https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki)
  lists `Requires: TBD Post Quantum Signature BIP`. Value held in classical L1
  outputs is on Bitcoin's timeline, not the L2's.
- **The L2 borrows a consensus stack** from a third ecosystem, and inherits that
  ecosystem's migration path and constraints.
- **The L2 owns a bridge** whose trust root is cryptography of its own choosing,
  and which is the highest-value target in the system.

The goal is to make both the L2 and the base layer post-quantum secure, while
recognising that the L2 can schedule only part of the work. The contributions of
this paper are a taxonomy of the surfaces and their owners (section 2); a roadmap
placing each surface against the four stages of a migration (section 3); a review
of the base layers (section 4); an analysis of the options for each surface
(sections 5 to 8); a case study of one deployed L2 (section 9); and the pitfalls
that recur across them (section 10).

**Method.** The analysis is drawn from primary sources: BIP and EIP texts, the
Cosmos SDK and CometBFT release notes, the design documents and security analyses
of the protocols discussed, and the published designs and code of the L2
examined. Cost figures for hash-based verifiers are measured on real proofs
rather than estimated. One L2 is examined in depth, so the taxonomy rests on
structural reasoning supported by a single case rather than on a survey.

## 2. Surfaces and their owners

A Bitcoin L2 that settles to L1 and runs its own consensus has at least the
following cryptographic surfaces. The taxonomy makes the ownership question
explicit before the question of which scheme to adopt.

| # | Surface | Typical scheme | Broken by Shor? | Who can fix it |
| --- | --- | --- | --- | --- |
| 1 | L1 settlement outputs | secp256k1 / Schnorr | yes | **the base layer only** |
| 2 | Bridge attestation and custody | secp256k1, Schnorr, MuSig2, BLS | yes | **the L2**, within the base layer's output types |
| 3 | Bridge proof system | Groth16, PLONK (pairing-based) or STARK (hash-based) | if pairing-based, including a pairing-based wrap of a STARK | **the L2**, with its proof-system provider |
| 4 | Bridge data commitments | Merkle, Winternitz, Lamport | **no** (hash-based) | — |
| 5 | L2 consensus keys | ed25519, secp256k1, BLS12-381 | yes | **the L2**, through its consensus stack |
| 6 | L2 execution accounts | secp256k1 ECDSA | yes | **the L2**, following its execution ecosystem |
| 7 | EVM cryptographic precompiles | ecrecover, BN254 pairing, BLS12-381, KZG | yes | **upstream**, and not at all for immutable callers |

Three parties hold the fixes. The base layer alone can change rows 1 and part of
row 2; the L2 controls rows 2, 3 and 5; the execution ecosystem, Ethereum for an
EVM-compatible L2, determines rows 6 and 7.

## 3. A migration roadmap

A cryptographic migration passes through four stages, as formulated in 
[NIST CSWP 39](https://csrc.nist.gov/pubs/cswp/39/upd1/considerations-for-achieving-crypto-agility/final):
*identify dependencies*; *introduce and test* the replacement; *move users and
data*; and *retire old paths*, disabling the vulnerable authorisation path so
that it cannot be forced back into use. The roadmap applies the four stages to
each surface of section 2.

| Order | Surface | Identify dependencies | Introduce and test | Move users and data | Retire old paths | Gated by |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | All surfaces | Inventory every signature and proof verification path, including fixed-length signature checks | — | — | — | the L2 |
| 1 | Bridge attestation keys (row 2) | Operator and committee signing keys | A post-quantum signature for attestations | Rotate the operator set, with dual signing | Stop accepting classical attestations | the L2 |
| 2 | Bridge proof system (row 3) | Every layer that asserts a pairing relation: wrap, verifier, witness encryption | A hash-based verifier, in script or garbled (section 6) | Move the dispute path to the new verifier | Disable the pairing-based dispute path | the L2, with its prover |
| 3 | Consensus keys (row 5) | Validator key types | A post-quantum key type in the consensus stack | Per-validator key rotation | Disallow new classical consensus keys | the L2, with its consensus stack |
| 4 | Execution accounts (row 6) | Accounts whose public keys are revealed; signature authority accepted by contracts | Post-quantum transactions and verification | Each holder migrates its own account | Revoke both the transaction and the `ecrecover` powers of the old key (section 8) | the execution ecosystem |
| 4 | EVM precompiles (row 7) | Callers of pairing and KZG precompiles, and their upgradeability | Additive post-quantum precompiles | Upgradeable callers move | Immutable callers cannot be retired | the execution ecosystem |
| 5 | Settlement and custody outputs (rows 1–2) | Outputs that commit a public key on chain | A post-quantum output type and signature | Move value into the new output type | Sunset of legacy spends | the base layer |

The order follows ownership and severity rather than novelty.

1. **Bridge attestation keys** carry the highest value per unit of control: the
   operator set is the bridge's trust root, the change is entirely the L2's, and
   dual signing gives rollback at every step.
2. **The bridge proof system** is the most severe exposure, since its failure
   breaks soundness rather than individual keys, and the hardest, since it means
   replacing the verifier the bridge is built around.
3. **Consensus keys** often have the mechanism upstream already, so the work is a
   dependency upgrade rather than cryptographic design.
4. **Execution accounts and precompiles** follow the execution ecosystem; an L2
   that diverges ahead of it breaks wallet and tooling compatibility, which for
   an L2 is usually the product. The precompile inventory is the exception and
   can be taken now, since an immutable caller can be identified before it
   matters but never migrated after.
5. **Settlement outputs** cannot be fixed by the L2. Exposure is bounded instead
   (section 5) until the base layer provides an output type and a signature.

The fourth stage is the one most often omitted. A surface on which a new
algorithm has been introduced but the old path has not been retired is no more
secure than before, since an adversary uses the old path.

## 4. The base layers

| | **Bitcoin** | **Ethereum** | **Cosmos** |
| --- | --- | --- | --- |
| Core problem | public-key exposure on an immutable ledger | signature size at validator scale | key-type negotiation across chains |
| Layer addressed first | output type and script | consensus signatures, and accounts separately | validator consensus keys |
| Post-quantum scheme chosen | none | hash-based, aggregated | ML-DSA-65 (FIPS 204) |
| Furthest artifact | draft BIPs | prototypes | shipped, opt-in |
| Governance | soft fork, rough consensus | coordinated upgrades | per-chain opt-in through genesis parameters |

### 4.1 Bitcoin

Two draft BIPs address the problem; neither is activated.
[BIP-360](https://github.com/bitcoin/bips/blob/master/bip-0360.mediawiki)
defines an output type that commits no public key on chain, and
[BIP-361](https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki)
proposes a pre-announced sunset of legacy ECDSA and Schnorr spends. BIP-361
depends on a post-quantum signature BIP that does not yet exist. The first
candidate, [SHRINCS](https://github.com/SHRINCS/shrincs-bip/blob/main/SHRINCS.md),
posted as a draft in August 2026, is hash-based: a stateful signature from 548
bytes with an SLH-DSA fallback of 5,777 bytes under one public key. It defines no
output type, and it supports no multi-signature aggregation, which bears directly
on bridge custody (section 5).

### 4.2 Ethereum

Ethereum treats consensus migration as an aggregation problem: hash-based
signatures rest only on hash security but are too large to place on chain one per
validator, so recursive aggregation in a hash-based zkVM,
[leanVM](https://github.com/leanEthereum/leanVM), compresses them, with
[leanSig](https://github.com/leanEthereum/leanSig) as the prototype signature
scheme. Accounts follow a separate path. Account abstraction is the vehicle:
[EIP-8141](https://eips.ethereum.org/EIPS/eip-8141) (frame transactions) admits an
arbitrary post-quantum signature scheme, and
[EIP-7885](https://eips.ethereum.org/EIPS/eip-7885) exposes the number-theoretic
transform as a precompile rather than welding one scheme into the protocol.
Section 8 takes up the consequences for an EVM-compatible L2. Nothing
post-quantum has shipped to Ethereum mainnet; the proposals cited are drafts.

### 4.3 Cosmos

[Cosmos SDK v0.55](https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md)
ships ML-DSA-65 as a validator consensus key type, opt-in through the chain's
permitted key types, together with validator key rotation, so an existing chain
can move its validators without a new genesis. Public keys grow from 32 to 1,952
bytes and signatures from 64 to 3,309. CometBFT does not aggregate commit
signatures: a commit carries one signature per voting validator, so its size is
linear in the validator set.

| Validators | ed25519 | ML-DSA-65 |
| --- | ---: | ---: |
| 50 | 3.2 KB | 165 KB |
| 100 | 6.4 KB | 331 KB |
| 150 | 9.6 KB | 496 KB |

One hazard generalises beyond Cosmos: an IBC light client verifies a counterparty's commit
signatures with its own compiled-in cryptography, so enabling a key type before
counterparties can verify it stops packet flow and eventually expires the
client. Migration in an interoperable ecosystem is a coordination problem before
it is a cryptographic one.

## 5. Settlement and custody outputs

A bridge holds deposited value in base-layer outputs, and the post-quantum
security of those outputs is set by the output types and signatures the base
layer offers. On Bitcoin none is post-quantum yet: the outputs a BitVM-style
bridge uses commit a public key on chain, and only a new output type, as BIP-360
proposes, together with a post-quantum signature removes that exposure. Both are
on the base layer's timeline.

What an L2 can do unilaterally is bound the exposure. It grows with the time value
remains in such outputs, so a custody policy that rotates outputs on a schedule,
caps the value in any one output and shortens the lifetime of connector outputs
reduces it without waiting on the base layer. Such a policy bounds the risk
rather than removing it. Custody that rests on MuSig2 has a further dependency:
its post-quantum replacement needs either a multi-signature construction for a
post-quantum scheme, which SHRINCS does not provide, or a script that checks
several individual signatures, at a corresponding cost in size.

## 6. The bridge proof system

### 6.1 The wrapper pattern

A BitVM-style bridge convinces Bitcoin script that a proof of the L2's state
transition verified. In current designs a zkVM produces a hash-based STARK, the
STARK is wrapped in a constant-size Groth16 proof over BN254, and the Groth16
verifier is what the dispute protocol evaluates. Hash-based layers surround the
wrap: the STARK below it, and the Winternitz or Lamport commitments that carry
values into script above it. Every layer between them asserts, or is keyed to,
the BN254 pairing relation, and a forged pairing relation is a forged peg-out:
the failure is one of soundness rather than of an individual key.

### 6.2 Generations of the on-chain verifier

The designs that bring a SNARK verifier to Bitcoin form one line of work. Each
answers the question *how is a Bitcoin script convinced that a Groth16 proof
verified*, and each moves more of the pairing arithmetic off chain. What varies
across generations is the execution of verification; the relation verified is
invariant.

| Generation | Mechanism | Cost addressed | Relation asserted |
| --- | --- | --- | --- |
| BitVM2 | Groth16 verifier compiled to Bitcoin script | on-chain: a Disprove script of several hundred KB, **over \$14,000** in a reported unhappy-path experiment | Groth16 / BN254 |
| [BitVM3](https://eprint.iacr.org/2026/933) | verifier circuit garbled; the dispute evaluates the garbling | on-chain cost collapses, but a garbled circuit is **42 GiB** | Groth16 / BN254 |
| [BABE](https://eprint.iacr.org/2026/065) | witness encryption for linear pairing relations, with a garbled scalar multiplication for the non-linear part | off-chain storage and setup, **~1000× lower** than BitVM3 | Groth16 / BN254 |
| [Deferred Binding](https://hackmd.io/@goatresearch/HkKp2g1Zfl) | BABE extended so that part of the public input may be fixed after garbling | usable when L2 state is known only at withdrawal | Groth16 / BN254 |

[BABE](https://eprint.iacr.org/2026/065) (Garg, Kolonelos, Sergeevitch, Sridhar
and Tse) keeps BitVM3's on-chain savings and "reduces its off-chain storage and
setup costs by three orders of magnitude", using "a witness encryption scheme for
linear pairing relations to verify Groth16 proofs" together with "a very
efficient garbled circuit for scalar multiplication on elliptic curves", which
builds on [Argo MAC](https://eprint.iacr.org/2026/049) (Eagen and Lai);
[Duty-Free Bits](https://eprint.iacr.org/2026/476) (Khambhati, Bhattacharya and
Heath) improves both encodings further. Deferred Binding, with its formal
primitive
[partial-binding witness encryption](https://github.com/GOATNetwork/bitvm-gc/blob/main/docs/partial_binding_we.tex),
lets the dynamic part of the public input (L2 state, sequencer commitments) be
bound at withdrawal rather than at garbling.

### 6.3 The relation and the security reduction

The witness encryption is keyed to the Groth16 verification equation on BN254,

```text
e(pi_1, pi_2) = e(alpha, beta) * e(vk_x, gamma) * e(pi_3, delta_2)
```

so the witness that decrypts the payout secret is a valid Groth16 proof, and the
scheme's correctness guarantees that anyone holding one can decrypt. The on-chain
side is hash-based: Bitcoin script performs hash commitments, garbled-gate
unlocking and a hashlock on the payout secret, and "No pairings or target-group
arithmetic are evaluated on chain." The security analysis of partial-binding
witness encryption states the reduction directly:

```text
Adv_pbWE(lambda)  <=  Adv_BABE(lambda)  +  Adv_DLog[G1](lambda)
```

and concludes that it "stays within standard BABE assumptions plus GC-security
and Lamport one-wayness". Against a quantum adversary `Adv_DLog[G1]` on BN254 goes
to 1 and `Adv_BABE` rests on the same curve, so the bound becomes vacuous: the
adversary forges a Groth16 proof for a withdrawal that never occurred, decrypts
the payout secret through the scheme's own correctness, and spends the hashlocked
output, without breaking any garbling or inverting any hash.

This is the wrapper pattern in its least visible form. Witness encryption,
garbled circuits, hashlocks and Lamport commitments are post-quantum primitives,
and three of the four on-chain primitives are post-quantum; the pairing persists
in the relation being encrypted against. The garbling layer has a parameter of its
own: 128-bit wire labels under an AES-128 PRF place it at NIST post-quantum
category 1, the floor of the approved range. Where garbling gates decryption of a
payout secret rather than serving as an optimisation, the label width warrants an
explicit decision; 256-bit labels restore category-5 margin at roughly twice the
garbling cost.

### 6.4 Replacing the pairing-based verifier

Witness encryption does not transfer to a hash-based verifier. BABE encrypts
against a linear pairing relation; a STARK verifier has no pairing relation, no
target group and no short algebraic acceptance predicate, so there is nothing for
a ciphertext to be keyed to, and witness encryption for hash-based proof systems
is an open problem. Garbling does transfer: a garbled circuit requires only that
the verifier be a Boolean circuit, which a hash-based verifier is. BABE thereby
creates a tension between cost and security: it lowers the cost of verifying
Groth16 on Bitcoin, and with it the pressure to replace Groth16, whereas
replacement is the post-quantum requirement.

A hash-based verifier can reach Bitcoin by two routes.

**In script.** Bitcoin script has no multiplication and no concatenation
opcodes; multiplication is emulated, as BitVM already does at 254 bits, but a
script cannot assemble the two-child preimage a SHA-256 Merkle step hashes
without `OP_CAT` ([BIP-347](https://github.com/bitcoin/bips/blob/master/bip-0347.mediawiki),
specified but not activated). The dependency belongs to the hash rather than to
the proof system: an algebraic hash such as Poseidon2 over a 31-bit field is
field arithmetic, which script emulates, so a FRI or WHIR verifier over such a
hash needs no soft fork. Its cost is the hash: a Poseidon2 permutation is about
572 KB of script, a Merkle path costs one permutation per level, and a full
verifier at 100-bit security is several hundred megabytes of script, about 97%
of it query work. Divided into transaction-sized segments for a disprove
protocol, of which a dispute executes one, it is of the order of two thousand
segments, the same order as BitVM2's own. Lattice-based arguments avoid the
per-level hash but pay in verifier work and require changing the prover's
commitment scheme; their cost in script is unmeasured.

**Garbled.** A binary-field STARK verifier, WHIR over GF(2^128) with Blake3,
garbled as in BitVM3, has 9.4 × 10^7 non-free gates and garbles to 1.5 GB for a
2^18-row trace at 104 bits of soundness, about thirty times fewer non-free gates
than the garbled Groth16 verifier
([Garbling a Hash-Based STARK Verifier for Bitcoin](https://github.com/GOATNetwork/bitcoin-stark-verifier/blob/whir-gc/paper/garbled-stark-verifier.pdf)).
The field and the hash, more than the proof system, determine the cost of a
garbled verifier: as a Boolean circuit, a Poseidon2 permutation over a 31-bit
prime field costs about 120 Blake3 compressions, so a proof committed with an
algebraic hash over a prime field is better wrapped into a binary-field proof
than garbled directly. Ziren, the zkVM in the case study, defines exactly this
route alongside its pairing-based wrap: a binary stage over GF(2^128) with Blake3
and a narrow recursion reduce its proof to one whose verifier is a compact
Boolean circuit ([Duan et al.](https://eprint.iacr.org/2026/2330)).

Both routes are feasible without a soft fork. What remains is integration into
the dispute protocol and a choice by cost, and, because the garbling stacks in
use are organised around the Groth16 verifier, the integration is a rebuild of
that component rather than the removal of a wrapper.

## 7. Consensus keys

Consensus keys admit the least costly migration. Where the consensus stack ships
a post-quantum key type, as Cosmos SDK v0.55 does with ML-DSA-65 (section 4.3),
the work is a dependency upgrade, a key rotation and a re-tuning of block
parameters. Two threats are distinct. Forged votes in live consensus become
possible when a cryptographically relevant quantum computer exists, and not
before. Forgery of recorded history, an alternative chain assembled from archived
commit signatures, is a harvest-now, forge-later threat: blocks signed before a
migration remain forgeable after it.

A migration that avoids a network-wide switch rotates validators individually.
In the design published for the case-study L2
([GOAT Research, *Post-Quantum Consensus Keys on GOAT*](https://hackmd.io/MPLgpnyATaiFfskWDXumRQ)),
each validator keeps a permanent identity mapped to its current consensus key,
rotates with a single transaction carrying the new key and a proof of possession,
and is limited to one rotation per retention period; the validator set may remain
mixed indefinitely, and the classical key is retained as an account key rather
than combined with the new one in a hybrid signature. Activation is gated by fork
heights, and each validator then rotates at a time of its choosing.

The measured costs reverse a common expectation:

| | secp256k1 | ML-DSA-65 |
| --- | ---: | ---: |
| Verification, per signature | 173 µs | 23 µs |
| Signing, per signature | 46 µs | 196 µs |
| Commit signatures per block, 20 validators | 1.3 KB | 66 KB |
| Validator public keys in state, 20 validators | 0.7 KB | 39 KB |

Verification becomes faster, so the migration costs bandwidth and storage rather
than computation, linearly in the validator count: at 20 validators the commit
signatures add about 585 GB per year, and at 100 about 3 TB. Aggregation of
post-quantum consensus signatures, as in Ethereum's recursive designs, would
remove the linear term, but no such mechanism exists in CometBFT, where two
attempts to add BLS aggregation were closed without merging. Two points of a
rotation design need deliberate treatment: the interaction between the evidence
window and the retention period, since misbehaviour under a rotated-out key must
remain attributable; and the final stage, disallowing new classical consensus
keys once rotation is complete.

## 8. The execution layer

### 8.1 Accounts

An EVM-compatible L2 inherits Ethereum's account model, and with it two findings
of a cost analysis of Ethereum's proposals
([AppliedPQC, *Post-Quantum Migration in Ethereum's Execution Layer*](https://github.com/AppliedPQC/pqc-research/issues/35)).
First, the cost of a post-quantum transaction lies in its bytes rather than its
verification: with a verification precompile, verification is a few percent of a
transaction's gas, and the signature's size sets the rest. Second, obtaining a
post-quantum key does not complete a migration. The old key retains two
independent powers, originating transactions from the address and authorising
actions through `ecrecover` in contracts, and each requires its own revocation.
Installing code at the account
([EIP-8298](https://eips.ethereum.org/EIPS/eip-8298)) brings
[EIP-3607](https://eips.ethereum.org/EIPS/eip-3607) into force and ends ECDSA
transaction origination, whereas a delegation under
[EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) alone does not;
[EIP-8151](https://eips.ethereum.org/EIPS/eip-8151) restricts `ecrecover` for
such accounts, closing signature-based authorisation through immutable contracts
such as ERC-20 `permit`. Until both revocations exist, abandoning the address is
the only action that removes both powers. For an L2 the consequence is to follow
Ethereum's account semantics rather than diverge ahead of them, and to keep its
execution client close enough to upstream that the changes can be inherited when
they ship.

### 8.2 Precompiles

The EVM exposes elliptic-curve and pairing operations as consensus-level
precompiles: secp256k1 recovery, BN254 addition, multiplication and pairing,
BLS12-381 operations, KZG point evaluation and secp256r1 verification. A
precompile cannot be migrated by its users, since it is a consensus rule and its
callers are often immutable. What decides whether an exposure can be migrated is
the caller set rather than the precompile.

| Precompile | Caller set | Migratable |
| --- | --- | --- |
| `ecrecover` | every account, gated by per-account state | yes: narrow it, as EIP-8151 does |
| KZG point evaluation | rollup contracts: bounded, known and maintained | plausibly: coordinate the set |
| BN254 and BLS12-381 pairing | every SNARK verifier deployed: unbounded and largely unmaintained | no |

A precompile bound to an account identity can be narrowed conditionally on that
account's state; a stateless mathematical precompile cannot, since there is no
account whose migration status could gate it. For the latter, the levers are
additive post-quantum precompiles, which solve the forward problem only;
repricing, which breaks callers silently through exhaustion of gas; removal on a
fixed date, without precedent on Ethereum; or nothing. The question for a given
L2 is therefore whether its own verifier contracts are upgradeable. A pairing
precompile also appears on the Bitcoin side of the same bridge, through the
Groth16 wrap of section 6, so a single broken assumption compromises the bridge
from both directions and should be tracked as one dependency.

## 9. Case study: GOAT

GOAT is a Bitcoin L2 with a CometBFT and Cosmos SDK consensus layer, an EVM
execution layer, and a BitVM2-style bridge whose dispute protocol is moving to
garbled verification. Its surfaces map onto the taxonomy as follows.

| Row | Surface | Current scheme | Quantum status | Roadmap stage | Owner |
| --- | --- | --- | --- | --- | --- |
| 1 | Settlement outputs | secp256k1 / Schnorr | broken | bounded only | Bitcoin |
| 2 | Custody | MuSig2 | broken | bounded only | GOAT, within Bitcoin's output types |
| 3 | Bridge proof system | Ziren STARK, Groth16/BN254 wrap, garbled verifier and witness encryption | broken at the wrap and the verifiers built on it | replacement introduced and measured | GOAT, with ZKM |
| 4 | Bridge commitments | Winternitz one-time signatures | post-quantum | — | — |
| 5 | Consensus keys | secp256k1 | broken until rotated | design published | GOAT |
| 6 | Execution accounts | secp256k1 ECDSA | broken | follows Ethereum | Ethereum, inherited |
| 7 | EVM precompiles | upstream set | broken for pairing and curve precompiles | inventory | Ethereum, inherited |

Three observations follow. The bridge's Bitcoin-side plumbing is already
post-quantum while its cryptographic content is not, which reframes the work on
row 3 from a rewrite into the replacement of one relation: the bridge carries
values into script through Winternitz commitments, but the relation those
commitments carry is a pairing equation. Ziren, the bridge's
zkVM, already defines the hash-based route that replaces the wrap, so the
remaining work on row 3 lies in the garbling stack and the dispute protocol rather
than in the prover. And the consensus keys, the only surface with a published
migration design, are also the least costly, which makes them a natural first
deployment of post-quantum cryptography in the stack even though the bridge is
the more severe exposure.

## 10. Recurring pitfalls

- **Symmetric layers protect only what they carry.** Garbled circuits, hash
  commitments and MPC layers survive Shor, but a composition is only as secure as
  the statement it garbles or commits to. The symmetric layer's own parameter
  needs checking separately: 128-bit labels sit at the floor of the approved
  post-quantum range.
- **Read the security theorem, not the list of primitives.** A scheme built from
  witness encryption, garbled circuits, hashlocks and Lamport commitments can
  still reduce to discrete logarithms, because the pairing survives in the
  relation encrypted against. When a design claims to remove an on-chain
  verifier, the question is what its ciphertext is keyed to.
- **The wrapper pattern runs in both directions.** It appears in existing
  designs, a hash-based commitment carrying a curve assertion, and it is as easily
  introduced during a migration, by proving inside a post-quantum system that a
  broken primitive verified. A migration step helps only if it changes what is
  asserted, not merely what attests to it.
- **A component's orientation is a structural property.** A garbling or proving
  stack built for one verifier makes the proof system part of its structure;
  replacing it is a rebuild, and a dependency graph does not show this.
- **Introducing a scheme is not retiring the old one.** An account with a
  post-quantum key, or a chain with a post-quantum key type, remains exposed
  through every path the old key still authorises.
- **Fixed-length signature checks.** Code that validates a 64-byte signature
  cannot accept a 3,309-byte one; such checks are small, load-bearing and
  invisible until something fails.
- **Size lands on the hot path.** Post-quantum signatures are one to two orders of
  magnitude larger; on a consensus layer every validator signs every block, so
  block, gossip and commit budgets need re-tuning together with the key type.
- **Forked dependencies defer upstream work.** An L2 on forked dependencies must
  catch up with upstream before it can inherit upstream post-quantum work, and
  the size of that gap often sets the schedule more than the cryptography does.
- **Prefer exposing a primitive to standardising a scheme.** A precompile bound to
  one construction becomes unremovable once contracts depend on it; exposing the
  shared operation, as EIP-7885 does with the number-theoretic transform, keeps
  the choice of scheme in upgradeable code.
- **Interoperability makes migration a coordination problem.** Where light
  clients verify a counterparty's signatures with their own cryptography, enabling
  a new key type before counterparties can verify it breaks connectivity silently.

## 11. Conclusion

A Bitcoin L2's post-quantum migration is a set of migrations owned by different
parties. The L2 controls its bridge and its consensus keys and can carry both
through every stage of the roadmap; it inherits its execution layer from Ethereum;
and on its settlement outputs it can only bound exposure until Bitcoin defines a
post-quantum output type and signature. The bridge is the decisive surface. Its
post-quantum position is set not by the primitives it is built from, most of which
are already post-quantum, but by the relation those primitives carry, and
replacing that relation is feasible today under garbling at a measured cost of the
order of 10^8 gates.

The analysis has three limits. One L2 has been examined in depth, so the taxonomy
rests on a single case. The hash-based verifiers are measured on proofs of
Plonky3's systems and of Ziren's experimental garbled route, not yet on a deployed
dispute protocol. And several designs discussed, among them BIP-360, BIP-361,
SHRINCS, Deferred Binding and the Ethereum account proposals, are drafts whose
details may change.

### References

**Base layers**

- BIP-360, *Pay-to-Merkle-Root (P2MR)* — <https://github.com/bitcoin/bips/blob/master/bip-0360.mediawiki>
- BIP-361, *Post Quantum Migration and Legacy Signature Sunset* — <https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki>
- BIP-347, *OP_CAT in Tapscript* — <https://github.com/bitcoin/bips/blob/master/bip-0347.mediawiki>
- SHRINCS Working Group, *SHRINCS: an efficient hash-based signature scheme for Bitcoin (first draft)*, bitcoin-dev, 2026-08-27 — <https://groups.google.com/g/bitcoindev/c/HbVboXIFiG8>, <https://github.com/SHRINCS/shrincs-bip/blob/main/SHRINCS.md>
- NIST, FIPS 204 (ML-DSA) and FIPS 205 (SLH-DSA) — <https://csrc.nist.gov/pubs/fips/204/final>, <https://csrc.nist.gov/pubs/fips/205/final>
- Post-Quantum Ethereum — <https://pq.ethereum.org/>
- `leanEthereum/leanVM` and `leanEthereum/leanSig` — <https://github.com/leanEthereum/leanVM>, <https://github.com/leanEthereum/leanSig>
- Cosmos SDK, `UPGRADING.md`, and PR #26436 (ML-DSA-65 consensus keys) — <https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md>, <https://github.com/cosmos/cosmos-sdk/pull/26436>
- CometBFT issue #3455 and PRs #3632 and #4763, BLS signature aggregation — <https://github.com/cometbft/cometbft/issues/3455>, <https://github.com/cometbft/cometbft/pull/3632>, <https://github.com/cometbft/cometbft/pull/4763>

**Migration framework**

- NIST CSWP 39-upd1, *Considerations for Achieving Cryptographic Agility* — <https://csrc.nist.gov/pubs/cswp/39/upd1/considerations-for-achieving-crypto-agility/final>

**Bridge proof systems**

- R. Linus et al., *BitVM3: Efficient Bitcoin Bridges via Garbled Circuits* — <https://eprint.iacr.org/2026/933>
- S. Garg, D. Kolonelos, M. Sergeevitch, S. Sridhar, D. Tse, *BABE: Verifying Proofs on Bitcoin Made 1000x Cheaper* — <https://eprint.iacr.org/2026/065>
- L. Eagen, Y. T. Lai, *Argo MAC: Garbling with Elliptic Curve MACs* — <https://eprint.iacr.org/2026/049>
- N. Khambhati, A. Bhattacharya, D. Heath, *Duty-Free Bits: Projectivizing Garbling Schemes* — <https://eprint.iacr.org/2026/476>
- GOAT Research, *Deferred Binding: Extending BABE for Dynamic Public Inputs in GOAT BitVM3* — <https://hackmd.io/@goatresearch/HkKp2g1Zfl>
- GOAT, *Partial-Binding Witness Encryption over Groth16* — <https://github.com/GOATNetwork/bitvm-gc/blob/main/docs/partial_binding_we.tex>
- S. Duan, J. Yin, B. Hou, V. Pham, E. Zhu, M. Guo, *Ziren: Succinct Arguments for MIPS32 Execution*, Cryptology ePrint Archive 2026/2330 — <https://eprint.iacr.org/2026/2330>
- *Garbling a Hash-Based STARK Verifier for Bitcoin*, and the measured verifiers in script and garbled — <https://github.com/GOATNetwork/bitcoin-stark-verifier/blob/whir-gc/paper/garbled-stark-verifier.pdf>, <https://github.com/GOATNetwork/bitcoin-stark-verifier>

**Consensus and execution layers**

- GOAT Research, *Post-Quantum Consensus Keys on GOAT* — <https://hackmd.io/MPLgpnyATaiFfskWDXumRQ>
- AppliedPQC, *Post-Quantum Migration in Ethereum's Execution Layer: The Full Picture* — <https://github.com/AppliedPQC/pqc-research/issues/35>
- EIP-3607, EIP-7702, EIP-7885, EIP-8141, EIP-8151 and EIP-8298 — <https://eips.ethereum.org/EIPS/eip-3607>, <https://eips.ethereum.org/EIPS/eip-7702>, <https://eips.ethereum.org/EIPS/eip-7885>, <https://eips.ethereum.org/EIPS/eip-8141>, <https://eips.ethereum.org/EIPS/eip-8151>, <https://eips.ethereum.org/EIPS/eip-8298>
