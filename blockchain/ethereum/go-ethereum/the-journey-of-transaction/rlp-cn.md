## 2. 交易的序列化方式

### 2.1 RLP 编码

RLP（Recursive Length Prefix）编码是以太坊生态中**最核心的序列化协议**，它的设计和使用有非常明确的原因和优势：

---

#### 2.1.1 为什么以太坊要用 RLP 编码？

##### 2.1.1.1 适用于任意嵌套、变长结构
- 区块链数据（如交易、区块、账户状态等）本质上是**嵌套的、变长的数组和二进制数据**。
- RLP 能高效、递归地编码任意复杂的嵌套结构（如交易数组、区块头、账户树等）。

##### 2.1.1.2 简单高效，易于实现
- RLP 算法极其简单，只有“长度前缀+内容”，没有复杂类型系统。
- 只需几十行代码即可实现，**跨语言实现容易**（Go、JS、Rust、Python等都有实现）。
- 没有类型信息，节省空间，适合区块链对带宽和存储的极致要求。

##### 2.1.1.3 保证跨平台、跨语言兼容
- 以太坊网络是**去中心化、异构节点**，不同实现（geth、besu、nethermind、parity、前端钱包等）都要能互相理解数据。
- RLP 作为唯一标准，保证了**所有节点、钱包、工具之间的兼容性**。

##### 2.1.1.4 适合 Merkle Trie、哈希等底层结构
- RLP 编码的输出是**确定性字节流**，适合做哈希、Merkle Trie 等底层数据结构的输入。
- 以太坊的账户树、存储树、区块头等都依赖 RLP 编码的确定性。

---

##### RLP 的优势总结

- **递归、紧凑、无类型、易实现、跨平台兼容、适合哈希和存储**
- 以太坊全生态（节点、钱包、区块链浏览器、DApp）都用 RLP 作为底层数据交换协议


#### 2.1.2 RLP 在哪里进行编码和解码

*以钱包交易的流程为例:*

1. **用户在钱包（如 MetaMask、imToken、硬件钱包、DApp 前端）发起交易：**
   - 钱包会本地组装交易结构体（from, to, value, gas, data, nonce...）。
   - 钱包会本地用用户私钥对交易进行签名。
   - 签名后，钱包会将交易**RLP 编码**成原始二进制数据（通常以 hex 字符串形式，如 `0xf86c...`）。

2. **钱包通过 RPC 的 `eth_sendRawTransaction` 把 RLP 编码的原始交易数据发送给节点：**
   - 发送内容就是 RLP 编码后的、已签名的原始交易。

3. **节点收到后：**
   - 节点会**反序列化（RLP 解码）**这笔交易，验证签名、检查合法性，然后加入交易池，等待打包进区块。

---

##### 关键点

- **钱包负责：** 组装、签名、RLP 序列化。
- **节点负责：** RLP 反序列化、校验、广播、打包。

---

#### 2.1.3 序列化和反序列化实例

##### 2.1.3.1 Viem 的 RLP 序列化

- Viem 在本地将交易结构体（nonce, gas, to, value, data, v, r, s...）组装成数组，然后用 RLP 算法编码成字节流，最后转成 hex 字符串。

###### 2.1.3.1.1 Viem 的 RLP encode 过程（伪代码）

```js
import { serializeTransaction } from 'viem';

const tx = {
  nonce: 1,
  gasPrice: 1000000000n,
  gas: 21000n,
  to: '0xabc...',
  value: 1000000000000000000n,
  data: '0x',
  v: 27,
  r: '0x...',
  s: '0x...'
};

const rawTx = serializeTransaction(tx); // 返回 RLP 编码的 hex 字符串
```

- Viem 内部会把 tx 转成数组（顺序严格），然后用 RLP 算法编码。
- RLP 编码的核心是：对每个元素，先写长度前缀，再写内容，嵌套结构递归处理。

###### 2.1.3.1.2 Viem RLP encode 的实现
- Viem 源码里有 RLP encode 实现，和 ethers.js/web3.js 类似。
- 你可以参考 [viem RLP encode 源码](https://github.com/wagmi-dev/viem/blob/main/src/utils/encoding/toRlp.ts)。
- 关键点：所有字段都要转成 Buffer/Uint8Array，顺序不能错，空值要用 0x80 表示。

##### 2.1.3.2. go-ethereum 的 RLP 反序列化

- 节点收到 rawTx（RLP 编码的 hex 字符串）后，先 hex decode 成字节流，然后用 RLP decode 算法还原为交易结构体。
- RLP decode 会递归解析长度前缀和内容，恢复出原始的字段数组，再映射到结构体字段。

###### 2.1.3.2.1 go-ethereum 的 RLP decode 过程（伪代码）

```go
import "github.com/ethereum/go-ethereum/rlp"

var tx types.Transaction
err := rlp.DecodeBytes(rawTxBytes, &tx)
```

- go-ethereum 的 `Transaction` 结构体实现了 `DecodeRLP` 方法。
- 对于 LegacyTx，直接 RLP decode；对于 EIP-2718 类型，先读 type byte，再 decode 剩下的内容。

###### 2.1.3.2.2 go-ethereum RLP decode 的实现
- 见源码 `core/types/transaction.go` 的 `DecodeRLP` 方法。
- 递归解析每个字段，自动映射到结构体。
- 处理嵌套、变长、空值等情况。


### 2.2 JSON 序列化

- 对外 RPC 接口（如 JSON-RPC）使用 JSON 格式，见 `core/types/transaction_marshalling.go`。
- 结构体 `txJSON` 定义了 JSON 字段，`MarshalJSON`/`UnmarshalJSON` 实现了序列化/反序列化。

---
