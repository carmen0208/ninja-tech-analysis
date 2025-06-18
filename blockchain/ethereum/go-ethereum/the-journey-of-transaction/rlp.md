## 2. Transaction Serialization Methods

### 2.1 RLP Encoding

RLP (Recursive Length Prefix) encoding is the **core serialization protocol** in the Ethereum ecosystem, designed and used for very clear reasons and advantages:

---

#### 2.1.1 Why Does Ethereum Use RLP Encoding?

##### 2.1.1.1 Suitable for Arbitrary Nested, Variable-Length Structures
- Blockchain data (such as transactions, blocks, account states, etc.) are essentially **nested, variable-length arrays and binary data**.
- RLP can efficiently and recursively encode arbitrarily complex nested structures (such as transaction arrays, block headers, account trees, etc.).

##### 2.1.1.2 Simple, Efficient, and Easy to Implement
- The RLP algorithm is extremely simple, consisting only of "length prefix + content," with no complex type system.
- It can be implemented in just a few dozen lines of code, making **cross-language implementation easy** (Go, JS, Rust, Python, etc. all have implementations).
- There is no type information, saving space, which is suitable for blockchain's extreme requirements for bandwidth and storage.

##### 2.1.1.3 Ensures Cross-Platform and Cross-Language Compatibility
- The Ethereum network is **decentralized and heterogeneous**, and different implementations (geth, besu, nethermind, parity, frontend wallets, etc.) must be able to understand each other's data.
- RLP, as the sole standard, ensures **compatibility among all nodes, wallets, and tools**.

##### 2.1.1.4 Suitable for Merkle Trie, Hashing, and Other Underlying Structures
- The output of RLP encoding is a **deterministic byte stream**, suitable as input for hashing, Merkle Trie, and other underlying data structures.
- Ethereum's account tree, storage tree, block header, etc., all rely on the determinism of RLP encoding.

---

##### Summary of RLP Advantages

- **Recursive, compact, typeless, easy to implement, cross-platform compatible, suitable for hashing and storage**
- The entire Ethereum ecosystem (nodes, wallets, block explorers, DApps) uses RLP as the underlying data exchange protocol


#### 2.1.2 Where Does RLP Encoding and Decoding Occur?

*Taking the wallet transaction process as an example:*

1. **User initiates a transaction in a wallet (such as MetaMask, imToken, hardware wallet, DApp frontend):**
   - The wallet locally assembles the transaction struct (from, to, value, gas, data, nonce, etc.).
   - The wallet locally signs the transaction with the user's private key.
   - After signing, the wallet **RLP encodes** the transaction into raw binary data (usually as a hex string, e.g., `0xf86c...`).

2. **The wallet sends the RLP-encoded raw transaction data to the node via the RPC method `eth_sendRawTransaction`:**
   - The content sent is the RLP-encoded, signed raw transaction.

3. **After the node receives it:**
   - The node **deserializes (RLP decodes)** the transaction, verifies the signature, checks its validity, then adds it to the transaction pool, waiting to be included in a block.

---

##### Key Points

- **Wallet is responsible for:** assembling, signing, and RLP serialization.
- **Node is responsible for:** RLP deserialization, validation, broadcasting, and packaging.

---

#### 2.1.3 Serialization and Deserialization Examples

##### 2.1.3.1 Viem's RLP Serialization

- Viem locally assembles the transaction struct (nonce, gas, to, value, data, v, r, s, etc.) into an array, then encodes it into a byte stream using the RLP algorithm, and finally converts it to a hex string.

###### 2.1.3.1.1 Viem's RLP Encode Process (Pseudocode)

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

const rawTx = serializeTransaction(tx); // Returns RLP-encoded hex string
```

- Internally, Viem converts tx into an array (with strict order), then encodes it using the RLP algorithm.
- The core of RLP encoding is: for each element, write the length prefix first, then the content, and handle nested structures recursively.

###### 2.1.3.1.2 Viem RLP Encode Implementation
- Viem's source code contains the RLP encode implementation, similar to ethers.js/web3.js.
- You can refer to [Viem RLP encode source code](https://github.com/wagmi-dev/viem/blob/main/src/utils/encoding/toRlp.ts).
- Key points: all fields must be converted to Buffer/Uint8Array, the order must not be wrong, and empty values must be represented as 0x80.

##### 2.1.3.2 go-ethereum's RLP Deserialization

- After the node receives rawTx (RLP-encoded hex string), it first hex decodes it into a byte stream, then uses the RLP decode algorithm to restore it to a transaction struct.
- RLP decode recursively parses the length prefix and content, restores the original field array, and then maps it to struct fields.

###### 2.1.3.2.1 go-ethereum's RLP Decode Process (Pseudocode)

```go
import "github.com/ethereum/go-ethereum/rlp"

var tx types.Transaction
err := rlp.DecodeBytes(rawTxBytes, &tx)
```

- go-ethereum's `Transaction` struct implements the `DecodeRLP` method.
- For LegacyTx, it directly RLP decodes; for EIP-2718 types, it first reads the type byte, then decodes the remaining content.

###### 2.1.3.2.2 go-ethereum RLP Decode Implementation
- See the source code for the `DecodeRLP` method in `core/types/transaction.go`.
- Recursively parses each field and automatically maps them to the struct.
- Handles nesting, variable length, empty values, etc.


### 2.2 JSON Serialization

- External RPC interfaces (such as JSON-RPC) use JSON format, see `core/types/transaction_marshalling.go`.
- The struct `txJSON` defines the JSON fields, and `MarshalJSON`/`UnmarshalJSON` implement serialization/deserialization.

--- 