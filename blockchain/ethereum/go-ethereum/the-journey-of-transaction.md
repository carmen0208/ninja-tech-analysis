# The Journey of the Transaction
## 1. What is the format of a Transaction?
An Ethereum transaction is essentially a struct containing the following main fields:

- `Nonce`: The transaction sequence number of the sender's account, used to prevent replay attacks.
- `GasPrice` or `MaxFeePerGas`/`MaxPriorityFeePerGas` (after EIP-1559): The price per unit of gas.
- `GasLimit`: The maximum amount of gas this transaction can consume.
- `To`: The recipient address (empty when creating a contract).
- `Value`: The amount to transfer (in wei).
- `Data`: Input data for contract calls, empty for simple transfers.
- `V, R, S`: Signature fields to ensure non-repudiation of the transaction.

In the go-ethereum codebase, the transaction struct is defined in [core/types/transaction.go](https://github.com/ethereum/go-ethereum/blob/master/core/types/transaction.go#L55-L106):

```go
type Transaction struct {
    // ... existing code ...
    inner TxData    // Consensus contents of a transaction
    // ... existing code ...
}
```
The `txdata` contains the above fields.

[Detailed introduction](./the-journey-of-transaction/transaction-structure.md)
---

## 2. How is a transaction submitted to a node?

Users typically submit transactions in two ways:

- Via JSON-RPC interface (such as `eth_sendRawTransaction` or `eth_sendTransaction`)
- Via the P2P network (transactions broadcasted from other nodes)

### 2.1 JSON-RPC Submission

- The user signs the transaction with their private key, obtaining the RLP-encoded raw transaction data (raw tx).
- The raw tx is sent to the node via HTTP/WebSocket/IPC using `eth_sendRawTransaction`.

In go-ethereum, the RPC entry point is in `eth/api.go`:

```go
func (api *PublicTransactionPoolAPI) SendRawTransaction(ctx context.Context, rawTx hexutil.Bytes) (common.Hash, error)
```

- This method decodes the rawTx, verifies the signature, and then calls `txpool.AddLocal(tx)` to add the transaction to the local transaction pool.

- **eth_sendRawTransaction**: You send the RLP-encoded raw transaction (already signed).
- **eth_sendTransaction**: You send structured JSON, and the node signs and encodes it as RLP for you.

---

#### 2.1.1 eth_sendRawTransaction

- **You must sign the transaction locally with your private key** to obtain the RLP-encoded raw transaction (usually a hex string, e.g., `0xf86c808504a817c80082520894...`).
- You send it via RPC:
  ```json
  {
    "method": "eth_sendRawTransaction",
    "params": ["0xf86c808504a817c80082520894..."], // This is the RLP-encoded raw transaction
    ...
  }
  ```
- The node receives it, decodes the RLP, verifies the signature, and adds it to the transaction pool.

#### 2.1.2 eth_sendTransaction

- You send structured JSON fields (such as from, to, value, data, gas, gasPrice, etc.), **without a signature**.
- The node uses its local wallet (keystore) to find the private key for the from address, signs the transaction, encodes it as RLP, and then broadcasts it.
- This method only works if the node has the account's private key locally.

---

#### Code Reference

- The `SendRawTransaction` method in `eth/api.go` first decodes the RLP, then processes it.
- The `SendTransaction` method assembles the transaction struct, signs it with the local private key, and finally encodes it as RLP.

This is a critical question, as it relates to real-world usage and security boundaries of Ethereum nodes. Here is a detailed explanation:

---

#### Note

- **In production environments and most wallets/front-end DApps, 99% use `eth_sendRawTransaction`.**
  - That is, **the user signs locally**, then sends the signed, RLP-encoded raw transaction to the node.
  - This way, the private key never leaves the user's local environment, ensuring high security.

- `eth_sendTransaction` is only used when **the node has the account's private key locally** (e.g., geth console, test environments, private chain development).
  - This requires the node's keystore to have the private key for the from address, and the node signs for you.
  - It is rarely used in production, as the private key is exposed on the node, which is a significant risk.

---

- **Production/Front-end/Wallet:**  
  - User signs locally, uses `eth_sendRawTransaction`, and the node only broadcasts and packages the transaction.
- **Node-local accounts (rare):**  
  - `eth_sendTransaction`, the node's keystore has the private key and signs for you.


### 2.2 RLP Decoding

- The node decodes the hex string into a byte stream.
- RLP decoding restores it to a `Transaction` struct (see `core/types/transaction.go`'s `DecodeRLP`).

[Detailed introduction](./the-journey-of-transaction/rlp.md)

### 2.3 Signature and Validity Checks

- Verifies the transaction signature (V, R, S) and recovers the sender address.
- Checks if nonce, gas, balance, and format are valid.

### 2.4 Transaction Pool (TxPool)

- After passing validation, the transaction is **added to the local transaction pool** (`core/tx_pool.go`).
- The transaction pool manages all pending transactions, sorting by account, nonce, gas price, etc.
- All new transactions (whether received locally or from the network) are added to the **transaction pool (TxPool)**.
- The core code for the transaction pool is in `core/txpool/txpool.go`.
[Detailed introduction](./the-journey-of-transaction/transaction-pool.md)

### 2.5 Broadcast to the P2P Network

- Nodes broadcast transactions to each other via the devp2p protocol (eth/66, eth/67, etc.).
- The code entry point is `eth/handler.go`'s `handleMsg`, with message type `TransactionsMsg`.
- Other nodes, upon receiving, also perform the same validation and add to their pool.

[Detailed introduction](./the-journey-of-transaction/transaction-broadcast.md)

---

## 3. The "Destination" of a Transaction

- **Local transaction pool**: Waiting to be packaged into a block by a miner/validator.
- **P2P network**: Synchronously propagated to all other nodes to ensure network-wide consensus.
- **Final goal**: Selected by a miner/validator, packaged into a new block, and written to the blockchain.


## 4. Key Code Paths (go-ethereum)

1. `eth/api.go` → `SendRawTransaction`
2. `core/types/transaction.go` → `DecodeRLP`
3. `core/tx_pool.go` → `AddLocal`/`AddRemotes`
4. `p2p/protocols/eth/handler.go` → Transaction broadcast

---

## 5. Summary Flowchart

```mermaid
flowchart TD
    User/Wallet -- eth_sendRawTransaction --> NodeRPC
    NodeRPC -- Parse/Decode/Validate --> TxPool
    TxPool -- Broadcast --> P2P Network Other Nodes
    TxPool -- Wait --> Block Packaging
    Block Packaging -- Success --> Blockchain
```

--- 