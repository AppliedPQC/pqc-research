# Figures for privacy-pass-post-quantum.md

One directory per post; these belong to `privacy-pass-post-quantum.md` in the repository root.

Each figure is committed twice: the `.svg` is the editable source, and the `.png` is what
the document references, since raw SVG does not render reliably from a raw URL on GitHub
or through the blog build.

| Figure | Shows |
|---|---|
| `privacy-pass` | the issuer, client and origin, what a quantum computer breaks and what it does not, and what the replacement swaps |
| `uov-trapdoor` | a secret UOV polynomial as a coefficient grid with the oil-times-oil block zero, the signing steps, and the uov-Ip sizes |
| `sizes` | signature and public-key sizes on a log scale, UOV against ECDSA, RSA, ML-DSA, SLH-DSA, FAEST-EM and MAYO |
| `vole-quicksilver` | a VOLE correlation as a commitment, committing to the witness, and QuickSilver's check at the secret point Δ |
| `voleith-tree` | the all-but-one seed tree, the opened siblings, and how the verifier rebuilds q without the hidden seed |
| `voleith-flow` | the seven steps of a VOLE-in-the-head proof, with Δ drawn last |
| `pomfrit` | PoMFRIT between client, issuer and verifier |
| `mole` | MOLE's anchors, moderator and credentials, with the sizes of all three schemes |

To change one, edit the `.svg` and re-export the `.png` at twice the display size, which is
what keeps the labels sharp when the image is scaled down to the prose column.

The numbers come from the sources the report cites: the UOV Round 3 specification for the
uov-Ip sizes, faest.info and pqmayo.org for FAEST and MAYO, FIPS 204 and 205 for ML-DSA and
SLH-DSA, and the Cloudflare post for the token sizes.
