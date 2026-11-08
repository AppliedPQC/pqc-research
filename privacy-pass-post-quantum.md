# Post-quantum Privacy Pass: anonymous credentials from UOV and VOLE-in-the-head

> Privacy Pass lets a site rate-limit or vouch for a client without learning who it
> is: an issuer hands out tokens, and a token's redemption cannot be linked to its
> issuance. Today's tokens rest on RSA blind signatures and elliptic curves. On
> 6 November 2026 Cloudflare Research published three post-quantum replacements built
> from the UOV multivariate signature and the VOLE-in-the-head proof system behind
> FAEST, with an implementation. This note reads the post and the code.
> Post: <https://blog.cloudflare.com/vole-mole/> (Bas Westerbaan).
> Code: <https://github.com/cloudflareresearch/vole-mole>.
>
> Checked 8 November 2026 against the post, the repository README and the papers it
> cites.

## The short version

1. **A quantum computer breaks forgery, not privacy.** Privacy Pass is
   unconditionally private: once Q-day arrives, already-issued tokens stay unlinkable.
   What falls is the issuer's signature, so an attacker could mint tokens and walk
   past the rate limits they enforce.
2. **The building blocks are UOV and VOLE-in-the-head.** UOV signatures are 135
   bytes with 46.6 KB public keys; VOLE-in-the-head proves statements about them in
   zero knowledge efficiently, because UOV verification is a quadratic map over a
   small binary field. All estimates use FAEST-EM-128s parameters.
3. **Three constructions cover the Privacy Pass family.**

   | Scheme | What it is | Issuance | Presentation |
   | --- | --- | ---: | ---: |
   | PoMFRIT (with UOV) | blind signature | < 1 KB | 4.3 KB |
   | VOLE-IHAT | token that also hides which issuer signed it | 5.6 KB | 6.5 KB |
   | VOLE-ACT | anonymous credit token with a hidden balance | 5.4 KB | 12.0 KB |

4. **The cost is in line with the rest of the migration.** The authors put the
   overhead of replacing RSA blind signatures with PoMFRIT on a par with replacing
   ECDSA with ML-DSA, and every operation runs in milliseconds.
5. **It is research code.** The repository describes itself as experimental and
   unaudited, with APIs and wire formats expected to change, and one of the
   optimisations rests on a new assumption, one-more-UOV.

## 1. What Privacy Pass protects

An issuer signs a token for a client after some check, such as a challenge or an
attestation. Later the client presents the token to a site, which verifies the
signature. Two properties matter:

- **Unlinkability.** The issuer cannot tell which issuance a redemption came from.
  RSA blind signatures give this unconditionally, so it holds against any attacker,
  quantum or not.
- **Unforgeability.** Only the issuer can make tokens. This rests on RSA, and Shor's
  algorithm breaks it.

The post also cites a recent classical result: researchers at UC San Diego and Inria
forged a 1024-bit RSA signature by using a Privacy Pass issuer as a signing oracle
(ePrint 2026/2131). Privacy Pass deployments use 2048-bit RSA, which the authors
consider a comfortable margin with the other defences in place.

The tokens also reveal their issuer, because verification needs the issuer's public
key. Hiding that is what VOLE-IHAT adds.

## 2. The building blocks

**VOLE-in-the-head** (Baum et al., Crypto 2023, ePrint 2023/996) is a
zero-knowledge proof system that is efficient for statements written as equations
over small binary fields. Its proofs grow linearly with the witness and the
statement. It is the basis of the FAEST signature, whose AES proofs are about 4 KB at
the round-3 128s parameter set, and of the KuMQuat signature.

**UOV** is a multivariate signature, among the nine schemes in the third round of
NIST's additional-signatures process. Its signatures are 135 bytes and its public
keys 46.6 KB. Verifying a UOV signature means evaluating a public quadratic map,
which costs about as much in VOLE-in-the-head as an MQ instance. The large key is
acceptable here because an issuer's key is fetched once, cached and used for many
tokens. MAYO, a UOV variant, has much smaller public keys and slightly larger
signatures, and gives larger proofs in this setting.

## 3. The three constructions

**PoMFRIT** (Baum et al., USENIX Security 2026, ePrint 2026/109) turns a signature
into a blind signature:

1. The client commits to its message, `c = H(m, r)`.
2. The issuer signs the commitment, `s = Sign(sk, c)`.
3. The client proves in zero knowledge that it knows `s` and `r` such that
   `Verify(pk, H(m, r)) = 1`. That proof is the blind signature.

The issuer saw only `c`, and the verifier sees only a proof, so the two cannot be
linked. The paper uses MAYO, for a 6.7 KB signature; with UOV it is 4.3 KB, and
issuance stays under 1 KB. A further optimisation has the client mask the hash and
the issuer find a UOV preimage directly, removing one hash from the proof; it changes
the security assumption to *one-more-UOV*.

**MOLE** (Moderation Of unLinkable Endorsements) is the protocol shape the other two
schemes target, which the post links to Mozilla Hacks. An *anchor* issues an
*endorsement*; a *moderator* accepts its redemption, where the client proves it holds
an endorsement from one key in the agreed anchor set without saying which; the
moderator then issues a *credential* that governs the client's interaction with the
site. Credentials need not hide their issuer or be publicly verifiable.

- **VOLE-IHAT** is the endorsement candidate: an issuer-hiding token whose redemption
  proves a valid signature under one of the anchor keys `P₁, …, P_N`, with the index
  kept in the witness. 5.6 KB to issue, 6.5 KB to redeem.
- **VOLE-ACT** is the credential candidate: an anonymous credit token whose balance
  the issuer updates on each spend without learning it. Proving that a balance lies
  in a range is cheap in VOLE-in-the-head. 5.4 KB to issue, 12.0 KB to spend.

## 4. The code

[`cloudflareresearch/vole-mole`](https://github.com/cloudflareresearch/vole-mole) is a
C++ library built with Meson, under the BSD 3-Clause licence. It implements PoMFRIT,
VOLE-IHAT (bound and unbound) and VOLE-ACT, with examples for each, and bundles
`faest-arch-opt` (MIT) and `pqov` (CC0 or Apache-2.0).

- **Build.** Meson 1.4+, Ninja, Python 3 and Clang 21+ or GCC 16+.
- **Performance**, on an Apple M4 Pro by the README: client messages with proofs of
  5–17 KB, or 4–12 KB with the smaller-proof parameter option; every step under 2 ms
  except VOLE-IHAT redemption, about 7 ms with 64 anchors.
- **Status.** "Experimental and not audited. Do not use in production." Double-spend
  checks belong to the caller, who tracks nullifiers.

## 5. The work the authors set out

The post closes by calling post-quantum anonymous credentials a move "from
cryptography design to protocol engineering", and lists what comes next:

- security proofs that reduce each protocol to the security of UOV;
- more cryptanalysis of multivariate schemes, which may change parameters and costs,
  with NIST Level 3 parameters for UOV suggested as a hedge for the one-more-UOV
  optimisation;
- amortisation, so one redemption covers several requests.

## What it means in practice

- **Operators of Privacy Pass issuers.** Today's tokens stay private after Q-day;
  forgery is the exposure, and it matters only once a quantum computer exists, since
  tokens are short-lived. That puts this migration on the signature clock, not the
  harvest-now-decrypt-later clock that drove key exchange.
- **Protocol designers.** A 4–12 KB token is workable on the web. Size, not compute,
  is the cost to plan for, and caching the 46.6 KB issuer key is part of the design.
- **Implementers.** Use `vole-mole` to measure and to experiment. The authors list
  security proofs and stable parameters as the next step.

## Sources

- Bas Westerbaan, [The post-quantum future of web privacy](https://blog.cloudflare.com/vole-mole/), Cloudflare, 6 November 2026.
- [`cloudflareresearch/vole-mole`](https://github.com/cloudflareresearch/vole-mole), README.
- Baum et al., [Publicly verifiable zero-knowledge and post-quantum signatures from VOLE-in-the-head](https://eprint.iacr.org/2023/996), Crypto 2023.
- Baum et al., [Concretely efficient blind signatures based on VOLE-in-the-head proofs and the MAYO trapdoor](https://eprint.iacr.org/2026/109) (PoMFRIT), USENIX Security 2026.
- [Forging 1024-bit RSA signatures in nearly SNFS time](https://eprint.iacr.org/2026/2131), ePrint 2026/2131.
- [FAEST](https://faest.info/); [MAYO](https://pqmayo.org/); [UOV](https://csrc.nist.gov/projects/pqc-dig-sig) in NIST's additional signatures process.
- IETF [Privacy Pass working group](https://datatracker.ietf.org/wg/privacypass/about/).
