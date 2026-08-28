# Figures for shrincs-construction.md

One directory per post; these belong to `shrincs-construction.md` in the repository root.

Each figure is committed twice: the `.svg` is the editable source, and the `.png` is what
the document references, since raw SVG does not render reliably from a raw URL on GitHub
or through the blog build.

| Figure | Shows |
|---|---|
| `architecture` | how a signature is composed on both paths, and the cross-binding between them |
| `cross-binding` | the byte layout of the two message digests |
| `wots-c-chain` | one WOTS+C chain at every Winternitz position, and all 32 indexes of one signature |
| `wots-c-grinding` | the constant-sum counter search |
| `fxmss-shapes` | UXMSS and BXMSS side by side |
| `fxmss-signature` | a worked signature and its verification |

To change one, edit the `.svg` and re-export the `.png` at twice the display size, which is
what keeps the labels sharp when the image is scaled down to the prose column.

The figures are drawn from the draft's own constants and algorithms rather than sketched:
the 32 chain indexes are a real grinding result at counter 22, and the trees follow the
draft's leaf-selection rule, so the worked example's authentication path and index bits are
the ones a signer would actually produce.
