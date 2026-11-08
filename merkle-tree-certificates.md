# Merkle Tree Certificates: how the public web gets post-quantum authentication

> Key exchange on the web went post-quantum by swapping one algorithm for another.
> Certificates cannot follow the same path: post-quantum signatures are too large to
> carry several of them in every TLS handshake. Chrome, Let's Encrypt and Cloudflare
> have instead chosen Merkle Tree Certificates (MTCs), a new certificate format that
> the IETF PLANTS working group is standardising. This note reads the draft, Chrome's
> plan, the Cloudflare–Chrome experiment, and the two certificate authorities that
> have committed to issuing MTCs.
> Draft: <https://datatracker.ietf.org/doc/draft-ietf-plants-merkle-tree-certs/>
> (-08, 2 November 2026).
>
> Checked 8 November 2026 against the draft, the Chrome and Chromium announcements,
> and the Let's Encrypt and Cloudflare posts listed under Sources.

## The short version

1. **The problem is size.** An ML-DSA-44 signature is 2,420 bytes and its public key
   1,312 bytes, against 64 and 64 for ECDSA P-256. A typical web handshake carries
   five signatures and two public keys, so a direct swap pushes it past 10 kB, which
   Let's Encrypt expects to slow some connections and break others.
2. **MTCs sign batches, not certificates.** A CA appends every certificate to a
   Merkle tree and signs the tree. A browser that already holds a recent tree head,
   a *landmark*, needs only the server's public key, one signature from that key and
   an inclusion proof: 736 bytes with no signatures in the draft's example.
3. **Transparency comes with issuance.** Every certificate is in the CA's log by
   construction, and independent cosigners sign its checkpoints, so Certificate
   Transparency is no longer a separate step.
4. **Chrome has chosen MTCs for the public web.** On 27 February 2026 it announced a
   Chrome Quantum-resistant Root Store that accepts only MTCs, rolled out in three
   phases through Q3 2027, alongside the existing root program.
5. **The format works on real traffic.** Cloudflare and Chrome served billions of
   MTCs to half of Chrome Beta 146 on selected domains; at the median, landmark MTCs
   were 9% faster than a classical chain. The experiment used classical signatures
   and ended in August 2026.
6. **Two CAs have committed.** Let's Encrypt chose MTCs in June 2026 and received
   more than €1M from the Sovereign Tech Agency in November 2026 to build its MTC CA
   through 2027. Cloudflare announced a public CA in September 2026, with its first
   MTCs in the first quarter of 2027 and standard MTC issuance at no cost.

## 1. Why certificates are the hard half

Post-quantum key exchange added one ML-KEM share to each side of the handshake,
about a kilobyte, and it shipped by default in browsers, operating systems and CDNs.
Authentication multiplies instead. A server sends its certificate chain, usually a
leaf and an intermediate, and each carries a public key and a CA signature; the
handshake adds the server's own signature and the Signed Certificate Timestamps that
Certificate Transparency requires. Let's Encrypt counts five signatures and two
public keys in a typical handshake.

| | Signature | Public key |
| --- | ---: | ---: |
| ECDSA P-256 | 64 B | 64 B |
| RSA-2048 | 256 B | 256 B |
| ML-DSA-44 | 2,420 B | 1,312 B |

Replacing each with ML-DSA-44 takes the handshake past 10 kB. Cloudflare puts the
growth at roughly forty times per signature, and the same factor applies to the
storage of every Certificate Transparency log.

Private PKI can absorb this: it controls both ends, and ML-DSA certificates are
specified for X.509 in RFC 9881 and issued by rustls, Go and OpenSSL. The public web
cannot, because every browser and every server has to agree.

## 2. How Merkle Tree Certificates work

The draft, `draft-ietf-plants-merkle-tree-certs-08`, is a Standards Track document of
the PKI, Logs, And Tree Signatures (PLANTS) working group. Its authors are David
Benjamin (Google), Devon O'Brien (Apple), Bas Westerbaan and Luke Valenta
(Cloudflare) and Filippo Valsorda (Geomys). It replaces
`draft-davidben-tls-merkle-tree-certs`.

**The log.** An MTC CA keeps an append-only log of everything it issues, as a Merkle
tree. The CA signs checkpoints of the tree, and independent *cosigners* sign them
too. The draft leaves cosigner roles open and names witnesses and mirrors as
examples; relying parties require a quorum of cosigners, who must also enforce a
consistent view of the log. Chrome's draft policy, as Cloudflare reports it, asks
for at least two cosignatures, one of them from a Chrome-recognised mirroring
cosigner.

**Two kinds of certificate.**

- A *standalone* certificate carries the cosignatures on a small subtree that
  contains it, so any client can verify it. In the draft's example its inclusion
  proof is 12 hashes, 384 bytes, plus those signatures.
- A *landmark-relative* certificate replaces the signatures with an inclusion proof
  to a larger subtree the client already trusts. In the draft's example that proof
  is 736 bytes, with no signatures at all. Proofs are bounded at 32 hashes, 1,024
  bytes, for subtrees of up to 2^32 certificates.

**Landmarks.** A CA allocates landmarks periodically: each has a number, a tree size
and an expiry, and together they cover every certificate still valid. Clients fetch
them out of band, ahead of any connection. The draft's example allocates a landmark
every hour for certificates of up to seven days, which gives at most 169 active
landmarks and, with SHA-256, 10,816 bytes of state on the client for 338 subtrees.

**In the handshake.** A server holds both forms and sends the landmark-relative one
when the client signals that it has the matching landmark, and the standalone one
otherwise. What travels in the common case is the server's public key, the server's
handshake signature and one inclusion proof, which Cloudflare puts at under 1 kB for
the proof. With a classical server key, as in the experiment, that is already smaller
than today's chain; with an ML-DSA server key it keeps the handshake to one
post-quantum signature and one post-quantum public key.

## 3. Chrome's plan

On 27 February 2026 the Chrome Secure Web Networking Team announced that Chrome has
no immediate plan to add post-quantum X.509 certificates to its root store, and that
it is building the **Chrome Quantum-resistant Root Store** (CQRS) for MTCs only,
operated alongside the existing Chrome Root Program.

| Phase | When | What |
| --- | --- | --- |
| 1 | Under way in February 2026 | Feasibility study with Cloudflare, each MTC connection backed by a trusted X.509 certificate |
| 2 | Q1 2027 | Certificate Transparency log operators with at least one usable log in Chrome before 1 February 2026 are invited to bootstrap public MTCs |
| 3 | Q3 2027 | Requirements for onboarding CAs into CQRS finalised, and sites can opt in to downgrade protection |

The announcement lists the direction for CAs in the new store: ACME-only issuance;
prospective MTC CAs first proving themselves as mirroring cosigners and domain
control validation monitors; revocation reduced to key-compromise events in place of
CRLs; reproducible domain validation with public, persistent proofs; and continuous,
externally verifiable monitoring that could replace annual audits.

The companion Chromium roadmap (David Benjamin and Joe DeBlasio, same date) sets out
four stages without dates:

1. **Add post-quantum options** alongside classical ones. On its own this protects
   nothing, because an attacker targets the weakest option a client accepts.
2. **Opt in individual sites** with an HSTS-like `Require-Post-Quantum` header, a
   stopgap the roadmap describes along with its risks.
3. **Require a post-quantum PKI**: clients trust post-quantum CAs, such as MTC CAs,
   and no classical ones, while still accepting classical server keys inside those
   certificates.
4. **Require post-quantum TLS keys**, drop classical key agreement and TLS 1.2. The
   roadmap calls this a far-future stage.

Chrome's TLS stack is BoringSSL, which moved its MTC code to draft-07 in November
2026 and dropped draft-05, which Chrome no longer configures.

## 4. The Cloudflare–Chrome experiment

Cloudflare ran a bootstrap CA that issued MTCs backed by a traditional certificate
chain, and served them to 50% of Chrome Beta 146 on selected free-plan domains.
Cloudflare's report, from September 2026:

- **Scale.** Billions of MTCs served over the course of the experiment.
- **Speed.** At the median, connections with landmark MTCs were 9% faster than with
  a classical signature chain. Cloudflare attributes most of that to leaving out the
  intermediate certificate.
- **Scope.** The experiment used classical signatures, so the post-quantum saving,
  which is the larger one, was outside what it measured. It wound down in August
  2026.

## 5. The certificate authorities

**Let's Encrypt.** On 3 June 2026 Let's Encrypt chose MTCs as its path to
post-quantum certificates, targeting a staging environment that issues MTCs in late
2026 and production in 2027. Its argument for the design is the one in section 2:
one signature per batch keeps the handshake small, and transparency becomes a
property of issuance. On 5 November 2026 the Sovereign Tech Agency contracted more
than €1M for Let's Encrypt's MTC CA, through the end of 2027, covering
standardisation at the IETF, C2SP and the root programs, the move from staging to a
compliant production launch, and the global rollout to subscribers. Let's Encrypt
calls it its largest single investment in post-quantum readiness.

**Cloudflare.** On 29 September 2026 Cloudflare announced it is becoming a public CA.
It signed a definitive agreement to acquire a broadly trusted root from GlobalSign,
applied to the Chrome, Apple, Microsoft and Mozilla root programs, and will issue
through ACME only, to clients that support ACME Renewal Information (RFC 9773). Its
first MTCs are scheduled for the first quarter of 2027, targeting inclusion in
Chrome's Quantum-resistant Root Store. The companion post describes the CA: an ACME
server forked from Boulder, the software behind Let's Encrypt; mirroring cosigners
built on Azul, Cloudflare's Rust transparency log; and standard MTC issuance at no
cost.

## What it means in practice

- **Server operators.** Turn on hybrid key exchange (X25519MLKEM768) now; Let's
  Encrypt calls it one of the highest-leverage steps available. Expect to hold two
  certificates per name, a landmark-relative MTC and a standalone fallback, and let
  the TLS library choose between them.
- **ACME client maintainers.** Implement ACME Renewal Information: Cloudflare will
  issue only to clients that use it, and Chrome's store is ACME-only. Follow the
  PLANTS working group and the `mtcs@chromium.org` list, where the client-side
  changes are being worked out.
- **Private PKI.** ML-DSA X.509 certificates under RFC 9881, or composites under
  RFC 10073, work today where you control both ends. MTCs are the public web's
  answer, built for browsers and public CAs.
- **Planning.** Chrome's dates give the outline: public MTC infrastructure from Q1
  2027 and CA onboarding requirements in Q3 2027.

## Sources

- IETF, [`draft-ietf-plants-merkle-tree-certs`](https://datatracker.ietf.org/doc/draft-ietf-plants-merkle-tree-certs/), version -08, 2 November 2026.
- Chrome Secure Web Networking Team, [Cultivating a robust and efficient quantum-safe HTTPS](https://blog.google/security/cultivating-a-robust-and-efficient-quantum-safe-https/), 27 February 2026.
- David Benjamin and Joe DeBlasio, [Post-quantum HTTPS authentication roadmap](https://www.chromium.org/Home/chromium-security/post-quantum-auth-roadmap/), Chromium, 27 February 2026.
- BoringSSL, [move MTC support to draft-07](https://github.com/google/boringssl/commit/fc6504a623d8cf5b92fafec714ac28396f67c01b), November 2026.
- Let's Encrypt, [The path to post-quantum certificates](https://letsencrypt.org/2026/06/03/pq-certs.html), 3 June 2026.
- Let's Encrypt, [Sovereign Tech Agency funding for post-quantum certificates](https://letsencrypt.org/2026/11/05/sta-pq-funding), 5 November 2026.
- Cloudflare, [Building a certificate authority for the whole Internet](https://blog.cloudflare.com/cloudflare-certificate-authority/), 29 September 2026, and the [press release](https://www.cloudflare.com/press/press-releases/2026/cloudflare-announces-public-certificate-authority-for-the-post-quantum-web/).
- Cloudflare, [Building a post-quantum CA with Merkle Tree Certificates](https://blog.cloudflare.com/pq-ca-with-mtcs/), 29 September 2026.
- [RFC 9773](https://www.rfc-editor.org/rfc/rfc9773.html), ACME Renewal Information; [RFC 9881](https://www.rfc-editor.org/rfc/rfc9881.html), ML-DSA in X.509; [RFC 10073](https://www.rfc-editor.org/rfc/rfc10073.html), composite ML-DSA in X.509.
