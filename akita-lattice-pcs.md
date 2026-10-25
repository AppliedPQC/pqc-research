# Akita: what a lattice commitment buys a post-quantum zkVM

> Akita is a lattice-based polynomial commitment scheme from LayerZero Labs, a16z
> crypto research, Carnegie Mellon and USC, released on 9 September 2026 together
> with Lattice Jolt, the version of the Jolt zkVM that can use it. Its claim is that a
> lattice commitment can now do what hash-based commitments cannot: keep proofs in
> the tens of kilobytes, commit to sparse data at the cost of its nonzero entries, and
> still verify in milliseconds, all from standard Module-SIS. This note sets out how
> it gets there, what its own measurements show against both lattice and hash-based
> schemes, what has changed in the code since the paper, and what it does not yet
> provide.
>
> Checked 2026-10-25 against: the paper as hosted by LayerZero
> ([`akita.pdf`](https://assets.layerzero.network/pdf/akita.pdf), dated 11 September
> 2026), the abstract of the current ePrint revision
> ([2026/1983](https://eprint.iacr.org/2026/1983), revised 15 October 2026), the
> [`LayerZero-Labs/akita`](https://github.com/LayerZero-Labs/akita) repository at
> `e423a39` (22 October), the [`a16z/jolt`](https://github.com/a16z/jolt) repository
> at `f3265b5` (24 October), and the Aerie paper. The detailed tables below come from
> the September paper; section 6 explains why its headline numbers have moved since.

## Terminology

| Term | Meaning |
|---|---|
| **PCS**, polynomial commitment scheme | Commits to a polynomial, then proves its value at a point chosen later. In most SNARKs it is the only cryptographic component. |
| **PIOP**, polynomial interactive oracle proof | The information-theoretic half of a SNARK. Jolt's sum-check machinery is one. Compiled with a PCS, it becomes an argument. |
| **Ajtai commitment** | A lattice commitment `t = A·m mod q` to a vector `m` of small integers. Binding rests on Module-SIS. Its cost is linear in the data, so zero entries cost nothing. |
| **Module-SIS** | The short-integer-solution problem over a module lattice: find a short nonzero `x` with `A·x = 0`. The forgery assumption behind ML-DSA. |
| **Fold** | One recursive step of the opening proof. The prover folds a large witness with verifier challenges into a smaller one and commits to it. Akita folds to completion, then sends a short final witness in the clear. |
| **Sum-check** | The interactive protocol that reduces a claim about a sum over the Boolean hypercube to one evaluation at a random point. |
| **One-hot polynomial** | A concatenation of standard basis vectors, one per execution step. Jolt's Twist and Shout memory checking commits to these, and almost every entry is zero. |
| **Core-SVP, ADPS16** | The cost model that prices a lattice attack by its BKZ block size `β`: `2^(0.292β)` classically, `2^(0.265β)` quantumly. NIST's lattice standards state their security levels in it. |
| **ROM, QROM** | Random oracle model, and its quantum version. A Fiat–Shamir argument is only proven post-quantum once its soundness is proved in the QROM. |

## 1. Why the commitment is the migration

Most deployed SNARKs are a PIOP composed with a PCS. The PIOP is
information-theoretic, so it has no quantum exposure of its own. The PCS is where
the cryptographic assumption lives. Akita's paper opens on exactly this point: swap
an elliptic-curve PCS for a post-quantum one and the whole argument becomes
post-quantum. Jolt makes the point unusually cleanly, because its PIOP is entirely
sum-check and its PCS was Dory, a pairing-based scheme.

The paper attaches one qualification in a footnote, and it matters for how the
result should be read. A rigorous post-quantum claim also needs Fiat–Shamir
soundness in the quantum random oracle model. Akita's knowledge soundness is proved
in the classical ROM, and a QROM proof is listed as future work. Its Module-SIS
parameters are, separately, priced against quantum lattice attacks. So the binding
assumption is post-quantum by construction, while the proof of the compiled
argument is not yet a quantum proof. This is a real gap, not one every scheme
shares: SNARGs built from interactive oracle proofs with round-by-round soundness,
which covers the hash-based schemes, have been proved secure in the QROM since
Chiesa, Manohar and Spooner (TCC 2019). It should not be dropped from summaries.

## 2. What hash-based commitments give up

The post-quantum PCS slot has so far been filled by hash-based schemes: FRI, STIR,
WHIR, BaseFold, Ligerito and the Binius line. They are transparent, their verifiers
are fast, and they rest on nothing but a hash function. Ethereum's leanVM and every
deployed STARK use them.

The Akita paper names two properties they lose relative to elliptic-curve
commitments.

- **Proof size.** With proven soundness, meaning no reliance on the conjectured
  proximity gaps, and at practical code rates of 1/2 or 1/4, the paper puts the
  smallest hash-based proofs, WHIR's, at no less than about 160 KB for non-trivial
  instances.
- **Pay-per-bit commitment.** A curve commitment `Σ mᵢGᵢ` costs only the nonzero
  entries of `m`. A Reed–Solomon-and-Merkle commitment encodes the whole vector
  into a codeword first, so a sparse vector costs as much as a dense one.

The second point is the one that motivates Akita. Jolt's Twist and Shout prover
commits to one-hot polynomials that are almost entirely zero. A hash-based PCS
would make Jolt pay for every zero. An Ajtai commitment, like Dory, would not.

## 3. The lattice trilemma

Lattice PCSs recover both properties in principle, and the paper counts a run of
recent constructions that have pushed them toward practice. Its argument is that
none was deployable, because each achieved at most two of three requirements:
small proofs, fast verification, and security from standard Module-SIS.

| PCS | Binding assumption | Proof size, as reported by each work | Verifier | Artifact |
|---|---|---|---|---|
| Greyhound | M-SIS | 46–53 KB, `N = 2^26`–`2^30` | `√N` | End-to-end C, AVX-512 only |
| Hachi | M-SIS | 55 KB, `N = 2^30` (estimate) | `√N` | Partial Rust, single fold |
| Jindo | M-SIS | 315–1,207 KB, `N = 2^14`–`2^20` | `N^(1/3)` | End-to-end Go |
| RoKoKo | vanishing SIS | 115 KB, `N = 2^26`–`2^30` | polylog | End-to-end Rust |
| Grand Danois | vanishing SIS | 80–90 KB, `N = 2^32` (estimate) | polylog | None |
| Akita | M-SIS | 61–65 KB, `N = 2^22`–`2^30` | `N^(1/K)`, any fixed `K ≥ 2` | Rust |

*Source: the paper's Table 1. The authors note that these sizes are each work's own
benchmark or estimate, not a parameter-matched comparison.*

Greyhound and Hachi have small proofs from standard Module-SIS, but a square-root
verifier. RoKoKo and Grand Danois reach polylogarithmic verification by giving the
public matrices extra algebraic structure, under vanishing SIS, which has seen far
less cryptanalysis. The constructions that reach polylogarithmic verification
from standard assumptions either give no concrete parameters or report proofs in
the megabytes, 5.17 MB and 767 MB at degree `2^30` for the two that do. Akita's place in
the table is the bottom row: Greyhound-sized proofs, a verifier below square root,
and plain Module-SIS.

## 4. How Akita gets there

Akita is Hachi with its verifier bottleneck removed and every fold re-engineered.
Hachi itself came from Greyhound, which came from LaBRADOR.

### Setup offloading, the main idea

Hachi splits an `N`-coefficient witness into `B` blocks of length `M ≈ N/B`. In
each fold the verifier must do two kinds of work: evaluate `B` folding challenges,
and scan the public commitment matrices, whose active width grows with `M + B`. The
total is about `B + N/B`, minimised at `B ≈ √N`. Fewer blocks mean fewer
challenges but a longer matrix scan, so the verifier is stuck at square root.

Akita moves the matrix scan into the proof. The setup is public and fixed, so every
setup prefix a fold might need is committed during preprocessing. At opening time
the prover claims the setup's contribution, an inner product over one flat setup
vector, and a degree-two sum-check reduces that claim to a single evaluation of the
setup polynomial. That evaluation is not checked directly, since that would restore
the scan. It is carried into the next fold as a second opening, beside the recursive
witness, and checked there against its preprocessing commitment.

With the scan gone, a fold can use far fewer blocks. Choosing `B ≈ N^(1/K)` over the
first `K − 1` folds gives a verifier that runs in about `N^(1/K)` time for any fixed
`K ≥ 2`, while proof size stays logarithmic and prover time quasi-linear. The cost
is paid in the next fold, which must certify the setup opening, and in 3 to 6 KB of
extra proof. In practice Akita offloads only one or two levels.

### The other changes

The rest of the paper is a fold-by-fold optimisation, each piece aimed either at
verifier work near the root or at witness size near the tail.

- **A cheaper digit range check.** Hachi proves each digit lies in a balanced
  alphabet with a degree-`b` vanishing polynomial. Akita uses the symmetry of that
  alphabet to halve the degree, then a GKR-style product tree. Prover work drops
  from `O(b²L)` to `O(bL)`.
- **Commitment compression.** Each fold sends two commitments, normally 1 to 8 KB.
  Akita repeatedly digit-decomposes and re-commits them down to 128 bytes, folding
  the binary-digit obligation into the range check it already runs.
- **Exact Euclidean norm checks.** Sum-checks certify the response's exact total
  squared norm rather than an approximation, which can lower the module rank of the
  tail witnesses by one or two.
- **Two ways to check ring relations.** Early folds keep Hachi's quotient-lifting
  method. The tail uses a quotient-free check that avoids carrying quotient
  polynomials into the next witness.
- **A repaired base-field shortcut.** Akita uses Diamond and Posen's tensor
  reduction to connect base-field evaluations to packed ring claims, which closes a
  binding gap in Hachi's published shortcut. Section 7 returns to this.

### What a zkVM needs beyond the protocol

- **Batching across time.** Jolt commits its trace, its advice and its program
  data at different stages, some offline, and opens them in one proof. Akita fixes
  the digit alphabet before any commitment is formed so that they stay batchable.
- **Distributed proving.** Forming the folded response normally needs an
  all-reduce across machines. Response chunking lets each machine fold its own
  blocks, at the cost of a larger witness, and only until communication is cheap.
- **An offline planner and a separate validator.** A schedule fixes dozens of
  interacting parameters across every fold. The planner searches complete
  schedules, and an independent validator rebuilds every fold transition and
  re-checks the security conditions. A proof carries only the 32-byte digest of a
  schedule row from a trusted catalog. It cannot supply its own parameters.

## 5. The numbers

All figures in this section are the authors' measurements from the paper dated 11
September 2026.

### Against lattice PCSs

These are single-thread measurements on an AMD Ryzen 9 9950X. Akita and Greyhound
use the same 32-bit field. The authors recalibrated Greyhound to a 128-bit quantum
target and fixed its Johnson–Lindenstrauss check before measuring it. RoKoKo runs
its own native instances over a roughly 50-bit field.

| Bits committed | Scheme | Commit (s) | Open (s) | Verify (ms) | Proof (KB) | Peak RAM (GiB) |
|---|---|---|---|---|---|---|
| `2^31` | Akita, direct | 1.29 | 2.64 | 12.7 | 63.1 | 0.69 |
| | Akita, offloaded | 1.29 | 3.28 | 12.1 | 66.3 | 0.72 |
| | Greyhound | 2.20 | 1.07 | 305 | 64.6 | 4.66 |
| | RoKoKo | 0.863 | 0.712 | 4.9 | 114.9 | 4.07 |
| `2^35` | Akita, direct | 24.6 | 16.1 | 33.3 | 64.6 | 4.69 |
| | Akita, offloaded | 24.5 | 18.6 | 16.2 | 67.3 | 4.74 |
| | Greyhound | 59.4 | 21.3 | 1,525 | 66.1 | 104.8 |
| | RoKoKo | 17.9 | 4.38 | 7.4 | 115.1 | 35.6 |

*Excerpt from the paper's Table 9. `2^35` bits is `2^30` coefficients of a 32-bit field.*

Three things stand out. Against Greyhound, verification is 17 to 94 times faster at
the same proof size, which is the paper's central claim. At the largest input,
offloading halves verification, from 33.3 to 16.2 ms, for 2.7 KB more proof and 2.5
seconds more opening. Peak memory is the clearest win: 4.7 GiB against 105 GiB for
Greyhound and 36 GiB for RoKoKo. The losses are stated too. RoKoKo opens faster and
verifies faster, with proofs nearly twice the size, and the conclusion concedes that
Akita's prover "is not yet on par with Greyhound and RoKoKo".

### Against hash-based PCSs

This is the comparison most readers will want, and it has had almost no coverage.
The measurements use eight prover threads and single-thread verification, at `2^31`
bits committed, the largest size every implementation completed.

| Scheme | Commit (s) | Open (s) | Verify (ms) | Proof (KB) | Peak RAM (GiB) |
|---|---|---|---|---|---|
| Akita, 32-bit field | 0.185 | 0.538 | 12.7 | 63.1 | 0.70 |
| Akita, 64-bit field | 0.126 | 0.422 | 10.9 | 66.7 | 0.50 |
| Akita, 128-bit field | 0.346 | 0.376 | 10.5 | 68.3 | 0.83 |
| WHIR (Plonky3) | 0.476 | 6.25 | 2.1 | 113.6 | 2.80 |
| STIR (Plonky3), batched | 0.674 | 1.27 | 3.3 | 164.9 | 3.16 |
| FRI (Plonky3), batched | 0.672 | 1.04 | 8.0 | 431.7 | 3.94 |
| BaseFold (Binius64) | 0.215 | 0.103 | 0.8 | 489.2 | 1.42 |
| Ligerito (Flock) | 0.124 | 0.300 | 1.1 | 494.6 | 1.37 |
| WHIR (ProveKit) | 1.88 | 4.11 | 2.8 | 563.2 | 4.57 |
| BaseFold (SP1) | 2.49 | 2.30 | 20.3 | 933.0 | 2.63 |
| FRI (Plonky2) | 38.4 | 29.6 | 2.5 | 117.6 | 44.6 |

*Excerpt from the paper's Table 10.*

The picture is a trade, not a sweep, and the paper says so. Akita's proofs are 63 to
68 KB, against 114 to 933 KB for the hash-based implementations. Every hash-based
verifier except SP1's is faster, by factors from about 1.3 for Plonky3 FRI to about
13 for Binius64 BaseFold. Binius64 BaseFold
and Flock's Ligerito open faster as well.

Two caveats from the paper apply. First, the implementations do not share a
security target or a proximity analysis, so the table compares measured software,
not equal-security instances. Second, equal committed bits is a coarse metric. A
32-bit field at `2^31` bits has four times the coefficients of a 128-bit field, and
the fields differ in arithmetic cost. The commitments in this table are dense,
moreover, so they do not show Akita's sparse-commitment advantage at all. That
advantage appears only in Jolt.

### In Jolt

The paper proves SHA-256 chains with Jolt on an Apple M4 Max, three runs per trace
length. Prover times include commitments, sum-checks and the joint opening, and
exclude preprocessing and setup.

| Padded trace | Backend | Prove (s) | Verify (ms) | Proof (KB) | Peak RAM (GiB) |
|---|---|---|---|---|---|
| `2^17` | Akita | 0.50 | 9.96 | 85.9 | 0.33 |
| | Dory | 1.08 | 69.38 | 81.9 | 0.32 |
| `2^23` | Akita | 11.22 | 15.57 | 94.0 | 2.93 |
| | Dory | 16.35 | 83.10 | 94.9 | 3.45 |
| `2^27` | Akita | 90.85 | 39.82 | 98.1 | 35.47 |
| | Dory | 175.18 | 88.29 | 95.9 | 34.76 |

*Excerpt from the paper's Table 11. Medians of three runs.*

Here Akita is not trading against anything. Over Dory it proves 1.3 to 2.2 times
faster and verifies 2.2 to 7.4 times faster, with complete Jolt proofs between 0.9%
smaller and 8.9% larger and all below 100 KB. Post-quantum security arrives as a
side effect of a faster system, which is unusual in this migration. The comparison
is with a pairing-based scheme, though. No hash-based Jolt was measured.

## 6. The numbers move

The headline figures differ across the paper's versions and the launch posts. The
detailed tables above come from the earliest version, so the drift is worth setting
out.

| Source | Date | Proof size | Verify vs Greyhound | Jolt prover vs Dory | Jolt verifier vs Dory |
|---|---|---|---|---|---|
| Paper hosted by LayerZero | 2026-09-11 | 61–70 KB | 10–94× | 1.3–2.2× | 2.2–7.4× |
| ePrint, archived copy | 2026-09-19 | 61–72 KB | 19.2–89.7× | 1.3–2.2× | 2.2–7.4× |
| ePrint, current revision | 2026-10-15 | 58–68 KB | 22.1–108.7× | 2.2–4.2× | 2.8–8.5× |
| LayerZero launch post | 2026-09-09 | 65–80 KB | — | 2–3× | — |
| a16z launch post | 2026-09-09 | under 100 KB | — | 2–3× | 2–3× |

The current revision's note says its last change left "statements, bounds and
measurements" unchanged, so the improved figures arrived in an intermediate
revision. This note could not retrieve the current PDF, so it does not reproduce its
tables.

A larger change postdates every version of the paper. On 10 October 2026 Akita
raised its lattice security target. The paper prices every Module-SIS instance at
128 bits under quantum Core-SVP, which sits between ML-DSA-44 and ML-DSA-65. The new
policy, [`sis-nist-level3-policy`](https://github.com/LayerZero-Labs/akita/blob/main/specs/sis-nist-level3-policy.md),
requires at least 169 quantum bits, the ML-DSA-65 forgery figure for NIST category
3. Every instance now needs a BKZ block size of at least 638. Jolt adopted it on 21
October ([#2107](https://github.com/a16z/jolt/pull/2107)).

It has a price. The compression ladder no longer fits at 169 bits, so compressed
commitments double from 128 to 256 bytes. Across the 115 catalogued schedules,
proofs grow by a median of 14.9%, about 10 KB. Four small recursive schedules now
produce proofs of 190 to 320 KB. Prove and verify times barely move.

The policy also states what it does not raise. Fiat–Shamir grinding, the challenge
fields and the challenge entropy stay at 128 bits. The spec says a full category 3
claim would need 256-bit challenge sets and 48-byte transcript digests, which is a
protocol change rather than a parameter change. The lattice is now at category 3
while the argument around it stays at 128 bits.

## 7. Akita's audit of its own field

Appendix F of the paper is an audit of the schemes Akita compares against, and it is
worth reading on its own. The authors state each finding narrowly, and none is
presented as an end-to-end attack.

- **Hachi.** The base-field shortcut in the public ePrint does not bind the
  individual partial evaluations. For extension degree 4, the setting Hachi and Akita
  both use, a prover can alter the claimed value and still pass both displayed
  checks. The Hachi authors told the Akita authors that later, not yet public,
  versions fix it. Hachi's generic transformation is unaffected.
- **Grand Danois.** Its first version sampled a compression challenge before the
  commitment that bound the witness, so a prover could pick a residual in its kernel.
  The current version removes that step and credits Quang Dao, Akita's lead author.
- **RoKoKo.** When recomposing base-`2^k` digits, the released implementation used
  `k^(ℓ−1)` as the norm factor where the correct operator norm is about
  `2^(k(ℓ−1))`: 216 instead of about 262,176 for `(k, ℓ) = (6, 4)`. Corrected, its root commitment drops
  to an estimated 77 bits against a stated 128, and four of five projected rounds
  fail the integer-lifting condition. The upstream issue,
  [`rokoko#94`](https://github.com/lattice-arguments/rokoko/issues/94), was closed
  as completed on 1 September 2026.
- **Greyhound and LaBRADOR.** Their implementations use a truncated-output Ring-SIS
  map, which the Akita authors treat as an assumption distinct from standard
  Module-SIS. The comparison tables carry that distinction rather than resolve it.
- **Orthus.** Its outer commitments in compressed mode fall below 128 bits under
  the Core-SVP convention used for the comparison.

This matches an earlier zkSecurity review that found
[soundness failures in LaBRADOR implementations](https://blog.zksecurity.xyz/posts/labrador-bugs/).
Lattice argument systems have many interacting norm bounds, and the gap between
paper and code is where they have failed. Akita's answer is the separate validator
and the policy-tagged schedule catalog. That is a better process, not an
independent check: Akita itself has not been externally audited.

## 8. What Akita does not yet provide

- **Zero knowledge.** Akita has none. Its book says sum-check messages, fold
  commitments and the final witness all leak information, and an earlier partial
  attempt was removed rather than shipped. In Jolt, the `akita` and `zk` features
  are mutually exclusive, so private Jolt proofs remain on Dory. The current
  release is a proof of correct computation, not a zero-knowledge proof.
- **A quantum soundness proof.** Knowledge soundness is proved in the classical
  ROM. The paper names the missing QROM analysis as the main open soundness issue.
- **Rigorous completeness for the tuned schedules.** The response model behind the
  optimised later folds is heuristic, by the paper's own account.
- **An audit.** The repository's `SECURITY.md` says Akita "is not yet a formally
  audited production release" and should be treated as research software. The
  LayerZero launch post calls it production-ready. The repository is the more
  careful source.
- **Default status in Jolt.** Dory is still Jolt's default backend. Akita is
  selected with the `akita` Cargo feature, and the generated SDK project does not
  yet support it. Lattice Jolt is therefore a supported configuration of Jolt, not
  what a new Jolt user gets.
- **Evidence for on-chain verification.** A 10 to 40 ms native verifier says
  nothing directly about EVM cost. The paper measures no Solidity or EVM verifier.

## 9. Lattice-native statements: Aerie

The paper's conclusion points at an application where a lattice PCS has a
structural advantage: proving statements that are themselves lattice relations.
Falcon and ML-DSA verification are norm bounds and ring equations, which is what
Akita's sum-checks already check.

[Aerie](https://github.com/a16z/aerie), released on 16 October 2026, does this for
Falcon. One proof covers a batch of Falcon-512 or Falcon-1024 signatures without
recursion, and its security argument reduces to Falcon, Module-SIS and Akita's
composition theorem. The authors report 32,768 Falcon-512 signatures aggregated in
about 0.65 seconds on an M4 Max, a proof of about 66 KiB, and about 29 times the
throughput of leanVM's XMSS aggregation on the same machine. Its README marks it as a
research release, with zero knowledge not claimed and ML-DSA support still to come.

This is the strongest form of the lattice argument. A hash-based system that
aggregates lattice signatures must arithmetise NTTs and norm checks inside a
hash-friendly circuit. A lattice PCS proves them close to natively.

## 10. Reading Akita against the hash-based route

The two routes now make different bets, and Akita's own tables show where each one
wins.

**Where Akita wins**

- **Proof size.** It holds 60 to 70 KB at sizes where proven-soundness hash proofs
  are 110 to 900 KB. That size goes on chain, over the network, or into a recursive
  verifier.
- **Sparse commitments.** It pays per nonzero entry, which is why it beats Dory
  inside Jolt and why a hash-based PCS would struggle there.
- **Prover memory.** It needs a few GiB where comparable schemes need tens.
- **Lattice-native statements.** Signature aggregation for Falcon now, and ML-DSA
  later.

**Where the hash-based route still wins**

- **Verification speed.** At `2^31` bits, most hash verifiers are 3 to 13 times
  faster.
- **Assumption and maturity.** A hash function is the most conservative assumption
  available, and hash-based PCSs have years of deployment and attack history.
  Lattice PCS implementations have a recent record of parameter bugs, Akita's
  comparison set included.
- **Ecosystem.** Ethereum's consensus plan, EIP-8288 and leanVM are built on
  recursive hash-based STARKs. Akita has no comparable integration path there yet.

Akita does not settle the question. It reopens it. Until September 2026 there was
no lattice PCS that a production zkVM could adopt on the paper's three criteria.
There is now one, measured against the hash-based schemes and published with code.
What remains is the work that turns a measured system into a trusted one: a QROM
proof, zero knowledge, an external audit, and a verifier cost model for the places
where proofs are actually checked.

## What could not be verified

- The tables of the current ePrint revision. The PDF is behind a Cloudflare
  challenge. Its abstract figures appear in section 6, and the detailed tables in
  this note are from the 11 September version.
- Akita's measurements under the category 3 policy beyond the spec's summary. The
  evidence directory is cited as the spec reports it, and the benchmarks were not
  rerun.
- Any benchmark in this note. All figures are the authors' own, on their hardware.
- Whether the Hachi base-field fix described in Appendix F has been published.

## References

Akita and its integration. Checked 2026-10-25.

- Q. Dao, O. Bodaghi, A. Khajehpour, G. Vitto, M. Badakhshan, M. Georghiades, F. Liu, J. Zhang, J. Thaler, *Akita: A High-Performance Lattice-Based Polynomial Commitment Scheme* — ePrint <https://eprint.iacr.org/2026/1983>; LayerZero-hosted PDF <https://assets.layerzero.network/pdf/akita.pdf>
- `LayerZero-Labs/akita`, the Rust implementation, at `e423a39` — <https://github.com/LayerZero-Labs/akita>
- Akita spec, *NIST Category 3 Module-SIS Policy* — <https://github.com/LayerZero-Labs/akita/blob/main/specs/sis-nist-level3-policy.md>
- Akita book, *Zero knowledge* roadmap — <https://github.com/LayerZero-Labs/akita/blob/main/book/src/roadmap/zero-knowledge.md>
- Akita `SECURITY.md` — <https://github.com/LayerZero-Labs/akita/blob/main/SECURITY.md>
- `LayerZero-Labs/pcs-benchmarks`, the harness behind the comparison tables — <https://github.com/LayerZero-Labs/pcs-benchmarks>
- `a16z/jolt`, at `f3265b5`; Jolt book chapter *Akita* (`book/src/how/akita.md`) — <https://github.com/a16z/jolt>
- Jolt PR #2107, *NIST category 3 Module-SIS and 2^31 schedule coverage* — <https://github.com/a16z/jolt/pull/2107>
- LayerZero, *Introducing Akita* (9 September 2026) — <https://layerzero.network/blog/introducing-akita>
- M. Georghiades, J. Thaler, A. Tretyakov, M. Zhu, *Entering the era of lattice SNARKs – with a faster, post-quantum Jolt* (9 September 2026) — <https://a16zfrontier.com/posts/article/lattice-snarks-jolt-post-quantum-faster>

Application.

- `a16z/aerie` — <https://github.com/a16z/aerie>; paper <https://assets.layerzero.network/pdf/aerie.pdf>
- a16z, *Aerie: post-quantum signature aggregation with lattice SNARKs* (16 October 2026) — <https://a16zfrontier.com/posts/article/fast-post-quantum-signature-aggregation-with-lattice-snarks>

Prior and concurrent work.

- N. K. Nguyen, G. O'Rourke, J. Zhang, *Hachi: Efficient lattice-based multilinear polynomial commitments over extension fields* — <https://eprint.iacr.org/2026/156>
- N. K. Nguyen, G. Seiler, *Greyhound: Fast Polynomial Commitments from Lattices*, CRYPTO 2024 — <https://eprint.iacr.org/2024/1293>
- M. Klooss, R. W. F. Lai, N. K. Nguyen, M. Osadnik, L. Tucci, *RoKoko: Lattice-based Succinct Arguments, a Committed Refinement* — <https://eprint.iacr.org/2026/575>
- `lattice-arguments/rokoko` issue #94, *Incorrect radix-recomposition norm factor invalidates SIS estimates and several q/2 injectivity checks* — <https://github.com/lattice-arguments/rokoko/issues/94>
- G. Raikos, D. Wong, *Soundness failures in LaBRADOR implementations from NTT-friendly rings*, zkSecurity — <https://blog.zksecurity.xyz/posts/labrador-bugs/>
- A. Chiesa, P. Manohar, N. Spooner, *Succinct Arguments in the Quantum Random Oracle Model*, TCC 2019 — <https://eprint.iacr.org/2019/834>
- G. Arnon, A. Chiesa, G. Fenzi, E. Yogev, *WHIR: Reed–Solomon Proximity Testing with Super-Fast Verification*, EUROCRYPT 2025 — <https://eprint.iacr.org/2024/1586>
