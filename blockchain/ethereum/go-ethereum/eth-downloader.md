## **What does the downloader do?**
The downloader package is responsible for the full chain synchronization in go-ethereum. It's a critical component that handles:
1. Block synchronization from peers
2. Different sync modes (Full Sync and Snap Sync)
3. State synchronization
4. Header and block body downloads
5. Receipt synchronization
6. Peer management for downloads

#### 1. **Block Synchronization from Peers**

When it starts:
- On node startup
- When node detects it's behind the network
- When receiving new block announcements

Sync Modes:
```mermaid
sequenceDiagram
    participant Node as Local Node
    participant Peer as Remote Peer
    participant Chain as Blockchain
    participant DL as Downloader

    Note over Node,DL: Sync Start
    Node->>DL: Initialize Sync
    DL->>Peer: Request Headers
    Peer-->>DL: Send Headers
    DL->>Chain: Validate Headers
    
    par Full Sync
        DL->>Peer: Request Bodies
        DL->>Peer: Request Receipts
        Peer-->>DL: Send Bodies
        Peer-->>DL: Send Receipts
    and Snap Sync
        DL->>Peer: Request State
        Peer-->>DL: Send State
    end

    DL->>Chain: Process & Insert
    Note over Node,DL: Sync Complete
```

Code paths:
- Main entry: `downloader.go:New()`
- Sync initiation: `downloader.go:synchronise()`
- Block processing: `downloader.go:processBlocks()`

#### 2. **Different Sync Modes**

Full Sync vs Snap Sync:
- Full Sync:
  - Downloads all blocks from genesis
  - Executes all transactions
  - Builds state from scratch
  - Result: Complete blockchain history

- Snap Sync:
  - Downloads recent state directly
  - Only processes recent blocks
  - Uses pivot point for state sync
  - Result: Recent state + recent history

#### 3. **State Synchronization**

Definition: Process of downloading the current state trie of the Ethereum network.

```mermaid
sequenceDiagram
    participant Node as Local Node
    participant Peer as Remote Peer
    participant SS as State Sync
    participant DB as State DB

    Note over Node,DB: State Sync Start
    Node->>SS: Initialize State Sync
    SS->>Peer: Request State Root
    Peer-->>SS: Send State Root
    
    loop State Download
        SS->>Peer: Request State Chunks
        Peer-->>SS: Send State Chunks
        SS->>DB: Store State Data
    end

    SS->>DB: Verify State Root
    Note over Node,DB: State Sync Complete
```

Code paths:
- Entry: `statesync.go:NewStateSync()`
- Processing: `statesync.go:Process()`
- Storage: `statesync.go:CommitState()`

#### 4. **Header and Block Body Downloads**

When and Why:
- Headers: First step in sync to establish chain structure
- Bodies: Required for transaction execution and state updates

```mermaid
sequenceDiagram
    participant DL as Downloader
    participant Queue as Queue
    participant Peer as Remote Peer
    participant Chain as Blockchain

    DL->>Queue: Schedule Headers
    Queue->>Peer: Request Headers
    Peer-->>Queue: Return Headers
    Queue->>Chain: Validate Headers

    par Body Download
        Queue->>Peer: Request Bodies
        Peer-->>Queue: Return Bodies
        Queue->>Chain: Process Bodies
    end
```

Code paths:
- Header download: `downloader.go:fetchHeaders()`
- Body download: `fetchers_concurrent_bodies.go`

#### 5. **Receipt Synchronization**

Receipts: Transaction execution results containing logs and status.

```mermaid
sequenceDiagram
    participant DL as Downloader
    participant Queue as Queue
    participant Peer as Remote Peer
    participant DB as Database

    DL->>Queue: Schedule Receipt Download
    Queue->>Peer: Request Receipts
    Peer-->>Queue: Return Receipts
    Queue->>DB: Store Receipts
```

Code paths:
- Receipt sync: `fetchers_concurrent_receipts.go`
- Processing: `queue.go:processReceipts()`

#### 6. **Peer Management**

Role:
- Manages peer connections
- Tracks peer reliability
- Distributes download requests
- Handles peer scoring

```mermaid
sequenceDiagram
    participant PM as PeerManager
    participant Peer as Remote Peer
    participant DL as Downloader
    participant Queue as Queue

    PM->>Peer: Register Peer
    PM->>DL: Update Peer List
    
    loop Download
        Queue->>PM: Request Best Peer
        PM->>Queue: Return Selected Peer
        Queue->>Peer: Request Data
        alt Success
            Peer-->>Queue: Return Data
            PM->>Peer: Update Score (+)
        else Failure
            PM->>Peer: Update Score (-)
            PM->>DL: Drop Peer
        end
    end
```

Code paths:
- Management: `peer.go`
- Selection: `queue.go:selectPeer()`




## 2. **Interaction with the whole repo:**

### a) **Role played:**
The downloader is a core component in the Ethereum client that ensures the node stays synchronized with the network. It:
- Manages blockchain synchronization
- Handles different sync strategies
- Coordinates with peers for data retrieval
- Ensures data consistency and validity

### b) **Workflow with whole repo:**

```mermaid
graph TD
    A[Ethereum Node] --> B[Downloader]
    B --> C[Peer Management]
    B --> D[Sync Modes]
    D --> E[Full Sync]
    D --> F[Snap Sync]
    B --> G[Block Processing]
    G --> H[Header Processing]
    G --> I[Body Processing]
    G --> J[Receipt Processing]
    B --> K[State Sync]
    K --> L[Trie Sync]
    B --> M[Event Management]
    M --> N[Progress Updates]
    M --> O[Error Handling]
```

### c) **Architecture in the whole repo:**

```mermaid
graph TD
    A[Ethereum Protocol] --> B[Downloader]
    B --> C[BlockChain]
    B --> D[StateDB]
    B --> E[P2P Network]
    
    subgraph "Core Components"
        C --> F[Block Processor]
        C --> G[State Manager]
        D --> H[Trie Database]
    end
    
    subgraph "Network Layer"
        E --> I[Peer Discovery]
        E --> J[Data Transfer]
    end
    
    B --> K[Event System]
    B --> L[Metrics]
```

## 3. **Within the folder itself:**

### a) **Detailed features:**
- Header synchronization
- Block body fetching
- Receipt synchronization
- State synchronization
- Skeleton chain sync
- Beacon chain sync support
- Concurrent fetching
- Progress tracking
- Peer management
- Error handling and recovery

### b) **Major sub-components:**
- Downloader: Main orchestrator
- Queue: Manages download scheduling
- Skeleton: Handles header skeleton sync
- Fetchers: Concurrent download handlers
- ResultStore: Manages downloaded data
- BeaconSync: Handles beacon chain sync
- StateSync: Manages state synchronization

#### - **Skeleton Chain**

Purpose:
- Lightweight chain structure
- Enables efficient sync coordination
- Manages chain reorganizations

```mermaid
sequenceDiagram
    participant Skeleton as Skeleton
    participant DL as Downloader
    participant Chain as Blockchain
    participant Peer as Remote Peer

    Skeleton->>Peer: Request Headers
    Peer-->>Skeleton: Return Headers
    Skeleton->>Chain: Build Skeleton
    
    loop Fill Gaps
        Skeleton->>DL: Request Missing Segments
        DL->>Peer: Fetch Segments
        Peer-->>DL: Return Segments
        DL->>Chain: Fill Skeleton
    end
```

Code paths:
- Implementation: `skeleton.go`
- Processing: `skeleton.go:Process()`

#### - **Beacon Chain Sync**

Purpose:
- Post-merge synchronization
- Coordinates with consensus layer
- Handles finalized blocks

```mermaid
sequenceDiagram
    participant BC as BeaconClient
    participant DL as Downloader
    participant Chain as Blockchain
    participant State as StateSync

    BC->>DL: New Finalized Header
    DL->>Chain: Verify Ancestor
    
    par Sync
        DL->>Chain: Sync Headers
        DL->>State: Sync State
    end

    DL->>Chain: Process Blocks
    DL->>BC: Sync Complete
```

Code paths:
- Implementation: `beaconsync.go`
- Processing: `beaconsync.go:BeaconSync()`



### c) **Sub-component workflow:**

```mermaid
graph TD
    A[Downloader] --> B[Queue]
    A --> C[Skeleton]
    A --> D[Fetchers]
    
    subgraph "Queue Management"
        B --> E[Schedule Tasks]
        B --> F[Track Progress]
    end
    
    subgraph "Fetchers"
        D --> G[Header Fetcher]
        D --> H[Body Fetcher]
        D --> I[Receipt Fetcher]
    end
    
    subgraph "State Management"
        A --> J[StateSync]
        J --> K[Trie Sync]
    end
    
    C --> L[Header Validation]
    C --> M[Chain Assembly]
```

## 4. **Key Concepts:**
The downloader implements several important concepts:

- **Snap Sync**: An optimized synchronization mode that downloads the state directly, rather than reconstructing it from historical blocks. It's more efficient than full sync.
- **Skeleton Chain**: A lightweight representation of the blockchain's structure used to validate and organize block downloads.
- **Pivot Point**: In snap sync, this is the block from which state downloading begins, allowing for faster synchronization.
- **Concurrent Fetching**: The ability to download multiple pieces of data (headers, bodies, receipts) simultaneously from different peers.

## 5. **User Interaction:**
As a user, you can interact with the downloader through:

### 1. **Sync Mode Selection:**
```go
// Choose sync mode when starting geth
geth --syncmode "snap" // or "full"
```

### 2. **API Endpoints:**
```go
// Through RPC
eth.syncing // Check sync status
eth.syncProgress // Get detailed sync progress

// Through Management APIs
downloader.Progress() // Get sync progress
downloader.Cancel() // Cancel ongoing sync
```

### 3. **Monitoring:**
- Track sync progress through metrics
- Monitor sync status through logs
- Get sync statistics through API calls

### 4. **Configuration:**
- Set sync parameters through client configuration
- Configure sync thresholds and limits
- Manage peer connections and timeouts


## [Purge Mode](https://github.com/ethereum/go-ethereum/pull/31414)

#### Mermaid Diagram: Sync Process
it shows the major difference:
> if there is a cutoff, it will only insert header before cutoff and body receipt after, not the full data anymore

```mermaid
graph TD
    A[Start Sync] --> B{Check Cutoff}
    B -->|No Cutoff| C[Full Sync]
    B -->|With Cutoff| D[Verify Cutoff Header]
    D --> E[Insert Headers Before Cutoff]
    E --> F[Insert Bodies/Receipts After Cutoff]
    C --> G[Insert All Data]
    F --> H[Sync Complete]
    G --> H
```
    
#### Sequence Diagram: InsertReceiptChain
```mermaid
sequenceDiagram
    participant Caller
    participant Blockchain
    participant DB
    
    Caller->>Blockchain: InsertReceiptChain(blocks, receipts)
    Blockchain->>Blockchain: Validate headers
    Blockchain->>Blockchain: Split by ancient limit
    alt Ancient blocks
        Blockchain->>DB: Write ancient blocks/receipts
    else Live blocks
        Blockchain->>DB: Write live blocks/receipts
    end
    Blockchain->>DB: Update head markers
    Blockchain->>Caller: Return result
```