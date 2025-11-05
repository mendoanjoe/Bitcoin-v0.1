# Ringkasan Teknis Bitcoin v0.1

## Gambaran Umum Arsitektur

```
┌─────────────────────────────────────────────────────────────────┐
│                     BITCOIN v0.1 ARCHITECTURE                    │
└─────────────────────────────────────────────────────────────────┘

┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   UI Layer   │  │  RPC Layer   │  │  Script VM   │  │ Cryptography │
│  (ui.cpp)    │  │              │  │ (script.cpp) │  │  (key.h)     │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │                 │
       └─────────────────┴─────────────────┴─────────────────┘
                                │
                   ┌────────────┴────────────┐
                   │   CORE LAYER (main.cpp) │
                   │  - Blockchain Logic     │
                   │  - Transaction Pool     │
                   │  - Mining               │
                   └────────────┬────────────┘
                                │
       ┌────────────────────────┼────────────────────────┐
       │                        │                        │
┌──────┴───────┐     ┌──────────┴──────────┐   ┌────────┴────────┐
│  P2P Network │     │   Storage Layer     │   │  Mining Engine  │
│  (net.cpp)   │     │   (db.cpp)          │   │  (SHA-256)      │
│  - IRC Seed  │     │   - BerkeleyDB      │   │  - PoW Loop     │
│  - Messages  │     │   - Block Files     │   │  - Difficulty   │
└──────────────┘     └─────────────────────┘   └─────────────────┘
```

## Flow Diagram: Transaction Lifecycle

```
1. USER CREATES TRANSACTION
   │
   ├─→ SelectCoins()
   │   └─→ Find UTXO dengan balance cukup
   │
   ├─→ CreateTransaction()
   │   ├─→ Build inputs dari UTXO
   │   ├─→ Build outputs (payment + change)
   │   └─→ SignSignature() untuk setiap input
   │
   ├─→ CommitTransactionSpent()
   │   ├─→ AddToWallet()
   │   └─→ Mark old coins as spent
   │
   └─→ BroadcastTransaction()
       └─→ RelayWalletTransaction()

2. NETWORK PROPAGATION
   │
   ├─→ Tx diterima oleh Node A
   ├─→ Node A validates transaction
   │   ├─→ Check inputs exist & unspent
   │   ├─→ Verify signatures
   │   └─→ Check fees
   │
   └─→ Node A relay ke peers
       └─→ Add to mempool

3. MINING PROCESS
   │
   ├─→ Miner creates block template
   │   ├─→ Create coinbase tx
   │   └─→ Select tx from mempool
   │
   ├─→ Calculate merkle root
   │
   ├─→ Proof-of-Work loop
   │   ├─→ nonce = 0
   │   ├─→ hash = SHA256(SHA256(header))
   │   ├─→ if hash <= target → BLOCK FOUND!
   │   └─→ else nonce++ dan repeat
   │
   └─→ Broadcast block

4. BLOCK CONFIRMATION
   │
   ├─→ Network validates block
   │   ├─→ Check PoW
   │   ├─→ Verify all transactions
   │   └─→ Check merkle root
   │
   ├─→ Add to blockchain
   │   ├─→ ConnectBlock()
   │   └─→ Update best chain
   │
   └─→ Transaction confirmed! (1 confirmation)
       └─→ Wait 5 more blocks untuk 6 confirmations
```

## Flow Diagram: Key Generation to Address

```
1. GENERATE KEYPAIR
   │
   EC_KEY_generate_key(secp256k1)
   │
   ├─→ Private Key: 256-bit random number
   └─→ Public Key: Point on curve (x, y)

2. SERIALIZE PUBLIC KEY
   │
   0x04 || X (32 bytes) || Y (32 bytes)
   │
   └─→ Uncompressed Public Key (65 bytes)

3. HASH PUBLIC KEY
   │
   SHA-256(public_key)
   │
   └─→ RIPEMD-160(sha256_hash)
       │
       └─→ Hash160 (20 bytes)

4. ADD VERSION & CHECKSUM
   │
   ├─→ Add version byte: 0x00 || hash160
   │
   └─→ Calculate checksum:
       ├─→ SHA-256(SHA-256(version || hash160))
       └─→ Take first 4 bytes

5. BASE58 ENCODE
   │
   Base58Encode(version || hash160 || checksum)
   │
   └─→ Bitcoin Address
       Example: 1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa
```

## Mining Flow Diagram

```
BitcoinMiner() MAIN LOOP
│
├─→ [1] Generate mining key
│   └─→ CKey key; key.MakeNewKey();
│
├─→ [2] Wait for peers
│   └─→ while(vNodes.empty()) Sleep(1000);
│
├─→ [3] Get difficulty
│   └─→ nBits = GetNextWorkRequired(pindexBest);
│
├─→ [4] Create coinbase transaction
│   ├─→ Input: prevout.SetNull() (no input)
│   ├─→ Input script: nBits || extraNonce
│   └─→ Output: mining_key + block_reward
│
├─→ [5] Create block template
│   ├─→ Add coinbase tx
│   ├─→ Collect pending transactions
│   │   ├─→ Validate each tx
│   │   ├─→ Check fees
│   │   └─→ Add to block (max 500KB)
│   └─→ Calculate block reward + fees
│
├─→ [6] Build merkle tree
│   └─→ hashMerkleRoot = BuildMerkleTree(vtx)
│
├─→ [7] Setup block header
│   ├─→ nVersion = 1
│   ├─→ hashPrevBlock = pindexBest->GetBlockHash()
│   ├─→ hashMerkleRoot (from step 6)
│   ├─→ nTime = GetAdjustedTime()
│   ├─→ nBits (from step 3)
│   └─→ nNonce = 1
│
├─→ [8] Calculate target
│   └─→ target = CBigNum().SetCompact(nBits)
│
└─→ [9] PROOF-OF-WORK LOOP ⟳
    │
    ├─→ hash = SHA256(SHA256(block_header))
    │
    ├─→ if (hash <= target)
    │   ├─→ ✓ BLOCK FOUND!
    │   ├─→ SaveKey(key)
    │   ├─→ ProcessBlock()
    │   └─→ RelayBlock()
    │
    ├─→ else if (new_transactions_available)
    │   └─→ Break, rebuild block (go to step 4)
    │
    └─→ else
        ├─→ nNonce++
        └─→ Repeat (back to start of loop)

Average time to find block: ~10 minutes
Hashes per block (at genesis): ~2^32 ≈ 4.3 billion tries
```

## Script Validation Flow

```
STANDARD P2PKH TRANSACTION VALIDATION
│
Script Structure:
├─→ scriptSig (Input):    <signature> <pubKey>
└─→ scriptPubKey (Output): OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG

Execution Stack:
│
Step 1: Empty stack
   Stack: []

Step 2: Push signature from scriptSig
   Stack: [signature]

Step 3: Push pubKey from scriptSig
   Stack: [signature, pubKey]

Step 4: OP_DUP (duplicate pubKey)
   Stack: [signature, pubKey, pubKey]

Step 5: OP_HASH160 (hash top of stack)
   Stack: [signature, pubKey, Hash160(pubKey)]

Step 6: Push pubKeyHash from scriptPubKey
   Stack: [signature, pubKey, Hash160(pubKey), pubKeyHash]

Step 7: OP_EQUALVERIFY (check equality, remove if equal)
   Stack: [signature, pubKey]
   ├─→ if Hash160(pubKey) != pubKeyHash → FAIL ✗
   └─→ if Hash160(pubKey) == pubKeyHash → continue ✓

Step 8: OP_CHECKSIG (verify ECDSA signature)
   ├─→ Extract signature and pubKey from stack
   ├─→ Reconstruct transaction hash to sign
   ├─→ ECDSA_verify(hash, signature, pubKey)
   └─→ Push result to stack
       Stack: [true] or [false]

Step 9: Check final stack
   ├─→ if top == true → VALID TRANSACTION ✓
   └─→ if top == false → INVALID TRANSACTION ✗
```

## P2P Network Message Flow

```
NODE STARTUP & PEER DISCOVERY
│
├─→ [1] Start TCP socket listener (port 8333)
│
├─→ [2] IRC Peer Discovery
│   ├─→ Connect to irc.lfnet.org
│   ├─→ Join channel #bitcoin
│   ├─→ Encode IP as nickname: "u" + Base58(IP)
│   ├─→ Monitor channel for other peers
│   └─→ Extract IP addresses from nicknames
│
├─→ [3] Connect to peers
│   └─→ ThreadOpenConnections()
│       ├─→ Maintain 8 outbound connections
│       └─→ Accept inbound connections
│
└─→ [4] Version handshake
    │
    Node A ──[version]──→ Node B
    Node A ←─[version]─── Node B
    Node A ──[verack]──→ Node B
    Node A ←─[verack]─── Node B
    │
    └─→ Connected! Start message exchange

MESSAGE TYPES & FLOW
│
├─→ "addr"
│   Purpose: Share known peer addresses
│   Flow: Node A → [addr: list of IPs] → Node B
│         Node B adds to address book
│
├─→ "inv" (inventory)
│   Purpose: Announce new block/transaction
│   Flow: Node A finds new block
│         Node A → [inv: MSG_BLOCK, hash] → Peers
│         Peers check if they have it
│         Peers → [getdata: hash] → Node A
│
├─→ "getblocks"
│   Purpose: Request block inventory
│   Flow: New node joins network
│         Node → [getblocks: locator] → Peer
│         Peer → [inv: list of blocks] → Node
│         Node → [getdata: blocks] → Peer
│         Peer → [block: data] → Node
│
├─→ "tx"
│   Purpose: Broadcast transaction
│   Flow: User creates transaction
│         Node → [tx: transaction] → Peers
│         Peers validate transaction
│         Peers add to mempool
│         Peers relay to their peers
│
└─→ "block"
    Purpose: Broadcast mined block
    Flow: Miner finds block
          Miner → [block: data] → Peers
          Peers validate block
          Peers add to blockchain
          Peers relay to their peers
```

## Database Structure

```
BERKELEYDB FILES
│
├─→ wallet.dat
│   ├─→ ["key", pubKey] → privKey
│   ├─→ ["tx", hash] → CWalletTx
│   ├─→ ["name", address] → string (address book)
│   └─→ ["setting", key] → value
│
├─→ addr.dat
│   └─→ ["addr", ip] → CAddress (peer info)
│
└─→ blkindex.dat
    ├─→ ["blockindex", hash] → CBlockIndex
    ├─→ ["hashBestChain"] → uint256
    └─→ ["tx", hash] → CTxIndex

FLAT FILES (Block Storage)
│
├─→ blk0001.dat (raw block data)
├─→ blk0002.dat
└─→ blk000n.dat
    │
    Format: [4 bytes: magic] [4 bytes: size] [block data]
```

## Key Components Summary

| Component | File | Purpose |
|-----------|------|---------|
| **Cryptography** | key.h | ECDSA signing/verification dengan secp256k1 |
| **Transactions** | main.h, main.cpp | Transaction creation, validation, UTXO tracking |
| **Blockchain** | main.h, main.cpp | Block validation, chain management, reorganization |
| **Mining** | main.cpp | Proof-of-work, block creation, difficulty adjustment |
| **Script** | script.h, script.cpp | Stack-based script execution dan validation |
| **Network** | net.h, net.cpp | P2P communication, message handling, peer management |
| **Storage** | db.h, db.cpp | BerkeleyDB wrapper untuk persistent storage |
| **Wallet** | main.cpp, db.cpp | Key management, transaction history |
| **Base58** | base58.h | Address encoding/decoding |
| **SHA-256** | sha.h, sha.cpp | Hashing untuk PoW dan transaction IDs |
| **IRC** | irc.h, irc.cpp | Peer discovery via IRC |
| **UI** | ui.h, ui.cpp | wxWidgets GUI |

## Constants & Parameters

```cpp
// Consensus parameters
COIN = 100000000 satoshi = 1 BTC
MAX_BLOCK_SIZE = 500 KB
COINBASE_MATURITY = 100 blocks
TARGET_TIMESPAN = 2 weeks (2016 blocks)
TARGET_SPACING = 10 minutes
INITIAL_BLOCK_REWARD = 50 BTC

// Network
DEFAULT_PORT = 8333
MAX_OUTBOUND_CONNECTIONS = 8
PROTOCOL_VERSION = 209

// Cryptography
CURVE = secp256k1
HASH_ALGORITHM = SHA-256
ADDRESS_VERSION = 0x00 (mainnet)
```

## Thread Architecture

```
MAIN THREADS
│
├─→ Thread 1: Main UI Thread
│   └─→ wxWidgets event loop
│
├─→ Thread 2: ThreadSocketHandler
│   └─→ Socket I/O (send/receive)
│
├─→ Thread 3: ThreadMessageHandler  
│   └─→ ProcessMessages() untuk setiap peer
│
├─→ Thread 4: ThreadOpenConnections
│   └─→ Maintain peer connections
│
├─→ Thread 5: ThreadIRCSeed
│   └─→ IRC peer discovery
│
└─→ Thread 6: ThreadBitcoinMiner (optional)
    └─→ Mining loop jika fGenerateBitcoins=true
```

## Security Mechanisms

1. **Double-Spend Prevention**
   - UTXO model - coins can only be spent once
   - ConnectInputs() validates unspent outputs
   - Longest chain rule resolves forks

2. **Sybil Attack Resistance**
   - Proof-of-Work makes fake identities expensive
   - One-CPU-one-vote (bukan one-IP-one-vote)

3. **51% Attack Protection**
   - PoW makes attack computationally expensive
   - Economic incentive untuk honest mining

4. **Signature Verification**
   - ECDSA ensures only key owner can spend
   - Script system validates ownership

5. **Transaction Malleability** (Note: v0.1 had issues)
   - Signature covering transaction hash
   - Later fixed in subsequent versions

## Performance Characteristics

```
Mining Speed (Genesis, 2009):
├─→ Difficulty: 1.0
├─→ Target: 0x00000000FFFF0000...
├─→ Expected hashes: ~2^32 (4.3 billion)
└─→ Time: ~10 minutes (on 2009 hardware)

Block Size:
├─→ Average: ~1-10 KB (mostly empty in 2009)
└─→ Maximum: 500 KB (consensus limit)

Transaction Throughput:
├─→ ~7 transactions per second (theoretical max)
└─→ Block time: 10 minutes

Database:
├─→ wallet.dat: Grows with transactions
├─→ blkindex.dat: ~200 bytes per block
└─→ blk*.dat: Sum of all block sizes
```

---

## Kesimpulan Teknis

Bitcoin v0.1 adalah implementasi yang elegant dari konsep cryptocurrency dengan:

✓ **Cryptographic Security**: ECDSA + SHA-256  
✓ **Decentralized Consensus**: Proof-of-Work  
✓ **Trustless System**: No central authority  
✓ **Economic Incentives**: Mining rewards  
✓ **Flexible Scripts**: Turing-incomplete VM  
✓ **P2P Network**: Direct peer communication  
✓ **Immutable Ledger**: Blockchain structure  

Walaupun sederhana dibandingkan dengan versi modern, v0.1 sudah mengimplementasikan semua konsep fundamental yang membuat Bitcoin revolutionary.
