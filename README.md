# TypedTXTemplate

**比特币 SV 类型化交易输出（typed transaction template）规范与参考 SDK。**

每一种链上动作都有它自己的**帧类型**（frame type），携带一份公开、有序、逐字段文档化的数据结构，
使得任何独立实现都能解析、验证，并产出**逐字节相同**的输出。

本仓库**只负责格式层**。它回答"这个输出是什么意思"，不回答"它在哪里"—— 链上数据获取、索引、广播、
发现，全部明确不在范围内。

## 文档

| | |
|---|---|
| [docs/README.md](docs/README.md) | 规范总索引与范围边界 |
| [docs/core/encoding.md](docs/core/encoding.md) | 脚本编码：数据压栈惯例与共识限制 |
| [docs/core/framing.md](docs/core/framing.md) | 帧布局，以及构建与解析规则 |
| [docs/core/field-encodings.md](docs/core/field-encodings.md) | 每种字段类型的规范编码 |
| [docs/core/registry.md](docs/core/registry.md) | 标签命名空间与帧类型注册表 |
| [docs/subprotocols/README.md](docs/subprotocols/README.md) | 子协议如何在核心层之上定义 |
| [docs/subprotocols/bbs/](docs/subprotocols/bbs/) | **子协议 `bbs`** —— 论坛（询价见 ChannelProtocol）|
| [docs/conformance/](docs/conformance/) | Go ⇄ TypeScript 差分测试框架与共享测试向量 |

## 实现

两套实现消费同一份测试向量
[`docs/conformance/vectors.json`](docs/conformance/vectors.json)，并且**必须**逐字节一致。

- `go/` —— Go
- `ts/` —— TypeScript

## 一行看懂帧格式

```
<marker> OP_DROP  (<field> OP_DROP)*  <ownerPub33> OP_CHECKSIG
```

数据载荷写进锁定脚本，因此被交易 id 承诺、无法篡改；但由于每个字段都在签名检查之前被丢弃，
这个输出仍然是一个**正常可花费的 UTXO**。全项目**永不产生** `OP_RETURN`。

## 许可证

AGPL-3.0，见 [LICENSE](LICENSE)。
