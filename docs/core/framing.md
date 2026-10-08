# 核心层 —— 帧布局

规范条款。定义类型化输出的结构，以及构建与解析它的精确规则。

## 1. 帧

一个**帧**是这样形状的一个交易输出锁定脚本：

```
<marker>  OP_DROP
<field₁>  OP_DROP
<field₂>  OP_DROP
……
<fieldₙ>  OP_DROP
<owner>   OP_CHECKSIG
```

其中：

- `<marker>` 是**帧类型标签** —— ASCII，例如 `BSV8:POST:1.0`。它标识类型。
- `<fieldᵢ>` 是该类型声明的第 *i* 个字段，顺序严格照声明。
- `<owner>` 是**33 字节压缩 secp256k1 公钥**。该输出可被、且仅可被持有对应私钥的人花费。

marker 自身编码了类型、主版本与次版本，用 `:` 分隔。标签命名空间见
[registry.md](registry.md)。

## 2. 不变量

一个脚本要成为格式正确的帧，三条不变量**必须**成立。

**I1 —— 字段数。** 字段压栈的数量**必须**等于该类型声明的字段数。构建器在数量不符时**必须**抛错；
解析器在数量不符时**必须**返回"不是帧"。不允许填充、不允许截断、没有可选字段。

**I2 —— 所有者后缀。** 脚本的最后三个字节**必须**是 `OP_CHECKSIG`，并且紧邻其前的那次压栈
**必须**恰好压入 33 字节。那次压栈就是所有者。不以 `<33 字节> OP_CHECKSIG` 结尾的脚本不是帧。

**I3 —— 520 字节信封。** 脚本总长**禁止**超过 520 字节。见
[encoding.md §4](encoding.md#4-共识限制)。

## 3. 解析

解析器**必须**通过尝试下述过程来识别帧，**禁止**先按 marker 字节做预筛。

```
parse(script):
    p ← 0
    marker ← readPush(script, p)          # 失败 → 不是帧
    if script[p++] ≠ OP_DROP              → 不是帧
    kind ← lookup(marker)                 # 未知标签 → 不是帧
    if kind 未定义                          → 不是帧

    fields ← []
    loop:
        data ← readPush(script, p)        # 失败 → 不是帧
        if p < len(script) and script[p] == OP_DROP:
            p ← p + 1
            fields.append(data)
            continue
        # 这次压栈后面没有 OP_DROP ⇒ 它就是所有者
        if data.length == 33
           and p < len(script) and script[p] == OP_CHECKSIG
           and p + 1 == len(script):
            if fields.length == kind.fieldCount:   # I1
                return Frame(kind, fields, data)
            return 不是帧
        return 不是帧
```

这个过程有两个性质值得单独说明，因为它们极易写错：

- **所有者靠位置识别，不靠长度。** 单看长度，33 字节的字段和 33 字节的所有者无法区分。
  区分依据是**后面没有跟 `OP_DROP`**，再加上收尾的 `OP_CHECKSIG`（I2）。
- **解析在结构上就是严格的。** 没有恢复路径。压栈被截断、多余尾字节、缺少 `OP_DROP`、
  非最短压栈、字段数与 schema 不符 —— 全部返回"不是帧"，**绝不**返回一个字段部分填充的帧。

`readPush` **必须**实现 [encoding.md §3](encoding.md#3-规范压栈编码) 的规范表，
并且**必须**强制最短性，包括 `OP_0`、`OP_1`–`OP_16`、`OP_1NEGATE` 这几个特例。

## 4. 构建

构建器**必须**：

1. 在字段列表为空或长度不符时拒绝（I1）；
2. 拒绝不是恰好 33 字节的所有者公钥；
3. 在组装出的脚本会超过 520 字节时拒绝（I3）；
4. 每次压栈都按最短形式产出（[encoding.md §3](encoding.md#3-规范压栈编码)）。

它**禁止**静默强转字段 —— 不填充、不截断、也不对手工传入的字节数组做 UTF-8 重编码。

## 5. 字段 schema

帧类型在注册表中声明：

- 它的**标签**；
- **有序字段名列表**；
- 每个字段在 [field-encodings.md](field-encodings.md) 中的**类型**。

该声明是规范条款，并发布在注册表里。实现**必须**拒绝宽度与所声明类型不符的字段（对于定宽类型）。

## 6. 签名

帧末尾的所有者检查是**花费条件**，它和**载荷签名**是两回事。两者是独立机制，应用协议可以只用其一、
或两者并用。

| | 所有者检查 | 载荷签名 |
|---|---|---|
| 位置 | 输出中的 `OP_CHECKSIG` | 帧内部的一个字段 |
| 由谁产生 | 花费这个 UTXO | 持有被指名的密钥的人 |
| 承诺了什么 | 这笔交易的输入与输出 | 应用协议定义的规范消息 |
| 断言的是 | "所有者授权了**这次花费**" | "密钥 K 的持有者断言**命题 S**" |

由于 BSV 使用 FORKID sighash，哈希类型为 `SIGHASH_ALL | SIGHASH_FORKID`（`0x41`），
而 `SIGHASH_ALL` 承诺 `hashPrevouts`、`hashSequence` **以及 `hashOutputs`**。这正是让花费签名
成为一种"认可"的原因：

> 在花费一个帧时产生的签名，承诺了**创建它的那整笔交易，包括该帧的每一个字段**。
> 一个在签名前审阅过那笔交易的签名者 thereby 就对其内容作出了担保。

需要**显式**的作者签名或运营方签名的应用协议，**必须**把该签名作为一个字段携带，
并用域分离定义它的规范消息。见
[subprotocols/README.md](../subprotocols/README.md#6-签名与域分离)。
