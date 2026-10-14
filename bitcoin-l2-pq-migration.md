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
elliptic-curve assertion. Settlement and bridge outputs depend on the base
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
5. Settlement and bridge outputs
6. The bridge proof system
7. Consensus keys
8. The execution layer
9. Case study: GOAT
10. Conclusion

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
(sections 5 to 8), each with the pitfalls specific to it; and a case study of one
deployed L2 (section 9).

**Method.** The analysis is drawn from primary sources: BIP and EIP texts, the
Cosmos SDK and CometBFT release notes, the design documents and security analyses
of the protocols discussed, and the published designs and code of the L2
examined. Cost figures for hash-based verifiers are measured on real proofs
rather than estimated. One L2 is examined in depth, so the taxonomy rests on
structural reasoning supported by a single case rather than on a survey.

## 2. Surfaces and their owners

A Bitcoin L2 that settles to L1 and runs its own consensus has at least the
following cryptographic surfaces, listed in Table 1. The taxonomy makes the
ownership question explicit before the question of which scheme to adopt.

**Table 1.** Cryptographic surfaces of a Bitcoin L2 and the party that can fix
each.

| # | Surface | Typical scheme | Broken by Shor? | Who can fix it |
| --- | --- | --- | --- | --- |
| 1 | L1 settlement outputs | secp256k1 / Schnorr | yes | **the base layer only** |
| 2 | Bridge attestation and covenant keys | secp256k1, Schnorr, MuSig2, BLS | yes | **the L2**, within the base layer's output types |
| 3 | Bridge proof system | Groth16, PLONK (pairing-based) or STARK (hash-based) | if pairing-based, including a pairing-based wrap of a STARK | **the L2**, with its proof-system provider |
| 4 | Bridge data commitments | Merkle, Winternitz, Lamport | **no** (hash-based) | — |
| 5 | L2 consensus keys | ed25519, secp256k1, BLS12-381 | yes | **the L2**, through its consensus stack |
| 6 | L2 execution accounts | secp256k1 ECDSA | yes | **the L2**, following its execution ecosystem |
| 7 | EVM cryptographic precompiles | ecrecover, BN254 pairing, BLS12-381, KZG | yes | **upstream**, and not at all for immutable callers |

*Notes.* Row numbers are used throughout the paper. "Broken by Shor?" asks
whether Shor's algorithm, run on a cryptographically relevant quantum computer,
recovers keys or forges proofs; the hash-based schemes of row 4 face only the
quadratic speed-up of Grover's algorithm, which parameter choice absorbs. Row 2
has two owners: the L2 chooses the signing scheme and the signer set, while the
base layer fixes the output types that hold the value.

Three parties hold the fixes. The base layer alone can change rows 1 and part of
row 2; the L2 controls rows 2, 3 and 5; the execution ecosystem, Ethereum for an
EVM-compatible L2, determines rows 6 and 7.

## 3. A migration roadmap

A cryptographic migration passes through four stages, as formulated in 
[NIST CSWP 39](https://csrc.nist.gov/pubs/cswp/39/upd1/considerations-for-achieving-crypto-agility/final):
*identify dependencies*; *introduce and test* the replacement; *move users and
data*; and *retire old paths*, disabling the vulnerable authorisation path so
that it cannot be forced back into use. The roadmap applies the four stages to
each surface of Table 1. Figure 1 gives its shape, and Table 2 states each step.

[![The migration roadmap: six surfaces in order against the four NIST CSWP 39 stages, coloured by the party that gates each row](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/bitcoin-l2-pq-migration/roadmap.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/bitcoin-l2-pq-migration/roadmap.png)

**Figure 1.** The migration roadmap. Rows are surfaces in migration order and
columns are the four stages; the fill marks the party that gates the row. Dashed
cells await a party other than the L2, and the hatched cell marks the one step
that cannot be completed, the retirement of immutable precompile callers.

**Table 2.** The four migration stages applied to each surface.

| Order | Surface | Identify dependencies | Introduce and test | Move users and data | Retire old paths | Gated by |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | All surfaces | Inventory every signature and proof verification path, including fixed-length signature checks | — | — | — | the L2 |
| 1 | Bridge attestation keys (row 2) | Operator and committee signing keys | A post-quantum signature for attestations | Rotate the operator set, with dual signing | Stop accepting classical attestations | the L2 |
| 2 | Bridge proof system (row 3) | Every layer that asserts a pairing relation: wrap, verifier, witness encryption | A hash-based verifier, in script or garbled (section 6) | Move the dispute path to the new verifier | Disable the pairing-based dispute path | the L2, with its prover |
| 3 | Consensus keys (row 5) | Validator key types | A post-quantum key type in the consensus stack | Per-validator key rotation | Disallow new classical consensus keys | the L2, with its consensus stack |
| 4 | Execution accounts (row 6) | Accounts whose public keys are revealed; signature authority accepted by contracts | Post-quantum transactions and verification | Each holder migrates its own account | Revoke both the transaction and the `ecrecover` powers of the old key (section 8) | the execution ecosystem |
| 4 | EVM precompiles (row 7) | Callers of pairing and KZG precompiles, and their upgradeability | Additive post-quantum precompiles | Upgradeable callers move | Immutable callers cannot be retired | the execution ecosystem |
| 5 | Settlement and bridge outputs (rows 1–2) | Outputs that commit a public key on chain | A post-quantum output type and signature | Move value into the new output type | Sunset of legacy spends | the base layer |

*Notes.* Row references are to Table 1; row 4, the hash-based data commitments,
needs no migration and is omitted. Order 0 is a precondition for every other row,
and rows sharing an order number can proceed in parallel. A dash marks a stage
that does not apply. "Gated by" names the party whose action the row waits on,
which is not always the party that performs the work: the holders of execution
accounts perform stage 3 themselves.

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
secure than before, since an adversary uses the old path: an account that adds a
post-quantum key still answers to its ECDSA key until both of that key's powers
are revoked (section 8.1), and a chain that admits ML-DSA-65 consensus keys still
accepts secp256k1 ones until they are disallowed (section 7). The inventory of
stage 0 should also include fixed-length signature checks: code that validates a
64-byte signature rejects a 3,309-byte ML-DSA-65 signature, and such checks sit in
transaction decoding, gossip limits and commit validation, where they surface only
at activation.

## 4. The base layers

An L2 draws on up to three ecosystems: Bitcoin for settlement, Ethereum for
execution and Cosmos for consensus. Table 3 compares their positions.

**Table 3.** The post-quantum position of the three base layers, as of October
2026.

| | **Bitcoin** | **Ethereum** | **Cosmos** |
| --- | --- | --- | --- |
| Core problem | public-key exposure on an immutable ledger | signature size at validator scale | key-type negotiation across chains |
| Layer addressed first | output type and script | consensus signatures, and accounts separately | validator consensus keys |
| Post-quantum scheme chosen | none | hash-based, aggregated | ML-DSA-65 (FIPS 204) |
| Furthest artifact | draft BIPs | prototypes | shipped, opt-in |
| Governance | soft fork, rough consensus | coordinated upgrades | per-chain opt-in through genesis parameters |

*Notes.* "Post-quantum scheme chosen" records a scheme adopted by the protocol, not
a candidate: SHRINCS is a draft proposal for Bitcoin (section 4.1), and Ethereum's
hash-based direction rests on prototypes (section 4.2). "Furthest artifact" is the
most advanced stage reached, from specification through prototype to release.
Cosmos's "shipped" means available in a Cosmos SDK release for chains that opt in;
no chain is required to adopt it.

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
on a bridge whose covenant is emulated by a multi-signature (section 5).

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
linear in the validator set (Table 4).

**Table 4.** Signature bytes in one CometBFT commit, by validator count and key
type.

| Validators | ed25519 | ML-DSA-65 |
| --- | ---: | ---: |
| 50 | 3.2 KB | 165 KB |
| 100 | 6.4 KB | 331 KB |
| 150 | 9.6 KB | 496 KB |

*Notes.* Computed from signature sizes of 64 bytes for ed25519 and 3,309 bytes for
ML-DSA-65, assuming that every validator votes; 1 KB is 1,000 bytes. Vote
metadata, which is the same under both key types, is excluded.

One hazard generalises beyond Cosmos: an IBC light client verifies a counterparty's commit
signatures with its own compiled-in cryptography, so enabling a key type before
counterparties can verify it stops packet flow and eventually expires the
client. Migration in an interoperable ecosystem is a coordination problem before
it is a cryptographic one.

## 5. Settlement and bridge outputs

A bridge holds deposited value in base-layer outputs, and the post-quantum
security of those outputs is set by the output types and signatures the base
layer offers. On Bitcoin none is post-quantum yet: the outputs a BitVM-style
bridge uses commit a public key on chain, and only a new output type, as BIP-360
proposes, together with a post-quantum signature removes that exposure. Both are
on the base layer's timeline.

In a federated bridge these outputs are held in custody by a threshold of
signers. A BitVM-style bridge holds no custody: Bitcoin has no covenant opcode, so
the bridge emulates one. An N-of-N MuSig2 key of the signer set presigns the
transaction graph that may spend a deposit, and each signer deletes its share, so
that a single honest deletion confines the deposit to the presigned graph. The
emulation rests on the deleted key being unrecoverable, which is the assumption a
quantum adversary removes: the discrete logarithm of the aggregated public key is
a key that signs any transaction, and the presigned graph then no longer
constrains the deposit. A post-quantum emulation needs a post-quantum
multi-signature to presign with, which SHRINCS does not provide, or a script that
checks N individual post-quantum signatures, at a size linear in N. A native
covenant opcode, such as `OP_CHECKTEMPLATEVERIFY`
([BIP-119](https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki)),
would enforce the graph without a signing key, and so without the exposure,
wherever the graph can be expressed as templates. Each option is on the base
layer's timeline.

What an L2 can do unilaterally is bound the exposure. It grows with the time value
remains in such outputs, so a policy that rotates outputs on a schedule, caps the
value in any one output and shortens the lifetime of connector outputs reduces it
without waiting on the base layer. Such a policy bounds the risk rather than
removing it.

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

The pattern is as easily introduced during a migration as inherited from an
existing design. Wrapping a classical proof in a post-quantum one leaves the
classical assertion in place: a STARK proving that a Groth16 proof verified, or a
zkVM proving that an ECDSA signature verified, is forged as soon as the inner
pairing or signature is. A migration step helps only if it changes what is
asserted.

### 6.2 Generations of the on-chain verifier

The designs that bring a SNARK verifier to Bitcoin form one line of work. Each
answers the question *how is a Bitcoin script convinced that a Groth16 proof
verified*, and each moves more of the pairing arithmetic off chain. What varies
across generations is the execution of verification; the relation verified is
invariant (Table 5).

**Table 5.** Generations of the Groth16 verifier on Bitcoin.

| Generation | Mechanism | Cost addressed | Relation asserted |
| --- | --- | --- | --- |
| BitVM2 | Groth16 verifier compiled to Bitcoin script | on-chain: a Disprove script of several hundred KB, **over \$14,000** in a reported unhappy-path experiment | Groth16 / BN254 |
| [BitVM3](https://eprint.iacr.org/2026/933) | verifier circuit garbled; the dispute evaluates the garbling | on-chain cost collapses, but a garbled circuit is **42 GiB** | Groth16 / BN254 |
| [BABE](https://eprint.iacr.org/2026/065) | witness encryption for linear pairing relations, with a garbled scalar multiplication for the non-linear part | off-chain storage and setup, **~1000× lower** than BitVM3 | Groth16 / BN254 |
| [Deferred Binding](https://hackmd.io/@goatresearch/HkKp2g1Zfl) | BABE extended so that part of the public input may be fixed after garbling | usable when L2 state is known only at withdrawal | Groth16 / BN254 |

*Notes.* Costs are those reported for each design and are not measured on a
common workload. The BitVM2 and BitVM3 figures are as quoted by the BABE authors,
and the BABE figure is their reduction relative to BitVM3. The last column is
constant across the generations, so none of them changes the quantum exposure of
row 3 of Table 1.

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
the dispute protocol and a choice by cost. The integration is a rebuild rather
than the removal of a wrapper, because the garbling stacks in use are organised
around the Groth16 verifier: a garbled-circuit library whose gadgets implement
BN254 field and pairing arithmetic offers nothing to a hash-based verifier, which
needs binary-field multipliers and hash gadgets, so replacing the proof system
replaces the circuit library.

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

The measured costs, in Table 6, reverse a common expectation.

**Table 6.** Cost of moving the case-study L2's consensus keys from secp256k1 to
ML-DSA-65.

| | secp256k1 | ML-DSA-65 |
| --- | ---: | ---: |
| Verification, per signature | 173 µs | 23 µs |
| Signing, per signature | 46 µs | 196 µs |
| Commit signatures per block, 20 validators | 1.3 KB | 66 KB |
| Validator public keys in state, 20 validators | 0.7 KB | 39 KB |

*Notes.* Measurements as reported in the published design. Timings depend on
implementation and hardware and are comparable within a row only. Sizes count
signature and key bytes alone; an ML-DSA-65 public key is 1,952 bytes and a
signature 3,309 bytes.

Verification becomes faster, so the migration costs bandwidth and storage rather
than computation, linearly in the validator count: at 20 validators the commit
signatures add about 585 GB per year, and at 100 about 3 TB, so block, gossip and
storage budgets must be re-tuned together with the key type. Aggregation of
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
Ethereum's account semantics rather than diverge ahead of them. An execution
client forked from upstream inherits changes such as EIP-8141 and EIP-8151 only
after rebasing onto the release that ships them, so the fork's divergence sets the
earliest date at which they can reach the L2.

### 8.2 Precompiles

The EVM exposes elliptic-curve and pairing operations as consensus-level
precompiles: secp256k1 recovery, BN254 addition, multiplication and pairing,
BLS12-381 operations, KZG point evaluation and secp256r1 verification. A
precompile cannot be migrated by its users, since it is a consensus rule and its
callers are often immutable. What decides whether an exposure can be migrated is
the caller set rather than the precompile (Table 7).

**Table 7.** Migratability of the EVM's elliptic-curve and pairing precompiles,
determined by their caller sets.

| Precompile | Caller set | Migratable |
| --- | --- | --- |
| `ecrecover` | every account, gated by per-account state | yes: narrow it, as EIP-8151 does |
| KZG point evaluation | rollup contracts: bounded, known and maintained | plausibly: coordinate the set |
| BN254 and BLS12-381 pairing | every SNARK verifier deployed: unbounded and largely unmaintained | no |

*Notes.* A precompile is migratable if its exposure can be removed without
breaking a caller that cannot be changed. The rows are representative: the BN254
and BLS12-381 group operations share the caller set of the pairing row.

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
garbled verification. Table 8 maps its surfaces onto the taxonomy.

**Table 8.** The surfaces of GOAT against the taxonomy of Table 1 and the roadmap
of Table 2.

| Row | Surface | Current scheme | Quantum status | Roadmap stage | Owner |
| --- | --- | --- | --- | --- | --- |
| 1 | Settlement outputs | secp256k1 / Schnorr | broken | bounded only | Bitcoin |
| 2 | Peg-in covenant | MuSig2 N-of-N, presigned | broken | bounded only | GOAT, within Bitcoin's output types |
| 3 | Bridge proof system | Ziren STARK, Groth16/BN254 wrap, garbled verifier and witness encryption | broken at the wrap and the verifiers built on it | replacement introduced and measured | GOAT, with ZKM |
| 4 | Bridge commitments | Winternitz one-time signatures | post-quantum | — | — |
| 5 | Consensus keys | secp256k1 | broken until rotated | design published | GOAT |
| 6 | Execution accounts | secp256k1 ECDSA | broken | follows Ethereum | Ethereum, inherited |
| 7 | EVM precompiles | upstream set | broken for pairing and curve precompiles | inventory | Ethereum, inherited |

*Notes.* Row numbers follow Table 1. "Broken" is conditional on a
cryptographically relevant quantum computer. "Roadmap stage" is the furthest step
of Table 2 reached as of October 2026. "Bounded only" means that no replacement
exists yet and exposure is limited by the output policy of section 5.

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

## 10. Conclusion

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
