公钥解析 ip
## 1. 帧

```
BSV8:PR:1.0  OP_DROP
PublickeyHex        OP_DROP
MultipleAddresses       OP_DROP
last区块高度 OP_DROP
operatorSig    OP_DROP
nameserverSig  OP_DROP
<posterPub>    OP_CHECKSIG
```

- BSV8:PR:1.0 标识和版本号一起
- PublickeyHex 要解析的公钥
- MultipleAddresses  多地址编码
- operatorSig 针对 PublickeyHex对应私钥对 
{
    version: BSV8:PR:1.0
    PublickeyHex: xxx
    MultipleAddresses: yyy
} 
的签名
- nameserverSig nameserver 对解析服务费认可的签名
{
    version: BSV8:PR:1.0
    PublickeyHex: xxx
    MultipleAddresses: yyy
    price: zzz
} 
因为 nameserver 价格是变动的，所以需要签名确认
