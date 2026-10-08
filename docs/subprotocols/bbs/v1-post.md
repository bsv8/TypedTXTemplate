# `bbs` v1 —— 帖子规范

规范条款。定义 `BSV8:POST:1.0` 帧，以及不依赖链上状态即可判定的验证规则。

## 1. 帧

```
BSV8:POST:1.0  OP_DROP
version        OP_DROP
boardId        OP_DROP
postHash       OP_DROP
parentHash     OP_DROP
posterPub      OP_DROP
fee            OP_DROP
notAfterHeight OP_DROP
operatorSig    OP_DROP
body           OP_DROP
<posterPub>    OP_CHECKSIG
```

**所有者就是 `posterPub`。** 这一点是刻意的：它使作者的"认可"成为一个花费行为，
而由于 BSV 的 `SIGHASH_ALL` 承诺 `hashOutputs`，那次花费的签名会连带承诺本帧的每一个字段。
见 [core/framing.md §6](../../core/framing.md#6-签名) 与
[v1-lifecycle.md](v1-lifecycle.md)。

## 2. 字段

| # | 字段 | 类型 | 宽度 | 说明 |
|---|---|---|---|---|
| 0 | `version` | `u8` | 1 | 固定为 `1` |
| 1 | `boardId` | `bytes` | 16 | 板块标识，由索引方分配；`00…00` 表示默认板块 |
| 2 | `postHash` | `hash256` | 32 | **完整规范文档**的 `SHA256d`；链下投递的正文必须匹配它 |
| 3 | `parentHash` | `hash256` | 32 | 父帖的 `postHash`；主题帖为 32 个 `0x00` |
| 4 | `posterPub` | `pubkey` | 33 | 作者的压缩公钥 |
| 5 | `fee` | `u64le` | 8 | 本帖应付的索引费用，单位 satoshi；来自一次签名报价 |
| 6 | `notAfterHeight` | `u32le` | 4 | 本帖认可截止的**区块高度**；来自同一次报价 |
| 7 | `operatorSig` | `sig64` | 64 | 索引方对字段 0–6 的签名 |
| 8 | `body` | `utf8` | 0–274 | 链上预览（标题 / 摘要）；可为 0 长度 |

**主题帖与回复用同一个帧类型**，靠 `parentHash` 区分。这是刻意的：多一个帧类型要多占 12 字节
（marker 更长 + 一个长度前缀），而"回复"与"发帖"在证据结构上完全相同。

字段约束：

- `version` **必须**为 `1`；其他值解析器**必须**返回"不是帧"。
- `parentHash` 为全 0 时表示主题帖。实现**不得**把它当作"父帖哈希恰好为零"。
- `postHash` **不得**与 `parentHash` 相同（自引用）。
- `fee` **必须**大于 0。一个零费用帖子不构成"付费帖"，索引方**不得**把它计为一次消费。
- `notAfterHeight` **必须**大于帖子被打包时的高度。
- `boardId` 全 0 合法，表示默认板块。

## 3. 字节预算

```
元素                          占用字节
────────────────────────────────────────
marker "BSV8:POST:1.0"（13 B）   1+13 + 1 = 15
version      （OP_1 单字节形式）  1   + 1 =  2
boardId                        1+16 + 1 = 18
postHash                       1+32 + 1 = 34
parentHash                     1+32 + 1 = 34
posterPub                      1+33 + 1 = 35
fee                            1+ 8 + 1 = 10
notAfterHeight                 1+ 4 + 1 =  6
operatorSig                    1+64 + 1 = 66
body                           1+ N + 1 =  N+2
owner (33)                     1+33    = 34
OP_CHECKSIG                           =  1
────────────────────────────────────────
固定开销（body = 0）                   = 255 B
520 上限下 body 上限                   = 265 B ≈ 88 个中文字 / 265 个 ASCII 字符
```

**注意 `version` 那一行。** 字段值是 `0x01`，落在 `OP_1`…`OP_16` 区间内，因此按
[core/encoding.md §3](../../core/encoding.md#3-规范压栈编码) 的最短规则，它必须编码为**单字节**
`0x51`，而不是 `0x01 0x01`。这一字节之差，正是"最短性是共识要求"在字节账上的体现。

**`body` 刻意保持很小。** 完整正文由 `postHash` 承诺、存放在链下；链上的 `body` 只是预览。
这就是 `bbs` 处理 520 字节限制的方式 —— 选项一，"对它取哈希"。

## 4. 规范消息

### 4.1 签名

被签名的消息按 schema 顺序拼接字段 0–6，**不含 `body`**：

```
preimage = "bsv8/bbs/v1/operator|"          (23 B, ASCII)
         ‖ version          (1 B)
         ‖ boardId          (16 B)
         ‖ postHash         (32 B)
         ‖ parentHash       (32 B)
         ‖ posterPub        (33 B)
         ‖ fee              (8 B, 小端)
         ‖ notAfterHeight   (4 B, 小端)

msg          = SHA256d(preimage)             (32 B)
operatorSig  = Sign(operatorPriv, msg)       (64 B, 紧凑 r‖s)
```

`body` **不**被签名，因为它的作用是预览；内容完整性由 `postHash` 承担，而 `postHash` **在**签名范围内。

### 4.2 域分离前缀

`bbs` 定义两个前缀，**每个角色一个**：

| 前缀 | 角色 | 用于 |
|---|---|---|
| `bsv8/bbs/v1/operator\|` | 运营方 | `BSV8:POST:1.0` 的 `operatorSig` |
| `bsv8/bbs/v1/poster\|` | 作者 | 显式作者签名（v1 预留，v2 启用）|

**为什么只有两个。** 报价由运营方经 [ChannelProtocol](README.md#5-跨仓库边界) 的
`bsv8.bbs.quote.v1` 直接发出，那条消息的签名用的是 **CP 的 domain**
（`bsv8.public-message.v1`），**不是**这里的。报价从"链下签名消息"变成"链上被承诺的数值"
这一步，由 `operatorSig` 完成 —— 它覆盖 `fee`，所以价格不可篡改。
两个前缀因此已经足够：运营方签一次（连同价格一起），作者签一次（可选）。

## 5. 验证

以下判定**只依赖这一个输出脚本**，不查询任何链上状态，也不联系任何服务方，
因此任何实现都能独立完成：

```
verify(frame):
    f ← TxTemplates.Parse(frame.script)
    if f is not { kind: BSV8:POST:1.0, fieldCount: 9 }  → 无效
    if f.fields[0] != 0x01                                → 无效（版本）
    if f.fields[8] 不是合法 UTF-8                         → 无效
    if f.fields[3] == f.fields[2]                         → 无效（自引用）
    if f.fields[5] == 0                                   → 无效（零费用）
    if f.fields[6] == 0                                   → 无效（无有效期）
    if f.owner != f.fields[4]                              → 无效（所有者必须是作者）
    if !Verify(f.fields[7], SHA256d("bsv8/bbs/v1/operator|" ‖ fields[0..6]))
                                                          → 无效（签名）
    if 脚本总长 > 520                                      → 无效
    return 有效
```

**这一步就是"价格不可篡改"的全部含义。** 验过 `operatorSig` 就等于证明了
"运营方为这篇帖子签下了 `fee = 5`、`notAfterHeight = 3002`"。不需要反查任何价格表，
不需要知道当时"市价"是多少。

**明确不在此判定内的**（需要链上状态，属于索引服务）：

- `notAfterHeight` 是否已经过期 —— 需要这笔认可交易被打包的高度。
- `fee` 是否真的付了 —— 需要检查付给板块注册地址的输出。
- 本帧是否已被花费 —— 需要反向索引。
- `postHash` 对应的链下正文是否可获得、是否匹配。

## 6. 示例

`postHash` 取一个全 `0xAB` 的占位摘要，`boardId` 取 8 个 `0x00` + 8 个 `0x11`，
`fee = 5`，`notAfterHeight = 3002`，`body = "标题：费用与证据链"`（UTF-8，21 字节）。

字段字节：

```
version        01
boardId        00000000000000001111111111111111
postHash       abababab…（32 字节）
parentHash     00…00（32 字节，主题帖）
posterPub      02a1b2…（33 字节）
fee            0500000000000000
notAfterHeight b2 0b 00 00        （3002 = 0x00000BB2 小端）
operatorSig    r‖s（64 字节）
body           UTF-8
```

由 `BBS POST:1.0` 生成的脚本长度 = 255 + 1 + 27 + 1 = **284 字节**（上限 265，预留 12 字节余量）。

完整字节向量见 [`conformance/vectors.json`](../../conformance/vectors.json)。

## 7. 非规范性说明

**不要把密钥或口令的哈希放进 `postHash`。** 链上数据是永久、全球复制、无法删除的。
发布 `SHA256(seed)` 等于永久公开一个"此人拥有该 seed"的验证器：如果 seed 由低熵来源派生
（口令、助记词），这个哈希就是一个公开且永久的口令检查器，可被离线爆破，且**你无法撤回**。
`postHash` 应当承诺一份**文档**，而不是一份秘密。

**`boardId` 不是访问控制。** 它是 16 个字节的路由标签，任何人都能构造任意 `boardId` 的帧。
私密板块必须靠正文加密来实现，而不是靠 `boardId`。
