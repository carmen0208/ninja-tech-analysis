# The Transaction Structure and Serialization

## 1. Detailed Definition of the Transaction Struct

An Ethereum transaction is essentially a struct containing the following main fields:

- `Nonce`: The transaction sequence number of the sender's account, used to prevent replay attacks.
- `GasPrice` or `MaxFeePerGas`/`MaxPriorityFeePerGas` (after EIP-1559): The price per unit of gas.
- `GasLimit`: The maximum amount of gas that can be consumed by this transaction.
- `To`: The recipient address (empty when creating a contract).
- `Value`: The amount to transfer (unit: wei).
- `Data`: Input data when calling a contract; empty for regular transfers.
- `V, R, S`: Signature fields to ensure the transaction is non-repudiable.

In the go-ethereum code, the transaction struct is defined in `[core/types/transaction.go](https://github.com/ethereum/go-ethereum/blob/master/core/types/transaction.go#L55-L106)`:

```go
type Transaction struct {
    inner TxData    // Core content of the transaction (polymorphic, supports different types)
    time  time.Time // The time when first seen locally

    // Cache
    hash atomic.Pointer[common.Hash]
    size atomic.Uint64
    from atomic.Pointer[sigCache]
}
```

- The core content of the transaction is implemented by the `TxData` interface, supporting multiple transaction types (LegacyTx, DynamicFeeTx, AccessListTx, BlobTx, etc).

### 1.2 Main Transaction Types

#### LegacyTx (Legacy Transaction)

In `core/types/tx_legacy.go`:

```go
type LegacyTx struct {
    Nonce    uint64
    GasPrice *big.Int
    Gas      uint64
    To       *common.Address // nil means contract creation
    Value    *big.Int
    Data     []byte
    V, R, S  *big.Int // Signature
}
```

#### DynamicFeeTx (EIP-1559 Dynamic Fee Transaction)

In `core/types/tx_dynamic_fee.go`:

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
    V, R, S    *big.Int // Signature
}
```

#### BlobTx (EIP-4844 Data Extension Transaction)

In `core/types/tx_blob.go`:

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
    Sidecar    *BlobTxSidecar // Optional
    V, R, S    *uint256.Int // Signature
}
```

---

### 1.3. Source of the `data` Field

- The `data` field is essentially a `[]byte`, and is called `Data` in transaction structs (such as `LegacyTx`, `DynamicFeeTx`, `AccessListTx`, etc).
- **Normal Transfer**: `data` is empty (`nil` or `[]`).
- **Contract Call**: `data` is the ABI-encoded contract method and parameters (e.g., the encoding of `transfer(address,uint256)`).
- **Contract Deployment**: `data` is the contract bytecode.

---

#### 1.3.1. Encoding and Decoding of the `data` Field

##### 1.3.1.1 RLP Encoding/Decoding

- The entire transaction uses RLP encoding, and `data` is one of the fields being encoded.
- See `func (tx *LegacyTx) encode(b *bytes.Buffer) error { return rlp.Encode(b, tx) }`
- Decoding is similar: `rlp.DecodeBytes(input, tx)`.

##### 1.3.1.2 JSON Encoding/Decoding

- The RPC layer uses JSON, where the `data` field is called `input` in JSON and is of type `hexutil.Bytes`.
- See the `txJSON` struct in `core/types/transaction_marshalling.go`.
- Encoding: `enc.Input = (*hexutil.Bytes)(&itx.Data)`
- Decoding: `itx.Data = *dec.Input`

#### 1.3.2 `data` Examples

##### 1.3.2.1 Contract Creation

**Encoding Process**

- **data** = contract bytecode, e.g., `0x6080604052...`
- Encoding process: directly use the compiled contract bytecode as the data field, no ABI encoding needed.

* (A contract (usually Solidity source code) is compiled to EVM bytecode using the Solidity compiler (solc))

```sh
solc --bin MyContract.sol)
```

**In viem/ethers.js/web3.js:**
- You just need to pass `data: "0x6080604052..."`, which will be used directly as the transaction's input field.

**Decoding Process**

- When the node receives the transaction, it directly executes `data` as the contract initialization code, no ABI decode needed.
- Only contract calls require ABI decode.

The contract will be executed via [calling evm.create(...), then evm.initNewContract(...)](https://github.com/ethereum/go-ethereum/blob/master/core/vm/evm.go/#L522-L523)

---

##### 1.3.2.2. Contract Call

Take `transfer(address,uint256)` as an example, with parameters `to=0x1234...`, `amount=1000`.

**Encoding Process (in viem/ethers.js/web3.js)**

- Method signature: `transfer(address,uint256)`
- Method selector: `keccak256("transfer(address,uint256)")[:4]`, e.g., `0xa9059cbb`
- Parameter encoding (ABI encode):
  - address: left-padded to 32 bytes
  - uint256: left-padded to 32 bytes
- Concatenate: `0xa9059cbb` + address encoding + amount encoding

**Example using ethers.js:**
```js
const iface = new ethers.utils.Interface(abi);
const data = iface.encodeFunctionData("transfer", [to, amount]);
// data is the input field
```

**Decoding Process (go-ethereum)**

- When the node receives the transaction, it will use ABI to decode the input when executing the contract.
- Related code in `accounts/abi/abi.go`:
  - Use `MethodById(data[:4])` to find the method
  - Use `method.Inputs.UnpackValues(data[4:])` to decode parameters

**Pseudocode:**
```go
method, _ := abi.MethodById(data[:4])
params, _ := method.Inputs.UnpackValues(data[4:])
```

---

##### 1.3.2.3 Summary

- **Contract Creation**: data is directly the bytecode, no ABI encode/decode needed.
- **Contract Call**: data = method selector + ABI-encoded parameters
  - Encoding: ethers.js/viem uses `encodeFunctionData`, go-ethereum uses `abi.Pack`
  - Decoding: go-ethereum uses `abi.MethodById` + `method.Inputs.UnpackValues`

---

###### Key go-ethereum Methods for Reference

- **Encoding**: `accounts/abi/abi.go` → `func (abi ABI) Pack(name string, args ...interface{}) ([]byte, error)`
- **Decoding**: `accounts/abi/abi.go` → `func (abi ABI) MethodById(sigdata []byte) (*Method, error)` and `func (method Method) Inputs.UnpackValues(data []byte)`

---

If you want to see specific encode/decode code snippets or have concrete parameter examples, feel free to ask!

### 1.4 Signature Generation and Verification

#### 1.4.1 Signature Generation

- Transaction signatures use the secp256k1 elliptic curve, with signature fields `V, R, S`.
- Signature process:
  1. First, hash the transaction content (excluding the signature fields), see the `sigHash` method.
  2. Use the private key to sign the hash, resulting in a 65-byte signature (r[32] + s[32] + v[1]).
  3. Fill the signature into the `V, R, S` fields to get the complete transaction.

Code (`core/types/transaction_signing.go`):
```go
func SignTx(tx *Transaction, s Signer, prv *ecdsa.PrivateKey) (*Transaction, error) {
    h := s.Hash(tx) // Calculate the hash to be signed
    sig, err := crypto.Sign(h[:], prv) // Sign with the private key
    if err != nil {
        return nil, err
    }
    return tx.WithSignature(s, sig) // Fill in VRS
}
```

#### 1.4.2 Signature Verification

- During verification, the public key is recovered from `V, R, S`, and then the sender address is recovered.
- See `Sender(signer, tx *Transaction)`, which internally uses `crypto.Ecrecover`.


---