# SHRINCS 的构造：一种半有状态的哈希签名方案

> SHRINCS（*Shrunken SPHINCS*）是 2026 年 8 月 27 日发布到 bitcoin-dev 邮件列表上的一份密码学 BIP 草案。它把 FIPS 205 中标准化的哈希签名方案 SLH-DSA 降为回退路径，而在主路径上采用一种更小的**有状态**签名。本文沿着 FIPS 205 描述 SLH-DSA 时所用的几个维度来拆解该方案：哪些部分原样复用，哪些部分只改变了参数，哪些部分是新构造的，以及这一设计放弃了什么。规范见 <https://github.com/SHRINCS/shrincs-bip/blob/main/SHRINCS.md>。
>
> 状态：初稿，标注为“Do NOT use in production”（请勿用于生产环境）。安全性证明、测试向量和单元测试均尚未完成。为检验下文的各项论断而编写的整个方案的 Rust 实现位于 [`AppliedPQC/shrincs-rs`](https://github.com/AppliedPQC/shrincs-rs)。术语沿用 FIPS 205，下一节先确定这些术语，再加以使用。

## 记号与术语

本文通篇沿用 FIPS 205 的术语，另外加上该标准中没有的两种构造。下面大致按照用到的先后顺序给出这些术语。

| 术语 | 含义 |
|---|---|
| **一次性签名**，OTS | 秘密钥仅对单条消息安全的方案。在同一密钥下生成第二个签名会泄露足够多的密钥信息，使第三个签名可以被伪造。WOTS+、WOTS-TW 和 WOTS+C 都属于此类。 |
| **少次签名**，FTS | 在同一密钥下可容忍少量签名的方案，其安全性随签名数量增加而下降。FORS 属于此类。 |
| **Winternitz 链** | 对一个链秘密反复应用哈希所得的序列。公布第 `d` 次迭代值即编码了索引 `d`，验证者可以沿链前进，但不能回退。 |
| **Winternitz 参数** `w` | 链长，在 SHRINCS 中始终为 16，因此每条链承载摘要的 `lg w = 4` 比特。 |
| **校验和链** | 额外的链，编码消息索引之和的补数，使得提高任一消息索引都会迫使校验和索引降低，而这需要对链求逆。WOTS-TW 附加三条这样的链。 |
| **常数和编码** | WOTS+C 采用的替代方案。不附加校验和，而是只接受各分量之和等于某个固定常数的索引向量，签名者通过搜索计数器，直到摘要产生这样一个向量。于是提高一个索引就必须降低另一个索引，链步数也因此固定。也称 target-sum Winternitz（目标和 Winternitz）。 |
| **Merkle 树** | 一棵二叉树，其叶子承诺待认证的值，内部节点是其两个子节点的哈希，因此根承诺了所有叶子。 |
| **认证路径** | 从叶子到根的路径上每个节点的兄弟节点。它与叶子一起确定根，签名正是携带它来证明成员关系。 |
| **XMSS** | 一棵叶子为 OTS 公钥的 Merkle 树，每个节点都用可调哈希计算。每个签名消耗一片叶子，这使该方案成为有状态的。 |
| **超树** | 由 XMSS 树构成的树，每一层用其某个 OTS 密钥对下一层的根签名。这样无需任何单棵树高到不切实际，就能达到很大的签名预算。 |
| **FORS** | 由 `k` 棵高为 `a` 的 Merkle 树构成的少次签名。摘要在每棵树中选出一片叶子，签名公开这些叶子及其认证路径。 |
| **可调哈希** | 以公开种子和位置为密钥的哈希，使得在某一位置应用的函数与在任何其他位置应用的函数相互独立。 |
| **ADRS** | 编码上述位置的地址。为每次调用赋予不同的 ADRS，正是这些哈希彼此独立的原因。 |
| **域分离** | 通过让函数本身随角色不同而不同，确保扮演不同角色的输入不会发生碰撞。 |
| **多目标攻击** | 同时针对许多已公布哈希中任意一个寻找原像的搜索。优势随目标数量增长，这正是要分离位置、并为树的边规定方向的原因。 |
| **有状态方案** | 签名者必须记录哪些叶子已被使用的方案，因为重复使用会破坏安全性。 |
| **无状态方案** | 签名者不保留任何持久记录的方案，代价是签名更大。 |
| **半有状态方案** | SHRINCS 引入的安排：一个密钥对同时承载二者，只要状态不可用，就可使用无状态组件。 |
| **签名预算** | 一个密钥对在叶子耗尽之前能够生成的签名数量。 |

## 1. 与 SLH-DSA 的关系

| 层面 | 内容 |
|---|---|
| **原样复用** | 无状态路径就是 SLH-DSA。`slh_dsa_sign` 和 `slh_dsa_verify` 与 FIPS 205 的算法 22 和算法 24 逐行对应，唯一的区别是签名随机化值由调用者提供，而非在内部生成。一个接受自定义参数集的 SLH-DSA 实现只需一层薄封装 |
| **重新参数化** | 无状态路径使用一个**非标准参数集**：签名预算从 2⁶⁴ 降到 2⁴⁰，签名从 7,856 字节降到 5,776 字节 |
| **新构造** | 有状态路径：FXMSS，一种形状可变的 Merkle 树，建立在常数和一次性签名 WOTS+C 之上。二者都不源自 SPHINCS+ |
| **共享机制** | SHA-256 作为唯一的底层哈希；ADRS 可调哈希机制；所有内部输出截断为 16 字节 |

SLH-DSA 之所以被设计为无状态，是因为 SP 800-208 认定 XMSS 和 LMS“not suitable for general use”（不适合通用场景）。SHRINCS 朝相反方向走：它重新引入状态以换取更小的尺寸，然后借助无状态回退，把丢失状态的后果从损失资金降为签名变大。

## 2. 架构

| 组件 | 结构 | 路径 |
|---|---|---|
| **FXMSS**，一种形状可变的 Merkle 树签名 | 建立在**任意形状** Merkle 树上的哈希链 | 有状态（主路径） |
| **WOTS+C**，FXMSS 所用的一次性签名 | 采用**常数和编码**的哈希链 | 有状态（主路径） |
| **SLH-DSA**，无状态回退 | 建立在 FORS 之上的超树 | 无状态（回退） |
| WOTS-TW，SLH-DSA 内部的一次性签名 | 带校验和的哈希链 | 无状态（回退） |
| FORS，一种少次签名 | 许多小型 Merkle 树 | 无状态（回退） |

两条路径中任何一条产生的签名都能通过验证。签名者在日常使用有状态路径，当状态丢失、损坏或不确定时（例如从静态备份恢复之后）回退到无状态路径。

### 签名流程

[![SHRINCS 签名的组成：有状态路径和无状态回退，二者都归约到 48 字节的公钥](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/architecture.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/architecture.png)

每条路径都归约到一个根，两个根合起来构成公钥。二者并非彼此独立；参见第 6 节中的交叉绑定。

### 区分两条路径

签名的第一个字节是判别符：

- `= 255`（`FXMSS_HEIGHT`）选择无状态路径；
- `0 ≤ x < 255` 选择有状态路径，该字节是签名所用叶子的 `leaf_height`。

255 之所以可以用作标签，是因为 `leaf_height = 255` 对应深度 0，而深度为 0 的树被禁止用于签名。长度也能区分二者：有状态签名的最大长度 4,619 字节小于固定的无状态签名长度 5,777 字节，因此存在两个相互独立的判别依据。

## 3. 密钥

密钥生成从调用者处获取一个 **48 字节的种子**，并将其拆分为三个 16 字节的部分：

```
seed(48) = sk_seed(16) ‖ sk_prf(16) ‖ pk_seed(16)
```

| | 组成 | 长度 |
|---|---|---|
| **公钥** | `pk_seed ‖ sl_root ‖ sf_root` | **48 B** |
| **秘密钥** | `sk_seed ‖ sk_prf ‖ pk_seed ‖ sl_root ‖ sf_structure ‖ sf_root` | **82 B** |

| 字段 | 作用 |
|---|---|
| `pk_seed` | 每个可调哈希的公开参数 |
| `sk_seed` | 派生 WOTS-TW、WOTS+C 和 FORS 的秘密钥 |
| `sk_prf` | 派生消息随机化值 R |
| `sl_root` | 无状态组件：SLH-DSA 超树顶层 XMSS 树的根 |
| `sf_root` | 有状态组件：FXMSS 的根，位于高度 255 |
| `sf_structure` | **2 字节**，`(shape, depth)`，描述 FXMSS 树；只保存在秘密钥中 |

两种编码都是有意为之：

- 截去公钥的**最后 16 字节**（`sf_root`），剩下的就是无状态组件在其参数集下的标准 SLH-DSA 公钥；
- 截去秘密钥的**最后 18 字节**（`sf_structure ‖ sf_root`），剩下的就是一个有效的 SLH-DSA 秘密钥。

因此，SHRINCS 密钥是 SLH-DSA 密钥的后缀扩展。

## 4. 参数集

### 有状态参数

| 参数 | 取值 | 含义 |
|---|---|---|
| `WOTS_C_CHAIN_BITS` | 4 | 每条 Winternitz 链编码的比特数（w = 16） |
| `WOTS_C_CHAIN_COUNT` | **32** | Winternitz 链的数量，**不含校验和链** |
| `FXMSS_HEIGHT` | 255 | FXMSS 树的假想高度，因而也是叶子的最大深度 |

派生值：`WOTS_C_CONSTANT_SUM = ⌈32 × 15 / 2⌉ = 240`，即各链索引的目标和，也是其期望值。

### 无状态参数，与 FIPS 205 对照

| 参数 | FIPS 205 名称 | SHRINCS | SLH-DSA-SHA2-**128s** |
|---|---|---|---|
| 安全参数（字节） | `n` | 16 | 16 |
| Winternitz | `lg_w` | 4 (w=16) | 4 (w=16) |
| 超树层数 | `d` | **5** | 7 |
| 每层 XMSS 高度 | `h'` | 9 | 9 |
| 超树总高度 | `h` | **45** | 63 |
| FORS 树高 | `a` | **13** | 12 |
| FORS 树数量 | `k` | **10** | 14 |
| 消息摘要长度 | `m` | **24** | 30 |
| **签名预算** | | **2⁴⁰** | 2⁶⁴ |
| **签名长度** | | **5,776 B** | 7,856 B |

WOTS-TW 的链数：`len1 = ⌈128/4⌉ = 32`，`len2 = ⌈bitlen(32×15)/4⌉ = ⌈9/4⌉ = 3`，共 **35** 条链，比 WOTS+C 多三条。

2,080 字节的缩减可以精确分解如下：

| 组成部分 | SLH-DSA-128s | SHRINCS | 差值 |
|---|---|---|---|
| 随机化值 R | 16 | 16 | 0 |
| FORS 签名，`16·k·(a+1)` | 16×14×13 = 2,912 | 16×10×14 = **2,240** | −672 |
| 超树签名，`d × (35×16 + 16×h')` | 7×704 = 4,928 | 5×704 = **3,520** | −1,408 |
| **合计** | **7,856** | **5,776** | **−2,080** |

三分之二的节省来自削减预算，从而缩短了超树；三分之一来自把 FORS 改造成数量更少、高度更高的树。

草案指出，在同样 2⁴⁰ 的预算下，SPHINCS+C 或 PORS+FP 还能再节省 15%，但为了保留对 FIPS 205 实现的复用和现有的安全性分析，两者均未采用。

## 5. ADRS

与 SLH-DSA 一样，每次哈希调用都必须是唯一的，使得某一位置上的输入无法在另一位置上重放而得到相同的输出。SHRINCS 的 ADRS 为 **22 字节**，每条路径各有一种解释：

| 无状态 ADRS | 大小 | | 有状态 ADRS | 大小 |
|---|---|---|---|---|
| `layer` | 1 B | | `node_height` | 1 B |
| `tree_address` | 8 B | | `node_index` | 8 B |
| `type` | 1 B | | `type` | 1 B |
| `payload` | 12 B | | `payload` | 12 B |

`type` 的取值把两条路径完全分开，数值上没有任何重叠：

| 无状态 | 取值 | | 有状态 | 取值 |
|---|---|---|---|---|
| `SL_WOTS_TW_HASH` | 0 | | `SF_WOTS_C_HASH` | 16 |
| `SL_WOTS_TW_PK` | 1 | | `SF_WOTS_C_PK` | 17 |
| `SL_XMSS_TREE` | 2 | | `SF_FXMSS_TREE` | 18 |
| `SL_FORS_TREE` | 3 | | `SF_WOTS_C_PRF` | 21 |
| `SL_FORS_ROOTS` | 4 | | `SF_WOTS_C_GRIND` | 22 |
| `SL_WOTS_TW_PRF` | 5 | | | |
| `SL_FORS_PRF` | 6 | | | |

在 `SF_WOTS_C_PRF` 下，payload 的前两个字节存放 `sf_structure`。树的形状正是在这里进入秘密钥派生的；参见第 8 节。

## 6. 哈希函数

所有哈希函数都由 SHA-256 构建，采用统一的形式：

```
sha256(pk_seed ‖ zeros(48) ‖ ADRS ‖ M)[:16]
```

`zeros(48)` 填充把 `pk_seed`（16 字节）补足到 64 字节，恰好是一个 SHA-256 分组，因此对该分组的压缩可以预先计算一次并重复使用。

### 可调哈希

| 函数 | 用途 | 路径 |
|---|---|---|
| `F` | 哈希链的一步 | 两者 |
| `H` | 合并两个 Merkle 子节点 | 两者 |
| `T_sl` | 把 35 个 WOTS-TW 链端压缩为公钥 | 无状态 |
| `T_sf` | 把 32 个 WOTS+C 链端压缩为公钥 | 有状态 |
| `T_k` | 压缩 k 个 FORS 根 | 无状态 |
| `H_grind` | WOTS+C 常数和碾磨 | 有状态 |

`H_grind` 是唯一的例外，它只使用 `ADRS[:10]`：

```
sha256(pk_seed ‖ zeros(48) ‖ ADRS[:10] ‖ digest ‖ zeros(4) ‖ counter)[:16]
```

这使整个输入保持在单次压缩之内，这一点很重要，因为碾磨最多要运行 2¹⁶ 次。由此带来的一个结果是：此时签名者在 `ADRS[10:12]` 中保存着 `sf_structure`，而验证者让这些字节保持为零；由于它们落在被使用的窗口之外，双方计算出的值相同。

### 伪随机函数

| 函数 | 定义 | 作用 |
|---|---|---|
| `PRF(pk_seed, sk_seed, ADRS)` | `sha256(pk_seed ‖ zeros(48) ‖ ADRS ‖ sk_seed)[:16]` | 派生 WOTS 和 FORS 的秘密钥。**使用完整的 22 字节 ADRS** |
| `PRF_msg_sl` / `PRF_msg_sf` | HMAC-SHA256 | 派生消息随机化值 R |

### 消息摘要与交叉绑定

两个消息摘要函数在结构上是对称的：**各自以自己的根作为专门参数，并在 `M` 中接收另一条路径的根**。草案参考实现中的两次调用如下：

```py
# stateless: what is actually signed is sf_root ‖ message
#   M = 0x00 ‖ len(ctx) ‖ ctx ‖ sf_root ‖ message
H_msg_sl(R, pk_seed, sl_root, M)

# stateful:
#   M = 0x00 ‖ len(ctx) ‖ ctx ‖ sl_root ‖ message
H_msg_sf(R, pk_seed, sf_root, ADRS, M)
```

[![两个消息摘要的字节布局，显示各自以自己的根作为参数、并在 M 中包含另一条路径的根](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/cross-binding.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/cross-binding.png)

因此，两个组件是相互咬合的。任何一个根都不能从公钥中单独取出、作为独立的密钥使用，一条路径下的签名也不能转移到另一个公钥上。

## 7. WOTS+C：以常数和编码代替校验和

### Winternitz 签名简述

这里的两种一次性签名都是 Winternitz 签名，因此在讨论变体之前，有必要先看一下基础构造。

**链**是对一个秘密反复应用哈希所得的序列：`s`、`H(s)`、`H²(s)`，依此类推。公布第 `d` 个链节即对数字 `d` 签名，因为验证者可以沿链向前走到末端，却无法在不对 `H` 求逆的情况下往回走。链的末端 `H^(w−1)(s)` 是公开值。设 `w` 为 Winternitz 参数，一条链承载 `lg w` 比特；对摘要签名的方法是把它切分成若干数字，并为每个数字分配一条链。

仅凭这些，签名是可伪造的。持有 `Hᴰ(s)` 的敌手可以继续哈希，得到任意 `D′ > D` 的 `Hᴰ′(s)`，这对每个更大的数字都是有效签名。补救办法是引入第二类链，承载数字之和的补数，使得提高一个消息数字就会降低一个校验和数字，而向上伪造就意味着要沿校验和链*往回*走。

[![w = 8 时的 WOTS+：密钥生成、对数字 2 签名、验证，以及为何需要校验和链](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/wots-plus.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/wots-plus.png)

这就是 WOTS+；WOTS-TW 是同一构造，只是把链函数换成了可调哈希，这使每条链中的每个位置都成为一个不同的函数，也正是这一点给了 SPHINCS+ 紧致的安全性证明。上面的一切对它都成立，在 `w = 16` 时，32 条消息链之上附加三条校验和链。

WOTS+C 走的是另一条路，目的相同。它不添加校验和，而是只接受各链索引之和已经等于某个常数的摘要。

```
requirement: Σ indexes == WOTS_C_CONSTANT_SUM = 240
```

在单条链上，签名值 σᵢ 是从秘密钥出发沿链走 dᵢ 步所得的中间结果。验证者只能继续向前：


共有 32 条链，每条 4 比特，取值范围 0–15，索引和的期望值恰好为 32×15/2 = 240。签名者对一个 16 比特计数器进行**碾磨**，直到索引和恰好落在这个值上：

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

计数器与索引一并返回，因为签名要携带它：验证者无法搜索它，只能根据给定的值重新计算一次。

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

两段代码均原样取自 [`AppliedPQC/shrincs-rs`](https://github.com/AppliedPQC/shrincs-rs)。`wots_c_grind` 最后一行的 `None` 就是草案认为概率低于 2⁻¹⁴⁵⁰ 的那种情形；`wots_c_map_digest` 返回的 `None` 则是验证者拒绝一个索引和不等于目标值的计数器，正是这项检查使该编码具有约束性。

[![WOTS+C 的三个部分：没有校验和链的密钥生成，碾磨计数器直到 32 个数字之和为 240，以及对每条消息链运算量都是相同 240 步的验证者](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/wots-c.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/wots-c.png)

碾磨失败的概率低于 2⁻¹⁴⁵⁰，这正是草案选取参数时所针对的目标。验证者根据签名中的计数器重新计算一次，如果和不等于 240 就拒绝。

由此得出两个结果：

1. **少三条链。** 32 条对比 WOTS-TW 的 35 条，每个签名节省 48 字节，约 8%。
2. **最坏情况验证开销等于平均开销。** 由于索引和固定，验证者走过的链步总数也固定：Σ(15 − dᵢ) = 32×15 − 240 = 240，与签名者执行的步数相同。因此共识验证者可以精确地为最坏情况定价。

第二条论断值得去检验而不是凭信任接受，而它也确实经过了检验。Rust 实现中的 `verify_parameter_consistency` 从定义方程重新推导它，而不是简单复述；已知答案测试在其覆盖的每个签名上都检验了它。用规范自带的 `H_grind` 和 `base_2b` 对 300 条消息运行，平均碾磨 64.4 个计数器，而双方的链步数每次都是同一个值 240。

> 同样的思路，即 Target-Sum Winternitz（目标和 Winternitz），也出现在 Ethereum 的 LeanSig 中，但动机不同。LeanSig 需要它，是因为聚合电路的规模按最坏情况确定；SHRINCS 需要它，是因为共识必须为最坏情况下的验证定价。两者有一位共同作者 Mikhail Kudinov。

## 8. FXMSS：形状由签名者选择的 XMSS

### XMSS 简述

一次性密钥只能签名一次，因此需要多次签名的方案就需要许多一次性密钥，以及一种把它们全部作为单个值公布的方法。XMSS 是标准答案：把一次性公钥的哈希放在 Merkle 树的叶子上，然后公布根。

于是签名携带三样东西：一次性签名本身；生成它的叶子的**索引**；以及**认证路径**，即从该叶子到根的每一层各一个兄弟节点。验证者从一次性签名恢复出叶子，然后逐层向上，依次与每个兄弟节点合并，索引告诉它在哪一侧合并。到达所公布的根，就是验证的含义。

[![高度为 3 的 XMSS，用叶子 2 签名：树、认证路径、签名携带的内容，以及验证者的逐层向上](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/xmss.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/xmss.png)

每片叶子只能使用一次，这使 XMSS 成为有状态的：签名者必须记录哪些叶子已被用掉。这正是 FXMSS 继承下来、由第 9 节的状态计数器管理的约束，也是该方案之所以要带一个无状态回退的原因。

### 与 XMSS 的区别

| | XMSS，位于无状态路径内部 | **FXMSS**，有状态路径 |
|---|---|---|
| 树 | 始终平衡，固定高度 `h' = 9` | **任意形状**；叶子可以位于不同深度 |
| 叶子 OTS | WOTS-TW，35 条链 | **WOTS+C，32 条链** |
| 签名对象 | SLH-DSA 内部：**签名者自己生成的可信消息** | **本身就是一个独立方案，直接对不可信消息签名** |

最后一行是规范所强调的一项安全要求，也是二者接口不同的原因。

### 假想高度 255

根固定在高度 255，因此 `depth = 255 − height`。只有在最大深度 255 处，叶子才位于高度 0。规范称 255 为假想高度，因为没有任何真实的树能填满 2²⁵⁶ 个节点。

由此得出三个性质：

1. 叶子的 `height` 占**一个字节**，由签名携带，并直接给出验证者所需的深度。
2. 认证路径必须**恰好**为 `depth × 16` 字节，这与所声明的高度相互校验。
3. 255 在有状态路径上从不出现，因此可以腾出来作为无状态签名的标签。

### 签名格式与大小

```
FXMSS signature = 2 (grind counter) + 512 (32 chains × 16 B) + 16 × depth
                = 514 + 16·depth

SHRINCS stateful signature = 1 (leaf_height) + 16 (R)
                           + ⌈min(depth,64)/8⌉ (leaf_index) + FXMSS signature
```

- depth = 1：1+16+1+530 = **548 B**，最小值
- depth = 255：1+16+8+4,594 = **4,619 B**，最大值

每增加一层增加 16 字节，而当深度越过 8、16，依此类推直到 64 时，索引字段各增加一个字节。这就是草案中“每个签名 16 或 17 字节”这一数字的来源。深度超过 64 之后，索引保持 8 字节，增长稳定为每层 16 字节。

### 两种规定的形状

形状由秘密钥中的两个 `(shape, depth)` 字节描述，叶子则由状态计数器选择。

**UXMSS，左倾且不平衡，`shape = 0`：**

```
ctr <  depth → (index=1, height=255−1−ctr)   ⇒ leaf_depth = ctr+1
ctr == depth → (index=0, height=255−depth)   ⇒ leaf_depth = depth
```

预算为 **depth+1**，各叶子深度依次为 1, 2, …, d, d，最后两个相等。早期的签名非常小，后面的签名线性增长。主干沿左侧下降（`index = 0`），并在每一层向右挂一片叶子（`index = 1`）；在最底层，两个子节点都是叶子。

**BXMSS，平衡，`shape = 1`：**

```
ctr < 2^depth → (index=ctr, height=255−depth)
```

预算为 **2^depth**，每个签名长度相同：所有叶子位于同一层，因此认证路径总是 `depth` 个兄弟节点。

[![UXMSS 与 BXMSS 并列，标出每片叶子处的签名大小，并附预算与开销表](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/fxmss-shapes.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/fxmss-shapes.png)

圆圈是内部节点，方框是 WOTS+C 叶子，每片叶子上标有它所产生的签名大小。


| 配置 | 签名大小 | 预算 | 密钥生成（SHA-256 压缩次数） | 平均签名开销 |
|---|---|---|---|---|
| UXMSS d=255 | 548 → 4,619 B | 256 | 425,982 | 133,326 |
| BXMSS d=5 | 612 B | 32 | 309,054 | 16,522 |
| BXMSS d=8 | 660 B | 256 | 425,982 | 133,447 |
| BXMSS d=10 | 693 B | 1,024 | 826,878 | 534,341 |
| BXMSS d=20 | 854 B | 2²⁰ | 547,649,022 | 547,356,475 |

d=255 的 UXMSS 与 d=8 的 BXMSS 生成开销相同，二者都有 256 片叶子。在给定预算下，形状不改变开销，改变的是大小曲线。UXMSS 把小签名集中在前面，BXMSS 则让所有签名的大小保持一致。

### 与形状无关的验证

验证者从不看到 `shape`。它读取签名的第一个字节，推出 `depth`，然后向上爬升相应的层数。下图按照与上面各图相同的三个部分阐述整个方案，并在一棵深度为 4、状态计数器为 2 的 UXMSS 树上完整演示了这一爬升过程：

[![FXMSS 的三个部分：签名者选择形状，状态计数器选择叶子，以及仅凭深度向上爬升的验证者](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/fxmss.png)](https://raw.githubusercontent.com/AppliedPQC/pqc-research/main/figures/shrincs-construction/fxmss.png)

同样取自草案参考实现的 `fxmss_pubkey_from_sig` 只接收 `(leaf_index, leaf_height, signature)`：

```py
assert len(xmss_auth) == leaf_depth * 16          # path length must match the depth
assert leaf_index < 2 ** min(64, leaf_depth)      # index must fit at that depth
node = wots_c_pubkey_from_sig(...)
for k in range(leaf_depth):                        # direction comes from the index bits
    if (leaf_index >> k) & 1: node = H(..., sibling + node)
    else:                     node = H(..., node + sibling)
```

其中没有引用任何形状字节。无论树是 UXMSS、BXMSS 还是其他形状，验证者都只有一条代码路径，这限定了共识代码需要审查和优化的范围。

“一条代码路径覆盖所有形状”这一论断同样经过了测试，而不只是断言。该 crate 的测试向量覆盖四种树形状下的 46 个有状态签名，即 UXMSS 和 BXMSS 各取两种深度，每一个都通过同一个从不接收形状字节的 `fxmss_pubkey_from_sig` 验证成功。这些向量所含的大小正是上面两幅图中的数值，由草案的参考实现生成，而非从中抄录。

### 形状在何处绑定

形状被绑定到秘密钥派生中，而不是绑定到验证中：

- `fxmss_sign` 把 `sf_structure` 放入 `ADRS[10:12]`；
- `wots_c_sign` 和 `wots_c_pubkey_gen` 在调用 `PRF` 时使用它，而 `PRF` 使用**完整的 22 字节 ADRS**，因此每个链秘密都依赖于形状；
- 随后 `ADRS[10:14]` 被清零，链迭代和 `T_sf` 都在清零之后运行，因此验证者不需要形状。

其结果是，同一个种子在不同的 `(shape, depth)` 取值下会产生完全不同的秘密、叶子和根。这就是域分离：导入密钥时若形状字节有误，不会与原树的秘密发生碰撞，而是产生一个不可用的密钥。

### 有向树

规范明确反对 Taproot 所采用的那种无方向 Merkle 树。如果没有左右之分，持有同一层两个节点的敌手可以同时针对这两个目标进行原像搜索，使多目标优势翻倍，而且该优势随节点数量增长。因此方向由 `(leaf_index >> k) & 1` 显式给出。

## 9. 状态管理

有状态路径通过**状态计数器**（即已签发签名的数量）选择叶子。重复使用某个计数器值，就会在同一个 WOTS+C 密钥下对两条不同消息签名，任何观察到这两个签名的人都能进行伪造。

草案中 MUST 级别的规则：

- 计数器不得备份或恢复，不得导出或导入，也不得并发使用；
- 必须在返回签名**之前**将其持久化；
- 当状态不确定时，必须拒绝使用有状态路径，转而回退到无状态路径。

有两项缓解措施使其区别于 XMSS 和 LMS：

1. **每个密钥对在构造上都带有无状态回退。** 丢失状态的代价是签名大小（从 548 字节增至 5,777 字节），而不是资金。
2. **叶子重复使用可被公开检测。** 每个有状态签名都暴露其 `(index, height)`，因此同一公钥下的重复叶子可以在第二笔交易广播之前被发现。

## 10. 大小与验证开销

| 项目 | 大小 |
|---|---|
| 公钥 | 48 B |
| 有状态签名 | 548 – 4,619 B |
| 无状态签名 | 5,777 B |
| **公钥加签名，最小值** | **596 B** |

作为对比，SLH-DSA-SHA2-128s 为 32 + 7,856 = 7,888 B，是其 **13.23** 倍；ML-DSA-44 为 1,312 + 2,420 = 3,732 B，是其 **6.26** 倍，不过 ML-DSA-44 属于 Level 2，而 SHRINCS 属于 Level 1。

### 验证开销

| | 压缩次数 | 每字节 |
|---|---|---|
| 有状态 | 255 – 509 | **0.465** |
| 无状态 | 467 – 2,792 | **0.483** |
| BIP-340 Schnorr | 约 127 次等效 | **1.98** |

按每字节计，这比 Schnorr 快 4 到 16 倍。有状态参数经过调整，使两条路径的每字节开销彼此接近，从而一条见证折扣规则即可同时覆盖二者。草案据此主张为见证折扣留出空间，但并未给出具体定义。

开销落在别处：密钥生成需要 3.1×10⁵ 到 5.5×10⁸ 次压缩，无状态签名平均需要 1.7×10⁶ 次。

## 11. 与 SLH-DSA 的区别

| | SLH-DSA-SHA2-128s | SHRINCS |
|---|---|---|
| 状态 | 无状态 | **半有状态**：有状态主路径加无状态回退 |
| 公钥 | 32 B | 48 B，增加了 `sf_root` |
| 签名 | 7,856 B | 主路径 548–4,619 B，回退 5,777 B |
| 一次性签名 | WOTS-TW，35 条链，校验和 | **WOTS+C，32 条链，常数和**，回退路径中另有 WOTS-TW |
| 树结构 | 固定超树，`d=7, h'=9` | **形状可变的 FXMSS**，回退路径中另有 `d=5, h'=9` 的超树 |
| 签名预算 | 2⁶⁴ | 回退 2⁴⁰；主路径 32–2²⁰，取决于形状 |
| 验证开销 | 取决于签名者的选择 | 主路径上**最坏情况等于平均情况** |
| 安全级别 | Level 1 | Level 1：经典 128 比特，量子 64 比特 |
| 底层哈希 | SHA-2 或 SHA-3 | **仅 SHA-256**，复用 Bitcoin 共识中已有的哈希 |
| 标准化 | **FIPS 205** | **草案**，未提交到 BIPs 仓库，没有安全性证明 |

## 12. 该方案不提供的性质

草案为自己列出的第三个缺点：

> SHRINCS 缺乏任何代数结构，无法支持**公钥再随机化**、**多重签名方案（如 MuSig）**等。

不存在对应 BIP-32 xpub 的机制，也没有 MuSig 式的多重签名。这是哈希签名的共同特点，它们没有可用于聚合的代数结构。因此，对于托管依赖 n-of-n MuSig2 的协议（例如 BitVM2 peg），后量子替代方案并不在更换输出类型的范围之内。

## 13. 开放问题

这份草案是初稿，它自己也这样说明。尚待完成的工作分为三类，难度并不相同。

**缺失的产物，属于工作量而非研究问题。** 目前没有安全性证明；草案将其记为 TODO，并感谢 Andreas Hülsing 就此提供的意见。没有完整的测试向量，而草案说明，要把该提案从 Draft 推进到 Complete，这些是必需的。没有单元测试，也没有优化实现，参考实现被其作者自己描述为朴素、低效且非常数时间的。这些都是已知的、范围明确的，只是尚未完成。

**属于 Bitcoin 而非方案本身的部署问题。** 草案规定的密码学与共识解耦，因此至少还需要一份定义新输出类型的 BIP，而它尚未撰写。每字节验证开销支持给予见证折扣，草案提出了这一论点，但没有给出定义。参数集是大小与速度之间的多维权衡，工作组预计会有争议；两个组件可以独立地重新参数化，因此可选空间大于单一选择。

**缺乏代数结构所带来的问题。** 第 12 节陈述了这一缺失本身；悬而未决的是如何应对。与 BIP-340 的混合通过简单拼接即可实现，而草案不打算定义专门的组合器，因此希望在部分密钥泄露下仍具有强不可伪造性的部署需要自行设计这一点。对于没有可再随机化公钥的方案，能否构建 HD 钱包式的安排，是草案提出但留待解决的问题。

除此之外，状态计数器仍是最危险的环节。参考实现完全不做状态管理，而是从调用者处获取计数器，这坦率地表明了困难所在：第 9 节的规则是 MUST 级别的，而方案本身无法强制执行。钱包如何呈现一个随密钥使用而增大的签名，以及在禁止恢复状态的情况下备份和恢复如何进行，这些都是草案提出但没有回答的用户体验问题。

## 14. 在 BIP 流程中的位置

BIP-361 的 `Requires` 字段写着“TBD Post Quantum Signature BIP”。SHRINCS 是这份缺失文档的第一个公开候选，而它尚未提交到 BIPs 仓库，因此 BIP-361 本身仍是 BIPs 索引中唯一的后量子条目。

## 参考实现

整个方案已用 Rust 实现于 [`AppliedPQC/shrincs-rs`](https://github.com/AppliedPQC/shrincs-rs)，遵循草案自身的结构。它对照两个权威来源进行检验，而不是自我检验：一是草案附带的参考实现，它确定了测试向量，本实现还与之进行双向交叉验证；二是 NIST 的 ACVP 向量，无状态组件在实例化为标准 SLH-DSA 参数集后可以复现这些向量。第二项很重要，因为草案的核心复用论断是该组件就是 FIPS 205，而来自草案的向量无法检验这一论断。

编写实现的过程揭示了本文此前未明确说明的一个行为。耗尽有状态预算并不是错误：`shrincs_sign` 找不到叶子，于是走与根本未提供计数器时相同的分支，因此签名由无状态组件以 5,777 字节返回，并能通过验证。叶子用尽与状态文件丢失的降级方式完全相同。

## 附录：工作组

Mike Casey（OpenChain）、Conduition（Brink）、Ethan Heilman（Cloudflare，BIP-347 和 BIP-360 的作者）、Mikhail Kudinov（Blockstream）、Oleksandr Kurbatov（Blockstream）、Boris Nagaev（独立）、Jonas Nick（Blockstream）、remix7531（OpenSats）。

该方案源自 Jonas Nick 和 Mikhail Kudinov 早先的提案。Kudinov 也是 Ethereum LeanSig 的作者之一，这就是两条路线都采用 Target-Sum（即常数和）Winternitz 编码的原因。
