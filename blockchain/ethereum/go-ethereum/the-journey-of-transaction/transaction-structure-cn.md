## 1. 交易结构体的详细定义

### 1.1 顶层结构

在 `core/types/transaction.go`：

```go
type Transaction struct {
    inner TxData    // 交易的核心内容（多态，支持不同类型）
    time  time.Time // 首次本地见到的时间

    // 缓存
    hash atomic.Pointer[common.Hash]
    size atomic.Uint64
    from atomic.Pointer[sigCache]
}
```

- 交易的核心内容由 `TxData` 接口实现，支持多种交易类型（LegacyTx、DynamicFeeTx、AccessListTx、BlobTx等）。

### 1.2 主要交易类型

#### LegacyTx（传统交易）

在 `core/types/tx_legacy.go`：

```go
type LegacyTx struct {
    Nonce    uint64
    GasPrice *big.Int
    Gas      uint64
    To       *common.Address // nil 表示创建合约
    Value    *big.Int
    Data     []byte
    V, R, S  *big.Int // 签名
}
```

#### DynamicFeeTx（EIP-1559 动态手续费交易）

在 `core/types/tx_dynamic_fee.go`：

```go
type DynamicFeeTx struct {
    ChainID    *big.Int
    Nonce      uint64
    GasTipCap  *big.Int // maxPriorityFeePerGas
    GasFeeCap  *big.Int // maxFeePerGas
    Gas        uint64
    To         *common.Address
    Value      *big.Int
    Data       []byte
    AccessList AccessList
    V, R, S    *big.Int // 签名
}
```

#### BlobTx（EIP-4844 数据扩展交易）

在 `core/types/tx_blob.go`：

```go
type BlobTx struct {
    ChainID    *uint256.Int
    Nonce      uint64
    GasTipCap  *uint256.Int
    GasFeeCap  *uint256.Int
    Gas        uint64
    To         common.Address
    Value      *uint256.Int
    Data       []byte
    AccessList AccessList
    BlobFeeCap *uint256.Int
    BlobHashes []common.Hash
    Sidecar    *BlobTxSidecar // 可选
    V, R, S    *uint256.Int // 签名
}
```

---

### 1.3. `data` 字段的来源

- `data` 字段本质上是一个 `[]byte`，在交易结构体（如 `LegacyTx`、`DynamicFeeTx`、`AccessListTx` 等）中都叫 `Data`。
- **普通转账**：`data` 为空（`nil` 或 `[]`）。
- **合约调用**：`data` 是 ABI 编码后的合约方法和参数（如 `transfer(address,uint256)` 的编码）。
- **合约部署**：`data` 是合约字节码。

---

#### 1.3.1. `data` 字段的编码和解码

##### 1.3.1.1 RLP 编码/解码

- 交易整体采用 RLP 编码，`data` 作为其中一个字段被编码。
- 见 `func (tx *LegacyTx) encode(b *bytes.Buffer) error { return rlp.Encode(b, tx) }`
- 解码时同理，`rlp.DecodeBytes(input, tx)`。

##### 1.3.1.2 JSON 编码/解码

- RPC 层用 JSON，`data` 字段在 JSON 里叫 `input`，类型为 `hexutil.Bytes`。
- 见 `core/types/transaction_marshalling.go` 的 `txJSON` 结构体。
- 编码时：`enc.Input = (*hexutil.Bytes)(&itx.Data)`
- 解码时：`itx.Data = *dec.Input`

#### 1.3.2 data 实例

##### 1.3.2.1 合约创建（Contract Creation）

**encode 过程**

- **data** = 合约字节码（bytecode），比如 `0x6080604052...`
- encode 过程：直接将合约编译出来的字节码作为 data 字段，无需 ABI 编码。

* (合约（通常是 Solidity 源码）通过 Solidity 编译器（solc）编译为 EVM 字节码（bytecode

```sh
solc --bin MyContract.sol)
```

**在 viem/ethers.js/web3.js 里：**
- 你只需传 `data: "0x6080604052..."`，它会直接作为交易的 input 字段。

**decode 过程**

- 节点收到交易后，直接将 `data` 作为合约初始化代码执行，不需要 ABI decode。
- 只有在合约调用时才需要 ABI decode。

合约会通过[调用 evm.create(...)，再到 evm.initNewContract(...)](https://github.com/ethereum/go-ethereum/blob/master/core/vm/evm.go/#L522-L523)被执行

---

##### 1.3.2.2. 合约调用（Contract Call）

以 `transfer(address,uint256)` 为例，参数为 `to=0x1234...`，`amount=1000`。

**encode 过程 (在 viem/ethers.js/web3.js 里)**

- 方法签名：`transfer(address,uint256)`
- 方法选择器：`keccak256("transfer(address,uint256)")[:4]`，假设为 `0xa9059cbb`
- 参数编码（ABI encode）：
  - address: 左填充32字节
  - uint256: 左填充32字节
- 拼接：`0xa9059cbb` + address编码 + amount编码

**以 ethers.js 为例：**
```js
const iface = new ethers.utils.Interface(abi);
const data = iface.encodeFunctionData("transfer", [to, amount]);
// data 就是 input 字段
```

**decode 过程（go-ethereum**

- 节点收到交易后，执行合约时会用 ABI 解码 input。
- 相关代码在 `accounts/abi/abi.go`:
  - 先用 `MethodById(data[:4])` 找到方法
  - 用 `method.Inputs.UnpackValues(data[4:])` 解码参数

**伪代码：**
```go
method, _ := abi.MethodById(data[:4])
params, _ := method.Inputs.UnpackValues(data[4:])
```

---

##### 1.3.2.3 总结

- **合约创建**：data 直接是字节码，无需 ABI encode/decode。
- **合约调用**：data = 方法选择器 + ABI 编码参数
  - encode：ethers.js/viem 用 `encodeFunctionData`，go-ethereum 用 `abi.Pack`
  - decode：go-ethereum 用 `abi.MethodById` + `method.Inputs.UnpackValues`

---

###### 参考 go-ethereum 关键方法

- **编码**：`accounts/abi/abi.go` → `func (abi ABI) Pack(name string, args ...interface{}) ([]byte, error)`
- **解码**：`accounts/abi/abi.go` → `func (abi ABI) MethodById(sigdata []byte) (*Method, error)` 和 `func (method Method) Inputs.UnpackValues(data []byte)`

---

如果你想看具体的 encode/decode 代码片段或有具体参数举例，可以继续问！

### 1.4 签名的生成和验证

#### 1.4.1 签名生成

- 交易签名用 secp256k1 椭圆曲线，签名字段为 `V, R, S`。
- 签名流程：
  1. 先对交易内容（不含签名字段）做哈希（见 `sigHash` 方法）。
  2. 用私钥对哈希签名，得到 65 字节的签名（r[32] + s[32] + v[1]）。
  3. 把签名填入 `V, R, S` 字段，得到完整交易。

代码（`core/types/transaction_signing.go`）：
```go
func SignTx(tx *Transaction, s Signer, prv *ecdsa.PrivateKey) (*Transaction, error) {
    h := s.Hash(tx) // 计算待签名哈希
    sig, err := crypto.Sign(h[:], prv) // 用私钥签名
    if err != nil {
        return nil, err
    }
    return tx.WithSignature(s, sig) // 填入VRS
}
```

#### 1.4.2 签名验证

- 验证时，从 `V, R, S` 恢复公钥，进而恢复发送者地址。
- 见 `Sender(signer, tx *Transaction)`，内部用 `crypto.Ecrecover`。


---

## 2. 交易的序列化方式

### 2.1 RLP 编码

- 以太坊的交易在网络和存储中主要采用 RLP（Recursive Length Prefix）编码。
- `Transaction` 实现了 `EncodeRLP` 和 `DecodeRLP` 方法。
- LegacyTx 直接 RLP 编码，EIP-2718 类型交易（如 EIP-1559、EIP-4844）采用“类型前缀+RLP编码内容”格式。

```go
func (tx *Transaction) EncodeRLP(w io.Writer) error {
    if tx.Type() == LegacyTxType {
        return rlp.Encode(w, tx.inner)
    }
    // EIP-2718 typed TX envelope
    buf := encodeBufferPool.Get().(*bytes.Buffer)
    defer encodeBufferPool.Put(buf)
    buf.Reset()
    if err := tx.encodeTyped(buf); err != nil {
        return err
    }
    return rlp.Encode(w, buf.Bytes())
}
```

### 2.2 JSON 序列化

- 对外 RPC 接口（如 JSON-RPC）使用 JSON 格式，见 `core/types/transaction_marshalling.go`。
- 结构体 `txJSON` 定义了 JSON 字段，`MarshalJSON`/`UnmarshalJSON` 实现了序列化/反序列化。

---

## 总结

- 交易结构体是多态的，支持多种类型（Legacy、EIP-1559、EIP-4844等）。
- 网络和存储用 RLP 编码，RPC 用 JSON 编码。
- 主要字段包括 nonce、gas、to、value、data、签名（v/r/s）等。

---
