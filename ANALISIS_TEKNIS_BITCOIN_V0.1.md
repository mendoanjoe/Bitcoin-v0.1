# Analisis Teknis Bitcoin v0.1 Source Code

## Pendahuluan

Dokumen ini berisi analisis mendalam terhadap source code Bitcoin versi 0.1 yang dirilis oleh Satoshi Nakamoto pada 9 Januari 2009. Analisis ini mencakup flow pembuatan kunci, transaksi, mining, dan aspek teknis lainnya.

## 1. Flow Pembuatan Kunci (Key Generation Flow)

### 1.1 Arsitektur Kriptografi

Bitcoin v0.1 menggunakan **Elliptic Curve Digital Signature Algorithm (ECDSA)** dengan kurva **secp256k1** untuk kriptografi kunci publik/privat.

#### Ukuran Kunci:
```cpp
// Dari key.h
// secp256k1:
// const unsigned int PRIVATE_KEY_SIZE = 279;
// const unsigned int PUBLIC_KEY_SIZE  = 65;
// const unsigned int SIGNATURE_SIZE   = 72;
```

### 1.2 Implementasi Class CKey

Class `CKey` (didefinisikan di `key.h`) adalah wrapper untuk OpenSSL EC_KEY:

```cpp
class CKey
{
protected:
    EC_KEY* pkey;  // OpenSSL EC_KEY object
    
public:
    CKey()
    {
        pkey = EC_KEY_new_by_curve_name(NID_secp256k1);
        if (pkey == NULL)
            throw key_error("CKey::CKey() : EC_KEY_new_by_curve_name failed");
    }
    
    void MakeNewKey()
    {
        if (!EC_KEY_generate_key(pkey))
            throw key_error("CKey::MakeNewKey() : EC_KEY_generate_key failed");
    }
```

### 1.3 Flow Pembuatan Kunci Baru

**Langkah-langkah pembuatan kunci baru:**

1. **Inisialisasi CKey object** → Membuat EC_KEY dengan kurva secp256k1
2. **Panggil MakeNewKey()** → Generate private key secara random
3. **Extract Public Key** → Public key diturunkan dari private key
4. **Simpan ke Database** → Key disimpan ke BerkeleyDB wallet

```cpp
// Dari main.cpp, line 75-82
vector<unsigned char> GenerateNewKey()
{
    CKey key;
    key.MakeNewKey();                    // 1. Generate random private key
    if (!AddKey(key))                    // 2. Simpan ke wallet
        throw runtime_error("GenerateNewKey() : AddKey failed\n");
    return key.GetPubKey();              // 3. Return public key
}

// AddKey menyimpan ke memory dan database
bool AddKey(const CKey& key)
{
    CRITICAL_BLOCK(cs_mapKeys)
    {
        mapKeys[key.GetPubKey()] = key.GetPrivKey();        // Memory map
        mapPubKeys[Hash160(key.GetPubKey())] = key.GetPubKey();
    }
    return CWalletDB().WriteKey(key.GetPubKey(), key.GetPrivKey());  // Database
}
```

### 1.4 Signing dan Verification

**Digital Signature Flow:**

```cpp
bool CKey::Sign(uint256 hash, vector<unsigned char>& vchSig)
{
    vchSig.clear();
    unsigned char pchSig[10000];
    unsigned int nSize = 0;
    // ECDSA signing menggunakan OpenSSL
    if (!ECDSA_sign(0, (unsigned char*)&hash, sizeof(hash), pchSig, &nSize, pkey))
        return false;
    vchSig.resize(nSize);
    memcpy(&vchSig[0], pchSig, nSize);
    return true;
}

bool CKey::Verify(uint256 hash, const vector<unsigned char>& vchSig)
{
    // -1 = error, 0 = bad sig, 1 = good
    if (ECDSA_verify(0, (unsigned char*)&hash, sizeof(hash), 
                     &vchSig[0], vchSig.size(), pkey) != 1)
        return false;
    return true;
}
```

## 2. Flow Pembuatan Transaksi (Transaction Creation Flow)

### 2.1 Struktur Transaksi

Transaksi Bitcoin terdiri dari:
- **vin (inputs)**: Array dari CTxIn - referensi ke output transaksi sebelumnya
- **vout (outputs)**: Array dari CTxOut - tujuan dan jumlah bitcoin

```cpp
class CTransaction
{
public:
    int nVersion;
    vector<CTxIn> vin;      // Transaction inputs
    vector<CTxOut> vout;    // Transaction outputs
    int nLockTime;
    
    uint256 GetHash() const  // SHA256(SHA256(transaction))
    {
        return SerializeHash(*this);
    }
}
```

### 2.2 Flow CreateTransaction

**Proses pembuatan transaksi (dari main.cpp, line 2514):**

```
CreateTransaction() Flow:
│
├─ 1. Pilih Coins (SelectCoins)
│   └─ Cari UTXO yang cukup untuk memenuhi nValue + nFee
│
├─ 2. Buat Output untuk Penerima
│   └─ vout[0] = CTxOut(nValueOut, scriptPubKey)
│
├─ 3. Buat Change Output (jika ada kembalian)
│   └─ vout[1] = CTxOut(nValueIn - nValue, scriptPubKeyChange)
│
├─ 4. Buat Inputs dari Coins yang dipilih
│   └─ Untuk setiap coin → vin.push_back(CTxIn(...))
│
├─ 5. Sign Setiap Input
│   └─ SignSignature(*pcoin, wtxNew, nIn++)
│
└─ 6. Validasi Fee
    └─ if (nFee < wtxNew.GetMinFee()) → ulangi dengan fee lebih besar
```

**Kode implementasi:**

```cpp
bool CreateTransaction(CScript scriptPubKey, int64 nValue, CWalletTx& wtxNew, int64& nFeeRequiredRet)
{
    CRITICAL_BLOCK(cs_main)
    {
        CTxDB txdb("r");
        CRITICAL_BLOCK(cs_mapWallet)
        {
            int64 nFee = nTransactionFee;
            loop
            {
                wtxNew.vin.clear();
                wtxNew.vout.clear();
                
                // 1. Pilih coins untuk digunakan
                set<CWalletTx*> setCoins;
                if (!SelectCoins(nValue, setCoins))
                    return false;
                
                // 2. Buat output untuk penerima
                wtxNew.vout.push_back(CTxOut(nValueOut, scriptPubKey));
                
                // 3. Buat change output jika ada kembalian
                if (nValueIn > nValue)
                {
                    // Gunakan key yang sama dengan salah satu input
                    vector<unsigned char> vchPubKey;
                    CTransaction& txFirst = *(*setCoins.begin());
                    foreach(const CTxOut& txout, txFirst.vout)
                        if (txout.IsMine())
                            if (ExtractPubKey(txout.scriptPubKey, true, vchPubKey))
                                break;
                    
                    CScript scriptPubKey;
                    scriptPubKey << vchPubKey << OP_CHECKSIG;
                    wtxNew.vout.push_back(CTxOut(nValueIn - nValue, scriptPubKey));
                }
                
                // 4. Buat inputs dari coins
                foreach(CWalletTx* pcoin, setCoins)
                    for (int nOut = 0; nOut < pcoin->vout.size(); nOut++)
                        if (pcoin->vout[nOut].IsMine())
                            wtxNew.vin.push_back(CTxIn(pcoin->GetHash(), nOut));
                
                // 5. Sign setiap input
                int nIn = 0;
                foreach(CWalletTx* pcoin, setCoins)
                    for (int nOut = 0; nOut < pcoin->vout.size(); nOut++)
                        if (pcoin->vout[nOut].IsMine())
                            SignSignature(*pcoin, wtxNew, nIn++);
                
                // 6. Cek fee
                if (nFee < wtxNew.GetMinFee(true))
                {
                    nFee = nFeeRequiredRet = wtxNew.GetMinFee(true);
                    continue;  // Ulangi dengan fee baru
                }
                
                break;
            }
        }
    }
    return true;
}
```

### 2.3 Script System

Bitcoin menggunakan stack-based scripting language untuk validasi transaksi:

**Script Pub Key (Output):**
```
<pubKey> OP_CHECKSIG
```

**Script Sig (Input):**
```
<signature>
```

**Validasi:**
```
Stack: []
1. Push signature      → Stack: [signature]
2. Push pubKey         → Stack: [signature, pubKey]
3. OP_CHECKSIG         → Verify signature dengan pubKey
   → Stack: [true/false]
```

## 3. Flow Mining (Bitcoin Mining Flow)

### 3.1 Proof-of-Work Concept

Bitcoin menggunakan **SHA-256 hash** untuk proof-of-work:
- Block valid jika: `Hash(block) <= target`
- Target berbanding terbalik dengan difficulty
- Miner mencoba berbagai nonce sampai menemukan hash yang valid

### 3.2 Mining Process Flow

**Flow lengkap BitcoinMiner() dari main.cpp, line 2183:**

```
BitcoinMiner() Loop:
│
├─ 1. Generate Mining Key
│   └─ CKey key; key.MakeNewKey();
│
├─ 2. Tunggu Peers
│   └─ while (vNodes.empty()) → Sleep(1000)
│
├─ 3. Buat Coinbase Transaction
│   ├─ txNew.vin[0].prevout.SetNull()  // No input (coinbase)
│   ├─ txNew.vin[0].scriptSig << nBits << bnExtraNonce
│   └─ txNew.vout[0].scriptPubKey << key.GetPubKey() << OP_CHECKSIG
│
├─ 4. Buat Block Baru
│   ├─ pblock->vtx.push_back(txNew)  // Coinbase tx
│   └─ Tambahkan pending transactions dari mempool
│
├─ 5. Hitung Target dari nBits
│   └─ hashTarget = CBigNum().SetCompact(pblock->nBits).getuint256()
│
├─ 6. Prebuild Hash Buffer
│   ├─ tmp.block.nVersion, hashPrevBlock, hashMerkleRoot
│   ├─ tmp.block.nTime, nBits, nNonce
│   └─ Setup untuk optimasi SHA-256
│
├─ 7. Mining Loop (Proof-of-Work)
│   │
│   └─ loop
│       ├─ BlockSHA256(&tmp.block, nBlocks0, &tmp.hash1)
│       ├─ BlockSHA256(&tmp.hash1, nBlocks1, &hash)
│       │
│       ├─ if (hash <= hashTarget)
│       │   ├─ Block ditemukan!
│       │   ├─ AddKey(key)  // Simpan mining key
│       │   ├─ ProcessBlock(pblock)  // Broadcast block
│       │   └─ Break
│       │
│       ├─ if (nTransactionsUpdated != nTransactionsUpdatedLast)
│       │   └─ Break dan buat block baru
│       │
│       └─ ++tmp.block.nNonce  // Coba nonce berikutnya
│
└─ 8. Ulangi dari Step 2
```

**Kode implementasi mining loop:**

```cpp
bool BitcoinMiner()
{
    printf("BitcoinMiner started\n");
    SetThreadPriority(GetCurrentThread(), THREAD_PRIORITY_LOWEST);
    
    CKey key;
    key.MakeNewKey();
    CBigNum bnExtraNonce = 0;
    
    while (fGenerateBitcoins)
    {
        // Tunggu peers
        while (vNodes.empty())
            Sleep(1000);
        
        // Dapatkan difficulty
        CBlockIndex* pindexPrev = pindexBest;
        unsigned int nBits = GetNextWorkRequired(pindexPrev);
        
        // Buat coinbase transaction
        CTransaction txNew;
        txNew.vin.resize(1);
        txNew.vin[0].prevout.SetNull();
        txNew.vin[0].scriptSig << nBits << ++bnExtraNonce;
        txNew.vout.resize(1);
        txNew.vout[0].scriptPubKey << key.GetPubKey() << OP_CHECKSIG;
        
        // Buat block
        auto_ptr<CBlock> pblock(new CBlock());
        pblock->vtx.push_back(txNew);
        
        // Tambahkan transactions dari mempool
        // ... (collect transactions logic)
        
        pblock->nBits = nBits;
        pblock->vtx[0].vout[0].nValue = pblock->GetBlockValue(nFees);
        
        // Setup hash buffer
        tmp.block.nVersion = pblock->nVersion;
        tmp.block.hashPrevBlock = pblock->hashPrevBlock;
        tmp.block.hashMerkleRoot = pblock->BuildMerkleTree();
        tmp.block.nTime = pblock->nTime;
        tmp.block.nBits = pblock->nBits;
        tmp.block.nNonce = 1;
        
        // Hitung target
        uint256 hashTarget = CBigNum().SetCompact(pblock->nBits).getuint256();
        uint256 hash;
        
        // MINING LOOP
        loop
        {
            // SHA256(SHA256(block header))
            BlockSHA256(&tmp.block, nBlocks0, &tmp.hash1);
            BlockSHA256(&tmp.hash1, nBlocks1, &hash);
            
            // Cek apakah hash <= target
            if (hash <= hashTarget)
            {
                pblock->nNonce = tmp.block.nNonce;
                
                printf("BitcoinMiner:\n");
                printf("proof-of-work found  \n  hash: %s  \ntarget: %s\n", 
                       hash.GetHex().c_str(), hashTarget.GetHex().c_str());
                
                // Simpan key dan broadcast block
                AddKey(key);
                key.MakeNewKey();
                ProcessBlock(NULL, pblock.release());
                
                break;
            }
            
            // Coba nonce berikutnya
            ++tmp.block.nNonce;
            
            // Cek apakah ada transactions baru
            if (nTransactionsUpdated != nTransactionsUpdatedLast)
                break;
        }
    }
    return true;
}
```

### 3.3 Difficulty Adjustment

Target difficulty disesuaikan setiap 2016 block untuk menjaga block time ~10 menit:

```cpp
unsigned int GetNextWorkRequired(const CBlockIndex* pindexLast)
{
    const int64 nTargetTimespan = 14 * 24 * 60 * 60; // 2 weeks
    const int64 nTargetSpacing = 10 * 60;             // 10 minutes
    const int64 nInterval = nTargetTimespan / nTargetSpacing; // 2016 blocks
    
    // Genesis block
    if (pindexLast == NULL)
        return bnProofOfWorkLimit.GetCompact();
    
    // Only change once per interval
    if ((pindexLast->nHeight+1) % nInterval != 0)
        return pindexLast->nBits;
    
    // Go back by what we want to be 2016 blocks worth of time
    const CBlockIndex* pindexFirst = pindexLast;
    for (int i = 0; pindexFirst && i < nInterval-1; i++)
        pindexFirst = pindexFirst->pprev;
    
    // Calculate time taken for last 2016 blocks
    int64 nActualTimespan = pindexLast->GetBlockTime() - pindexFirst->GetBlockTime();
    
    // Limit adjustment to 4x up or down
    if (nActualTimespan < nTargetTimespan/4)
        nActualTimespan = nTargetTimespan/4;
    if (nActualTimespan > nTargetTimespan*4)
        nActualTimespan = nTargetTimespan*4;
    
    // Retarget
    CBigNum bnNew;
    bnNew.SetCompact(pindexLast->nBits);
    bnNew *= nActualTimespan;
    bnNew /= nTargetTimespan;
    
    return bnNew.GetCompact();
}
```

## 4. Flow Validasi Block (Block Validation Flow)

### 4.1 Struktur Block

```cpp
class CBlock
{
public:
    // Header
    int nVersion;
    uint256 hashPrevBlock;
    uint256 hashMerkleRoot;
    unsigned int nTime;
    unsigned int nBits;
    unsigned int nNonce;
    
    // Transactions
    vector<CTransaction> vtx;
    
    uint256 GetHash() const
    {
        return Hash(BEGIN(nVersion), END(nNonce));
    }
}
```

### 4.2 Block Validation Process

**Flow ProcessBlock() - validasi dan penerimaan block baru:**

```
ProcessBlock():
│
├─ 1. Cek Block Hash vs PoW Target
│   └─ if (GetHash() > target) → REJECT
│
├─ 2. Cek Timestamp
│   └─ if (nTime > now + 2 hours) → REJECT
│
├─ 3. Cek apakah Block sudah ada
│   └─ if (mapBlockIndex.count(hash)) → Already have
│
├─ 4. Cek Previous Block
│   ├─ if (!mapBlockIndex.count(hashPrevBlock))
│   │   └─ Simpan sebagai orphan block
│   └─ else → Lanjut validasi
│
├─ 5. Validasi Transaksi-transaksi
│   ├─ Cek coinbase transaction
│   ├─ Validasi setiap transaction
│   └─ Cek merkle root
│
├─ 6. Cek Difficulty Target
│   └─ if (nBits != GetNextWorkRequired()) → REJECT
│
├─ 7. Connect ke Main Chain
│   ├─ CTxDB txdb
│   ├─ ConnectBlock(txdb, pindex)
│   └─ Update best chain jika perlu
│
└─ 8. Relay Block ke Peers
    └─ RelayInventory(CInv(MSG_BLOCK, hash))
```

## 5. Flow P2P Networking

### 5.1 Arsitektur Network

Bitcoin v0.1 menggunakan arsitektur P2P murni:
- Tidak ada server pusat
- Node berkomunikasi langsung satu sama lain
- Protocol berbasis TCP socket

### 5.2 Message Types

```cpp
// Message types yang didukung
MSG_TX           // Transaction
MSG_BLOCK        // Block
MSG_GETBLOCKS    // Request block inventory
MSG_GETDATA      // Request block/tx data
MSG_ADDR         // Share peer addresses
MSG_INV          // Inventory announcement
```

### 5.3 Message Processing Flow

```
ProcessMessage():
│
├─ "version"
│   └─ Handshake dengan peer, exchange protocol version
│
├─ "addr"
│   └─ Receive peer addresses, update address book
│
├─ "inv"
│   ├─ Receive inventory (block/tx announcements)
│   └─ Request data dengan "getdata" jika belum punya
│
├─ "getdata"
│   └─ Kirim block atau transaction yang diminta
│
├─ "getblocks"
│   ├─ Peer request block inventory
│   └─ Send "inv" dengan list blocks
│
├─ "tx"
│   ├─ Receive transaction
│   ├─ Validate transaction
│   ├─ Add to mempool
│   └─ Relay ke peers lain
│
└─ "block"
    ├─ Receive block
    ├─ ProcessBlock() untuk validasi
    └─ Relay jika valid
```

### 5.4 Peer Discovery

Bitcoin v0.1 menggunakan **IRC channel** untuk peer discovery:

```cpp
// Dari irc.cpp
void ThreadIRCSeed()
{
    // Connect ke IRC server
    // Channel: #bitcoin
    // Server: irc.lfnet.org
    
    // Encode IP address sebagai nickname
    // Format: "u" + Base58(IP)
    
    // Monitor channel untuk melihat peers lain
    // Parse nicknames untuk mendapatkan IP addresses
}
```

## 6. Flow Wallet Management

### 6.1 Wallet Storage

Wallet disimpan menggunakan **BerkeleyDB**:

```cpp
class CWalletDB : public CDB
{
public:
    // Simpan private key
    bool WriteKey(const vector<unsigned char>& vchPubKey, 
                  const CPrivKey& vchPrivKey)
    {
        return Write(make_pair(string("key"), vchPubKey), vchPrivKey);
    }
    
    // Simpan transaction
    bool WriteTx(uint256 hash, const CWalletTx& wtx)
    {
        return Write(make_pair(string("tx"), hash), wtx);
    }
}
```

### 6.2 Key Management

**In-memory key storage:**

```cpp
// Global variables di main.cpp
map<vector<unsigned char>, CPrivKey> mapKeys;           // PubKey -> PrivKey
map<uint160, vector<unsigned char> > mapPubKeys;        // Hash160 -> PubKey
CCriticalSection cs_mapKeys;                            // Thread safety
```

### 6.3 Address Generation

**Flow pembuatan Bitcoin address:**

```
GenerateNewKey():
│
├─ 1. Generate ECDSA keypair
│   └─ EC_KEY_generate_key(secp256k1)
│
├─ 2. Extract Public Key (65 bytes)
│   └─ 04 + X-coordinate (32 bytes) + Y-coordinate (32 bytes)
│
├─ 3. Hash Public Key
│   ├─ SHA-256(pubkey)
│   └─ RIPEMD-160(sha256)  → Hash160 (20 bytes)
│
├─ 4. Add Version Byte
│   └─ 0x00 + Hash160
│
├─ 5. Calculate Checksum
│   ├─ SHA-256(SHA-256(version + hash160))
│   └─ Take first 4 bytes
│
├─ 6. Encode Base58
│   └─ Base58Encode(version + hash160 + checksum)
│
└─ Result: Bitcoin Address (starts with "1")
```

**Implementasi:**

```cpp
// Dari base58.h
inline string PubKeyToAddress(const vector<unsigned char>& vchPubKey)
{
    return Hash160ToAddress(Hash160(vchPubKey));
}

inline string Hash160ToAddress(uint160 hash160)
{
    // Add version byte (0x00 untuk mainnet)
    vector<unsigned char> vch(1, ADDRESSVERSION);
    vch.insert(vch.end(), UBEGIN(hash160), UEND(hash160));
    return EncodeBase58Check(vch);
}

inline string EncodeBase58Check(const vector<unsigned char>& vchIn)
{
    // Add 4-byte checksum
    vector<unsigned char> vch(vchIn);
    uint256 hash = Hash(vch.begin(), vch.end());
    vch.insert(vch.end(), (unsigned char*)&hash, (unsigned char*)&hash + 4);
    return EncodeBase58(vch);
}
```

## 7. Flow Verifikasi Transaksi

### 7.1 Script Execution

Script evaluation adalah stack-based:

```cpp
bool EvalScript(const CScript& script, const CTransaction& txTo, 
                unsigned int nIn, int nHashType)
{
    vector<valtype> stack;
    vector<valtype> altstack;
    
    CScript::const_iterator pc = script.begin();
    while (pc < script.end())
    {
        opcodetype opcode;
        valtype vchPushValue;
        
        if (!script.GetOp(pc, opcode, vchPushValue))
            return false;
        
        switch (opcode)
        {
            case OP_DUP:
                if (stack.size() < 1)
                    return false;
                stack.push_back(stacktop(-1));
                break;
                
            case OP_HASH160:
                if (stack.size() < 1)
                    return false;
                {
                    valtype& vch = stacktop(-1);
                    vch = Hash160(vch);
                }
                break;
                
            case OP_EQUALVERIFY:
                if (stack.size() < 2)
                    return false;
                if (stacktop(-1) != stacktop(-2))
                    return false;
                stack.pop_back();
                stack.pop_back();
                break;
                
            case OP_CHECKSIG:
                // Verify ECDSA signature
                if (stack.size() < 2)
                    return false;
                {
                    valtype& vchSig = stacktop(-2);
                    valtype& vchPubKey = stacktop(-1);
                    bool fSuccess = CheckSig(vchSig, vchPubKey, scriptCode, txTo, nIn);
                    stack.pop_back();
                    stack.pop_back();
                    stack.push_back(fSuccess ? vchTrue : vchFalse);
                }
                break;
        }
    }
    
    return (!stack.empty() && CastToBool(stack.back()));
}
```

### 7.2 Transaction Verification Flow

```
VerifySignature():
│
├─ 1. Combine Scripts
│   ├─ scriptSig (dari input)
│   └─ scriptPubKey (dari previous output)
│
├─ 2. Execute Combined Script
│   └─ EvalScript(scriptSig + scriptPubKey, tx, nIn)
│
└─ 3. Check Result
    └─ Stack top must be TRUE
```

**Standard Transaction Script (P2PKH):**

```
scriptPubKey: OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
scriptSig: <signature> <pubKey>

Execution:
1. Push signature                → [sig]
2. Push pubKey                   → [sig, pubKey]
3. OP_DUP                        → [sig, pubKey, pubKey]
4. OP_HASH160                    → [sig, pubKey, hash(pubKey)]
5. Push pubKeyHash               → [sig, pubKey, hash(pubKey), pubKeyHash]
6. OP_EQUALVERIFY                → [sig, pubKey] (jika equal)
7. OP_CHECKSIG                   → [true/false]
```

## 8. Security Features

### 8.1 Double-Spend Prevention

**Mekanisme:**
1. Transaction harus reference UTXO yang belum di-spend
2. ConnectInputs() cek apakah input sudah di-spend
3. Longest chain rule untuk handle fork

```cpp
bool CTransaction::ConnectInputs(CTxDB& txdb, ...)
{
    for (int i = 0; i < vin.size(); i++)
    {
        COutPoint prevout = vin[i].prevout;
        
        // Cek apakah output sudah di-spend
        CTxIndex txindex;
        if (!txdb.ReadTxIndex(prevout.hash, txindex))
            return false;
            
        if (!txindex.vSpent[prevout.n].IsNull())
            return false;  // Already spent!
        
        // Mark as spent
        txindex.vSpent[prevout.n] = posThisTx;
        txdb.WriteTxIndex(prevout.hash, txindex);
    }
    return true;
}
```

### 8.2 51% Attack Protection

- Proof-of-Work membuat mahal untuk attack
- Longest chain rule
- Difficulty adjustment setiap 2016 blocks

### 8.3 Replay Attack Prevention

- SIGHASH types untuk flexible signing
- nSequence untuk transaction replacement
- nLockTime untuk delayed transactions

## 9. Data Structures

### 9.1 Merkle Tree

Block menggunakan Merkle tree untuk efficient verification:

```cpp
uint256 CBlock::BuildMerkleTree() const
{
    vMerkleTree.clear();
    foreach(const CTransaction& tx, vtx)
        vMerkleTree.push_back(tx.GetHash());
        
    int j = 0;
    for (int nSize = vtx.size(); nSize > 1; nSize = (nSize + 1) / 2)
    {
        for (int i = 0; i < nSize; i += 2)
        {
            int i2 = min(i+1, nSize-1);
            vMerkleTree.push_back(Hash(BEGIN(vMerkleTree[j+i]), END(vMerkleTree[j+i]),
                                      BEGIN(vMerkleTree[j+i2]), END(vMerkleTree[j+i2])));
        }
        j += nSize;
    }
    return (vMerkleTree.empty() ? 0 : vMerkleTree.back());
}
```

### 9.2 Blockchain Index

```cpp
class CBlockIndex
{
public:
    CBlockIndex* pprev;      // Pointer ke previous block
    CBlockIndex* pnext;      // Pointer ke next block
    unsigned int nFile;      // File number di disk
    unsigned int nBlockPos;  // Position di file
    int nHeight;             // Height di chain
    CBigNum bnChainWork;     // Total work di chain sampai block ini
    
    uint256 GetBlockHash() const
    {
        return block.GetHash();
    }
}
```

## 10. Optimasi dan Implementasi Detail

### 10.1 SHA-256 Optimization

Mining menggunakan optimasi SHA-256 dengan pre-computed buffer:

```cpp
// Prebuild hash buffer untuk optimasi
struct unnamed1
{
    struct unnamed2
    {
        int nVersion;
        uint256 hashPrevBlock;
        uint256 hashMerkleRoot;
        unsigned int nTime;
        unsigned int nBits;
        unsigned int nNonce;
    } block;
    unsigned char pchPadding0[64];
    uint256 hash1;
    unsigned char pchPadding1[64];
} tmp;

// Pre-format blocks sekali
unsigned int nBlocks0 = FormatHashBlocks(&tmp.block, sizeof(tmp.block));
unsigned int nBlocks1 = FormatHashBlocks(&tmp.hash1, sizeof(tmp.hash1));

// Mining loop - hanya update nonce
loop
{
    BlockSHA256(&tmp.block, nBlocks0, &tmp.hash1);
    BlockSHA256(&tmp.hash1, nBlocks1, &hash);
    
    if (hash <= hashTarget)
        break;  // Found!
        
    ++tmp.block.nNonce;
}
```

### 10.2 Database Design

BerkeleyDB digunakan untuk persistent storage:

```cpp
// Database files:
// - wallet.dat: Private keys dan transactions
// - addr.dat: Known peer addresses  
// - blkindex.dat: Block index
// - blk0001.dat, blk0002.dat, ...: Block data

class CDB
{
protected:
    Db* pdb;
    string strFile;
    
    // Transaction support
    DbTxn* activetxn;
    
    // Thread-safe access
    static CCriticalSection cs_db;
    static map<string, int> mapFileUseCount;
}
```

### 10.3 Thread Architecture

Bitcoin v0.1 menggunakan multiple threads:

```cpp
// Main threads:
1. ThreadSocketHandler    - Handle socket I/O
2. ThreadMessageHandler   - Process P2P messages
3. ThreadOpenConnections  - Maintain peer connections
4. ThreadIRCSeed         - IRC peer discovery
5. ThreadBitcoinMiner    - Mining (jika enabled)
6. ThreadDNSAddressSeed  - DNS-based peer discovery
```

## 11. Kesimpulan

### 11.1 Arsitektur Utama

Bitcoin v0.1 mendemonstrasikan arsitektur yang elegant untuk:
1. **Decentralized consensus** melalui Proof-of-Work
2. **Cryptographic security** menggunakan ECDSA dan SHA-256
3. **P2P network** tanpa central authority
4. **Script-based smart contracts** untuk flexible transactions

### 11.2 Key Insights Teknis

1. **Kriptografi:**
   - secp256k1 untuk ECDSA
   - SHA-256 untuk hashing
   - Base58Check untuk address encoding

2. **Consensus:**
   - Longest chain rule
   - Proof-of-Work dengan adjustable difficulty
   - Block time target: ~10 menit

3. **Security:**
   - UTXO model mencegah double-spending
   - Script system untuk flexible validation
   - Merkle trees untuk efficient verification

4. **Networking:**
   - Pure P2P architecture
   - IRC untuk initial peer discovery
   - Gossip protocol untuk block/tx propagation

5. **Storage:**
   - BerkeleyDB untuk wallet dan indexes
   - Flat files untuk block data
   - Merkle tree untuk compact proofs

### 11.3 Design Principles

Source code Bitcoin v0.1 menunjukkan prinsip design yang kuat:
- **Simplicity**: Implementasi yang straightforward
- **Security**: Multiple layers of verification
- **Decentralization**: No single point of failure
- **Incentive alignment**: Mining rewards align dengan network security

Analisis ini menunjukkan bagaimana Satoshi Nakamoto mengkombinasikan cryptography, distributed systems, dan economic incentives untuk menciptakan sistem mata uang digital yang truly decentralized.

---

## Referensi

- Bitcoin v0.1 Source Code: https://github.com/bitcoin/bitcoin/tree/v0.1.5
- Bitcoin Whitepaper: Satoshi Nakamoto, 2008
- OpenSSL ECDSA Documentation
- BerkeleyDB Documentation

**Catatan:** Analisis ini berdasarkan Bitcoin v0.1 yang dirilis 9 Januari 2009. Versi modern Bitcoin telah mengalami banyak perubahan dan improvement.
