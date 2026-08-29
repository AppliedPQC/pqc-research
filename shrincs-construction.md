# The construction of SHRINCS, a semi-stateful hash-based signature scheme

> SHRINCS (*Shrunken SPHINCS*) is a cryptographic BIP draft posted to the bitcoin-dev
> mailing list on 27 August 2026. It demotes SLH-DSA, the hash-based signature scheme
> standardised in FIPS 205, to a fallback path, and puts a smaller **stateful** signature
> on the primary path. This note takes the scheme apart along the axes FIPS 205 uses for
> SLH-DSA: what is reused verbatim, what only changes parameters, what is newly
> constructed, and what the design gives up. The specification is at
> <https://github.com/SHRINCS/shrincs-bip/blob/main/SHRINCS.md>.
>
> Status: first draft, marked "Do NOT use in production". Security proofs, test vectors
> and unit tests are all outstanding. A Rust implementation of the whole scheme, written
> to check the claims below, is at
> [`AppliedPQC/shrincs-rs`](https://github.com/AppliedPQC/shrincs-rs). The vocabulary is
> that of FIPS 205, and the section below fixes the terms before they are used.

## Notation and terminology

The note uses the vocabulary of FIPS 205 throughout, together with two constructions the
standard does not contain. The terms are given here in roughly the order they are needed.

| Term | Meaning |
|---|---|
| **One-time signature**, OTS | A scheme whose secret key is secure for a single message. A second signature under the same key reveals enough of the key that a third can be forged. WOTS+, WOTS-TW and WOTS+C are of this kind. |
| **Few-time signature**, FTS | A scheme that tolerates a small number of signatures under one key, with security degrading as that number grows. FORS is of this kind. |
| **Winternitz chain** | A sequence of iterated applications of a hash to a chain secret. Publishing the `d`-th iterate encodes the index `d`, and a verifier can advance the chain but not reverse it. |
| **Winternitz parameter** `w` | The chain length, 16 throughout SHRINCS, so each chain carries `lg w = 4` bits of the digest. |
| **Checksum chains** | Extra chains encoding the complement of the sum of the message indexes, so that raising any message index forces the checksum index down, which would require inverting a chain. WOTS-TW appends three. |
| **Constant-sum encoding** | The alternative WOTS+C uses. No checksum is appended; instead only index vectors summing to a fixed constant are admitted, and the signer searches a counter until the digest yields one. Raising one index then requires lowering another, and the number of chain steps becomes fixed. Also called target-sum Winternitz. |
| **Merkle tree** | A binary tree whose leaves commit to the values being authenticated and whose internal nodes are hashes of their two children, so that the root commits to every leaf. |
| **Authentication path** | The sibling of each node along the path from a leaf to the root. With the leaf it determines the root, and it is what a signature carries to prove membership. |
| **XMSS** | A Merkle tree whose leaves are OTS public keys, evaluated under a tweakable hash at every node. Each signature consumes one leaf, which makes the scheme stateful. |
| **Hypertree** | A tree of XMSS trees, each layer signing the root of the layer below with one of its OTS keys. This reaches a large signature budget without any single tree of impractical height. |
| **FORS** | A few-time signature of `k` Merkle trees of height `a`. The digest selects one leaf per tree, and the signature reveals those leaves with their authentication paths. |
| **Tweakable hash** | A hash keyed by a public seed and by a position, so that the function applied at one position is independent of the function applied at any other. |
| **ADRS** | The address that encodes such a position. Giving every call a distinct ADRS is what makes those hashes independent. |
| **Domain separation** | Arranging that inputs playing different roles cannot collide, by making the function itself differ with the role. |
| **Multi-target attack** | A search for a preimage of any one of many published hashes at once. The advantage grows with the number of targets, which is why positions are separated and tree edges directed. |
| **Stateful scheme** | One whose signer must record which leaves have been used, since reuse breaks security. |
| **Stateless scheme** | One whose signer keeps no persistent record, at the cost of a larger signature. |
| **Semi-stateful scheme** | The arrangement SHRINCS introduces: one key pair carrying both, with the stateless component available whenever the state is not. |
| **Signature budget** | The number of signatures a key pair can produce before its leaves are exhausted. |

## 1. Relationship to SLH-DSA

| Layer | Content |
|---|---|
| **Reused verbatim** | The stateless path is SLH-DSA. `slh_dsa_sign` and `slh_dsa_verify` correspond line for line to FIPS 205 Algorithms 22 and 24, differing only in that the signature randomiser is supplied by the caller rather than generated internally. An SLH-DSA implementation that accepts custom parameter sets needs only a thin wrapper |
| **Reparameterised** | The stateless path uses a **non-standard parameter set**: the signature budget drops from 2⁶⁴ to 2⁴⁰, and the signature from 7,856 to 5,776 bytes |
| **Newly constructed** | The stateful path: FXMSS, a Merkle tree of variable shape, over WOTS+C, a constant-sum one-time signature. Neither derives from SPHINCS+ |
| **Shared machinery** | SHA-256 as the sole underlying hash; the ADRS tweakable-hash mechanism; all internal outputs truncated to 16 bytes |

SLH-DSA was designed to be stateless, because SP 800-208 judged XMSS and LMS "not
suitable for general use". SHRINCS moves in the opposite direction: it reintroduces state
in exchange for size, then uses a stateless fallback to reduce the consequence of losing
that state from loss of funds to a larger signature.

## 2. Architecture

| Component | Structure | Path |
|---|---|---|
| **FXMSS**, a Merkle tree signature of variable shape | Hash chains over an **arbitrarily shaped** Merkle tree | Stateful (primary) |
| **WOTS+C**, the one-time signature FXMSS uses | Hash chains with **constant-sum encoding** | Stateful (primary) |
| **SLH-DSA**, the stateless fallback | Hypertree over FORS | Stateless (fallback) |
| WOTS-TW, the one-time signature inside SLH-DSA | Hash chains with a checksum | Stateless (fallback) |
| FORS, a few-time signature | Many small Merkle trees | Stateless (fallback) |

A signature from either path verifies. Signers use the stateful path routinely and fall
back to the stateless path when state is lost, corrupted, or uncertain, as after a
restore from a static backup.

### Signing flow

[![How a SHRINCS signature is composed: the stateful path and the stateless fallback, both reducing to the 48-byte public key](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/architecture.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/architecture.png)

Each path reduces to a root, and the two roots together form the public key. The two are
not independent of one another; see the cross-binding in section 6.

### Distinguishing the two paths

The first byte of a signature is the discriminator:

- `= 255` (`FXMSS_HEIGHT`) selects the stateless path;
- `0 ≤ x < 255` selects the stateful path, and the byte is the `leaf_height` of the leaf
  that signed.

255 is available as a tag because `leaf_height = 255` corresponds to depth 0, and trees of
depth 0 are forbidden from signing. Length separates the two as well: the stateful maximum
of 4,619 bytes is below the fixed stateless 5,777, giving two independent discriminators.

## 3. Keys

Key generation takes a **48-byte seed** from the caller, split into three 16-byte parts:

```
seed(48) = sk_seed(16) ‖ sk_prf(16) ‖ pk_seed(16)
```

| | Composition | Length |
|---|---|---|
| **Public key** | `pk_seed ‖ sl_root ‖ sf_root` | **48 B** |
| **Secret key** | `sk_seed ‖ sk_prf ‖ pk_seed ‖ sl_root ‖ sf_structure ‖ sf_root` | **82 B** |

| Field | Role |
|---|---|
| `pk_seed` | Public parameter of every tweakable hash |
| `sk_seed` | Derives WOTS-TW, WOTS+C and FORS secret keys |
| `sk_prf` | Derives the message randomiser R |
| `sl_root` | The stateless component: the root of the top XMSS tree of the SLH-DSA hypertree |
| `sf_root` | The stateful component: the FXMSS root, at height 255 |
| `sf_structure` | **2 bytes**, `(shape, depth)`, describing the FXMSS tree; carried only in the secret key |

Both encodings are deliberate:

- truncating the **last 16 bytes** of the public key (`sf_root`) leaves a standard SLH-DSA
  public key for the stateless component under its parameter set;
- truncating the **last 18 bytes** of the secret key (`sf_structure ‖ sf_root`) leaves a
  valid SLH-DSA secret key.

A SHRINCS key is thus a suffix extension of an SLH-DSA key.

## 4. Parameter sets

### Stateful parameters

| Parameter | Value | Meaning |
|---|---|---|
| `WOTS_C_CHAIN_BITS` | 4 | Bits encoded per Winternitz chain (w = 16) |
| `WOTS_C_CHAIN_COUNT` | **32** | Number of Winternitz chains, **with no checksum chains** |
| `FXMSS_HEIGHT` | 255 | The imaginary height of the FXMSS tree, and so the maximum leaf depth |

Derived: `WOTS_C_CONSTANT_SUM = ⌈32 × 15 / 2⌉ = 240`, the target sum of the chain indexes,
which is also their expected value.

### Stateless parameters, against FIPS 205

| Parameter | FIPS 205 name | SHRINCS | SLH-DSA-SHA2-**128s** |
|---|---|---|---|
| Security parameter (bytes) | `n` | 16 | 16 |
| Winternitz | `lg_w` | 4 (w=16) | 4 (w=16) |
| Hypertree layers | `d` | **5** | 7 |
| XMSS height per layer | `h'` | 9 | 9 |
| Total hypertree height | `h` | **45** | 63 |
| FORS tree height | `a` | **13** | 12 |
| FORS tree count | `k` | **10** | 14 |
| Message digest length | `m` | **24** | 30 |
| **Signature budget** | | **2⁴⁰** | 2⁶⁴ |
| **Signature length** | | **5,776 B** | 7,856 B |

WOTS-TW chain counts: `len1 = ⌈128/4⌉ = 32` and `len2 = ⌈bitlen(32×15)/4⌉ = ⌈9/4⌉ = 3`,
for **35** chains, three more than WOTS+C.

The 2,080-byte reduction decomposes exactly:

| Component | SLH-DSA-128s | SHRINCS | Difference |
|---|---|---|---|
| Randomiser R | 16 | 16 | 0 |
| FORS signature, `16·k·(a+1)` | 16×14×13 = 2,912 | 16×10×14 = **2,240** | −672 |
| Hypertree signature, `d × (35×16 + 16×h')` | 7×704 = 4,928 | 5×704 = **3,520** | −1,408 |
| **Total** | **7,856** | **5,776** | **−2,080** |

Two thirds of the saving comes from cutting the budget, which shortens the hypertree, and
one third from reshaping FORS into fewer and taller trees.

The draft notes that SPHINCS+C or PORS+FP would give a further 15 percent at the same 2⁴⁰
budget, and declines both in order to keep FIPS 205 implementation reuse and the existing
security analysis.

## 5. ADRS

As in SLH-DSA, every hash call must be made unique so that an input in one position
cannot be replayed in another to yield the same output. The SHRINCS ADRS is **22 bytes**,
with one interpretation per path:

| Stateless ADRS | Size | | Stateful ADRS | Size |
|---|---|---|---|---|
| `layer` | 1 B | | `node_height` | 1 B |
| `tree_address` | 8 B | | `node_index` | 8 B |
| `type` | 1 B | | `type` | 1 B |
| `payload` | 12 B | | `payload` | 12 B |

The `type` values separate the paths completely, with no numeric overlap:

| Stateless | Value | | Stateful | Value |
|---|---|---|---|---|
| `SL_WOTS_TW_HASH` | 0 | | `SF_WOTS_C_HASH` | 16 |
| `SL_WOTS_TW_PK` | 1 | | `SF_WOTS_C_PK` | 17 |
| `SL_XMSS_TREE` | 2 | | `SF_FXMSS_TREE` | 18 |
| `SL_FORS_TREE` | 3 | | `SF_WOTS_C_PRF` | 21 |
| `SL_FORS_ROOTS` | 4 | | `SF_WOTS_C_GRIND` | 22 |
| `SL_WOTS_TW_PRF` | 5 | | | |
| `SL_FORS_PRF` | 6 | | | |

The first two payload bytes under `SF_WOTS_C_PRF` hold `sf_structure`. That is where the
tree shape enters secret-key derivation; see section 8.

## 6. Hash functions

All are built from SHA-256, in a common form:

```
sha256(pk_seed ‖ zeros(48) ‖ ADRS ‖ M)[:16]
```

The `zeros(48)` padding brings `pk_seed` (16 bytes) to 64 bytes, exactly one SHA-256
block, so the compression of that block can be precomputed once and reused.

### Tweakable hashes

| Function | Use | Path |
|---|---|---|
| `F` | One step of a hash chain | Both |
| `H` | Combines two Merkle children | Both |
| `T_sl` | Compresses 35 WOTS-TW chain ends into a public key | Stateless |
| `T_sf` | Compresses 32 WOTS+C chain ends into a public key | Stateful |
| `T_k` | Compresses k FORS roots | Stateless |
| `H_grind` | WOTS+C constant-sum grinding | Stateful |

`H_grind` is the one exception, consuming only `ADRS[:10]`:

```
sha256(pk_seed ‖ zeros(48) ‖ ADRS[:10] ‖ digest ‖ zeros(4) ‖ counter)[:16]
```

This keeps the whole input within a single compression, which matters because grinding
runs up to 2¹⁶ times. One consequence is that the signer holds `sf_structure` in
`ADRS[10:12]` at this point while the verifier leaves those bytes zero; since they fall
outside the consumed window, both sides compute the same value.

### Pseudorandom functions

| Function | Definition | Role |
|---|---|---|
| `PRF(pk_seed, sk_seed, ADRS)` | `sha256(pk_seed ‖ zeros(48) ‖ ADRS ‖ sk_seed)[:16]` | Derives WOTS and FORS secret keys. **Consumes the full 22-byte ADRS** |
| `PRF_msg_sl` / `PRF_msg_sf` | HMAC-SHA256 | Derives the message randomiser R |

### Message digests and the cross-binding

The two message digest functions are structurally symmetric: **each takes its own root as
a dedicated parameter and receives the other path's root inside `M`**. The two calls, as
the draft's reference implementation makes them:

```py
# stateless: what is actually signed is sf_root ‖ message
#   M = 0x00 ‖ len(ctx) ‖ ctx ‖ sf_root ‖ message
H_msg_sl(R, pk_seed, sl_root, M)

# stateful:
#   M = 0x00 ‖ len(ctx) ‖ ctx ‖ sl_root ‖ message
H_msg_sf(R, pk_seed, sf_root, ADRS, M)
```

[![Byte layout of both message digests, showing each taking its own root as a parameter and the other path's root inside M](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/cross-binding.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/cross-binding.png)

The two components are therefore interlocked. Neither root can be lifted out of the
public key and used as a key in its own right, and a signature under one path cannot be
carried over to a different public key.

## 7. WOTS+C: constant-sum encoding in place of a checksum

### Winternitz signatures, briefly

Both of the one-time signatures here are Winternitz, so it is worth having the base
construction in view before the variation.

A **chain** is a hash applied repeatedly to a secret: `s`, `H(s)`, `H²(s)`, and so on.
Publishing the `d`-th link signs the digit `d`, because a verifier can walk the chain
forward to the end but cannot walk it back without inverting `H`. The end of the chain,
`H^(w−1)(s)`, is the public value. With `w` the Winternitz parameter, one chain carries
`lg w` bits, and a digest is signed by cutting it into digits and giving each its own
chain.

That much is forgeable on its own. An adversary holding `Hᴰ(s)` can hash further to get
`Hᴰ′(s)` for any `D′ > D`, which is a valid signature on every larger digit. The fix is a
second kind of chain carrying the complement of the digit sum, so that raising a message
digit lowers a checksum digit, and forging upwards means walking a checksum chain
*backwards*.

[![WOTS+ at w = 8: key generation, signing the digit 2, verification, and why the checksum chain is needed](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/wots-plus.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/wots-plus.png)

That is WOTS+, and WOTS-TW is the same construction with the chaining function replaced
by a tweakable hash, which is what makes each position in each chain a distinct function
and what gives SPHINCS+ its tight security proof. Everything above holds for it, with
three checksum chains over 32 message chains at `w = 16`.

WOTS+C takes a different route to the same end. It adds no checksum, and instead admits
only digests whose chain indexes already sum to a constant.

```
requirement: Σ indexes == WOTS_C_CONSTANT_SUM = 240
```

On a single chain, the signature value σᵢ is the intermediate result of walking dᵢ steps
along the chain from the secret key. A verifier can only continue forwards:


With 32 chains of 4 bits each, over 0–15, the expected index sum is exactly 32×15/2 = 240.
The signer **grinds** a 16-bit counter until the sum lands on it:

```rust
pub fn wots_c_grind<S: HashSuite>(
    pk_seed: &[u8],
    digest: &[u8],
    adrs: &mut Adrs,
) -> Option<(u16, Vec<u32>)> {
    adrs.set_type(SF_WOTS_C_GRIND);
    for i in 0..=u16::MAX {
        let hashed = S::h_grind(pk_seed, adrs, digest, i);
        let idx = base_2b(&hashed, WOTS_C_CHAIN_BITS, WOTS_C_CHAIN_COUNT);
        if idx.iter().map(|&x| x as usize).sum::<usize>() == WOTS_C_CONSTANT_SUM {
            return Some((i, idx));
        }
    }
    None
}
```

The counter is returned alongside the indexes because the signature carries it: a
verifier cannot search for it, and recomputes once from the value it is given.

```rust
pub fn wots_c_map_digest<S: HashSuite>(
    pk_seed: &[u8],
    digest: &[u8],
    adrs: &mut Adrs,
    counter: u16,
) -> Option<Vec<u32>> {
    adrs.set_type(SF_WOTS_C_GRIND);
    let idx = base_2b(
        &S::h_grind(pk_seed, adrs, digest, counter),
        WOTS_C_CHAIN_BITS,
        WOTS_C_CHAIN_COUNT,
    );
    (idx.iter().map(|&x| x as usize).sum::<usize>() == WOTS_C_CONSTANT_SUM).then_some(idx)
}
```

Both are from [`AppliedPQC/shrincs-rs`](https://github.com/AppliedPQC/shrincs-rs)
unchanged. The `None` on the last line of `wots_c_grind` is the case the draft puts
below 2⁻¹⁴⁵⁰; the `None` from `wots_c_map_digest` is a verifier rejecting a counter
whose indexes miss the sum, which is the check that makes the encoding binding.

[![WOTS+C in three parts: key generation without checksum chains, grinding a counter until the 32 digits sum to 240, and a verifier whose chain work is the same 240 steps for every message](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/wots-c.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/wots-c.png)

Grinding fails with probability below 2⁻¹⁴⁵⁰, which is the target the draft's parameters
were chosen against. The verifier recomputes once from the counter in the signature and
rejects if the sum is not 240.

Two consequences follow:

1. **Three chains fewer.** 32 against 35 for WOTS-TW, saving 48 bytes per signature, or
   roughly 8 percent.
2. **Worst-case verification cost equals the average.** Because the index sum is fixed,
   so is the total number of chain steps a verifier walks: Σ(15 − dᵢ) = 32×15 − 240 = 240,
   the same count the signer performs. A consensus verifier can therefore price the worst
   case exactly.

The second claim is the one worth checking rather than taking on faith, and it is
checked. `verify_parameter_consistency` in the Rust implementation re-derives it from
the defining equation rather than restating it, and the known-answer tests exercise it
on every signature they cover. Running the specification's own `H_grind` and `base_2b`
over 300 messages gives a mean of 64.4 counters ground, and a chain-step count on both
sides that is the single value 240 every time.

> The same idea, Target-Sum Winternitz, appears in Ethereum's LeanSig, under a different
> motivation. LeanSig needs it because an aggregation circuit is sized for the worst case;
> SHRINCS needs it because consensus must price worst-case verification. The two share an
> author in Mikhail Kudinov.

## 8. FXMSS: an XMSS of signer-chosen shape

### XMSS, briefly

A one-time key signs once, so a scheme that signs more than once needs many of them and
a way to publish them all as one value. XMSS is the standard answer: put the hashes of
the one-time public keys at the leaves of a Merkle tree and publish the root.

A signature then carries three things. The one-time signature itself; the **index** of
the leaf that made it; and the **authentication path**, one sibling per level from that
leaf to the root. A verifier recovers the leaf from the one-time signature and climbs,
combining with each sibling in turn, and the index tells it which side to combine on.
Reaching the published root is what verification means.

[![XMSS at height 3, signing with leaf 2: the tree, the authentication path, what the signature carries, and the verifier's climb](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/xmss.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/xmss.png)

Each leaf may be used once, which is what makes XMSS stateful: the signer has to record
which leaves are spent. That is the constraint FXMSS inherits and the state counter of
section 9 manages, and it is why the scheme carries a stateless fallback at all.

### Differences from XMSS

| | XMSS, inside the stateless path | **FXMSS**, the stateful path |
|---|---|---|
| Tree | Always balanced, at fixed height `h' = 9` | **Any shape**; leaves may sit at different depths |
| Leaf OTS | WOTS-TW, 35 chains | **WOTS+C, 32 chains** |
| What is signed | Internal to SLH-DSA: **trusted messages the signer generated** | **A scheme in its own right, signing untrusted messages directly** |

The last row is a security requirement the specification emphasises, and the reason the
two have different interfaces.

### The imaginary height of 255

The root is fixed at height 255, so `depth = 255 − height`. A leaf sits at height 0 only
at the maximum depth of 255. The specification calls 255 imaginary because no real tree
can fill 2²⁵⁶ nodes.

Three properties follow:

1. A leaf `height` occupies **one byte**, is carried in the signature, and gives the
   verifier the depth directly.
2. The authentication path must be **exactly** `depth × 16` bytes, which cross-checks
   against the declared height.
3. 255 never occurs on the stateful path, which frees it as the tag for stateless
   signatures.

### Signature format and size

```
FXMSS signature = 2 (grind counter) + 512 (32 chains × 16 B) + 16 × depth
                = 514 + 16·depth

SHRINCS stateful signature = 1 (leaf_height) + 16 (R)
                           + ⌈min(depth,64)/8⌉ (leaf_index) + FXMSS signature
```

- depth = 1: 1+16+1+530 = **548 B**, the minimum
- depth = 255: 1+16+8+4,594 = **4,619 B**, the maximum

Each additional level adds 16 bytes, and the index field grows by one byte as depth
crosses 8, 16, and so on up to 64. That is the origin of the draft's "16 or 17 bytes per
signature" figure. Beyond depth 64 the index stays at 8 bytes and growth is a steady 16.

### The two prescribed shapes

Shape is described by the two `(shape, depth)` bytes in the secret key, and the leaf is
selected by the state counter.

**UXMSS, left-leaning and unbalanced, `shape = 0`:**

```
ctr <  depth → (index=1, height=255−1−ctr)   ⇒ leaf_depth = ctr+1
ctr == depth → (index=0, height=255−depth)   ⇒ leaf_depth = depth
```

The budget is **depth+1**, with depths 1, 2, …, d, d, the last two equal. Early signatures
are very small and later ones grow linearly. The spine descends on the left (`index = 0`)
and hangs one leaf to the right (`index = 1`) at each level; at the bottom level both
children are leaves.

**BXMSS, balanced, `shape = 1`:**

```
ctr < 2^depth → (index=ctr, height=255−depth)
```

The budget is **2^depth** and every signature is the same length: all leaves sit on one
level, so the authentication path is always `depth` siblings.

[![UXMSS and BXMSS side by side, with the signature size at each leaf and a table of budget and cost](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/fxmss-shapes.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/fxmss-shapes.png)

Circles are internal nodes, boxes are WOTS+C leaves, and each leaf carries the signature
size it produces.


| Configuration | Signature size | Budget | Key generation (SHA-256 compressions) | Average signing |
|---|---|---|---|---|
| UXMSS d=255 | 548 → 4,619 B | 256 | 425,982 | 133,326 |
| BXMSS d=5 | 612 B | 32 | 309,054 | 16,522 |
| BXMSS d=8 | 660 B | 256 | 425,982 | 133,447 |
| BXMSS d=10 | 693 B | 1,024 | 826,878 | 534,341 |
| BXMSS d=20 | 854 B | 2²⁰ | 547,649,022 | 547,356,475 |

UXMSS at d=255 and BXMSS at d=8 cost the same to generate, both having 256 leaves. Shape
does not change the cost at a given budget; it changes the size curve. UXMSS front-loads
the small signatures, and BXMSS flattens size across all of them.

### Shape-agnostic verification

The verifier never sees `shape`. It reads the first signature byte, derives `depth`, and
climbs that many levels. The figure below sets out the whole scheme in the same three
parts as the ones above, and works the climb through on a UXMSS tree of depth 4 with the
state counter at 2:

[![FXMSS in three parts: the signer choosing a shape, the state counter selecting a leaf, and a verifier that climbs on depth alone](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/fxmss.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/fxmss.png)

`fxmss_pubkey_from_sig`, again from the draft's reference implementation, receives only
`(leaf_index, leaf_height, signature)`:

```py
assert len(xmss_auth) == leaf_depth * 16          # path length must match the depth
assert leaf_index < 2 ** min(64, leaf_depth)      # index must fit at that depth
node = wots_c_pubkey_from_sig(...)
for k in range(leaf_depth):                        # direction comes from the index bits
    if (leaf_index >> k) & 1: node = H(..., sibling + node)
    else:                     node = H(..., node + sibling)
```

No shape byte is referenced. Whether the tree is UXMSS, BXMSS, or some other shape, the
verifier has a single code path, which bounds what consensus code has to review and
optimise.

The claim that one code path covers every shape is likewise under test rather than
asserted. The crate's vectors cover 46 stateful signatures across four tree shapes,
UXMSS and BXMSS at two depths each, and every one of them verifies through a single
`fxmss_pubkey_from_sig` that is never given the shape byte. The sizes those vectors
carry are the ones in the two figures above, produced by the draft's reference
implementation rather than copied from it.

### Where the shape is bound

Shape is bound into secret-key derivation rather than into verification:

- `fxmss_sign` places `sf_structure` in `ADRS[10:12]`;
- `wots_c_sign` and `wots_c_pubkey_gen` use it when calling `PRF`, and `PRF` consumes the
  **full 22-byte ADRS**, so every chain secret depends on the shape;
- `ADRS[10:14]` is then zeroed, and chain iteration and `T_sf` both run after zeroing, so
  the verifier does not need the shape.

The consequence is that one seed under different `(shape, depth)` values yields entirely
different secrets, leaves and roots. This is domain separation: a wrong shape byte on key
import does not collide with the original tree's secrets, it yields an unusable key.

### Directed trees

The specification argues explicitly against directionless Merkle trees of the kind
Taproot uses. Without a left-right distinction, an adversary holding two nodes at the same
level can run a preimage search against both targets at once, doubling the multi-target
advantage, and the advantage grows with the number of nodes. Direction is therefore given
explicitly by `(leaf_index >> k) & 1`.

## 9. State management

The stateful path selects its leaf by a **state counter**, the number of signatures
issued. Reusing a counter signs two different messages under one WOTS+C key, and anyone
who observes both signatures can forge.

The draft's MUST-level rules:

- the counter must not be backed up or restored, must not be exported or imported, and
  must not be used concurrently;
- it must be persisted **before** the signature is returned;
- when state is uncertain the stateful path must be refused, falling back to the
  stateless one.

Two mitigations distinguish this from XMSS and LMS:

1. **Every key pair carries a stateless fallback by construction.** Losing state costs
   signature size, from 548 to 5,777 bytes, rather than funds.
2. **Leaf reuse is publicly detectable.** Every stateful signature exposes its
   `(index, height)`, so a repeated leaf under one public key can be spotted before the
   second transaction is broadcast.

## 10. Sizes and verification cost

| Item | Size |
|---|---|
| Public key | 48 B |
| Stateful signature | 548 – 4,619 B |
| Stateless signature | 5,777 B |
| **Public key plus signature, minimum** | **596 B** |

For comparison, SLH-DSA-SHA2-128s gives 32 + 7,856 = 7,888 B, a factor of **13.23**, and
ML-DSA-44 gives 1,312 + 2,420 = 3,732 B, a factor of **6.26**, though ML-DSA-44 is
Level 2 where SHRINCS is Level 1.

### Verification cost

| | Compressions | Per byte |
|---|---|---|
| Stateful | 255 – 509 | **0.465** |
| Stateless | 467 – 2,792 | **0.483** |
| BIP-340 Schnorr | ≈127 equivalent | **1.98** |

Per byte this is 4 to 16 times faster than Schnorr. The stateful parameters were tuned to
put the two paths' per-byte costs close together, so that one witness discount rule can
cover both. The draft argues from this for leaving room for a witness discount, without
defining one.

The cost falls elsewhere: key generation runs 3.1×10⁵ to 5.5×10⁸ compressions, and
stateless signing averages 1.7×10⁶.

## 11. Differences from SLH-DSA

| | SLH-DSA-SHA2-128s | SHRINCS |
|---|---|---|
| State | Stateless | **Semi-stateful**: stateful primary with stateless fallback |
| Public key | 32 B | 48 B, adding `sf_root` |
| Signature | 7,856 B | 548–4,619 B primary, 5,777 B fallback |
| One-time signature | WOTS-TW, 35 chains, checksum | **WOTS+C, 32 chains, constant sum**, plus WOTS-TW in the fallback |
| Tree structure | Fixed hypertree, `d=7, h'=9` | **Variable-shape FXMSS**, plus a `d=5, h'=9` hypertree in the fallback |
| Signature budget | 2⁶⁴ | 2⁴⁰ fallback; 32–2²⁰ primary, depending on shape |
| Verification cost | Depends on signer choices | **Worst case equals average** on the primary path |
| Security level | Level 1 | Level 1: 128-bit classical, 64-bit quantum |
| Underlying hash | SHA-2 or SHA-3 | **SHA-256 only**, reusing the hash Bitcoin consensus already has |
| Standardisation | **FIPS 205** | **Draft**, not submitted to the BIPs repository, no security proof |

## 12. Properties the scheme does not provide

The third drawback the draft lists for itself:

> SHRINCS lacks any algebraic structure allowing for **public key rerandomization**,
> **multisignature schemes (like MuSig)**, etc.

There is no counterpart to BIP-32 xpubs and no MuSig-style multisignature. This is common
to hash-based signatures, which have no algebraic structure to aggregate over. For
protocols whose custody rests on n-of-n MuSig2, such as the BitVM2 peg, a post-quantum
replacement is therefore not within the scope of changing an output type.

## 13. Open problems

The draft is a first draft, and says so. What is outstanding falls into three kinds, and
they are not equally hard.

**Missing artefacts, which are work rather than research.** There is no security proof;
the draft records one as TODO and thanks Andreas Hülsing for input on it. There are no
comprehensive test vectors, which the draft states are required to move the proposal from
Draft to Complete. There are no unit tests and no optimised implementation, the reference
being described by its own authors as naive, inefficient and non-constant-time. Each of
these is known, scoped, and merely undone.

**Deployment questions that belong to Bitcoin rather than to the scheme.** The draft
specifies cryptography decoupled from consensus, so at least one further BIP defining a
new output type is needed and is not written. The verification cost per byte argues for
a witness discount, and the draft makes that argument without defining one. The parameter
set is a multi-dimensional trade between size and speed which the working group expects
to be contested; the two components can be reparameterised independently, so the space is
larger than a single choice.

**What follows from the missing algebraic structure.** Section 12 states the absence
itself; what is open is what to do about it. A hybrid with BIP-340 works by simple
concatenation, and the draft declines to define a dedicated combiner, so a deployment
wanting strong unforgeability under partial compromise has that to design. Whether an
HD-wallet arrangement can be built for a scheme with no rerandomisable public key is a
question the draft raises and leaves open.

Alongside those, the state counter remains the sharp edge. The reference implementation
performs no state management at all and takes the counter from its caller, which is
honest about where the difficulty lives: the rules in section 9 are MUST-level and
unenforceable by the scheme itself. How a wallet presents a signature that grows as a key
is used, and how backup and recovery work when restoring state is forbidden, are user
experience questions the draft raises and does not answer.

## 14. Where this sits in the BIP process

The `Requires` field of BIP-361 reads "TBD Post Quantum Signature BIP". SHRINCS is the
first public candidate for that missing document, and it has not been submitted to the
BIPs repository, so BIP-361 itself remains the only post-quantum entry in the BIPs index.

## A reference implementation

The whole scheme is implemented in Rust at
[`AppliedPQC/shrincs-rs`](https://github.com/AppliedPQC/shrincs-rs), following the
draft's own structure. It is checked against two authorities rather than against
itself: the reference implementation the draft ships, which fixes the vectors and
which it also cross-verifies with in both directions, and NIST's ACVP vectors,
which the stateless component reproduces once instantiated at the standard SLH-DSA
parameter sets. The second matters because the draft's central reuse claim is that
this component is FIPS 205, and vectors from the draft cannot test that claim.

Writing it surfaced one behaviour this note had left implicit. Exhausting the
stateful budget is not an error: `shrincs_sign` finds no leaf and takes the same
branch as when no counter was supplied at all, so the signature comes back from the
stateless component at 5,777 bytes and verifies. Running out of leaves and losing
the state file degrade identically.

## Appendix: the working group

Mike Casey (OpenChain), Conduition (Brink), Ethan Heilman (Cloudflare, author of BIP-347
and BIP-360), Mikhail Kudinov (Blockstream), Oleksandr Kurbatov (Blockstream), Boris
Nagaev (independent), Jonas Nick (Blockstream), remix7531 (OpenSats).

The scheme originates in earlier proposals by Jonas Nick and Mikhail Kudinov. Kudinov is
also an author of Ethereum's LeanSig, which is why both lines adopt Target-Sum, or
constant-sum, Winternitz encoding.
