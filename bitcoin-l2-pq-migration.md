# Post-quantum migration for Bitcoin layer 2s

### Abstract

A Bitcoin layer 2 has a post-quantum problem that neither Bitcoin nor Ethereum
has alone. It settles to a base layer it cannot change, borrows consensus from a
third ecosystem, and operates a bridge whose trust root is cryptography of its
own choosing. The goal is to make both the L2 and the base layer post-quantum
safe, but only one of those is the L2's to schedule, so the work divides into
surfaces it can act on and surfaces where it can only bound exposure and press
upstream.

Part I sets out that structure and a seven-row surface taxonomy keyed on *who
can fix each row*. Part II surveys the three base layers an L2
inherits from, where the central finding is that progress inverts exposure:
Bitcoin, with the sharpest exposure, has specified no post-quantum signature
scheme at all, while Cosmos has one in shipped code. Part III reads one stack,
GOAT, in depth as the evidence the framework was derived from. Part IV gives
an ordering. Part V collects the recurring pitfalls and the remaining open
items. The roadmap that follows the abstract summarises the whole upgrade.

The finding of most practical consequence is that symmetric and hash-based
layers protect only what they carry. In the stack examined, the Bitcoin-side
commitment layer, the garbling layer and the witness encryption that replaces
the on-chain verifier are all post-quantum, yet the bridge is not, because each
of them carries an elliptic-curve assertion. Section 9 follows the pattern
through four generations of the same component and finds it stated in the
security proof of the newest one. Section 10 reports measured costs of
hash-based replacement verifiers, both in Bitcoin script and as garbled
circuits.

### Roadmap

The upgrade is organised in two dimensions. Each surface passes through the four
stages of a cryptographic migration that
[StarkWare's network comparison](https://quantum.starkware.co/networks) derives
from [NIST CSWP 39](https://csrc.nist.gov/pubs/cswp/39/upd1/considerations-for-achieving-crypto-agility/final):
*identify dependencies*; *introduce and test* the replacement; *move users and
data*; and *retire old paths*, so that the vulnerable authorisation path is
disabled and cannot be forced back into use. The surfaces are taken in the order
of section 14, which follows ownership and severity. The table states, for each
surface of the GOAT stack, where each stage stands and which party gates it.

| Phase | Surface | Identify dependencies | Introduce and test | Move users and data | Retire old paths | Gated by |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | All surfaces | Inventory of signature and proof paths, including fixed-length checks (section 15) | — | — | — | GOAT |
| 1 | Peg custody (rows 1–2) | Taproot key path, exposed from output creation (section 11) | No post-quantum signature exists on Bitcoin | Bound exposure: rotate custody outputs, cap value per output, shorten connector lifetimes | P2MR (BIP-360) and a post-quantum signature BIP; legacy sunset (BIP-361) | **Bitcoin** |
| 2 | Bridge proof system (row 3) | Groth16 wrap, garbled Groth16 verifier, BABE relation (sections 8 and 9) | Ziren's garbled-verifier route; hash-based garbled verifier measured (section 10) | Move the dispute path to the hash-based verifier and rebuild `bitvm-gc` around it | Disable the pairing-based dispute path; not yet designed | **GOAT**, with ZKM |
| 3 | Consensus keys (row 5) | secp256k1 validator keys (section 12) | ML-DSA-65 key type, through a CometBFT and Cosmos SDK upgrade at a fork height; design published | Per-validator rotation, one per retention period, with mixed sets allowed | Disallowing new secp256k1 consensus keys is not in the published design; history signed before activation remains forgeable | **GOAT** |
| 4 | Execution accounts (row 6) | Public keys revealed by use; signature authority accepted by contracts (sections 5 and 13) | Upstream: frame transactions and verification precompiles | Each holder migrates its own account | Two revocations: transaction origination and `ecrecover` authority (section 5) | **Ethereum**, delivered by closing `goat-geth`'s lag |
| 4 | EVM precompiles (row 7) | Callers of the pairing and KZG precompiles, and whether each is upgradeable (section 13) | Upstream post-quantum precompiles, additive | Upgradeable callers move | Immutable callers cannot be retired | **Ethereum** |
| — | Bridge data commitments (row 4) | Winternitz one-time signatures, hash-based (section 8) | — | — | — | — |

No surface has reached the fourth stage. GOAT controls every stage of the
consensus keys and of the bridge proof system, the latter with ZKM's prover; it
can only bound exposure on the custody surfaces until Bitcoin acts, and it
inherits the execution-layer stages from Ethereum. The final stage is the one
most often omitted: a surface on which a new algorithm has been introduced but
the old path has not been retired is no more secure than before, since an
adversary uses the old path.

### Sources and verification

The analysis is drawn from primary sources: BIP text from
[`bitcoin/bips`](https://github.com/bitcoin/bips),
[`UPGRADING.md`](https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md)
from [`cosmos/cosmos-sdk`](https://github.com/cosmos/cosmos-sdk), the
[GOAT repositories](https://github.com/GOATNetwork) and their dependency trees
read through the GitHub API and local clones, and, for section 9, the BABE
and Argo MAC papers alongside GOAT's own
[Deferred Binding design doc](https://github.com/GOATNetwork/bitvm-gc/blob/main/docs/partial_binding_we.tex).
Claims are stated as verified; checks still outstanding are listed under
*Remaining open items* (section 16).

---

## Part I: The general problem

### 1. Why a Bitcoin L2 is a distinct problem

Most post-quantum blockchain analysis treats a chain as a single cryptographic
system: one signature scheme securing funds, one consensus mechanism, one
governance process to change them. A Bitcoin L2 is not that. It is a composition
of systems that were designed separately, migrate on separate timelines, and are
owned by separate parties.

That composition produces a problem neither Bitcoin nor Ethereum has alone:

- The L2 cannot fix its own base layer. Bitcoin has specified no
  post-quantum signature scheme at all:
  [BIP-360](https://github.com/bitcoin/bips/blob/master/bip-0360.mediawiki)
  (Draft) says in its own text that it "does not include the introduction of
  post-quantum signature schemes", and
  [BIP-361](https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki)
  (Draft, Informational) lists `Requires: TBD Post Quantum Signature
  BIP`, a document that does not exist. Any BTC in a classical L1 output is on
  Bitcoin's timeline, not the L2's.
- The L2 usually borrows a consensus stack from a third ecosystem, and
  inherits that ecosystem's migration path, maturity and constraints.
- The L2 owns a bridge, whose trust root is cryptography of its own choosing,
  and it is the highest-value target in the system.

So the goal is *make both the L2 and the base layer post-quantum safe*, while
recognising that the L2 can schedule only half of it. The rest of this report
separates the surfaces it owns from the ones where the available course is to
bound exposure and track upstream roadmaps.

### 2. The surface taxonomy

Any Bitcoin L2 that settles to L1 and runs its own consensus has, at minimum,
these cryptographic surfaces. The taxonomy makes the ownership question
explicit before the scheme-selection question.

| # | Surface | Typical scheme | Fails to Shor? | Who can fix it |
| --- | --- | --- | --- | --- |
| 1 | L1 settlement outputs | secp256k1 / Schnorr | yes | **base layer only** |
| 2 | Bridge attestation / operator set | secp256k1, Schnorr, MuSig2, BLS | yes | **the L2** |
| 3 | Bridge proof system | Groth16, PLONK (pairing-based) or STARK (hash-based) | if pairing-based, including a pairing-based wrap of a STARK | **the L2**, with its proof-system provider |
| 4 | Bridge data commitments | Merkle / Winternitz / Lamport | **no (hash-based)** | already safe |
| 5 | L2 consensus keys | ed25519, BLS12-381 | yes | **the L2**, via its consensus stack |
| 6 | L2 execution accounts | secp256k1 ECDSA | yes | the L2, at tooling cost |
| 7 | **EVM crypto precompiles** | ecrecover, BN254 pairing, BLS12-381, KZG | yes | **upstream, and unfixable for deployed callers** |

The point of the grouping is that three different parties hold the fix,
and only the middle group is the L2's to schedule.

---

## Part II: The base layers an L2 inherits from

*Bitcoin, Ethereum and Cosmos are usually discussed as though they face one
shared problem and differ only in speed. They do not: they are solving
structurally different problems, and their relative progress inverts the usual
assumption.*

### 3. The three base-layer problems compared

| | **Bitcoin** | **Ethereum** | **Cosmos** |
| --- | --- | --- | --- |
| Core problem | public-key exposure on an immutable ledger | signature size at validator scale | key-type negotiation across chains |
| Layer attacked first | output type / script | consensus signatures, and accounts separately | validator consensus keys |
| PQ scheme chosen | **none yet** | hash-based (XMSS/Winternitz family) | **ML-DSA-65 (FIPS 204)** |
| Furthest artifact | Draft BIPs | working prototypes | **shipped, opt-in** |
| Governance style | soft fork, rough consensus | coordinated upgrades | per-chain opt-in + genesis params |

An independent comparison by StarkWare, which it discloses as the developer of
one of the networks assessed, reaches compatible placements for the two chains
it covers: Bitcoin and Ethereum both fall in its higher-exposure group, Bitcoin
with the stronger agility score but a migration that ends in a mandatory
interruption for accounts that do not act, Ethereum with a weaker score and a
scheduled interruption ([StarkWare, *Networks*](https://quantum.starkware.co/networks)).
Cosmos chains are not covered.

### 4. Bitcoin: an exposure problem, with the signature question deferred

Two Draft BIPs sit in the canonical
[`bitcoin/bips`](https://github.com/bitcoin/bips) repository. Neither is
activated.

[BIP-360, "Pay-to-Merkle-Root" (P2MR)](https://github.com/bitcoin/bips/blob/master/bip-0360.mediawiki), `Layer: Consensus (soft fork)`,
`Status: Draft`, v0.12.0, `Requires: 340, 341, 342`. It proposes an output type
that is Taproot with the key-path spend removed, so no bare public key is ever
committed on chain. The BIP is explicit about the limits of that: protection
"does not depend on the activation of post-quantum signatures", it defends
against *long* exposure only, and "P2MR does not, by itself, protect against
short exposure quantum attacks". Most decisively, the text states:
"While this proposal does not include the introduction of post-quantum
signature schemes"; the authors are "currently researching options".

[BIP-361, "Post Quantum Migration and Legacy Signature Sunset"](https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki),
`Status: Draft`, `Type: Informational`, assigned 2026-02-11. It proposes a
pre-announced sunset of legacy ECDSA/Schnorr, framing quantum security as "a
private incentive". Its `Requires` field reads: "TBD Post Quantum Signature
BIP".

That unresolved dependency is the central observation. Bitcoin has a hardening step and a deadline
plan, and the deadline plan formally depends on a document that does not yet
exist as a BIP. Checked again on 2026-07-31: the BIPs index still contains exactly one
post-quantum entry, BIP-361 itself. The exposure is not hypothetical: BIP-361
states that as of 1 March 2026 over 34% of all bitcoin have revealed a public
key on chain.

The sunset is not a blanket freeze. BIP-361's Phase A (160,000 blocks, roughly three years after
activation) stops sends to quantum-vulnerable address types. Phase B, two years
later, does not simply invalidate legacy signatures: it *encumbers* ECDSA and
Schnorr spends with a rescue protocol designed "to rule out quantum
attackers, but to permit spends from the authentic coin-holders". The mechanism
is an asymmetry of knowledge: most wallets since 2012 derive keys through
BIP-32 hardened derivation, so a holder can prove knowledge of a parent extended
private key that a quantum attacker recovering only the child key would not
have. The BIP points at ZK-STARK-based rescue protocols and at commit/reveal
schemes for this.

Two qualifications, both from the BIP itself. Coverage is unresolved: "it
remains to be seen how much of the legacy Bitcoin supply can be theoretically
covered by such rescue protocols", and only if the majority is covered will the
restriction be "at most mildly confiscatory". And P2PK has no rescue
protocol: "it's not currently believed possible to construct a rescue
protocol for P2PK UTXOs, as no knowledge asymmetry is known", which is why
BIP-361's authors separately support an Hourglass-style proposal for those
outputs. So the coins that genuinely become unspendable are the abandoned ones
and the ones where no secret asymmetry exists, not the legacy supply as a
whole.

#### The SHRINCS draft specification

A candidate for that missing document was posted to the bitcoin-dev list on
2026-08-27: [SHRINCS](https://groups.google.com/g/bitcoindev/c/HbVboXIFiG8)
("Shrunken SPHINCS"), a hash-based scheme published as a
[draft BIP](https://github.com/SHRINCS/shrincs-bip/blob/main/SHRINCS.md) by a
working group drawn from Blockstream, Brink, Cloudflare, OpenSats and OpenChain.
The draft specifies cryptography only. It states that deployment cannot proceed
"without introducing at least one new output type, which we do not define in this
BIP", carries the disclaimer "Do NOT use in production", and lists test vectors,
unit tests, a security proof and an optimised implementation as outstanding. The
dependency has therefore moved from a document that does not exist to one that
exists in draft and is explicitly incomplete.

The construction places two signature schemes under one 48-byte public key and
accepts a signature from either: a *stateful* path, a flexible XMSS tree (FXMSS)
whose leaves are WOTS+C one-time keys, at 548 to 4,619 bytes; and a *stateless*
path, SLH-DSA under a non-standard parameter set, at 5,777 bytes. The stateful
path is primary, the stateless one a fallback for when signing state is lost or
uncertain. The minimum combined public key and signature is 596 bytes, which the
draft records as 13.23 times smaller than SLH-DSA-SHA2-128s and 6.26 times
smaller than ML-DSA-44.

The stateless path is SLH-DSA as specified in FIPS-205, with its signing and
verification algorithms matching Algorithms 22 and 24, so an implementation
supporting custom parameter sets can be reused. The parameters cut the hypertree
from seven layers to five and reshape FORS from fourteen trees of height twelve
to ten of height thirteen, reducing the signature budget from 2⁶⁴ to 2⁴⁰ and
the signature from 7,856 to 5,776 bytes; 1,408 bytes of that saving come from the
shorter hypertree and 672 from the reshaped FORS. The draft notes that SPHINCS+C
or PORS+FP would give a further 15% at the same budget, and declines it to retain
FIPS-205 implementation reuse and its existing analysis.

The stateful path is not SPHINCS-derived. Its one-time signature is WOTS+C, which
replaces the Winternitz checksum with a constant-sum encoding reached by grinding
a 16-bit counter. That removes the three checksum chains, leaving 32 rather than
the 35 of WOTS-TW, and makes the verifier's work identical in the worst and
average case. FXMSS then leaves the tree shape to the signer, recorded as two
bytes of shape and depth in the secret key and bound into secret-key derivation
but not into verification: the verifier recomputes the root from the leaf
position and authentication path alone, and has one code path for every shape.
An unbalanced tree gives the smallest early signatures, 548 bytes, growing by 16
or 17 bytes each as the authentication path lengthens and exhausting after
depth + 1 signatures; a balanced tree of depth d gives 2ᵈ signatures of
constant size, roughly 660 bytes at d = 8 and 854 at d = 20, at a
key-generation cost that doubles with each level.

The property that motivates the design for Bitcoin is verification cost per
signature byte. The draft measures 0.465 SHA-256 compressions per byte for the
stateful path and 0.483 for the stateless one, against 1.98 for BIP-340 Schnorr
on the same benchmark, and states that this "leaves room for a witness discount
that partly compensates for the larger signature size". The stateful parameters
are chosen to hold that ratio close to the stateless one, so a single discount
rule would cover both paths. Key generation and signing are correspondingly
expensive: 3.1 × 10⁵ to 5.5 × 10⁸ compressions for key generation, and
1.7 × 10⁶ for an average stateless signature.

Statefulness is the cost. Reusing a state counter under one key lets anyone who
observes both signatures forge, and the draft's rules are correspondingly strict:
counters must not be backed up, restored, exported or used concurrently, and must
be incremented in persistent storage before the signature is returned. Two
mitigations distinguish the design from XMSS and LMS, which SP 800-208 declares
"not suitable for general use". The stateless fallback is present in every key
pair, so lost state costs signature size rather than access to funds, and an
implementation that cannot establish its counter must refuse the stateful path.
And each stateful signature discloses the leaf it used, so a repeated leaf under
one public key is detectable before the second signature is broadcast.

Two absences bear on the rest of this report. SHRINCS has no algebraic structure
supporting public-key rerandomisation, so BIP-32 extended public keys have no
direct analogue, and none supporting multi-signatures, so there is no MuSig-style
aggregation. The peg custody of section 11 rests on exactly that second property,
which places its post-quantum replacement outside what a hash-based output type
provides on its own.

### 5. Ethereum: an aggregation problem at consensus, an account problem above it

Ethereum's exposure is ordinary (accounts reveal a public key on first spend,
consensus uses BLS12-381), but its *constraint* is not. Hash-based signatures
are the most conservative post-quantum option available, resting only on hash
security, but at roughly 3 KB each they cannot go on chain one per validator.
Ethereum therefore treats post-quantum migration substantially as an
aggregation-engineering problem.

The concrete artifacts are named and public.
[`leanEthereum/leanVM`](https://github.com/leanEthereum/leanVM) describes
itself as a "minimal hash-based zkVM, for a Post-Quantum Ethereum" and exists to
do recursive aggregation.
[`leanEthereum/leanSig`](https://github.com/leanEthereum/leanSig) is a Rust
prototype of the proposed signature scheme, built on tweakable hash functions
and incomparable encodings, and grew out of the research implementation
[`b-wagn/hash-sig`](https://github.com/b-wagn/hash-sig)
([eprint 2025/055](https://eprint.iacr.org/2025/055)). `leanSig`'s README is
explicit that the code is unaudited and not for production.
[`pq.ethereum.org`](https://pq.ethereum.org/) is the coordination hub.

That is only the consensus front. Accounts have their own two-track story,
and it is the half that concerns user funds:

- an emergency track: Buterin's proposal to hard-fork on evidence of
  large-scale theft, revert to the last pre-theft block, freeze legacy ECDSA
  accounts, and let users recover through a transaction carrying a STARK proof
  of knowledge of the hash preimage behind their address, authenticating by a
  secret that was never exposed on chain;
- a structural track: account abstraction as the migration vehicle.
  [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) ("Set Code for EOAs",
  Final, shipped in Pectra) lets an EOA delegate to contract code, though its
  authorisations are themselves secp256k1-signed, so it is a stepping stone
  rather than a post-quantum mechanism. The dedicated vehicle is
  [EIP-8141](https://eips.ethereum.org/EIPS/eip-8141) ("Frame Transaction",
  Draft, created 2026-01-29), which supports a slot for an arbitrary
  post-quantum signature scheme alongside the classical ones.

At the execution layer the mechanism is deliberately scheme-agnostic.
Rather than adding an ML-DSA or Falcon precompile (which would repeat the
`bn256Pairing` mistake of welding a precompile to one construction),
[EIP-7885](https://eips.ethereum.org/EIPS/eip-7885) ("Precompile for NTT
operations", Draft, created 2025-02-12) exposes the *shared primitive*. Its
motivation states the reasoning directly: "Choosing to integrate NTT and InvNTT
instead of a specific algorithm provides agility, as DILITHIUM or FALCON or any
equivalent can be implemented with a modest cost from those operators", and the
same operator "is also of interest to speed-up STARK verifiers", so one
precompile serves both scaling and the quantum transition. This introduces
crypto-agility at the layer where it was previously absent.

A cost analysis of these proposals
([AppliedPQC, *Post-Quantum Migration in Ethereum's Execution Layer*](https://github.com/AppliedPQC/pqc-research/issues/35))
adds two findings that bear on any EVM chain. First, the cost of a post-quantum
transaction lies in its bytes rather than its verification: with a verification
precompile, verification is a few percent of the transaction's gas, and the
signature's size sets the rest, which favours the smallest schemes for
transactions that carry their own signatures. Second, obtaining a post-quantum
key does not complete a migration. The old key retains two independent powers,
originating transactions from the address and authorising actions through
`ecrecover` in contracts, and each requires its own revocation:
[EIP-8298](https://eips.ethereum.org/EIPS/eip-8298) lets an account install code,
after which [EIP-3607](https://eips.ethereum.org/EIPS/eip-3607) rejects its ECDSA
transactions, and [EIP-8151](https://eips.ethereum.org/EIPS/eip-8151) restricts
`ecrecover` for such accounts. Until both ship, abandoning the address is the
only action that removes both powers.

Nothing post-quantum has shipped to Ethereum mainnet on any of these fronts;
EIP-7885, EIP-8141, EIP-8298 and EIP-8151 are all Draft.

### 6. Cosmos: key-type negotiation, with a shipped implementation

Cosmos SDK v0.55 registers
[ML-DSA-65 (FIPS 204)](https://docs.cosmos.network/sdk/latest/keys/post-quantum-keys)
as a supported validator consensus key type. This is shipped code, not a
proposal ([PR #26436](https://github.com/cosmos/cosmos-sdk/pull/26436)): the
key type, its wiring and its gas parameters ship enabled by default, and the
chain tooling accepts it at initialisation.

It is deliberately *opt-in*: the set of validator key types a chain permits
still defaults to ed25519 alone, so nothing changes for an existing chain until
its governance says so. Existing chains can combine the new key type with
validator consensus key rotation, which ships enabled in the same release, to
move validators onto post-quantum keys without a new genesis.

The size consequences are stated plainly and match FIPS 204 exactly: public keys
grow from 32 to 1952 bytes and signatures from 64 to 3309 bytes, roughly 60x and
50x. CometBFT's per-signature and per-validator commit budgets were raised to
accommodate them, and chains are told to revisit block-size and gossip framing
limits.

And the cost is linear, because CometBFT does not aggregate: a commit carries
one signature for each validator that voted, in validator-address order. There
is no aggregate to grow; there is an array whose length is the validator set.
Signature bytes per commit therefore scale as N × 3309:

| Validators | ed25519 | ML-DSA-65 |
| --- | --- | --- |
| 50 | 3.2 KB | 165 KB |
| 100 | 6.4 KB | 331 KB |
| 150 | 9.6 KB | 496 KB |

CometBFT's response to that arithmetic is more deliberate than the upgrade note
suggests. The maximum signature size is not a fixed number but a maximum taken
over every supported key type, so admitting ML-DSA-65 to the key-type set
raises it from 96 to 3309 bytes globally. The default block-size parameter then
budgets the worst-case commit *separately from application data*: for a
maximum-size validator set of 10,000, that is 21 MiB of data plus roughly
32 MiB of commit headroom, about 53 MiB in total. The design choice is to make
the commit budget explicit and size it for the worst case, rather than leave
operators to discover the interaction; the source comments reason directly
about ML-DSA-65's 3309-byte signatures, so this was sized for post-quantum
keys on purpose.

One consequence is less obvious: the maximum is
over key types the build *supports*, not the ones a chain *enables*. Every chain
on a CometBFT build that supports ML-DSA-65 inherits the larger default block size whether
or not its validators ever use a post-quantum key. The cost of the option is
paid by everyone; only the per-commit bytes are paid by adopters.

[`UPGRADING.md`](https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md)
does not mention aggregation anywhere, and the upstream work to
add it has not landed: CometBFT issue
[#3455](https://github.com/cometbft/cometbft/issues/3455) (BLS signature
aggregation) has been open since 2024, issue
[#1305](https://github.com/cometbft/cometbft/issues/1305) (halving commit size
with partial ed25519 signatures) since 2023, and two pull requests adding
aggregation to `crypto/bls12381`,
[#3632](https://github.com/cometbft/cometbft/pull/3632) and
[#4763](https://github.com/cometbft/cometbft/pull/4763), were both closed without
merging. BLS12-381 exists in CometBFT as a *key type*, not as aggregated commits.
Two abandoned attempts is evidence about difficulty, not only about priority.

A finding that generalises beyond Cosmos is an IBC warning in the Cosmos
changelog that has no analogue in the Bitcoin or Ethereum material.
IBC light clients on a counterparty chain verify your validator set's commit
signatures using *the counterparty's own compiled-in crypto*. If validators
holding sufficient voting power sign with a key type a counterparty cannot
verify, headers fail verification there, packet flow stops, and the light client
eventually expires. The counterparty needs the verification code, not the key
type. Post-quantum migration in an interoperable ecosystem is therefore a
*coordination* problem before it is a cryptographic one, and the failure mode
is silent until connectivity breaks.

---

## Part III: The GOAT stack as evidence

### 7. Summary of findings

Five of the six surfaces below carry quantum-exposed cryptography, and they are
not equally GOAT's to fix. Row numbers refer to the taxonomy of section 2, so
that the case study and the framework can be read against each other.

| Row | Surface | Repo | Scheme | Quantum status | Owner of the fix |
| --- | --- | --- | --- | --- | --- |
| 1 | Bitcoin L1 outputs | — | secp256k1 / Schnorr | broken by Shor | **Bitcoin, not GOAT** |
| 1–2 | Peg custody | [`bitvm2-node`](https://github.com/GOATNetwork/bitvm2-node) | MuSig2 over a **Taproot key path** | broken by Shor, **and exposed from output creation** | **GOAT** |
| 3 | Bridge proof system | [`bitvm2-node`](https://github.com/GOATNetwork/bitvm2-node) | [Ziren](https://github.com/ProjectZKM/Ziren) STARK → **Groth16/BN254** wrap → garbled verifier | broken at the wrap and at the verifiers built on it | **GOAT** |
| 4 | Bridge bit commitments | [`bitvm2-node`](https://github.com/GOATNetwork/bitvm2-node) → [BitVM](https://github.com/GOATNetwork/BitVM) | **Winternitz OTS** | **already PQ-safe** | — |
| 5 | Consensus keys | [`goat`](https://github.com/GOATNetwork/goat) (CometBFT) | **secp256k1**, with rotation to ML-DSA-65 designed | broken by Shor until rotated | **GOAT** |
| 6 | EVM accounts | [`goat-geth`](https://github.com/GOATNetwork/goat-geth) | secp256k1 ECDSA | broken by Shor | GOAT, at tooling cost |

Peg custody spans two rows: the output is an L1 settlement output, but its
*construction* is GOAT's choice, which is what makes the mitigation in section 11
available without waiting on Bitcoin.

The principal result is in rows 3 and 4. BitVM2's Bitcoin-side plumbing is already
post-quantum; its cryptographic content is not. That reframes the work from a
rewrite into two contained swaps.

### 8. bitvm2-node: hash-based transport of elliptic-curve content

[`bitvm2-node`](https://github.com/GOATNetwork/bitvm2-node) is the bridge
node: a Bitcoin light client and proof pipeline with circuits for the header
chain, state chain, commit chain and operator claims. Its declared
cryptographic dependencies are the Groth16 proof system over the
pairing-friendly curve BN254, MuSig2 over secp256k1, SHA-2, and GOAT's fork of
the BitVM script library.

Consider first the quantum-vulnerable components. Groth16 over BN254 is a *pairing-based* proof system.
Pairing-friendly curves fall to Shor exactly as ECDSA does, so a
cryptographically relevant quantum computer could forge bridge proofs, not
merely steal keys. That is a more severe failure mode than key theft: it
compromises the bridge's soundness rather than an individual's funds. MuSig2
over secp256k1, used for peg-out authorisation, is broken in the ordinary way.

The remaining layer is hash-based, and it is structurally significant. BitVM2 carries values
between Bitcoin script fragments using *bit commitments* implemented as
Winternitz one-time signatures, hash-based and therefore post-quantum on the
same conservative assumption as SLH-DSA. The implementation lives in GOAT's
[BitVM](https://github.com/GOATNetwork/BitVM) fork and is referenced throughout
the bridge node. (Establishing this required enumerating the git tree directly:
GitHub code search does not index forks, a hazard recorded among the pitfalls
of section 15.)

Groth16 is a wrapper rather than the proving system. `bitvm2-node` depends on
[Ziren](https://github.com/ProjectZKM/Ziren), a zkVM for MIPS32 over the
KoalaBear field ([Duan et al.](https://eprint.iacr.org/2026/2330)). Each shard of an execution is proved with a LogUp-GKR bus
argument, a zerocheck and a jagged commitment opened with WHIR, all under
Poseidon2, and recursion composes the shard proofs into one compressed proof.
From that proof Ziren defines two verification routes: a pairing-based SNARK
wrap, Groth16 or PLONK over BN254, for verifiers that check pairings; and a
garbled-verifier route for Bitcoin rollups, in which a binary stage over
GF(2^128) with Blake3 and a narrow recursion reduce the proof to one whose
verifier is a compact Boolean circuit. `bitvm2-node` uses the first. Its Groth16
proof elements are committed into Bitcoin script through Winternitz signatures,
and the Groth16 verifier is what BitVM2 executes.

The garbling stack is organised around Groth16 specifically.
[`bitvm-gc`](https://github.com/GOATNetwork/bitvm-gc) is not a general circuit
garbler applied to Groth16: its principal component is a garbled *SNARK
verifier*, and its circuit tree is organised around the Groth16/BN254
verification circuit. Replacing the proof system therefore entails rebuilding
the garbled-circuit stack around a different verifier rather than removing a
wrapper, and it is the largest single item in this report.

Downstream of the STARK, the proof path comprises the following layers:

| Layer | Primitive | Quantum status | Where the fix lives |
| --- | --- | --- | --- |
| Groth16 wrap | **BN254 pairing** | **broken** | GOAT |
| `bitvm-gc` garbled verifier | AES-128 garbling of a **Groth16 verifier** | garbling safe, **statement broken** | GOAT, as a rebuild |
| BABE / Deferred Binding | witness encryption **against the Groth16 relation** | on-chain surface safe, **relation broken** | GOAT; see section 9 |
| WOTS commitment into script | hash-based | safe | — |

The three middle layers each assert, or are keyed to, the BN254 pairing
relation. Because the garbled verifier and the witness encryption are both
constructed around that relation, removing the wrap requires replacing all three
together. Ziren's second route is that replacement: the garbled verifier then
checks a hash-based proof rather than a pairing equation, and section 10 reports
its measured cost. The last of them is treated separately in section 9, since the
on-chain verifier has passed through four designs that are best read as a
single line of work.

The principal cost of replacement is size. Hash-based proofs are substantially
larger than Groth16's constant-size proof, and BitVM2's economics depend on what
fits in Bitcoin script and on the cost of a challenge transaction. Section 10
reports the measured costs.

### 9. The on-chain Groth16 verifier: script, garbling, witness encryption

The designs treated here form a single line of work. Each answers the
same question, *how is a Bitcoin script convinced that a Groth16 proof
verified*, and each generation pushes the pairing arithmetic further off chain.
Read together, they show that what varies across generations is the *execution*
of verification, while the *assertion* being verified is invariant.

| Generation | Mechanism | Cost it attacks | Relation asserted |
| --- | --- | --- | --- |
| BitVM2 | Groth16 verifier compiled to Bitcoin script | on-chain: Disprove script of several hundred KB, **over \$14,000** in a recent unhappy-path experiment | Groth16 / BN254 |
| [BitVM3](https://eprint.iacr.org/2026/933) | verifier circuit *garbled*; dispute evaluates the garbling | on-chain cost collapses, but the garbled circuit is **42 GiB** | Groth16 / BN254 |
| [BABE](https://eprint.iacr.org/2026/065) | witness encryption for linear pairing relations, plus a garbled EC scalar-mul for the non-linear part | off-chain storage and setup, **~1000× lower** than BitVM3 | Groth16 / BN254 |
| [Deferred Binding](https://hackmd.io/@goatresearch/HkKp2g1Zfl) (GOAT) | BABE extended so part of the public input may be fixed *after* garbling | makes BABE usable when L2 state is only known at peg-out | Groth16 / BN254 |

The right-hand column is constant across all four generations; the remainder
of this section documents that invariance.

#### The design lineage and GOAT's position in it

[BABE](https://eprint.iacr.org/2026/065) (Garg, Kolonelos, Sergeevitch, Sridhar
and Tse) states the motivation plainly: BitVM2 "suffers from very high on-chain
Bitcoin transaction fees in the unhappy path (over \$14,000 in a recent
experiment)", and BitVM3 fixes that "but each garbled circuit is 42 Gibytes in
size, so the off-chain storage and setup costs are huge". BABE keeps BitVM3's
on-chain savings and "reduces its off-chain storage and setup costs by three
orders of magnitude", using "a witness encryption scheme for linear pairing
relations to verify Groth16 proofs", augmented, because "Groth16 verification
involves non-linear pairings", with a 2PC protocol built on "a very efficient
garbled circuit for scalar multiplication on elliptic curves". That garbled
circuit builds on [Argo MAC](https://eprint.iacr.org/2026/049) (Eagen and Lai),
which "enables over 1000× more efficient garbled SNARK verifiers". Both encodings
were then improved again by
[Duty-Free Bits](https://eprint.iacr.org/2026/476) (Khambhati, Bhattacharya and
Heath), which reports "we improve BABE's encoding size by 45×, and Argo MAC's by
20×".

GOAT's contribution sits on top of that stack. Vanilla BABE requires the full
public input to be fixed before garbling, which the bridge cannot do: the
dynamic part `x_D` (L2 state, sequencer commitments, watchtower data) is only
known at peg-out, so by then the committed `vk_x` is stale and decryption fails.
[Deferred Binding](https://hackmd.io/@goatresearch/HkKp2g1Zfl), whose formal
write-up defines the primitive it rests on,
[partial-binding witness encryption](https://github.com/GOATNetwork/bitvm-gc/blob/main/docs/partial_binding_we.tex),
resolves this with a Dual-Scalar Garbled Circuit whose two outputs,
`r * pi_1` and `r * P_D + r * B`, "provide prefix binding on `x_S`
and proactive cryptographic binding on `x_D`", with `B` a verifier-chosen
blinding point hidden inside the circuit. The construction is implemented: the
`main` branch of [`bitvm-gc`](https://github.com/GOATNetwork/bitvm-gc) carries
the BABE implementation alongside the older garbled verifier.

#### The relation the scheme encrypts against

The design doc fixes the relation in its first paragraph: a Type-III pairing on
BN254, with the verifier accepting iff

```text
e(pi_1, pi_2) = e(alpha, beta) * e(vk_x, gamma) * e(pi_3, delta_2)
```

A ciphertext masks the payout secret under a key derived from the verification
equation's constants; decryption recovers that key by pairing the proof
elements, a step *justified by the Groth16 equation above*, and then unmasks
the secret.

So the witness that unlocks the payout is a valid Groth16 proof over BN254, and
the correctness property of the scheme guarantees that anyone holding one can
decrypt. This behaviour is the scheme's intended correctness property; it is
also the source of the post-quantum exposure.

The on-chain side is, as claimed, hash-based. The doc is explicit that Bitcoin
script is used only for hash commitments to the ciphertext components,
gate-by-gate garbled-circuit unlocking, and a hashlock on the payout secret
gating the final UTXO spend, concluding: "No pairings or target-group arithmetic are evaluated
on chain." Every on-chain primitive here (hashlocks, Lamport commitments, the
garbling) survives Shor. That is what makes the composition easy to
misclassify.

#### The security reduction and its discrete-logarithm terms

No inference is required here: GOAT's own security analysis states the
reduction directly:

```text
Adv_pbWE(lambda)  <=  Adv_BABE(lambda)  +  Adv_DLog[G1](lambda)
```

with a companion lemma bounding leakage of the secret scalar `r` by
`Adv_DLog[G1] + Adv_coDLog`,
because "public exposures of `r`-scaled samples ... are multi-instance DLog
challenges with a shared scalar". The conclusion states the assumption set:
"The reduction stays within standard BABE assumptions plus GC-security and
Lamport one-wayness."

Against a cryptographically relevant quantum computer, `Adv_DLog[G1]` on BN254
goes to 1, and
`Adv_BABE` rests on the same curve. The bound does not
degrade gracefully: it becomes vacuous. Concretely, an adversary who can solve
discrete log on BN254 forges a Groth16 proof for a peg-out that never happened,
decrypts the payout secret through the scheme's own correctness property, and
spends the hashlocked UTXO. No garbling is broken, no hash is inverted, and no
step of the protocol misbehaves.

This is the third instance of the wrapper pattern in the stack. The first two,
Winternitz commitments carrying a Groth16 proof and garbled circuits garbling a
Groth16 verifier, are recognisable as wrappers. In this case the terminology
suggests the opposite: witness encryption, garbled circuits, hashlocks and
Lamport commitments are post-quantum primitives, three of the four on-chain
primitives are post-quantum, and the security proof is a reduction. The pairing
persists in the *relation being encrypted against*, which the terminology does
not indicate.

#### The zkVM on the dispute path

Deferred Binding requires *soldering*, the translation of garbled labels across
cut-and-choose instances, and proves it in a zkVM: the soldering program in the
BABE implementation is a Ziren guest. The cut-and-choose argument, and with it
the 1-of-`n` honest-verifier assumption, therefore rests on the soundness of a
zkVM proof checked during the dispute. The zkVM consequently appears twice in
the protocol, in the proving pipeline and on the dispute path, and a change to
its proof system applies to both.

#### The garbling security parameter

The garbling layer uses 128-bit wire labels under an AES-128 PRF, in both the
older garbled verifier and the BABE implementation. 128-bit labels under AES-128 put the
garbling layer at NIST post-quantum security category 1, the floor of the
approved range, since Grover search over a 128-bit space is the reference for
that category. The choice was immaterial while garbling served as an
optimisation; under BABE the garbling gates decryption of the payout secret, so
the parameter warrants an explicit decision. Moving labels and PRF to 256 bits
restores category-5 margin at roughly twice the garbling cost.

The rest of `bitvm-gc` is unchanged in character: the circuit being garbled
remains the Groth16/BN254 verifier, with symmetric primitives appearing only
in the garbling machinery, and the bridge node still authorises peg-outs with
MuSig2 over Taproot, so the key-path exposure of section 11 is untouched by
any of this.

#### Non-transferability to a STARK verifier

A natural question is whether the same machinery can carry a hash-based
verifier: witness-encrypt against a STARK instead of Groth16, so that the BN254
dependency disappears. It cannot, for a structural reason.

BABE is a witness encryption scheme for linear pairing relations. Groth16 is
expressible because its verification equation is a pairing product, so `Y` can
be formed at encryption time from `vk` and the public input and recovered at
decryption time by pairing the proof elements. A STARK verifier has no pairing
relation, no target group and no short algebraic acceptance predicate: it is a
Merkle-and-hash argument with a large, non-algebraic verification transcript,
and there is nothing for the ciphertext to be keyed to. The analogue would
require witness encryption for hash-based proof systems, which is an open
problem rather than a change of parameters.

Garbling does not share this limitation. A garbled circuit requires only that
the verifier be expressible as a Boolean circuit, which a hash-based verifier
is; Ziren's garbled-verifier route is built on this, and section 10 reports such
verifiers measured. The limitation is specific to
witness encryption.

This creates a tension between cost and post-quantum security. BABE lowers the
cost of verifying Groth16 on Bitcoin, and with it the pressure to replace
Groth16, whereas replacing it is the post-quantum requirement. Engineering
invested in the BABE path deepens the dependence on a pairing-based statement,
and the conclusion of section 8 stands: `bitvm-gc` is Groth16-oriented by
construction, so moving off pairings is a rebuild of that component rather than
the replacement of a wrapper.

#### Assessment

This line of work leaves the post-quantum position of the bridge unchanged. Two
observations support the assessment. The pairing assumption resides in the
security theorem of the scheme that replaces the on-chain verifier, not merely
in a script the bridge could exchange; and the AES-128 garbling parameter gates
the payout secret rather than serving as an optimisation detail.

BABE answers the question it was designed for, on-chain cost, and its on-chain
surface is hash-based. The analysis above concerns its classification only: it
does not constitute post-quantum progress, and a hash-based proof system cannot
be substituted into the same protocol without a different encryption mechanism.

### 10. Post-quantum replacements for the on-chain verifier

If the Groth16 verifier is removed, something must take its place. What decides
the choice is not proof size but which opcodes Bitcoin script provides, and the
constraint is narrower than it first appears.

Script can do arithmetic; it cannot hash a concatenation. There is no
multiplication opcode (`OP_MUL` and its relatives are disabled), so
multiplication is emulated by double-and-add, and
[BitVM](https://github.com/bitvm/bitvm) already does this at 254 bits: the
whole Groth16 verifier BitVM2 puts in script is emulated multi-limb modular
arithmetic, with no use of `OP_CAT`. Emulated modular arithmetic needs no consensus
change. Concatenation does: `OP_CAT`, `OP_SUBSTR`, `OP_LEFT` and `OP_RIGHT` are
all disabled, so a script cannot assemble the two-child preimage that a
SHA256 Merkle step hashes.
[BIP-347](https://github.com/bitcoin/bips/blob/master/bip-0347.mediawiki)
would re-enable `OP_CAT`; its status is `Complete`, meaning specified, not
deployed.

The `OP_CAT` dependency is a choice of hash, not a property of the proof
system. A verifier needs concatenation only if its commitments are built on a
byte-oriented hash. An *algebraic* hash, Poseidon2 and its relatives, is a
permutation over field elements, so hashing becomes multiplication and addition,
which is the operation script already emulates. Choose the algebraic hash and
the soft-fork dependency disappears.

Ziren has already made that choice: its commitments, Merkle trees and
transcript are Poseidon2 over the KoalaBear field, not SHA-256 over
concatenated bytes. Both remaining candidate families are therefore expressible
under current consensus rules.

#### Reed–Solomon proximity tests over an algebraic hash

This is the proof GOAT already has: Ziren commits through Poseidon2 over
KoalaBear and opens with WHIR, a Reed–Solomon proximity test in the FRI family.
Verifying it on Bitcoin means writing a verifier for proofs the bridge produces
today: no new proof system and no change of prover.

The construction requires no `OP_CAT`, although hashing is where concatenation
would normally enter. A sponge digest is eight field elements, not thirty-two bytes, so compressing
two children feeds sixteen field elements into a width-16 permutation: sixteen
stack items handed to an arithmetic routine. There is nothing to concatenate;
the "concatenation" is the
order the state sits in. With SHA256 the two children are two 32-byte items and
`OP_SHA256` hashes only one item, so they must first be fused into a single
64-byte item, and that is `OP_CAT`. The dependency belongs to the byte hash, not
to Merkle verification.

A FRI or WHIR verifier (Merkle paths, transcript, folding checks) is therefore field
arithmetic over a 31-bit prime, which fits a script number directly. No
multi-limb representation and no soft fork.

The cost is the hash. A width-16 Poseidon2 permutation over KoalaBear costs
about 572 KB of script, dominated by its 148 S-box cubings in emulated field
arithmetic, and a Merkle path costs one permutation per level, so a depth-21
path occupies about three full blocks (measured in
[`bitcoin-stark-verifier`](https://github.com/GOATNetwork/bitcoin-stark-verifier),
below).

This is also why
[`bitcoin-circle-stark`](https://github.com/Bitcoin-Wildlife-Sanctuary/bitcoin-circle-stark)
builds its transcript and Merkle components on `OP_CAT` with `OP_SHA256`: a byte
hash is one opcode where an algebraic hash is hundreds of emulated
multiplications. `OP_CAT` is what makes such verifiers *cheap* on Bitcoin, not what makes them
*possible*.

A third family merits brief mention, if only to establish that it does not
constitute a separate option.
[BaseFold](https://eprint.iacr.org/2023/1705) generalises FRI to foldable codes
and would verify through the same Merkle-and-transcript machinery over the same
algebraic hash, so it falls in the same class on cost and inherits the same
per-level hash. It changes the prover's options, not the script's.

#### Module-lattice arguments

Schemes in the LaBRADOR line commit algebraically (Ajtai/module-SIS
commitments verified by ring arithmetic), so they have no hash in the opening at
all, and no `OP_CAT` dependency for the same reason Ziren's Poseidon2 has none.
Verification is ring multiplication over Z*q*[X]/(X^*d*+1), which is an NTT of
*d* points, (*d*/2)·log₂*d* butterflies built from the double-and-add primitive
BitVM already ships.

Two trades against the hash-based class. The first is field width: a 64-bit modulus needs three
29-bit limbs where a 31-bit field needs one, so each multiplication is dearer,
though still cheaper than the nine-limb BN254 multiplication BitVM2 performs
thousands of times, which is the evidence the class of work is affordable.

The second is about the prover, not the script. Ziren commits with Reed–Solomon
codes under WHIR, so a lattice argument is a change to its polynomial
commitment scheme: the commitment
itself, the opening argument, and the recursion circuits that verify them.

Wrapping remains a fallback: a lattice argument may prove that a STARK verified.
Unlike the Groth16 and BABE cases of section 9, the composition carries a
hash-based statement rather than a pairing assertion, so it is not an instance
of the wrapper pattern, but it costs a recursion layer that changing the
commitment scheme does not.

Against those, there is no per-level hash to pay, and proof sizes are the
smallest of any post-quantum family:
[LaBRADOR](https://eprint.iacr.org/2022/1341) is 58 KB for R1CS with 2²⁰
constraints, at the 128-bit level.

Size is not what decides it, though, and the published figures do not compare
directly. A script pays for verification, and that is where this line is
weakest: LaBRADOR's verifier is linear-time, and
[Greyhound](https://eprint.iacr.org/2024/1293)'s measured verification is
2.80 s at degree 2³⁰, which its own paper concedes is about twice Ligero's.
The headline numbers also measure different things: Greyhound's 53 KB is an
evaluation proof rather than an R1CS proof, and
[SALSAA](https://eprint.iacr.org/2025/2124)'s much-quoted 73 KB is one folding
step on linear relations rather than a standalone proof, against 979 KB for
its general-purpose argument at the same 2²⁸-element witness. So the comparison
has to be made by measuring.

Both are viable without a soft fork, and the choice is a measurement rather
than a research question. `OP_CAT` decides how *expensive* a hash-committed
verifier is, not whether one is possible. WHIR over Poseidon2 verifies what Ziren
already emits and pays a few hundred multiplications per Merkle level; a
module-lattice argument has no per-level hash and the smallest proofs available,
but pays for them in verifier work and changes Ziren's commitment scheme with
it. Neither fits a single transaction,
and neither needs to (BitVM already chunks a verifier across a disprove
pattern), so the question is how many chunks each option costs.

#### Measured verifiers

[`bitcoin-stark-verifier`](https://github.com/GOATNetwork/bitcoin-stark-verifier)
measures hash-based verifiers along both routes available to a BitVM-style
bridge, each on proofs produced by [Plonky3](https://github.com/Plonky3/Plonky3)'s
prover and accepted by Plonky3's own verifier.

- *In Bitcoin script.* Poseidon2 over KoalaBear and a WHIR verifier, using no
  disabled opcode. At 100-bit security the full verifier is several hundred
  megabytes of script, of which query work (Merkle paths and leaf hashing) is
  about 97%. Divided into segments of one transaction each, of which a dispute
  executes only one, with Winternitz commitments at the segment boundaries, it
  is on the order of two thousand segments, the same order as BitVM2's own.
- *As a garbled circuit.* A binary-field STARK verifier, Plonky3's multi-STARK
  over WHIR on GF(2^128) with Blake3, garbled as in BitVM3: the arithmetic and
  hash of Ziren's garbled-verifier route. For a 2^18-row Keccak-f trace at 104
  bits of soundness the verifier has 9.4 × 10^7 non-free gates and garbles to
  1.5 GB, about thirty times fewer non-free gates than the garbled Groth16
  verifier
  ([paper](https://github.com/GOATNetwork/bitcoin-stark-verifier/blob/whir-gc/paper/garbled-stark-verifier.pdf)).
  The verifier of the last proof of Ziren's narrow recursion is of the same
  order, about 2 × 10^8 non-free gates, on changes to Plonky3 and Ziren that
  are not yet reviewed.

Two consequences bear on the replacement decision. Garbling, unlike witness
encryption, carries a hash-based verifier without change to the protocol's
structure. And the field and the hash, more than the proof system, determine the
cost of a garbled verifier: as a Boolean circuit a Poseidon2 permutation over
KoalaBear costs about 120 Blake3 compressions, so a verifier for proofs
committed with Poseidon2 is better wrapped into a binary-field proof than
garbled directly, which is the order of Ziren's route.

### 11. Peg custody and the Taproot key path

The peg custody model is direct: the committee aggregates MuSig2 partial
signatures into a Taproot signature over an n-of-n aggregated Taproot public
key, used by the BitVM2 connector outputs. MuSig2 producing a Taproot
signature over an aggregate key is the canonical key-path spend pattern; a
script-path spend would not need aggregation. Script paths are present as
well, so the design uses both.

This is the sharpest exposure in the stack. A Taproot output commits its
output key in the `scriptPubKey` at the moment the output is *created*, not when
it is spent. So the peg's aggregated public key is on chain, in the clear, for
the entire lifetime of the UTXO. An adversary with a cryptographically relevant
quantum computer recovers the corresponding secret from that public key and
spends via the key path, without forging a proof, compromising any
committee member, or waiting for a spend to reveal anything. It is
strictly easier than attacking the Groth16 wrapper, and it is the textbook
long-exposure case that BIP-360 exists to remove.

One apparent mitigation is ineffective, and the reason generalises. A natural
move is to commit a provably unspendable NUMS point as the Taproot internal key,
disabling the key path and leaving only script paths, the construction BIP-360
proposes to standardise. Against a quantum adversary it buys nothing. A Taproot
key-path spend is validated by checking a Schnorr signature against the output
key `Q` in the `scriptPubKey`; consensus never examines the internal key, the
tweak, or how `Q` was constructed. An adversary who can solve discrete log
computes the secret for `Q` from what is already on chain and signs. NUMS stops
*classical* parties who do not know a discrete log: the committee, not
the attacker.

That is exactly why BIP-360 is a *consensus* soft fork rather than a wallet
convention: it defines an output type in which no curve point is committed at
all. The removal of the key path is the protection, and it cannot be obtained by
choosing a different internal key.

The stronger statement is structural. Any Taproot output is inherently
long-exposure, because `Q` is committed when the output is created and `Q` alone
is sufficient to spend it. No key choice changes that. What would change it is a
hash-committed output type (which BitVM2 cannot use, because it needs script
paths) or BIP-360, which is Bitcoin's timeline and not the L2's.

What is available unilaterally is bounding the window, not closing it. The
exposure is proportional to how long value sits in a Taproot UTXO, so peg
policy (rotating custody outputs on a schedule, capping the value in any one
output, avoiding long-lived connector outputs) reduces the attack surface
without waiting on anyone. Such policy is advisable and should be recorded
explicitly, but it should be ranked as risk-bounding rather than as a fix, and
it does not displace the proof-system work.

### 12. goat: consensus keys

The consensus keys admit the least costly migration in the stack, and they are
the one surface for which GOAT has published a design ([GOAT Research, *Post-Quantum Consensus Keys on GOAT*](https://hackmd.io/MPLgpnyATaiFfskWDXumRQ)). Validators move from
secp256k1 to ML-DSA-65 through a coordinated upgrade of CometBFT and the Cosmos
SDK, which ships the key type (section 6). The SDK is not forked, so the work is
a dependency upgrade rather than a rebase, although the dependency manifest can
suggest otherwise at first reading (section 15).

#### Threat model

The design separates two threats. Forged votes in live consensus become possible
when a cryptographically relevant quantum computer exists, and not before.
Forgery of recorded history, an alternative chain assembled from archived commit
signatures, is a harvest-now, forge-later threat, and the design states its
consequence directly: blocks signed before activation remain forgeable, because
they were signed with secp256k1.

#### Mechanism

Each validator keeps a permanent identity, and an index maps its current
consensus address to that identity, so existing state keeps its shape and a
validator that never rotates resolves to itself. A validator rotates by calling
a rotation contract on the EVM with the new public key and a proof of
possession. The execution layer forwards the request without verifying it; the
consensus layer performs every check and, at the end of the block, emits two
validator updates: the old key at zero power and the new key at the validator's
power. A rejected rotation is logged rather than halting the block.

Rotation is limited to one per retention period, 21 days on mainnet, and
validators rotate individually, so the validator set may remain mixed
indefinitely and no network-wide switch is required. The keys are dual rather
than hybrid: the secp256k1 key is retained as the validator's account key and
does not co-sign consensus messages.

#### Activation

Three gates activate the mechanism: a timestamp in the execution client, a fork
height that admits the new key type, and a fork height from which the consensus
layer accepts rotations. Each validator then rotates with a single transaction,
for which there is no deadline. Because CometBFT applies a validator update two
blocks after it is emitted, a rotation briefly exposes the validator to missed
blocks, and operators schedule it within the margin before downtime jailing.

#### Cost

GOAT's measurements, for ML-DSA-65 against the secp256k1 keys it replaces:

| | secp256k1 | ML-DSA-65 |
| --- | ---: | ---: |
| Verification, per signature | 173 µs | 23 µs |
| Signing, per signature | 46 µs | 196 µs |
| Commit signatures per block, 20 validators | 1.3 KB | 66 KB |
| Validator public keys in state, 20 validators | 0.7 KB | 39 KB |

Verification becomes faster, so the migration costs bandwidth and storage rather
than computation. Because CometBFT does not aggregate commit signatures
(section 6), the cost is linear in the number of validators: at 20 validators
the commit signatures add about 585 GB per year, and at 100 about 3 TB. The
configured block-size limit, about 6 MiB, accommodates the larger commit at the
current validator count. Aggregation of post-quantum signatures, as in
Ethereum's recursive-proof designs (section 5), is outside the design.

#### Limits

The design covers consensus keys only; accounts, the bridge and the Bitcoin-side
scripts are separate surfaces (sections 8 to 11 and 13). One interaction remains
to be settled: the effective evidence window, about four days at the current
block time, is shorter than the 21-day retention period, and because the
upstream staking module fixes a rotation's deadline at the time of rotation,
extending the evidence window could leave misbehaviour under a rotated-out key
unattributable. Finally, the chain does not appear to use IBC, so the
light-client hazard of section 6 appears not to apply; this should be confirmed
against deployment rather than the dependency manifest alone.

### 13. goat-geth: the divergence, measured

Measured against upstream
[`ethereum/go-ethereum`](https://github.com/ethereum/go-ethereum),
[`goat-geth`](https://github.com/GOATNetwork/goat-geth) is 37 commits ahead
and 377 commits behind, with 78 files changed, and had not been pushed to for
roughly four months at the time of measurement. Thirty-seven custom commits over seventy-eight files is a tractable
amount of divergence: a customised fork, not a rewrite. But 377 commits
behind is the number that matters for planning: any post-quantum work on the
execution layer inherits an upstream catch-up first, and that catch-up will only
grow.

#### The EVM's cryptographic substrate

Treating this surface as "accounts use ECDSA, follow Ethereum" understates it. The EVM exposes elliptic-curve and pairing operations as *consensus-level
precompiles*, and `goat-geth` carries the full upstream set:

| Address / type | Primitive | Quantum status |
| --- | --- | --- |
| `0x01` `ecrecover` | secp256k1 ECDSA | **broken** |
| `0x06`–`0x08` `bn256Add`, `bn256ScalarMul`, `bn256Pairing` | BN254 group ops and **pairing** | **broken** |
| `0x0a` `kzgPointEvaluation` | KZG over BLS12-381 | **broken** |
| `bls12381G1Add/G2Add/MultiExp/Pairing/MapG1/MapG2` | BLS12-381 ([EIP-2537](https://eips.ethereum.org/EIPS/eip-2537)) | **broken** |
| `p256Verify` | secp256r1 ECDSA ([RIP-7212](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7212.md)) | **broken** |
| `0x02`, `0x03`, `0x04`, `0x05`, `0x09` | SHA-256, RIPEMD-160, identity, modexp, BLAKE2F | safe |

Four curve families (secp256k1, secp256r1, BN254, BLS12-381) and two pairing
engines, all consensus-critical.

This is a harder migration than account signatures, though not a hopeless
one. Account keys can migrate through account abstraction, because the *user*
opts in. A precompile cannot be migrated that way: it is a consensus rule, and
the contracts calling it are immutable bytecode that in general nobody has the
authority to upgrade.

Ethereum's own work shows the one lever that does exist, and it is narrower than
"remove the precompile".
[EIP-8151](https://eips.ethereum.org/EIPS/eip-8151) ("Account Code
Restricted ecRecover",
Draft, created 2026-02-09) does not delete `ecRecover`; it *narrows* it, keyed on
state the protocol already controls. The attack it closes is instructive: an EOA
that installs real code, as [EIP-8298](https://eips.ethereum.org/EIPS/eip-8298)
allows, has its ECDSA *transaction* authority disabled by
[EIP-3607](https://eips.ethereum.org/EIPS/eip-3607) (Final), though a delegation
under [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) alone does not. But
`ecRecover` ignores account state, so a quantum attacker holding the derived
ECDSA key could still authorize transfers through immutable contracts, notably
ERC-20 `permit` implementations. EIP-8151 makes `ecRecover` return zero when the account has
non-empty code that is not a valid delegation indicator.

The generalisable rule is therefore sharper than "precompiles are unmigratable":

- a precompile bound to an account identity can be narrowed conditionally,
  because there is per-account state (has this account migrated?) to key the
  restriction on;
- a stateless mathematical precompile cannot. `bn256Pairing` answers a
  question about field elements; there is no account whose migration status
  could gate it, and every deployed verifier that calls it would break equally.

`ecRecover` is in the first category. The pairing, BLS12-381 and KZG precompiles
are in the second, and a search of the EIPs repository returns no proposal to
deprecate or replace any of them.

#### The caller set as the unit of analysis

Ethereum's own coverage is uneven in a way that turns out to be informative.
`ecRecover` has a plan. KZG has a stated plan: the roadmap acknowledges that
"KZG commitments rely on elliptic curve pairings, the same mathematical
structure that quantum computers can attack", and commits to "replace KZG with a
quantum-safe commitment scheme", with STARK-based and lattice-based commitments
named as candidates still under research. The BN254 and BLS12-381 group and
pairing precompiles are not addressed at all; the official quantum-resistance
page does not discuss precompiles as a category.

That asymmetry is not an oversight, and the reason generalises. What decides
whether an exposure can be migrated is not the precompile but who calls it:

| Precompile | Caller set | Migratable? |
| --- | --- | --- |
| `ecRecover` | every account, but gated by per-account state | yes: narrow it (EIP-8151) |
| KZG point evaluation | rollup contracts: bounded, known, actively maintained | plausibly: coordinate the set |
| BN254 / BLS12-381 pairing | every SNARK verifier ever deployed: unbounded, largely abandoned | no: nobody to coordinate with |

A bounded set of maintained callers can be coordinated through an upgrade. An
unbounded set of abandoned contracts cannot, by anyone, at any price. This is why
the plans exist where they exist.

#### Practical consequences

For stateless precompiles with unbounded callers, the available levers are
limited:

- Additive only: ship post-quantum precompiles alongside the old ones, as
  EIP-7885 proposes. This is the actual plan. It solves the *forward* problem
  and leaves the legacy one untouched.
- Gas repricing as soft deprecation: make the call prohibitively expensive
  rather than removing it. This breaks callers economically instead of
  technically, which fails *silently* through out-of-gas rather than loudly, and
  is arguably worse than a clean break.
- Flag-day removal: the shape of Bitcoin's BIP-361 sunset applied to a
  precompile. There is no Ethereum precedent for this, and it breaks every
  caller at once.
- Nothing: the default. When BN254 falls, every deployed verifier accepts
  forged proofs while behaving exactly as specified.

The actionable question for any given chain is therefore not "is the
precompile exposed" but "are my callers upgradeable". A project whose
verifier contracts are its own and upgradeable sits in the tractable category
regardless of what upstream does; one whose verifiers are immutable has a
deadline it cannot move. That is a question about deployment artifacts, and it
can be answered today from chain state, which is why the inventory in the
ordering below is dated work rather than a formality.

And the severity is soundness, not theft. If BN254 discrete log falls, a
forged proof satisfies `bn256Pairing`, and every on-chain verifier accepts it. For
a bridge that verifies withdrawal proofs on the L2 side, that is direct loss of
funds through an interface that is behaving exactly as specified.

The same assumption is load-bearing twice. BN254 appears on the Bitcoin side
through the BitVM2 Groth16 wrapper *and* on the EVM side through `bn256Pairing`.
One broken assumption compromises the bridge from both directions, so the two
should be tracked as a single dependency rather than two independent risks.

GOAT's own changes do not touch this. The fork's commits concentrate in type
definitions, tracing and its own modules, none of it cryptographic, and the
precompile set is inherited from upstream unchanged, which means the fix must
also come from upstream, which is the concrete argument for closing the
377-commit gap: the gap is the delivery channel for any future
post-quantum precompile work.

Recommendation for *account semantics* remains follow-not-lead: diverging ahead
of Ethereum's account-abstraction route breaks wallet and tooling compatibility,
which for an L2 is usually the product. But the precompile exposure is not an
account-semantics question, and it should not inherit that low priority.

---

## Part IV: Recommendations

### 14. Ordering the work

Ownership and severity, not novelty, should set the
order; the roadmap after the abstract places each step against the four
migration stages:

1. Bridge attestation keys (row 2). Highest value per unit of control. The
   operator set is the peg's trust root; compromising a threshold is
   equivalent to compromising the peg. It is entirely the L2's to change, and it
   can usually be rolled out with dual signing, giving rollback at every step.
2. Bridge proof system (row 3). Highest severity, and the hardest problem
   in the list: it fixes soundness rather than theft, and it means removing the
   Groth16 verifier the bridge is built around. Proof size and verification cost
   drive the L2's economics, so a constant-size pairing proof cannot simply be
   exchanged for a larger hash-based one. Sequenced second because it is
   expensive and requires rebuilding the verification stack, not because it is
   second in difficulty.
3. Consensus keys (row 5). Mechanism often already exists upstream (Cosmos
   SDK v0.55 ships ML-DSA-65 as an opt-in validator key type, for instance), so
   the work is frequently a dependency upgrade rather than cryptographic design.
4. Execution accounts (row 6). Follow the upstream ecosystem. Diverging
   account semantics ahead of it breaks wallet and tooling compatibility, which
   for an L2 is often the product itself.
5. L1 settlement (row 1). Not fixable. Bound it instead: avoid address
   reuse, prefer outputs that do not commit a bare public key, avoid long-lived
   outputs with revealed keys, and write custody policy so the script and key
   policy can migrate when the base layer can.

EVM precompiles (row 7) sit outside this sequence. They cannot be ordered
against the rest, because the fix is upstream and the callers are immutable.
What can be done now is an inventory: which deployed contracts call the pairing
and KZG precompiles, and which of those are consensus- or bridge-critical. A
contract that cannot be upgraded can at least be known about before it matters.

Applied to GOAT:

| Phase | Action | Why here |
| --- | --- | --- |
| 0 | Inventory every signature and proof verification path; add tests asserting no fixed signature-length assumptions | Fixed-length checks are load-bearing and invisible until a 3309-byte signature arrives (section 15) |
| 1 | Peg custody: write and enforce an exposure policy (rotate custody outputs, cap value per output, avoid long-lived connectors) | The Taproot output key is on chain from creation and is sufficient to spend, so no internal-key choice removes the exposure (section 11). Only the *window* is GOAT's to shrink; the fix is BIP-360 and Bitcoin's timeline |
| 2 | **Stop verifying a pairing on Bitcoin**: move the bridge from Ziren's pairing-based wrap to its garbled-verifier route, and re-target the `bitvm-gc` garbling stack from the Groth16/BN254 verifier to the hash-based one. That route is measured as a garbled circuit (about 10^8 non-free gates); the script route is measured as well (on the order of two thousand disprove segments at 100-bit) | The hardest item here, and the one that ships last. `bitvm-gc` is Groth16-verifier-oriented by construction, so this is a rebuild rather than a wrapper swap, and BABE cuts against it by lowering the cost of *keeping* Groth16. The prover side already exists in Ziren's design, and section 10 finds the verifier feasible on both routes without a soft fork, so what remains is integration and a choice by cost |
| 3 | Consensus keys: upgrade CometBFT and the Cosmos SDK together; activate ML-DSA-65 at the fork heights; validators rotate individually through the rotation contract, one rotation per retention period; provision bandwidth and storage for the larger commits | GOAT's published design (section 12). No SDK fork exists, so this is a dependency upgrade rather than a rebase; mixed validator sets are supported indefinitely, so no network-wide switch is needed; verification becomes faster, so the cost is bandwidth and storage rather than computation |
| 4 | Reduce `goat-geth`'s 377-commit lag; inventory callers of `0x06`–`0x08` and `0x0a`, and record for each whether it is **upgradeable** | The lag is the delivery channel for EIP-7885 and EIP-8151 when they land. The inventory's key column is upgradeability, not existence: an upgradeable verifier is tractable whatever upstream does, an immutable one has a deadline that cannot move |
| — | Peg: minimise Bitcoin-side key exposure; keep custody policy migratable | Blocked on Bitcoin, which by BIP-360's own text has no PQ signature scheme |

---

## Part V: Pitfalls and open items

### 15. Recurring pitfalls

The following patterns recur throughout the preceding analysis.

- Fixed-length signature checks. Code that validates `len(sig) == 64` is
  structurally incapable of accepting a post-quantum signature. ML-DSA-65
  signatures are 3309 bytes and public keys 1952. These checks are small,
  load-bearing and invisible until something fails.
- Size lands on the hot path. Post-quantum signatures are one to two orders
  of magnitude larger than what they replace. On a consensus layer every
  validator signs every block, so block size limits, gossip framing and
  commit-signature budgets all need re-tuning, not just the key type.
- Forks hide the real blocker. An L2 running forked dependencies inherits an
  upstream catch-up before it can inherit upstream post-quantum work. Measure
  the divergence early; it is often the schedule driver, not the cryptography.
- Code search under-reports on forks. GitHub code search does not index
  forks. In the one stack examined, searching for `winternitz` in a forked
  dependency returned zero hits while the files plainly exist in the tree.
  An audit built on code search over a fork-heavy stack will silently miss
  things. Enumerate the git tree instead.
- Prefer exposing a primitive to blessing a scheme. A precompile welded to
  one construction (a specific curve, a specific pairing) becomes unremovable
  the moment contracts depend on it. Exposing the shared underlying operation
  instead (NTT rather than ML-DSA, as Ethereum's EIP-7885 proposes) keeps the
  scheme choice in contract code, where it can still be changed.
- Precompiles are the exposure that is hardest to migrate. An account can
  adopt a new signature scheme; a deployed contract calling a pairing
  precompile cannot. On any EVM chain the elliptic-curve and pairing
  precompiles are consensus rules with immutable callers. One lever exists:
  a precompile bound to an *account identity* can be narrowed conditionally on
  per-account state, as EIP-8151 does for `ecRecover`. A stateless mathematical
  precompile such as a pairing check has no such state to key on, and for those
  the exposure is fixed at deployment time with only an inventory available.
- Ask what a component is *oriented around*, not just what it depends on. A
  garbling or proving stack purpose-built for one verifier makes the proof
  system a structural property rather than a configuration choice: swapping it
  is a rebuild of that component, and the dependency graph will not show this.
- The wrapper pitfall runs forwards as well as backwards. It is easy to spot
  in existing code (a hash-based commitment carrying a curve assertion), and
  just as easy to *introduce* while migrating, by wrapping a broken primitive in
  a post-quantum proof system and treating the composition as fixed. Proving
  that a broken signature verified is a true statement about a dead assumption.
  A migration step only helps if it changes what is asserted, not merely who
  attests to it.
- Read the security theorem, not the primitive list. The sharpest version of
  the wrapper pitfall hides behind vocabulary that is entirely post-quantum-safe.
  A scheme built from witness encryption, garbled circuits, hashlocks and
  Lamport commitments can still reduce to elliptic-curve discrete log, because
  the pairing survives in the *relation being encrypted against* rather than in
  any named primitive. Section 9's case states it in the proof itself: the
  advantage bound carries an explicit `Adv^DLog` term. When a design claims to
  remove an on-chain verifier, ask what the ciphertext is keyed to.
- Symmetric-primitive layers are not automatically "post-quantum done".
  Garbled circuits, hash commitments and MPC layers are built from symmetric
  primitives and survive Shor, but they only protect what they *carry*. If the
  statement being garbled or committed is an elliptic-curve or pairing
  assertion, the composition is only as quantum-safe as that assertion. Check
  the content, not the wrapper, and separately check the symmetric layer's own
  parameter, since 128-bit labels sit at the floor of the approved
  post-quantum range.
- A mitigation that stops classical misuse may stop nothing quantum. A NUMS
  Taproot internal key makes the key path unspendable by anyone who does not know
  a discrete log, which is everyone except the adversary the migration is about.
  Consensus checks a signature against the *output* key, so who chose the
  internal key is irrelevant to someone who can solve discrete log. Ask which
  adversary a mitigation excludes, not which spend path it closes.
- A commented-out `replace` directive reads as a fork. Anything that greps a
  `go.mod` for a module path finds live and commented lines alike. In the stack
  examined this inverted a schedule estimate: one dependency is genuinely forked
  and one is not, and the difference is a rebase versus a version bump.
- Interoperability makes migration a coordination problem. Where light
  clients verify a counterparty's signatures with their own compiled-in crypto,
  enabling a new key type before counterparties can verify it breaks
  connectivity, and the failure is silent at upgrade time.

### 16. Remaining open items

None of the following blocks the recommendations.

- **Precompile callers.** The deployed contracts that call the `0x06`–`0x08`
  and `0x0a` precompiles, and whether each is upgradeable, are not enumerated.
  This requires chain state rather than source, and it is the only item with a
  deadline: an immutable caller can be identified now but not migrated later.
- **The module-lattice verifier.** Its cost in Bitcoin script is unmeasured; the
  hash-based routes are measured (section 10).
- **One-time keys.** The cost of the one-time signatures that bind a
  verifier's inputs on chain, including their generation, is treated in
  [`bitcoin-stark-verifier`](https://github.com/GOATNetwork/bitcoin-stark-verifier/tree/whir-gc)
  rather than here.
- **Scope.** One L2 has been examined, so the taxonomy of Part I rests on
  structural reasoning supported by a single case.

### References

Primary sources, each verified live: the base-layer and GOAT sources on
2026-07-31, the script-cost sources on 2026-08-02, and, on 2026-08-10, `goat`'s `go.mod` `replace` block and every
script figure in section 10, which were re-measured rather than re-read.

- BIP-360, *Pay-to-Merkle-Root (P2MR)* — <https://github.com/bitcoin/bips/blob/master/bip-0360.mediawiki>
- BIP-361, *Post Quantum Migration and Legacy Signature Sunset* — <https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki>
- SHRINCS Working Group, *SHRINCS: an efficient hash-based signature scheme for Bitcoin (first draft)*, bitcoin-dev, 2026-08-27 — <https://groups.google.com/g/bitcoindev/c/HbVboXIFiG8>, <https://github.com/SHRINCS/shrincs-bip/blob/main/SHRINCS.md>
- NIST, FIPS 205 (SLH-DSA) — the standardised SPHINCS+ variant SHRINCS uses for its stateless path — <https://csrc.nist.gov/pubs/fips/205/final>
- NIST CSWP 39-upd1, *Considerations for Achieving Cryptographic Agility* — <https://csrc.nist.gov/pubs/cswp/39/upd1/considerations-for-achieving-crypto-agility/final>
- StarkWare, *Networks: Cryptographic Agility to PQ Migration* (Quantum Hub) — the four migration stages and the network comparison — <https://quantum.starkware.co/networks>
- AppliedPQC, *Post-Quantum Migration in Ethereum's Execution Layer: The Full Picture* — <https://github.com/AppliedPQC/pqc-research/issues/35>
- EIP-8298, *SETCODEFROM Code Reuse Instruction* — <https://eips.ethereum.org/EIPS/eip-8298>
- Bitcoin Optech, *Quantum resistance* — <https://bitcoinops.org/en/topics/quantum-resistance/>
- GOAT Research, *Post-Quantum Consensus Keys on GOAT* — the consensus-key migration design of section 12 — <https://hackmd.io/MPLgpnyATaiFfskWDXumRQ>
- Cosmos SDK, `UPGRADING.md` — <https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md>
- Cosmos SDK PR #26436, ML-DSA-65 consensus keys — <https://github.com/cosmos/cosmos-sdk/pull/26436>
- Cosmos docs, *Post-quantum keys* — <https://docs.cosmos.network/sdk/latest/keys/post-quantum-keys>
- Post-Quantum Ethereum — <https://pq.ethereum.org/>
- EIP-2537, *Precompile for BLS12-381 curve operations* — <https://eips.ethereum.org/EIPS/eip-2537>
- EIP-3607, *Reject transactions from senders with deployed code* — <https://eips.ethereum.org/EIPS/eip-3607>
- EIP-7702, *Set EOA account code* — <https://eips.ethereum.org/EIPS/eip-7702>
- EIP-7885, EIP-8141 and EIP-8151, the post-quantum account track — <https://eips.ethereum.org/EIPS/eip-7885>, <https://eips.ethereum.org/EIPS/eip-8141>, <https://eips.ethereum.org/EIPS/eip-8151>
- RIP-7212, *Precompile for secp256r1 curve support* — <https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7212.md>
- `leanEthereum/leanVM` — <https://github.com/leanEthereum/leanVM>
- `leanEthereum/leanSig` — <https://github.com/leanEthereum/leanSig>
- `b-wagn/hash-sig`, *Hash-Based Multi-Signatures for Post-Quantum Ethereum*, eprint 2025/055 — <https://eprint.iacr.org/2025/055>, <https://github.com/b-wagn/hash-sig>
- CometBFT, `types/block.go` (`Commit`, `CommitSig`, `MaxCommitBytes`) — <https://github.com/cometbft/cometbft/blob/main/types/block.go>
- CometBFT, `types/params.go` (`DefaultBlockParams`) — <https://github.com/cometbft/cometbft/blob/main/types/params.go>
- CometBFT issue #3455, *BLS signature aggregation* — <https://github.com/cometbft/cometbft/issues/3455>
- CometBFT issue #1305, *Halve commit size with partial ed25519 signatures* — <https://github.com/cometbft/cometbft/issues/1305>
- CometBFT PRs #3632 and #4763, `crypto/bls12381` signature aggregation, both closed unmerged — <https://github.com/cometbft/cometbft/pull/3632>, <https://github.com/cometbft/cometbft/pull/4763>
- W. Beullens, G. Seiler, *LaBRADOR: Compact Proofs for R1CS from Module-SIS* — <https://eprint.iacr.org/2022/1341>
- *SALSAA — Sumcheck-Aided Lattice-based Succinct Arguments and Applications* — <https://eprint.iacr.org/2025/2124>
- M. Klooss, R. W. F. Lai, N. K. Nguyen, M. Osadnik, L. Tucci, *RoKoko: Lattice-based Succinct Arguments, a Committed Refinement* — <https://eprint.iacr.org/2026/575>
- H. Zeilberger, B. Chen, B. Fisch, *BaseFold: Efficient Field-Agnostic Polynomial Commitment Schemes from Foldable Codes*, CRYPTO 2024 — <https://eprint.iacr.org/2023/1705>
- Bitcoin Core, `src/script/script.h` (`CScriptNum::nDefaultMaxNumSize`, the four-byte arithmetic operand limit) — <https://github.com/bitcoin/bitcoin/blob/master/src/script/script.h>
- Bitcoin Core, `src/script/interpreter.cpp` (the disabled opcodes, including `OP_MUL` and `OP_CAT`) — <https://github.com/bitcoin/bitcoin/blob/master/src/script/interpreter.cpp>
- Bitcoin Core, `src/policy/policy.h` (`MAX_STANDARD_TX_WEIGHT`) — <https://github.com/bitcoin/bitcoin/blob/master/src/policy/policy.h>
- `BitVM/BitVM`, `bitvm/src/bigint/mul.rs` — double-and-add multiplication in Bitcoin script — <https://github.com/bitvm/bitvm>
- `Bitcoin-Wildlife-Sanctuary/rust-bitcoin-m31`, measured M31 script weights — <https://github.com/Bitcoin-Wildlife-Sanctuary/rust-bitcoin-m31>
- S. Duan, J. Yin, B. Hou, V. Pham, E. Zhu, M. Guo, *Ziren: Succinct Arguments for MIPS32 Execution*, Cryptology ePrint Archive 2026/2330 — <https://eprint.iacr.org/2026/2330>
- *Greyhound: Fast Polynomial Commitments from Lattices* — <https://eprint.iacr.org/2024/1293>
- `GOATNetwork/bitcoin-stark-verifier` — hash-based STARK verifiers measured in Bitcoin script (Poseidon2 and WHIR, without `OP_CAT`) and as garbled circuits (binary-field WHIR with Blake3); source of the measurements in section 10 — <https://github.com/GOATNetwork/bitcoin-stark-verifier>
- *Garbling a Hash-Based STARK Verifier for Bitcoin* — the garbled-circuit measurements of section 10 — <https://github.com/GOATNetwork/bitcoin-stark-verifier/blob/whir-gc/paper/garbled-stark-verifier.pdf>
- Plonky3 — the WHIR prover and KoalaBear Poseidon2 instance the measurements are taken against — <https://github.com/Plonky3/Plonky3>
- Bitcoin Core, `src/consensus/consensus.h` (`MAX_BLOCK_WEIGHT`) — <https://github.com/bitcoin/bitcoin/blob/master/src/consensus/consensus.h>
- BIP-347, *OP_CAT in Tapscript* (E. Heilman, A. Sabouri; Consensus soft fork, Complete, not activated) — <https://github.com/bitcoin/bips/blob/master/bip-0347.mediawiki>
- `Bitcoin-Wildlife-Sanctuary/bitcoin-circle-stark`, a Circle STARK verifier in Bitcoin script — <https://github.com/Bitcoin-Wildlife-Sanctuary/bitcoin-circle-stark>
- `GOATNetwork/goat` — <https://github.com/GOATNetwork/goat>
- `GOATNetwork/goat-geth` — <https://github.com/GOATNetwork/goat-geth>
- `GOATNetwork/BitVM` — the fork carrying the Winternitz implementation (`bitvm/src/signatures/`) — <https://github.com/GOATNetwork/BitVM>
- `GOATNetwork/bitvm2-node` — <https://github.com/GOATNetwork/bitvm2-node>
- `GOATNetwork/bitvm-gc` — <https://github.com/GOATNetwork/bitvm-gc>

On-chain proof verification (section 9). Verified 2026-08-02.

- S. Garg, D. Kolonelos, M. Sergeevitch, S. Sridhar, D. Tse, *BABE: Verifying Proofs on Bitcoin Made 1000x Cheaper* — <https://eprint.iacr.org/2026/065>
- L. Eagen, Y. T. Lai, *Argo MAC: Garbling with Elliptic Curve MACs* — <https://eprint.iacr.org/2026/049>
- N. Khambhati, A. Bhattacharya, D. Heath, *Duty-Free Bits: Projectivizing Garbling Schemes* — <https://eprint.iacr.org/2026/476>
- R. Linus et al., *BitVM3: Efficient Bitcoin Bridges via Garbled Circuits* — <https://eprint.iacr.org/2026/933>
- GOAT Research, *Deferred Binding: Extending BABE for Dynamic Public Inputs in GOAT BitVM3* — <https://hackmd.io/@goatresearch/HkKp2g1Zfl>
- GOAT, *Partial-Binding Witness Encryption over Groth16* — the formal write-up of the primitive Deferred Binding rests on (`docs/partial_binding_we.tex`) — <https://github.com/GOATNetwork/bitvm-gc/blob/main/docs/partial_binding_we.tex>
- `bitvm-gc`, the BABE implementation (`verifiable-circuit-babe`, `babe-programs/soldering`) — <https://github.com/GOATNetwork/bitvm-gc>
