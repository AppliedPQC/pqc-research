# NIST IR 8547, from draft to final: what changed in the transition schedule

> NIST published the final version of IR 8547, *Transition to Post-Quantum
> Cryptography Standards*, on 27 October 2026, and withdrew the initial public draft
> of November 2024 the same day. The publication page lists no changes. This note
> reads the two documents side by side, sentence by sentence, and records what moved.
> Final: <https://csrc.nist.gov/pubs/ir/8547/final>
> ([PDF](https://nvlpubs.nist.gov/nistpubs/ir/2026/NIST.IR.8547.pdf),
> doi:10.6028/NIST.IR.8547). Draft:
> <https://nvlpubs.nist.gov/nistpubs/ir/2024/NIST.IR.8547.ipd.pdf>.
>
> Checked 28 October 2026 against both PDFs and against OMB M-26-15 itself. The
> final is 34 pages to the draft's 30.

## The short version

The headline dates did not move: quantum-vulnerable public-key cryptography is
deprecated after 2030 and disallowed after 2035. Three things did change.

1. **Deprecation now covers every key size.** The draft deprecated only the
   112-bit level; the final deprecates RSA, ECDSA, EdDSA, finite-field and
   elliptic-curve Diffie-Hellman and MQV at every size.
2. **Federal deadlines arrive earlier.** The final is written around
   Executive Order 14412 and OMB Memorandum M-26-15, both of June 2026. They put the
   most sensitive federal systems on PQC key establishment by the end of 2030 and
   PQC signatures by the end of 2031. NIST's 2030–2035 window is relief for
   everything else.
3. **NIST now gives defaults where the draft listed options.** ML-KEM-768 is the
   recommended parameter set, HQC is the only further KEM planned, and hybrid
   constructions get more specific room in FIPS 140 validation.

## 1. Deprecation at every key size

The draft's Table 2 split each signature family in two. At 112 bits of security
(RSA-2048, P-224) the entry read *deprecated after 2030, disallowed after 2035*;
at 128 bits and above (P-256, RSA-3072, Ed25519) it read only *disallowed after
2035*. The text explained why: SP 800-57 had planned to disallow 112-bit public-key
schemes on 1 January 2031, and NIST would "instead deprecate classical digital
signatures at the 112-bit security level".

The final's Table 2 has one row per family and no split:

| Family | Draft (2024) | Final (2026) |
|---|---|---|
| ECDSA | 112 bits: deprecated after 2030; ≥ 128 bits: disallowed after 2035 | ≥ 112 bits: deprecated after 2030, disallowed after 2035 |
| EdDSA | ≥ 128 bits: disallowed after 2035 | ≥ 128 bits: deprecated after 2030, disallowed after 2035 |
| RSA | 112 bits: deprecated after 2030; ≥ 128 bits: disallowed after 2035 | ≥ 112 bits: deprecated after 2030, disallowed after 2035 |

Table 4, for key establishment (finite-field DH and MQV, elliptic-curve DH and MQV,
RSA key transport), changes in the same way. The summary sentence now reads: "NIST
will deprecate quantum-vulnerable public-key cryptography after 2030 and disallow
its use under NIST standards and guidelines after 2035."

The practical meaning comes from SP 800-131A's vocabulary. *Deprecated* means the
algorithm may still be used, but the data owner accepts the risk. After 2030, a
P-256 signature is in the same position RSA-2048 was in under the draft: allowed,
but no longer something a new design can choose without justification.

## 2. The federal deadlines

The draft's only policy anchor was National Security Memorandum 10 and its goal of
mitigating quantum risk by 2035. The final is built around two newer documents:

- **Executive Order 14412**, *Securing the Nation Against Advanced Cryptographic
  Attacks*, signed 22 June 2026
  ([whitehouse.gov](https://www.whitehouse.gov/presidential-actions/2026/06/securing-the-nation-against-advanced-cryptographic-attacks/),
  [govinfo](https://www.govinfo.gov/app/details/DCPD-202600417)).
- **OMB Memorandum M-26-15**, *Execution of the Migration to Post-Quantum
  Cryptography*, 24 June 2026
  ([PDF](https://www.whitehouse.gov/wp-content/uploads/2026/06/M-26-15-Execution-of-the-Migration-to-Post-Quantum-Cryptography.pdf)).

Under these, high value assets, high impact systems and other systems holding
highly sensitive data must adopt PQC for key establishment by 31 December 2030 and
for digital signatures by 31 December 2031. Agencies must support TLS 1.3 by
2 January 2030, a requirement carried over from Executive Order 14306. The final
says plainly that its own 2030–2035 window "does not alter the earlier deadlines"
for these systems.

M-26-15 itself sets five phases:

| Phase | Years | Work |
|---|---|---|
| 1. Strategy, planning and discovery | 2026–2027 | Inventory, assessment, governance |
| 2. Pilots and early migration | 2027–2028 | Pilots and the first prioritised systems |
| 3. Prioritised migration | 2028–2030 | PQC key establishment for the priority systems |
| 4. Signature migration | 2031 | PQC signatures for the same systems |
| 5. Full migration | by 2035 | Everything else |

Each agency's migration plan was due within 120 days of the memo. Inventories are to
feed a cryptographic bill of materials, and systems that cannot support PQC or hybrid
cryptography are to be replaced or decommissioned. The memo makes IR 8547 "or
successor document" the schedule agencies must align with. That is why the final
reads less like a proposal and more like a reference.

Two other new passages follow from the executive order. NIST is revising
Cryptographic Module Validation Program processes to speed up validation of PQC
modules. And PIV credentials and the federal identity infrastructure (FICAM), with
their multi-year card and reader cycles, are called out as needing to start early.

## 3. Key establishment

- **ML-KEM-768 is the default.** "NIST recommends using ML-KEM-768 as the default
  parameter set, as it provides a large security margin at a reasonable performance
  cost." The draft listed the three parameter sets without a recommendation. (It
  also labelled them ML-DSA-768 and ML-DSA-1024 in one table row, which the final
  corrects.)
- **HQC is in, and nothing else is coming.** The draft said the fourth round was
  still under evaluation. The final says HQC was selected, that "a FIPS that
  specifies HQC-KEM is under development", that HQC will be approved once
  published, and that "NIST has no current plans to select any additional KEMs".
- **Hybrids get more specific room.** The draft already promised to accommodate
  hybrid key establishment and dual signatures in FIPS 140 validation. The final
  adds that the combination can be validated "even if other inputs are disallowed",
  provided one input comes from an approved scheme. It allows SP 800-56C's
  concatenated secret in either order, *T*‖*Z* as well as *Z*‖*T*. It also says the
  next revision of SP 800-52 will carry the guidance on hybrid key exchange in TLS.

## 4. Signatures

- **Urgency stated outright.** "While signatures are not vulnerable to HNDL attacks,
  transitioning to post-quantum signatures is also urgent due to the longer duration
  and greater complexity of the transition process and the need for a greater number
  of parties to coordinate." This sentence is new.
- **FN-DSA is still absent.** The approved-signature table holds ML-DSA, SLH-DSA, LMS
  and XMSS, as before. The wording moved from "NIST intends to develop" a
  Falcon-derived FIPS to "NIST is developing" one; there is still no FIPS 206 draft.
- **ML-DSA-44 has a footnote.** Its security category drops from 2 to 1 if it is
  used with an approved random bit generator offering less than 192 bits of
  security.
- **Pre-hash variants.** HashML-DSA and HashSLH-DSA must use a digest at least as
  strong as the parameter set.
- **No more classical signatures.** "NIST does not anticipate standardizing any
  further digital signature algorithms which are not quantum-resistant."
- **Stateful hash-based signatures have a stated use.** The draft said only that
  LMS and XMSS "are not intended for general use". The final adds that they "may be
  useful for applications like software and firmware signing".

## 5. Symmetric cryptography

The draft already said the few 112-bit symmetric standards will be disallowed in
2030, and that is unchanged. New in the final:

- Ascon from SP 800-232 appears in the tables: Ascon-AEAD128 as an approved AEAD at
  category 1, and Ascon-Hash256, Ascon-XOF128 and Ascon-CXOF128 as hash functions
  and XOFs.
- SHA-1's collision strength is given as "<80 bits" rather than 80. A new footnote
  says it is deprecated for every remaining use and will be disallowed in
  FIPS-validated modules after 2030.

## 6. What did not change

- The 2030 and 2035 dates.
- The five security categories and their reference attacks.
- The definitions of acceptable, deprecated, disallowed and legacy use.
- The 112-bit symmetric deadline.
- The commitment to hybrid modes and dual signatures in FIPS 140 validation.

One editorial slip survives from the draft. The table of contents still lists
section 4 as "Toward a PQC Standards Transition Timeline", while the heading in the
body now reads "PQC Standards Transition Timeline".

## What it means in practice

- **Outside the US government**, the 2035 date still binds nothing directly. But it
  is now the date a FIPS 140 module stops accepting P-256 and RSA-3072, not just the
  112-bit sizes. Anything that sells into regulated markets inherits it.
- **New designs** have a default to point to: ML-KEM-768 for key establishment,
  hybrid if wanted, and a validated path for the hybrid.
- **Signatures** are explicitly on the clock even though no one is harvesting them.
  The federal date for them is 2031.
- **Inventory comes first.** M-26-15's first phase and its bill-of-materials
  requirement say so. The EU roadmap (national inventories started by end-2026) and
  the UK NCSC (discovery by 2028) say the same.

## Sources

- NIST IR 8547, final, October 2026 — <https://csrc.nist.gov/pubs/ir/8547/final>
- NIST IR 8547, initial public draft, November 2024 (withdrawn) — <https://nvlpubs.nist.gov/nistpubs/ir/2024/NIST.IR.8547.ipd.pdf>
- NIST news release, 27 October 2026 — <https://www.nist.gov/news-events/news/2026/10/nist-report-provides-guidelines-transition-post-quantum-cryptography>
- Executive Order 14412 — <https://www.govinfo.gov/app/details/DCPD-202600417>
- OMB Memorandum M-26-15 — <https://www.whitehouse.gov/wp-content/uploads/2026/06/M-26-15-Execution-of-the-Migration-to-Post-Quantum-Cryptography.pdf>
- NIST SP 800-131A Rev. 2 — <https://csrc.nist.gov/pubs/sp/800/131/a/r2/final>
