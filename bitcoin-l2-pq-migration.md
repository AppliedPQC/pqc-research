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
items.

The finding of most practical consequence is that symmetric and hash-based
layers protect only what they carry. In the stack examined, the Bitcoin-side
commitment layer, the garbling layer, the FRI commitment and the witness
encryption that replaces the on-chain verifier are all post-quantum, and the
bridge is still not, because every one of them is wrapping or carrying an
elliptic-curve assertion. Section 9 follows that pattern through four
generations of the same component and finds it stated outright in the security
proof of the newest one.

### Sources and verification

The analysis is drawn from primary sources: BIP text from
[`bitcoin/bips`](https://github.com/bitcoin/bips),
[`UPGRADING.md`](https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md)
from [`cosmos/cosmos-sdk`](https://github.com/cosmos/cosmos-sdk), the
[Ziren issue tracker](https://github.com/ProjectZKM/Ziren/issues), the
[GOAT repositories](https://github.com/GOATNetwork) and their dependency trees
read through the GitHub API and local clones, and, for section 9, the BABE
and Argo MAC papers alongside GOAT's own
[Deferred Binding design doc](https://github.com/GOATNetwork/bitvm2-gc/blob/feat/goat-bitvm3/docs/partial_binding_we.tex).
Claims are stated as verified; checks still outstanding are listed under
*Remaining open items* (section 17).

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
| 3 | Bridge proof system | Groth16, PLONK (pairing-based) or STARK (hash-based) | wrap proof and offline memory check are EC-based | **ZKM**, with the L2 |
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

Nothing post-quantum has shipped to Ethereum mainnet on any of these fronts;
EIP-7885, EIP-8141 and
[EIP-8151](https://eips.ethereum.org/EIPS/eip-8151) are all Draft.

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

Six of the seven surfaces below carry quantum-exposed cryptography, and they are
not equally GOAT's to fix. Row numbers refer to the taxonomy of section 2, so
that the case study and the framework can be read against each other.

| Row | Surface | Repo | Scheme | Quantum status | Owner of the fix |
| --- | --- | --- | --- | --- | --- |
| 1 | Bitcoin L1 outputs | — | secp256k1 / Schnorr | broken by Shor | **Bitcoin, not GOAT** |
| 1–2 | Peg custody | [`bitvm2-node`](https://github.com/GOATNetwork/bitvm2-node) | MuSig2 over a **Taproot key path** | broken by Shor, **and exposed from output creation** | **GOAT** |
| 2 | Bridge attestation | [`goat`](https://github.com/GOATNetwork/goat) relayer module | secp256k1 / Schnorr | broken by Shor | **GOAT** |
| 3 | Bridge proof system | [`bitvm2-node`](https://github.com/GOATNetwork/bitvm2-node) | [Ziren](https://github.com/ProjectZKM/Ziren) STARK (**DLOG multiset memory check**) → **Groth16/BN254** wrap → garbled | broken at three layers | **GOAT and ZKM** |
| 4 | Bridge bit commitments | [`bitvm2-node`](https://github.com/GOATNetwork/bitvm2-node) → [BitVM](https://github.com/GOATNetwork/BitVM) | **Winternitz OTS** | **already PQ-safe** | — |
| 5 | Consensus keys | [`goat`](https://github.com/GOATNetwork/goat) (CometBFT) | **secp256k1** | broken by Shor | **GOAT** |
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
of section 16.)

Groth16 is a wrapper, not the proving system. `bitvm2-node` depends on
[Ziren](https://github.com/ProjectZKM/Ziren), a FRI/STARK zkVM over Poseidon
and a small prime field, and the Groth16 proof elements are committed into
Bitcoin script through Winternitz signatures. So the pipeline is: Ziren
produces a hash-based STARK proof, that
proof is wrapped into a constant-size Groth16 proof over BN254, and the Groth16
verifier is what BitVM2 runs in Bitcoin script.

The STARK core, however, is not post-quantum either. FRI and Poseidon are
hash-based, so the core appears safe; the issue tracker records otherwise. Ziren issue
[#276](https://github.com/ProjectZKM/Ziren/issues/276), *"Replace hash-to-curve
in multiset hash by quantum safe primitives"*, open since 2025-08-14:

> In our multiset hash, we rely on hash-to-curve to calculate the hash of values
> of the memory addresses, and consider each hash as the x of a point, then
> commit the accumulation of all the points in the EC subgroup. **If DLOG
> hardness is preserved**, prover can not forge another set of scalars for all
> the points respectively, and hence can not forge the proof of the offline
> memory checking. […] To achieve quantum safe, we need to remove the DLOG
> hardness assumption.

So it is the offline memory-consistency argument, not the polynomial commitment,
that rests on discrete log: a quantum adversary forges the memory check rather
than attacking FRI, and the execution proof falls with it.

The fix is a lattice-based multiset hash (LtHash), and it is already past the proposal stage.
Ziren's [`feat/lthash`](https://github.com/ProjectZKM/Ziren/tree/feat/lthash)
branch prototypes the replacement, reaching into the core machine and the
recursion circuits, following Zisk's
[lattice-based multiset hashing](https://zisk.technology/secure-challenge-derivation-in-zisk/).

This matters for how the layer should be ranked. Of the three quantum-exposed
layers in this proof pipeline, this is the tractable one: it is a primitive
swap inside an existing argument, not an architectural change, and someone has
already built it. It remains ZKM's work rather than something GOAT can fix in
`bitvm2-node`, and it still gates everything above it (a post-quantum wrapper
over a DLOG-dependent execution proof buys nothing), but it should be read as a
dependency to track and support, not as an open research problem.

A second solution exists: reverting the memory argument to LogUp. Ziren's own design documents record that memory consistency
was a LogUp lookup between the global and local memory tables before the
multiset hash replaced it, and Ziren still uses LogUp for every other cross-chip
check, so the machinery is in the tree today. A LogUp memory argument, with GKR
to prove the fractional sums as in the LogUp-GKR variant of Papini and
Haböck, is field arithmetic committed under FRI and nothing else. It removes
the discrete-log dependency without putting a lattice assumption in its place:
the proof system's assumption set returns to the hash alone. What it gives back
is what the multiset hash was adopted for. With execution split across shards,
the LogUp challenge has to be derived after commitments to every shard's read
and write sets, so the incremental, challenge-free accumulation the ECMH
provided has to be replaced by an accumulated or two-pass challenge derivation
across shards. That is an engineering cost inside a mechanism Ziren already
ships, not a new primitive. Either route closes the gap; the LogUp route closes
it with the smaller assumption set, and it is the one this report recommends
where the sharded challenge derivation is acceptable.

And the garbling stack is built around Groth16 specifically. `bitvm2-gc` is
not a general circuit garbler that happens to be pointed at Groth16; its
principal component is a garbled *SNARK verifier*, and its circuit tree is
organised around the Groth16/BN254 verification circuit. Swapping the proof
system therefore means rebuilding the garbled-circuit stack around a different
verifier rather than deleting a wrapper, which is the single largest
item in this report.

The proof path contains three independent Shor-vulnerable layers, not one:

| Layer | Primitive | Quantum status | Where the fix lives |
| --- | --- | --- | --- |
| Ziren FRI / Poseidon commitment | hash-based | safe | — |
| Ziren multiset memory check | **hash-to-curve (ECMH), DLOG** | **broken** | ZKM, [Ziren #276](https://github.com/ProjectZKM/Ziren/issues/276); **two solutions**: the [`feat/lthash`](https://github.com/ProjectZKM/Ziren/tree/feat/lthash) prototype, or reverting the memory argument to LogUp-GKR |
| Groth16 wrap | **BN254 pairing** | **broken** | GOAT |
| `bitvm2-gc` garbled verifier | AES-128 garbling of a **Groth16 verifier** | garbling safe, **statement broken** | GOAT, and it is a rebuild |
| BABE / Deferred Binding | witness encryption **against the Groth16 relation** | on-chain surface safe, **relation broken** | GOAT; see section 9 |
| WOTS commitment into script | hash-based | safe | — |

The two hash-based layers at the ends are fine. Everything between them depends
on either DLOG or pairings, and the first break sits *inside* the component
usually described as the hash-based one. Any credible post-quantum plan for this
bridge has to address all of them, in dependency order: Ziren's memory check
first, since a post-quantum wrapper over a DLOG-dependent execution proof buys
nothing.

The last row is the least obvious, and it is treated on its own in section 9,
because the on-chain verifier has been through four designs and they are best
read as one topic.

The cost is size and script weight. Hash-based proofs are substantially larger
than Groth16's constant-size proof, and BitVM2's economics depend on what fits
in Bitcoin script and what a challenge transaction costs. That trade is the real
engineering question, and it should be measured before it is decided.

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
[partial-binding witness encryption](https://github.com/GOATNetwork/bitvm2-gc/blob/feat/goat-bitvm3/docs/partial_binding_we.tex),
resolves this with a Dual-Scalar Garbled Circuit whose two outputs,
`r * pi_1` and `r * P_D + r * B`, "provide prefix binding on `x_S`
and proactive cryptographic binding on `x_D`", with `B` a verifier-chosen
blinding point hidden inside the circuit. It is implemented, not proposed: the
[`feat/goat-bitvm3`](https://github.com/GOATNetwork/bitvm2-gc/tree/feat/goat-bitvm3)
branch of `bitvm2-gc` carries the BABE implementation alongside the older
garbled verifier.

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

This is the fourth instance of the wrapper pitfall in this stack, and the least
conspicuous. The first three (Winternitz commitments carrying a
Groth16 proof, garbled circuits garbling a Groth16 verifier, and the prospective
error of proving BLS verification inside `leanVM`, section 13) at least look
like wrappers. Here the vocabulary actively argues the other way: *witness
encryption*, *garbled circuit*, *hashlock*, *Lamport* are all post-quantum-safe
terms, three of the four on-chain primitives are indeed post-quantum, and the
security proof is a clean reduction. The pairing survives in the *relation being
encrypted against*, which is the one place the terminology does not indicate.

#### Ziren on the dispute path

Deferred Binding needs *soldering* (translating garbled labels across
cut-and-choose instances) and proves it in a zkVM: the soldering program in the
BABE implementation is a Ziren guest. GOAT's note reports the resulting STARK
proof at 56.92 MB over roughly 94 million cycles.

That has a consequence the note does not draw. Ziren's offline memory-consistency
argument rests on an elliptic-curve multiset hash, the ECMH construction of
[Ziren #276](https://github.com/ProjectZKM/Ziren/issues/276), whose own text says
soundness holds only "if DLOG hardness is preserved" (section 8). So the
soldering proof, which is what makes the cut-and-choose argument and therefore
the 1-of-`n` honest-verifier assumption meaningful, inherits a second,
independent discrete-log dependency. Before this work, Ziren's exposure sat in
the proving pipeline. It now also sits on the bridge's dispute path.

Two DLog dependencies in one protocol, reached by different routes, is a
planning-relevant fact: [Ziren #276](https://github.com/ProjectZKM/Ziren/issues/276)
and its [`feat/lthash`](https://github.com/ProjectZKM/Ziren/tree/feat/lthash)
prototype are now load-bearing for two surfaces, not one, which strengthens the
case for phase 3 of the ordering in section 15.

One version detail affects which Ziren fixes are inherited: GOAT's note cites
Ziren 1.2.5, but the implementation pins 1.2.4.

#### The garbling security parameter

The garbling layer uses 128-bit wire labels under an AES-128 PRF, in both the
older garbled verifier and the BABE implementation. 128-bit labels under AES-128 put the
garbling layer at NIST post-quantum security category 1, the floor of the
approved range, since Grover search over a 128-bit space is the reference for
that category. That was a defensible default when garbling was an optimisation.
Under BABE it is the mechanism that gates decryption of the payout secret, so
the decision deserves to be made explicitly rather than inherited. Moving labels
and PRF to 256-bit restores category-5 margin at roughly double the garbling
cost.

The rest of `bitvm2-gc` is unchanged in character: the circuit being garbled
remains the Groth16/BN254 verifier, with symmetric primitives appearing only
in the garbling machinery, and the bridge node still authorises peg-outs with
MuSig2 over Taproot, so the key-path exposure of section 11 is untouched by
any of this.

#### Non-transferability to a Ziren STARK verifier

The natural next question is whether the same machinery can carry a hash-based
verifier: witness-encrypt against Ziren's STARK instead of Groth16, and the
BN254 dependency disappears. It does not follow, for a structural reason.

BABE is a witness encryption scheme for linear pairing relations. What makes
Groth16 expressible is that its verification equation is a pairing product, so
`Y` can be formed at encryption time from `vk` and the public input, and
recovered at decryption time by pairing the proof elements. A FRI/Poseidon
verifier has no pairing relation, no target group, and no short algebraic
acceptance predicate: it is a Merkle-and-hash argument with a large,
non-algebraic verification transcript. There is nothing for the ciphertext to be
keyed to. Constructing the analogue would mean a witness encryption scheme for
hash-based proof systems, which is not a parameter change but an open problem.

That produces a strategic tension. BABE makes
Groth16 dramatically cheaper to verify on Bitcoin, which reduces the pressure to
stop using Groth16, and stopping is the post-quantum requirement. The cost
curve and the quantum curve point in opposite directions. Every increment of
engineering invested in the BABE path deepens the commitment to a pairing-based
statement, and section 8's conclusion still stands: `bitvm2-gc` is
Groth16-oriented by construction, so moving off pairings is a rebuild of that
component rather than a swap of a wrapper.

#### Assessment

The net effect of this line of work on the post-quantum position is nil, and it
is not cost-free. Three observations support that assessment: the pairing
assumption lives in the security theorem of the scheme that replaces the
on-chain verifier, not merely in a script the bridge could swap out; Ziren's
discrete-log dependency is load-bearing in two places, the proving pipeline and
the dispute path; and the AES-128 garbling parameter gates the payout secret
rather than sitting as an optimisation detail.

None of this is an argument against BABE, which answers the question it was
asked — on-chain cost — and produces an on-chain surface that is genuinely
hash-based. It is an argument against *counting* it as post-quantum progress,
and against the assumption that a future hash-based proof system can be dropped
into the same protocol.

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
concatenated bytes.

That reopens the field: both remaining candidate families are expressible
under current consensus rules.

#### FRI over an algebraic hash

This is the proof GOAT already has: Ziren is a FRI zkVM committing through
Poseidon2 over KoalaBear. Verifying it on Bitcoin means writing a verifier for
proofs the bridge produces today: no new proof system, no change of prover,
nothing to re-audit upstream.

The construction requires no `OP_CAT`. The claim calls for justification,
because hashing is where concatenation normally enters; it does not enter here.
A sponge digest is eight field elements, not thirty-two bytes, so compressing
two children feeds sixteen field elements into a width-16 permutation: sixteen
stack items handed to an arithmetic routine. There is nothing to concatenate;
the "concatenation" is the
order the state sits in. With SHA256 the two children are two 32-byte items and
`OP_SHA256` hashes only one item, so they must first be fused into a single
64-byte item, and that is `OP_CAT`. The dependency belongs to the byte hash, not
to Merkle verification.

So a FRI verifier (Merkle paths, transcript, folding check) is field
arithmetic over a 31-bit prime, which fits a script number directly. No
multi-limb representation and no soft fork.

The cost is the hash, and it is measured rather than estimated.
[`bitcoin-stark-verifier`](https://github.com/AppliedPQC/bitcoin-stark-verifier)
implements the permutation for [Plonky3](https://github.com/Plonky3/Plonky3)'s
width-16 KoalaBear instance at roughly 572 KB of script, which in a taproot
witness is the same number of weight units: 1.43 standard transactions, or
14.3% of a block, and a Merkle path costs one permutation per level. The
measured instance uses more internal rounds than Ziren's, so the figure is an
upper bound for a Ziren-targeted verifier. The S-box dominates the cost: the
permutation performs 148 cubings, each built from emulated field
multiplications.

The number is a starting point rather than a floor: several known
inefficiencies in the implementation leave room to reduce it, but none changes
the order of magnitude. A depth-21 Merkle path is roughly 12 MB of script,
three full blocks.

This is also why
[`bitcoin-circle-stark`](https://github.com/Bitcoin-Wildlife-Sanctuary/bitcoin-circle-stark)
builds its transcript and Merkle components on `OP_CAT` with `OP_SHA256`: a byte
hash is one opcode where an algebraic hash is hundreds of emulated
multiplications. `OP_CAT` is what makes FRI on Bitcoin *cheap*, not what makes it
*possible*.

A third family merits brief mention, if only to establish that it does not
constitute a separate option.
[BaseFold](https://eprint.iacr.org/2023/1705) generalises FRI to foldable codes
and would verify through the same Merkle-and-transcript machinery over the same
algebraic hash, so it lands in the FRI column on cost and inherits the same
per-level hash. It changes the prover's options, not the script's.

#### Module-lattice arguments

Schemes in the LaBRADOR line commit algebraically (Ajtai/module-SIS
commitments verified by ring arithmetic), so they have no hash in the opening at
all, and no `OP_CAT` dependency for the same reason Ziren's Poseidon2 has none.
Verification is ring multiplication over Z*q*[X]/(X^*d*+1), which is an NTT of
*d* points, (*d*/2)·log₂*d* butterflies built from the double-and-add primitive
BitVM already ships.

Two trades against FRI. The first is field width: a 64-bit modulus needs three
29-bit limbs where a 31-bit field needs one, so each multiplication is dearer,
though still cheaper than the nine-limb BN254 multiplication BitVM2 performs
thousands of times, which is the evidence the class of work is affordable.

The second is about the prover, not the script. Ziren commits with FRI, so a
lattice argument is a change to its polynomial commitment scheme: the commitment
itself, the opening argument, and the recursion circuits that verify them. That
is the scope of the change, and Ziren is already being changed in exactly this
way: the [`feat/lthash`](https://github.com/ProjectZKM/Ziren/tree/feat/lthash)
prototype swaps ECMH for a lattice multiset hash through the same core
machinery and recursion circuits. The question is the scope of a prover
upgrade, not whether one is available.

Wrapping remains a fallback: proving inside a lattice argument that Ziren's STARK
verified is sound rather than circular, because the wrapped statement is itself
post-quantum. Unlike the Groth16 and BABE cases of section 9, the
composition inherits a post-quantum assumption rather than laundering a broken
one, so it is not the wrapper pitfall. But it costs a recursion layer that changing the
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
verifier is, not whether one is possible. FRI over Poseidon2 verifies what Ziren
already emits and pays a few hundred multiplications per Merkle level; a
module-lattice argument has no per-level hash and the smallest proofs available,
but pays for them in verifier work and changes Ziren's commitment scheme with
it. Neither fits a single transaction,
and neither needs to (BitVM already chunks a verifier across a disprove
pattern), so the question is how many chunks each option costs.

#### The full verifier, measured

The permutation cost above is one hash. What a bridge executes in a dispute is
a full verifier:
[`bitcoin-stark-verifier`](https://github.com/AppliedPQC/bitcoin-stark-verifier)
adds a WHIR opening verifier on top of the Poseidon2 script, with zero uses of
`OP_CAT` or any other disabled opcode. Its end-to-end test runs
[Plonky3](https://github.com/Plonky3/Plonky3)'s real prover, checks the proof
with Plonky3's own verifier, then re-derives the same claims in script and
executes them, with no hand-built vectors.

Composed from a Plonky3-derived configuration at 100-bit security with 22 bits
of grinding, the verifier is 979 permutations, 560 MB, 140 full blocks. At
80-bit it is 743 permutations and 106 blocks: lowering the security level buys
*fewer queries, not cheaper ones*, because a query's depth is set by the domain
size and not by λ.

Four observations follow from the measurement.

Query work is 97.1% of the script, of which the Merkle paths are 60% and the
leaf hashing 37%. The leaf-hashing term is the less expected of the two: binding an opened row to
its Merkle leaf (without which a spender can authenticate the committed leaf and
fold a different row) costs eight permutations per query at folding factor 4, on
top of the twenty-one the path costs. Sumcheck rounds at 95,100 bytes each are
negligible against that, and optimisation effort spent anywhere but hashing
addresses 3% of the problem.

Multilinear evaluation is exponential in the variable count, about
23.9 KB × 2ⁿ: 740 KB at n = 5, 24 MB at n = 10, 783 MB at n = 15. It passes
Merkle work at around twelve variables, so for a realistic witness it, not the
query count, is the binding term.

The closing identity costs 92.8 MB, 7.5% of the verifier, and no hash at
all. That is with its constraints derived from the transcript, which is what
stops a spender choosing them; against a supplied list it is 1.5 MB. The algebra
is cheap in the sense that matters (it buys no permutations) without being
negligible.

> *The disprove pattern, briefly.* The verifier is never run on chain in the
> happy path. An operator posts a claim and a bond; the script is cut into
> segments, each small enough for one transaction, and each segment's inputs and
> outputs are committed with Winternitz signatures so the segments chain
> together. If the claim is honest, nobody executes anything and the bond
> returns. If it is not, a challenger finds the one segment whose output
> contradicts its input and spends a Disprove transaction that runs *only
> that segment*, taking the bond. So the on-chain cost is one segment, not the
> whole verifier, but every segment must still be pre-signed and stored off
> chain, which is why total size sets the setup cost and the number of segments
> sets the challenge latency.

Next, the chunk count, with the commitment layer priced. A disprove step
cannot be one Merkle level, which already exceeds the standard transaction
weight; it must be a fraction of a permutation. A round predicate over
*supplied* states decides nothing about whose claim it is, so each chunk's
input and output states must be bound by Winternitz one-time signatures —
which fit Bitcoin without a soft fork, since a hash chain only ever hashes a
single item — and the signature parameters are forced by the stack limit
rather than chosen freely. Committing at chunk boundaries rather than at every
round, a chunk of about 22 of the permutation's 29 rounds fills a standard
transaction together with its two state commitments.

The 80-bit, 20-variable verifier is then 1,990 chunks, and 100-bit at 16
variables is 1,958.

Two thousand is the same order as BitVM2's own leaf count, which is what makes
the FRI route *viable* rather than merely cheaper than a pairing verifier nobody
can run either. Unlike the earlier figure it is a bound on the setup as well
as the spend, since the commitments are what setup consists of.

The measurement is a lower bound, not a simulation of GOAT's pipeline: it
verifies a WHIR opening and its sumcheck, not a zkVM execution proof. It settles
the cheap end, which is what the comparison needed.

The lattice side remains unmeasured. FRI's cost is known and large; the
module-lattice verifier's is unknown, but it has no per-level hash to pay. That is the remaining measurement,
and it is the one that decides the row.

Two things remain unestablished and should be settled with it. RoKoko's concrete
ring parameters are not in its abstract, and the modulus width drives its whole
cost. And Fiat-Shamir needs a hash in both designs: either an algebraic one
inside the script, or none at all if the challenge comes from BitVM's on-chain
challenge-response rather than from a transcript, which would remove the largest
cost in the FRI column and should be settled first.

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

### 12. goat: relayer and consensus

The surfaces in this section admit the least costly migrations in the stack.

Relayer attestation keys are the bridge trust root, held as a tagged union
over secp256k1 and Schnorr. The union is an extensibility point, so adding an
ML-DSA-65 variant extends an existing pattern. The blocker is a fixed-length
gate in signature verification, which rejects any signature that is not
exactly 64 bytes. An ML-DSA-65 signature is 3309 bytes and its public key
1952 bytes, both confirmed against an independent ACVP-verified FIPS 204
implementation. Until that check is per-variant, the verification path is
structurally incapable of accepting a post-quantum signature.

On consensus keys, Cosmos SDK v0.55 registers ML-DSA-65 as a validator
consensus key type, opt-in behind
`genesis.consensus_params.validator.pub_key_types`, with validator key rotation
shipping in the same release. GOAT pins SDK v0.53.8 from upstream, so this is
a dependency upgrade across two minors rather than a fork rebase.

This requires demonstration, because the dependency manifest suggests
otherwise at first reading: it carries a commented-out fork substitution for
the SDK next to a live one for the EVM client. So `goat-geth` is a real fork
and the SDK fork is not part of this build. A search that does not distinguish
the two concludes there are two forks, which inverts the schedule estimate for
this row: a rebase is the critical path, an upgrade is not.

GOAT's validators sign with secp256k1, not the Cosmos default of ed25519, and
the mainnet genesis sets the permitted key types explicitly rather than
inheriting a default. This changes nothing about the exposure, since both fall
to Shor, but it does mean the migration lever is already in use: adopting
ML-DSA-65 is an edit to a value the genesis already carries.

Two version facts bound that work. The pinned CometBFT release predates both
the ML-DSA-65 key type and the expanded signature budget described in
section 6. And the configured block-size limit is about 6 MiB, where the
arithmetic that produces it suggests 50 MiB was intended. Whatever the intent,
a chain moving to 3309-byte consensus signatures
should confirm this number deliberately: at 6 MiB the commit is a materially
larger fraction of the block than the CometBFT defaults assume.

Consensus keys lose nothing structurally in that move, because CometBFT never
aggregated them: a commit is an array of one signature per validator (section 6),
so adopting ML-DSA-65 costs bytes in proportion to the validator set and nothing
else. The work is a dependency upgrade and a block-parameter re-tune.

The relayer vote key is the opposite case, and the distinction is subtle. Its
aggregation is not inherited from Cosmos: it is GOAT's own code. Upstream has
no aggregation to extend, and the attempts to add it have not landed, so no
amount of tracking Cosmos releases produces an answer for this surface. It is
GOAT's alone, and it is the one place in this stack where the post-quantum move
costs a *property* rather than bytes.

The chain does not appear to use IBC, so the light-client hazard that
dominates Cosmos post-quantum migration (enabling a key type counterparties
cannot verify stops packet flow and expires the client) appears not to apply.
This should be confirmed against deployment reality rather than the dependency
manifest alone.

### 13. The relayer's BLS vote key

Section 12's recommendation addresses the attestation key. That is not the whole
picture, because the relayer carries three distinct key types, not one:

| Key | Scheme | Purpose |
| --- | --- | --- |
| attestation key (tagged union) | secp256k1 **or** Schnorr | attestation / proposals |
| transaction key | secp256k1 | transaction authorisation |
| vote key | **BLS12-381 G2, 96-byte compressed** | voting, verified in aggregate |

Adding an ML-DSA-65 variant to the attestation key type remains correct and remains the
cheapest first move, but it addresses only the attestation key. The vote key is a different problem, and a much harder one,
because its value is aggregation. BLS lets N relayer votes verify as one
48-byte signature. No standardised post-quantum signature aggregates: ML-DSA and
SLH-DSA have no aggregation, so replacing BLS naively turns one signature into
N, at 3309 bytes each. For twenty relayers that is roughly 66 KB where there was
48 bytes.

That aggregation is in use, not merely available, is settled at the
verification site rather than by the presence of an aggregate API. The
verification is specifically the many-signers, one-message case: a
participation bitmap selects voters, participation is checked against a
threshold, and one aggregate signature is verified against the selected public
keys over a single sign-document. Relayer consensus is therefore a threshold
vote carried by one 48-byte signature regardless of signer count, and the
bitmap design exists precisely so that signer count can grow.

BLS supplies two properties at once here, and the post-quantum replacement is
two questions rather than one. It aggregates: N independently generated keys
produce one constant-size object. Separately, the surrounding code supplies a
threshold, since the bitmap selects voters and the participation count is checked
against `Threshold()`. GOAT therefore uses BLS as an *aggregate signature* with
the threshold enforced in application logic, not as a threshold signature. The
distinction decides which replacements are like-for-like. A post-quantum
aggregation scheme preserves the bitmap, preserves per-signer attribution, and
needs no key-generation ceremony. A threshold signature moves the threshold into
the cryptography, requires a distributed key generation, and produces a signature
that does not identify who signed.

#### The limits of zkVM wrapping for BLS verification

A natural approach is to wrap the existing BLS verification in a post-quantum
zkVM: prove inside `leanVM` that the aggregate verified, and inherit the zkVM's
hash-based soundness. That does not work, in two distinct senses, and the second
is a restatement of this report's central pitfall.

Mechanically, there is nothing to build on. `leanVM` contains no BLS or
pairing code at all; it is built from KoalaBear field arithmetic, Poseidon,
WHIR and XMSS, with recursive aggregation as a dedicated component. It is
purpose-built to recursively aggregate *hash-based* signatures. A BLS verifier could be written as a guest program, but
emulating BLS12-381 pairing arithmetic over a 31-bit field is precisely the work
that EVM chains give a native precompile to avoid.

And even if it were built, it would buy nothing. Proving "this BLS aggregate
verified" inside a hash-based zkVM yields a post-quantum-sound *proof* of a
quantum-broken *statement*. An adversary who can forge BLS signatures forges one
and then honestly proves that it verified; the zkVM faithfully attests to a true
claim about a dead assumption. This is the same error as Winternitz commitments
carrying a Groth16 proof, garbled circuits garbling a Groth16 verifier, and
witness-encrypting against the Groth16 relation (section 9): the fourth
instance in this one stack, and the only one that would be a *prospective*
mistake rather than an existing one.

What `leanVM` offers is the removal of the need for BLS. BLS was
chosen for one property, aggregation. Hash-based signatures are post-quantum but
do not aggregate. Recursive proving restores that property. The path is
therefore:

```
BLS                    hash-based signatures       + recursive aggregation
(aggregates, broken) →  (safe, N x 3309 bytes)  →  (aggregation restored)
```

not "keep BLS and add a zkVM". The ordering matters: the signature scheme is
replaced first, and recursion recovers what the replacement costs.

Concretely for GOAT, the aggregate verification is not wrapped, it is
replaced, with the threshold-and-bitmap voting logic rebuilt around a hash-based
scheme plus recursion. GOAT already operates a zkVM (Ziren) and a garbling
stack, so the ingredients are unusually close to hand, but this is an
architectural workstream, not a key-type change, and it should be planned
separately from the attestation-key work.

#### Threshold signing

Recursion is not the only way to recover a constant-size vote. The requirement at
the verification site is narrower than aggregation in general: many signers, one
message, checked against a threshold. A threshold signature has that shape
natively, since N parties jointly emit one ordinary signature, and it leaves the
verifier untouched.

It is not, however, a smaller version of aggregation. A single signing key exists
as a mathematical object even where it is never materialised, which requires
either a distributed key generation or a dealer; the signer set is fixed at key
generation and changing it requires re-sharing; and the resulting signature does
not identify its contributors. Aggregation has none of the first three properties
and does have the last. The two are incomparable rather than ordered.

| | Parties | Verifier | Communication |
| --- | --- | --- | --- |
| [Hermine](https://eprint.iacr.org/2026/419) (Borin et al., ASIACRYPT 2026) | **N ≤ 64**; concrete parameters derived for N ≤ 16, T ≤ 8 | Raccoon, 11.3 KB signature | 73.2 KB per party |
| [Quorus](https://eprint.iacr.org/2025/1163) (Bienstock et al., USENIX 2026) | **any number**, t-of-n | unmodified ML-DSA; signature and key sizes match | ~100 KB per party per rejection-sampling round |
| [Efficient Threshold ML-DSA](https://eprint.iacr.org/2026/013) (Celi et al.) | up to **6** | unmodified ML-DSA | ≤ 1 MB per party |

Two of the three keep the chain verifying one standard signature with a stock
FIPS 204 verifier however many relayers voted, which is the constant-size
property that made BLS attractive. Hermine instead emits a Raccoon signature, so
adopting it changes the verification scheme as well as the signing protocol, and
Raccoon is itself a NIST additional-signature submission rather than a standard.

The price is a change in signing shape rather than in key type, and it is smaller
than it was. Today each relayer signs independently and offline, and the bitmap
records who did; a threshold protocol replaces that with a coordinated session
among the selected quorum. Hermine's first round is independent of both the
message and the signer set, so it can be preprocessed and the online phase is a
single round, and its non-interactive identifiable abort attributes a failed
session from the transcript alone, which is what a slashing design requires. The
residual cost is that the online round needs the quorum simultaneously available,
where the present design tolerates arbitrary staggering.

Two obstacles are larger than the protocol shape. The first is key generation.
Hermine assumes a trusted dealer and leaves distributed key generation for
Vandermonde sharing as an open problem; the compact construction of del Pino and
Niot omits it and notes that adding it would increase signature size. For a peg
trust root a dealer is not acceptable, because it means one party holds the whole
key at setup. Among lattice threshold schemes only Pelican and Olingo ship a
distributed key generation, and neither offers the feature set above. This, and
not signature size, is what currently blocks deployment.

The second is attribution. The bitmap does double duty: it selects voters and it
records participation. A threshold signature removes the second function, since
the output is indistinguishable from a single signer's. Where relayer rewards or
slashing depend on who voted, that logic has to be rebuilt or carried separately.

Standardisation has moved without arriving. NIST's first call for multi-party
threshold schemes, [IR 8214C](https://csrc.nist.gov/projects/threshold-cryptography),
reached final on 2026-01-20 with preview submissions due 2026-08-07, and Hermine
has been submitted to it. A submission is not a selection, and for a peg trust
root, depending on a construction with no FIPS number and no validation
programme remains a governance decision as much as a technical one. It argues for
sequencing the attestation key, which needs no aggregation at all, ahead of the
vote key.

Two adjacent results do not fit this surface. The compact scheme of del Pino and
Niot reaches 2.7 KB, close to a single Dilithium signature, but only at N = 8,
because its replicated secret sharing grows as a binomial coefficient; Hermine's
Vandermonde sharing is what raises the ceiling to 64, at roughly four times the
signature size. [MuSig-L](https://eprint.iacr.org/2022/1036) is the lattice
analogue of MuSig2 and needs no key-generation ceremony, but it is n-of-n rather
than t-of-n and reports no concrete parameters.

#### Post-hoc signature aggregation

The remaining option belongs with aggregation rather than with the threshold
schemes above: squash N existing ML-DSA signatures into one small object without
changing how anyone signs. It is the least developed of the three.
[Boudgoust and Takahashi](https://eprint.iacr.org/2023/159) (ESORICS 2023) gave
the first Fiat-Shamir-with-aborts aggregate signature, applicable to Dilithium,
and report "quite small compression rates" in their own words; it also aggregates
*distinct* messages, where this use case has one. The strong result in this line,
[aggregating with LaBRADOR](https://eprint.iacr.org/2024/311) (Aardal et al.,
CRYPTO 2024), is for Falcon, not ML-DSA. That asymmetry is a
design signal: if aggregability is a first-order requirement, it is an argument
about *which* post-quantum scheme to adopt, not a problem to be solved after
adopting ML-DSA.

### 14. goat-geth: the divergence, measured

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
that migrates to post-quantum authorization via
[EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) has its ECDSA
*transaction* authority disabled by
[EIP-3607](https://eips.ethereum.org/EIPS/eip-3607) (Final), but `ecRecover` ignores
account state, so a quantum attacker holding the derived ECDSA key could still
authorize transfers through immutable contracts, notably ERC-20 `permit`
implementations. EIP-8151 makes `ecRecover` return zero when the account has
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

### 15. Ordering the work

Ownership and severity, not novelty, should set the
order:

1. Bridge attestation keys (row 2). Highest value per unit of control. The
   operator or relayer set is the peg's trust root; compromising a threshold is
   equivalent to compromising the peg. It is entirely the L2's to change, and it
   can usually be rolled out with dual signing, giving rollback at every step.
2. Bridge proof system (row 3). Highest severity, and the hardest problem
   in the list: it fixes soundness rather than theft, and it means removing the
   Groth16 verifier the bridge is built around. Proof size and verification cost
   drive the L2's economics, so a constant-size pairing proof cannot simply be
   exchanged for a larger hash-based one. Sequenced second because it is
   expensive and depends on upstream work, not because it is second in
   difficulty.
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
| 0 | Inventory every signature and proof verification path; add tests asserting no fixed signature-length assumptions | The relayer's 64-byte signature gate shows these assumptions are load-bearing and invisible |
| 1 | Relayer: add an ML-DSA-65 variant to the attestation key type, make length checks per-variant, roll out with dual attestation | Highest value per unit of control; the key type is already extensible; dual signing gives rollback at every step |
| 2 | Peg custody: write and enforce an exposure policy (rotate custody outputs, cap value per output, avoid long-lived connectors) | The Taproot output key is on chain from creation and is sufficient to spend, so no internal-key choice removes the exposure (section 11). Only the *window* is GOAT's to shrink; the fix is BIP-360 and Bitcoin's timeline |
| 3 | Track and support [Ziren #276](https://github.com/ProjectZKM/Ziren/issues/276) through to merge, preferring the LogUp-GKR memory argument over [`feat/lthash`](https://github.com/ProjectZKM/Ziren/tree/feat/lthash) where sharded challenge derivation is acceptable | Gates everything above it, and is the tractable layer: either a primitive swap with a working prototype, or a reversion to the lookup argument Ziren already uses elsewhere, which leaves the hash as the only assumption. Influence and test rather than implement. Now load-bearing twice, since BABE soldering also proves in Ziren (section 9) |
| 4 | **Stop verifying a pairing on Bitcoin**: re-target the proof pipeline and the `bitvm2-gc` garbling stack away from Groth16/BN254. The FRI half is measured end to end (979 permutations and about 1,958 disprove chunks at 100-bit, with the commitment layer priced), so what remains is the module-lattice verifier | The hardest item here, and the one that ships last. `bitvm2-gc` is Groth16-verifier-oriented by construction, so this is a rebuild rather than a wrapper swap, and BABE cuts against it by lowering the cost of *keeping* Groth16. But section 10 finds both post-quantum candidates expressible with the opcodes Bitcoin has, with no soft fork, and the chunk count is the same order as BitVM2's own, so what gates the decision is the lattice verifier's cost rather than feasibility |
| 5 | Upgrade the Cosmos SDK to ≥ v0.55; opt into ML-DSA-65 consensus keys; rotate validators; re-tune block-size and gossip limits | No SDK fork exists, so this is a dependency upgrade rather than a rebase |
| 6 | Reduce `goat-geth`'s 377-commit lag; inventory callers of `0x06`–`0x08` and `0x0a`, and record for each whether it is **upgradeable** | The lag is the delivery channel for EIP-7885 and EIP-8151 when they land. The inventory's key column is upgradeability, not existence: an upgradeable verifier is tractable whatever upstream does, an immutable one has a deadline that cannot move |
| — | Peg: minimise Bitcoin-side key exposure; keep custody policy migratable | Blocked on Bitcoin, which by BIP-360's own text has no PQ signature scheme |

---

## Part V: Pitfalls and open items

### 16. Recurring pitfalls

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
- A "hash-based" proof system can still depend on DLOG. Check every
  sub-argument, not the headline commitment scheme: permutation arguments,
  lookup arguments and offline memory checks are all places where an
  elliptic-curve accumulator can hide inside an otherwise hash-based system.
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

### 17. Remaining open items

The list is short, and none of its items blocks the recommendations:

- The 37 `goat-geth` commits were classified by file area, not read line by
  line. Nothing in the changed areas is cryptographic, but a deliberate check
  of the type-definition changes against consensus-critical serialisation
  would close it fully.
- Which deployed L2 contracts call the `0x06`–`0x08` and `0x0a` precompiles is
  not enumerated, and neither is the more important question of whether each
  caller is upgradeable. That needs chain state rather than source, and it is
  the one item on this list with a deadline: an immutable caller can be
  identified now but never migrated later.
- Ziren's `feat/lthash` prototype has not been reviewed here for completeness or
  merged upstream; whether it covers every ECMH use site is unchecked. The
  LogUp-GKR alternative is stated from Ziren's design documents and the LogUp
  machinery in its tree, not from a prototype; its cost is the cross-shard
  challenge derivation, which is not measured here.
- The BABE work of section 9 lives on the `feat/goat-bitvm3` branch of
  `bitvm2-gc`, not on `main`, so the audit describes an in-flight design. The
  cut-and-choose and cost figures are taken from GOAT's note rather than
  reproduced; only the dependency structure, the Ziren pin and the garbling
  parameters were read from the tree.
- The module-lattice verifier has not been written in Bitcoin script, so its
  cost is unknown. The FRI side is measured end to end, including the chunk
  count, which makes this the one number that decides the proof-system row.
- The measured verifier covers a WHIR opening and its sumcheck, not a zkVM
  execution proof, and composes independent Merkle paths where the proof carries
  a pruned frontier. Both make it a lower bound; a batched verifier sharing
  internal nodes would be cheaper by some factor not yet established.
- The chunk count prices the on-chain spend and the number of one-time keys,
  but not generating them: the roughly 2,000 chunk boundaries require on the
  order of a quarter of a million hash chains, which is the cost BitVM2
  deployments actually struggle with and is not estimated here.
- The script figures are measured against Plonky3's Poseidon2 instance, which
  uses more internal rounds than Ziren's, so they are an upper bound for a
  Ziren-targeted verifier.
- No second L2 has been examined, so Part I's taxonomy is structural reasoning
  supported by one case, not a survey.

### References

Primary sources, each verified live: the base-layer and GOAT sources on
2026-07-31, the aggregation sources on 2026-08-01, the script-cost sources on
2026-08-02, and, on 2026-08-10, `goat`'s `go.mod` `replace` block and every
script figure in section 10, which were re-measured rather than re-read.

- BIP-360, *Pay-to-Merkle-Root (P2MR)* — <https://github.com/bitcoin/bips/blob/master/bip-0360.mediawiki>
- BIP-361, *Post Quantum Migration and Legacy Signature Sunset* — <https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki>
- SHRINCS Working Group, *SHRINCS: an efficient hash-based signature scheme for Bitcoin (first draft)*, bitcoin-dev, 2026-08-27 — <https://groups.google.com/g/bitcoindev/c/HbVboXIFiG8>, <https://github.com/SHRINCS/shrincs-bip/blob/main/SHRINCS.md>
- NIST, FIPS 205 (SLH-DSA) — the standardised SPHINCS+ variant SHRINCS uses for its stateless path — <https://csrc.nist.gov/pubs/fips/205/final>
- Bitcoin Optech, *Quantum resistance* — <https://bitcoinops.org/en/topics/quantum-resistance/>
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
- Ziren issue #276, *Replace hash-to-curve in multiset hash by quantum safe primitives* — <https://github.com/ProjectZKM/Ziren/issues/276>
- CometBFT, `types/block.go` (`Commit`, `CommitSig`, `MaxCommitBytes`) — <https://github.com/cometbft/cometbft/blob/main/types/block.go>
- CometBFT, `types/params.go` (`DefaultBlockParams`) — <https://github.com/cometbft/cometbft/blob/main/types/params.go>
- CometBFT issue #3455, *BLS signature aggregation* — <https://github.com/cometbft/cometbft/issues/3455>
- CometBFT issue #1305, *Halve commit size with partial ed25519 signatures* — <https://github.com/cometbft/cometbft/issues/1305>
- CometBFT PRs #3632 and #4763, `crypto/bls12381` signature aggregation, both closed unmerged — <https://github.com/cometbft/cometbft/pull/3632>, <https://github.com/cometbft/cometbft/pull/4763>
- G. Borin, S. Celi, R. del Pino, T. Espitau, S. Katsumata, G. Niot, T. Prest, K. Takemure, *Hermine: An Efficient Lattice-based FROST-like Threshold Signature*, ASIACRYPT 2026 — <https://eprint.iacr.org/2026/419>
- R. del Pino, G. Niot, *Finally! A Compact Lattice-Based Threshold Signature*, PKC 2025 — <https://eprint.iacr.org/2025/872>
- C. Boschini, A. Takahashi, M. Tibouchi, *MuSig-L: Lattice-Based Multi-Signature With Single-Round Online Phase*, CRYPTO 2022 — <https://eprint.iacr.org/2022/1036>
- A. Bienstock, L. de Castro, D. Escudero, A. Polychroniadou, A. Takahashi, *Quorus: Efficient, Scalable Threshold ML-DSA Signatures from MPC*, USENIX Security 2026 — <https://eprint.iacr.org/2025/1163>
- S. Celi, R. del Pino, T. Espitau, G. Niot, T. Prest, *Efficient Threshold ML-DSA* — <https://eprint.iacr.org/2026/013>
- K. Boudgoust, A. Takahashi, *Sequential Half-Aggregation of Lattice-Based Signatures*, ESORICS 2023 — <https://eprint.iacr.org/2023/159>
- M. Aardal, D. Aranha, K. Boudgoust, S. Kolby, A. Takahashi, *Aggregating Falcon Signatures with LaBRADOR*, CRYPTO 2024 — <https://eprint.iacr.org/2024/311>
- NIST, *Multi-Party Threshold Cryptography* (IR 8214C, final 2026-01-20) — <https://csrc.nist.gov/projects/threshold-cryptography>
- W. Beullens, G. Seiler, *LaBRADOR: Compact Proofs for R1CS from Module-SIS* — <https://eprint.iacr.org/2022/1341>
- *SALSAA — Sumcheck-Aided Lattice-based Succinct Arguments and Applications* — <https://eprint.iacr.org/2025/2124>
- M. Klooss, R. W. F. Lai, N. K. Nguyen, M. Osadnik, L. Tucci, *RoKoko: Lattice-based Succinct Arguments, a Committed Refinement* — <https://eprint.iacr.org/2026/575>
- H. Zeilberger, B. Chen, B. Fisch, *BaseFold: Efficient Field-Agnostic Polynomial Commitment Schemes from Foldable Codes*, CRYPTO 2024 — <https://eprint.iacr.org/2023/1705>
- Bitcoin Core, `src/script/script.h` (`CScriptNum::nDefaultMaxNumSize`, the four-byte arithmetic operand limit) — <https://github.com/bitcoin/bitcoin/blob/master/src/script/script.h>
- Bitcoin Core, `src/script/interpreter.cpp` (the disabled opcodes, including `OP_MUL` and `OP_CAT`) — <https://github.com/bitcoin/bitcoin/blob/master/src/script/interpreter.cpp>
- Bitcoin Core, `src/policy/policy.h` (`MAX_STANDARD_TX_WEIGHT`) — <https://github.com/bitcoin/bitcoin/blob/master/src/policy/policy.h>
- `BitVM/BitVM`, `bitvm/src/bigint/mul.rs` — double-and-add multiplication in Bitcoin script — <https://github.com/bitvm/bitvm>
- `Bitcoin-Wildlife-Sanctuary/rust-bitcoin-m31`, measured M31 script weights — <https://github.com/Bitcoin-Wildlife-Sanctuary/rust-bitcoin-m31>
- Zisk, *Secure challenge derivation* (lattice-based multiset hashing) — <https://zisk.technology/secure-challenge-derivation-in-zisk/>
- Ziren design documents, *Memory consistency checking* and *Lookup arguments* (the LogUp memory argument the multiset hash replaced) — <https://github.com/ProjectZKM/Ziren/blob/main/docs/src/design/memory-checking.md>, <https://github.com/ProjectZKM/Ziren/blob/main/docs/src/design/lookup-arguments.md>
- S. Papini, U. Haböck, *Improving logarithmic derivative lookups using GKR* (LogUp-GKR) — <https://eprint.iacr.org/2023/1284>
- Ziren, `crates/primitives/src/lib.rs` (`Poseidon2KoalaBear<16>`, `ROUNDS_F = 8`, `ROUNDS_P = 13`) — <https://github.com/ProjectZKM/Ziren>
- *Greyhound: Fast Polynomial Commitments from Lattices* — <https://eprint.iacr.org/2024/1293>
- `AppliedPQC/bitcoin-stark-verifier` — Poseidon2 and a WHIR opening verifier in Bitcoin script, without `OP_CAT`; source of every script-size figure in section 10 — <https://github.com/AppliedPQC/bitcoin-stark-verifier>
- Plonky3 — the WHIR prover and KoalaBear Poseidon2 instance the measurements are taken against — <https://github.com/Plonky3/Plonky3>
- Bitcoin Core, `src/consensus/consensus.h` (`MAX_BLOCK_WEIGHT`) — <https://github.com/bitcoin/bitcoin/blob/master/src/consensus/consensus.h>
- BIP-347, *OP_CAT in Tapscript* (E. Heilman, A. Sabouri; Consensus soft fork, Complete, not activated) — <https://github.com/bitcoin/bips/blob/master/bip-0347.mediawiki>
- `Bitcoin-Wildlife-Sanctuary/bitcoin-circle-stark`, a Circle STARK verifier in Bitcoin script — <https://github.com/Bitcoin-Wildlife-Sanctuary/bitcoin-circle-stark>
- `GOATNetwork/goat` — <https://github.com/GOATNetwork/goat>
- `GOATNetwork/goat-geth` — <https://github.com/GOATNetwork/goat-geth>
- `GOATNetwork/BitVM` — the fork carrying the Winternitz implementation (`bitvm/src/signatures/`) — <https://github.com/GOATNetwork/BitVM>
- `GOATNetwork/bitvm2-node` — <https://github.com/GOATNetwork/bitvm2-node>
- `GOATNetwork/bitvm2-gc` — <https://github.com/GOATNetwork/bitvm2-gc>

On-chain proof verification (section 9). Verified 2026-08-02.

- S. Garg, D. Kolonelos, M. Sergeevitch, S. Sridhar, D. Tse, *BABE: Verifying Proofs on Bitcoin Made 1000x Cheaper* — <https://eprint.iacr.org/2026/065>
- L. Eagen, Y. T. Lai, *Argo MAC: Garbling with Elliptic Curve MACs* — <https://eprint.iacr.org/2026/049>
- N. Khambhati, A. Bhattacharya, D. Heath, *Duty-Free Bits: Projectivizing Garbling Schemes* — <https://eprint.iacr.org/2026/476>
- R. Linus et al., *BitVM3: Efficient Bitcoin Bridges via Garbled Circuits* — <https://eprint.iacr.org/2026/933>
- GOAT Research, *Deferred Binding: Extending BABE for Dynamic Public Inputs in GOAT BitVM3* — <https://hackmd.io/@goatresearch/HkKp2g1Zfl>
- GOAT, *Partial-Binding Witness Encryption over Groth16* — the formal write-up of the primitive Deferred Binding rests on (`docs/partial_binding_we.tex`) — <https://github.com/GOATNetwork/bitvm2-gc/blob/feat/goat-bitvm3/docs/partial_binding_we.tex>
- `bitvm2-gc`, branch `feat/goat-bitvm3` (`verifiable-circuit-babe`, `babe-programs/soldering`) — <https://github.com/GOATNetwork/bitvm2-gc/tree/feat/goat-bitvm3>
