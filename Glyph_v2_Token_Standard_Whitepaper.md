# Glyph v2: A Unified Token Standard for the Radiant Blockchain

**Version:** 2.0-draft-8  
**Date:** February 2026  
**Status:** Release Candidate  

---

## Executive Summary

Glyph v2 is a comprehensive token standard for the Radiant blockchain that unifies fungible tokens, non-fungible tokens, smart assets, and naming services under a single, extensible framework. Building on the proven Glyph v1 foundation, v2 introduces enhanced features including on-chain royalty enforcement, explicit burn mechanisms, container/collection support, encrypted content, timelocked reveals, and integration with the WAVE naming system.

The standard leverages Radiant's unique UTXO model with induction proofs and reference system to provide true digital ownership with cryptographic guarantees. With the Radiant V2 hard fork (block 410,000), Glyph v2 dMint achieves fully on-chain proof-of-work validation via native hash opcodes (OP_BLAKE3, OP_K12), eliminating all indexer trust dependencies for decentralized minting.

Glyph v2 is designed for developers, creators, and indexers with a focus on clarity, extensibility, and ease of implementation.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Design Principles](#2-design-principles)
3. [Protocol Overview](#3-protocol-overview)
4. [Token Types](#4-token-types)
5. [Transaction Limits](#5-transaction-limits)
6. [Commit-Reveal Mechanism](#6-commit-reveal-mechanism)
7. [Envelope Formats](#7-envelope-formats)
8. [Metadata Schema](#8-metadata-schema)
9. [Fungible Tokens](#9-fungible-tokens)
10. [Non-Fungible Tokens](#10-non-fungible-tokens)
11. [Decentralized Minting (dMint)](#11-decentralized-minting-dmint)
    - 11.4.1 [On-Chain DAA Bytecode Specification](#1141-on-chain-daa-bytecode-specification)
12. [Burn Mechanism](#12-burn-mechanism)
13. [On-Chain Royalties](#13-on-chain-royalties)
14. [Containers and Collections](#14-containers-and-collections)
15. [Mutable State](#15-mutable-state)
16. [Encrypted Content](#16-encrypted-content)
17. [Timelocked Reveals](#17-timelocked-reveals)
18. [Authority Tokens](#18-authority-tokens)
19. [WAVE Naming Integration](#19-wave-naming-integration)
20. [Indexer Implementation](#20-indexer-implementation)
21. [Backwards Compatibility](#21-backwards-compatibility)
22. [Security Considerations](#22-security-considerations)
23. [Reference Implementation](#23-reference-implementation)
24. [Appendices](#24-appendices)
    - A. [Protocol ID Registry](#appendix-a-protocol-id-registry)
    - B. [CBOR Encoding Rules](#appendix-b-cbor-encoding-rules)
    - C. [Size Limits Summary](#appendix-c-size-limits-summary)
    - D. [Related REPs](#appendix-d-related-reps)
    - E. [V2 dMint Contract Bytecodes](#appendix-e-v2-dmint-contract-bytecodes)

---

## 1. Introduction

### 1.1 Background

The Radiant blockchain introduces a novel UTXO model with induction proofs that enables true digital asset ownership without requiring smart contract virtual machines. The reference system (`OP_PUSHINPUTREF` and `OP_PUSHINPUTREFSINGLETON`) provides a native mechanism for tracking assets across transactions.

Glyph v1 established the foundational token standard, enabling:
- Fungible tokens (FT) with value-sum validation
- Non-fungible tokens (NFT) with singleton references
- Decentralized minting via proof-of-work contracts
- Mutable NFT state with controller authorization

Glyph v2 extends this foundation with additional capabilities while maintaining full backwards compatibility.

### 1.2 Goals

1. **Unified Standard**: One framework for all token types and use cases
2. **Developer-Friendly**: Clear specifications, test vectors, and reference implementations
3. **Indexer-Optimized**: Layered schema for efficient querying at scale
4. **Future-Proof**: Extensible protocol with reserved identifiers
5. **Ecosystem Integration**: WAVE naming, encrypted content, and application profiles

### 1.3 Non-Goals

- Virtual machine or smart contract language
- Cross-chain bridges (separate REP)
- Token economics or governance models

> **Note:** The Radiant V2 hard fork introduces six consensus opcode changes to support
> Glyph v2 dMint: **OP_BLAKE3** (0xee) and **OP_K12** (0xef) for on-chain PoW validation,
> **OP_LSHIFT** (0x98) and **OP_RSHIFT** (0x99) for on-chain DAA bit-shift arithmetic,
> and **OP_2MUL** (0x8d) and **OP_2DIV** (0x8e) for efficient multiply/divide by 2.
> All six opcodes activate at block 410,000 on mainnet. See Section 11.4.

---

## 2. Design Principles

### 2.1 UTXO-Native

Glyph v2 is designed around Radiant's UTXO model:
- Tokens are UTXOs with embedded references
- Transfers are transaction spends
- Balances are the sum of unspent token outputs
- No account state or global storage

### 2.2 Layered Complexity

The standard supports three tiers of implementation:

| Layer | Contents | Indexing Cost |
|-------|----------|---------------|
| **L1: Core** | Ref, type, version | Minimal |
| **L2: Standard** | Name, description, content, policy | Normal |
| **L3: Extended** | App profiles, game items, custom data | Optional |

Indexers can choose their depth of support based on use case.

### 2.3 Deterministic Encoding

All metadata uses deterministic serialization:
- **Primary**: CBOR (RFC 8949) with canonical rules
- **Alternative**: Canonical JSON (RFC 8785)

This ensures commit hashes are reproducible across implementations.

### 2.4 Backwards Compatibility

- v2 indexers MUST support v1 tokens
- v1 wallets display v2 tokens as basic NFTs/FTs
- Migration path from v1 to v2 features

---

## 3. Protocol Overview

### 3.1 Magic Bytes

All Glyph transactions are identified by the magic bytes:

```
ASCII: gly
Hex:   676c79
```

### 3.2 Version Byte

The version byte follows the magic bytes:

| Version | Byte | Status |
|---------|------|--------|
| v1 | `0x01` | Legacy |
| v2 | `0x02` | Current |

### 3.3 GlyphID

The canonical identifier for a Glyph is the **reveal transaction outpoint**:

```
GlyphID = <reveal_txid>:<vout>
```

Example: `a1b2c3d4...f8:0`

This provides a unique, immutable reference that can be resolved by any indexer.

### 3.4 Protocol Identifiers

Glyph v2 defines the following protocol IDs:

| ID | Name | Description |
|----|------|-------------|
| 1 | `GLYPH_FT` | Fungible Token |
| 2 | `GLYPH_NFT` | Non-Fungible Token |
| 3 | `GLYPH_DAT` | Data Storage |
| 4 | `GLYPH_DMINT` | Decentralized Minting |
| 5 | `GLYPH_MUT` | Mutable State |
| 6 | `GLYPH_BURN` | Explicit Burn |
| 7 | `GLYPH_CONTAINER` | Container/Collection |
| 8 | `GLYPH_ENCRYPTED` | Encrypted Content |
| 9 | `GLYPH_TIMELOCK` | Timelocked Reveal |
| 10 | `GLYPH_AUTHORITY` | Issuer Authority |
| 11 | `GLYPH_WAVE` | WAVE Naming |
| 12-255 | Reserved | Future use |

Tokens specify protocols in the `p` array:
```json
{ "p": [2, 5, 8] }  // NFT + Mutable + Encrypted
```

### 3.5 Protocol Combination Rules

Some protocol combinations are invalid or require specific pairings:

| Combination | Valid | Notes |
|-------------|-------|-------|
| [1] FT only | ✅ | Standard fungible token |
| [2] NFT only | ✅ | Standard non-fungible token |
| [3] DAT only | ✅ | Data storage token |
| [1, 4] FT + dMint | ✅ | Mineable fungible token |
| [2, 5] NFT + Mutable | ✅ | Mutable NFT |
| [2, 7] NFT + Container | ✅ | Collection container |
| [2, 8] NFT + Encrypted | ✅ | Encrypted NFT content |
| [2, 10] NFT + Authority | ✅ | Authority token |
| [2, 5, 11] NFT + Mutable + WAVE | ✅ | WAVE name token |
| [1, 2] FT + NFT | ❌ | Invalid: mutually exclusive |
| [4] dMint alone | ❌ | Invalid: must include FT (1) |
| [5] Mutable alone | ❌ | Invalid: must include NFT (2) |
| [6] Burn alone | ❌ | Invalid: burn is an action marker, not a token type |
| [7] Container alone | ❌ | Invalid: must include NFT (2) |
| [8] Encrypted alone | ❌ | Invalid: must include NFT (2) |
| [9] Timelock alone | ❌ | Invalid: must include Encrypted (8) |
| [2, 9] NFT + Timelock | ❌ | Invalid: Timelock requires Encrypted (8) |
| [2, 8, 9] NFT + Encrypted + Timelock | ✅ | Timelocked encrypted NFT |
| [11] WAVE alone | ❌ | Invalid: must include NFT (2) and Mutable (5) |

**Protocol Dependency Rules**:
- `GLYPH_DMINT` (4) requires `GLYPH_FT` (1)
- `GLYPH_MUT` (5) requires `GLYPH_NFT` (2)
- `GLYPH_BURN` (6) is an **action marker** that must accompany a token type (FT or NFT)
- `GLYPH_CONTAINER` (7) requires `GLYPH_NFT` (2)
- `GLYPH_ENCRYPTED` (8) requires `GLYPH_NFT` (2)
- `GLYPH_TIMELOCK` (9) requires `GLYPH_ENCRYPTED` (8)
- `GLYPH_AUTHORITY` (10) requires `GLYPH_NFT` (2)
- `GLYPH_WAVE` (11) requires `GLYPH_NFT` (2) AND `GLYPH_MUT` (5)

Indexers MUST reject tokens with invalid protocol combinations.

---

## 4. Token Types

### 4.1 Fungible Tokens (FT)

Fungible tokens are divisible, interchangeable units:

- Use `OP_PUSHINPUTREF` (normal reference)
- Value tracked in photons (satoshis)
- Value-sum validation: `inputs >= outputs`
- Can be split and merged

**Use cases**: Currencies, points, shares, voting tokens

### 4.2 Non-Fungible Tokens (NFT)

Non-fungible tokens are unique, indivisible assets:

- Use `OP_PUSHINPUTREFSINGLETON` (singleton reference)
- Only one output can hold the reference at a time
- Transfers move the singleton to new owner

**Use cases**: Art, collectibles, tickets, certificates

### 4.3 Data Tokens (DAT)

Data tokens store information without creating a persistent asset:

- No reference created
- Payload committed to blockchain
- Cannot be transferred (consumed on creation)

**Use cases**: Timestamps, attestations, anchors

### 4.4 Containers (CONTAINER)

Containers group multiple tokens into collections:

- Parent token with child references
- Hierarchical ownership
- Collection-level metadata

**Use cases**: NFT collections, bundles, albums

---

## 5. Transaction Limits

Radiant's generous limits enable rich on-chain content:

| Limit | Value | Notes |
|-------|-------|-------|
| `MAX_TX_SIZE` | 12 MB | Consensus maximum |
| `MAX_STANDARD_TX_SIZE` | 20 MB | Policy limit for relay (see §7.2) |
| `MAX_SCRIPT_SIZE` | 10 KB | Maximum script length (standard) |
| `MAX_SCRIPT_ELEMENT_SIZE` | 520 bytes | Maximum bytes per push (standard) |

### 5.1 Glyph-Specific Limits

| Component | Limit | Rationale |
|-----------|-------|-----------|
| Metadata | 256 KB | Reasonable for rich content |
| Commit envelope | 100 KB | Fit in standard outputs |
| Reveal envelope (Style A) | 100 KB | OP_RETURN compatible |
| Reveal envelope (Style B) | 12 MB | Limited by MAX_TX_SIZE (consensus) |
| Update envelope | 64 KB | Incremental changes |
| Single inline file | 1 MB | Large media support |
| Total inline content | 10 MB | Practical limit |

---

## 6. Commit-Reveal Mechanism

### 6.1 Overview

Glyph uses a two-phase commit-reveal pattern:

1. **Commit**: Hash of metadata published in a transaction output
2. **Reveal**: Full metadata published in a spending transaction

This prevents front-running and enables efficient initial indexing.

### 6.2 Commit Hash

The commit hash is computed as:

```
commit_hash = SHA256(SHA256(canonical_metadata_bytes))
```

**Note**: Double SHA256 is used for backwards compatibility with Glyph v1 implementations. This aligns with Bitcoin/Radiant conventions (txids, merkle roots) and maintains a single code path for commit validation across both versions.

Where `canonical_metadata_bytes` is CBOR-encoded with:
- Map keys sorted lexicographically
- Minimal integer encoding
- No indefinite-length items

### 6.3 Commit Transaction

The commit transaction:
1. Creates the token reference(s)
2. Includes the commit envelope in an output
3. Locks to the creator's address

### 6.4 Reveal Transaction

The reveal transaction:
1. Spends the commit output
2. Includes the full metadata in scriptSig
3. Creates the final token output(s)

---

## 7. Envelope Formats

### 7.1 Style A: OP_RETURN

Compact format for smaller payloads:

**Commit**:
```
OP_RETURN <PUSHDATA(MAGIC || V2 || flags || commit_hash || ...)>
```

**Reveal**:
```
OP_RETURN <PUSHDATA(MAGIC || V2 || 0x00)>
           <PUSHDATA(canonical_metadata)>
           [<PUSHDATA(file_chunk)>...]
```

**Maximum size**: 100 KB

### 7.2 Style B: OP_3 Chunked

Extended format for larger payloads:

**Commit**:
```
OP_3 <PUSHDATA(MAGIC)> <PUSHDATA(V2 || flags || commit_hash || ...)>
```

**Reveal**:
```
OP_3 <PUSHDATA(MAGIC)>
     <PUSHDATA(canonical_metadata)>
     [<PUSHDATA(file_chunk)>...]
```

**Maximum size**: Limited by `MAX_TX_SIZE` (12 MB consensus limit). `MAX_STANDARD_TX_SIZE` (20 MB) is a policy constant but has no practical effect since consensus rejects transactions exceeding 12 MB.

### 7.3 Flags Byte

The flags byte encodes optional envelope components:

| Bit | Name | Description |
|-----|------|-------------|
| 0 | `has_content_root` | Merkle root of content present |
| 1 | `has_controller` | Mutable controller specified |
| 2 | `has_profile_hint` | App profile hint included |
| 3-6 | Reserved | Must be zero |
| 7 | `is_reveal` | Style A: distinguish commit/reveal |

### 7.4 Envelope Style Detection

Indexers MUST detect envelope style by examining the script prefix:

| Style | Detection | First Byte(s) |
|-------|-----------|---------------|
| A | `OP_RETURN` prefix | `0x6a` or `0x00 0x6a` |
| B | `OP_3` prefix | `0x53` |

**Detection Algorithm**:
```python
def detect_envelope_style(script: bytes) -> str:
    if len(script) < 1:
        return 'unknown'
    if script[0] == 0x6a:  # OP_RETURN
        return 'A'
    if len(script) >= 2 and script[0] == 0x00 and script[1] == 0x6a:  # OP_FALSE OP_RETURN
        return 'A'
    if script[0] == 0x53:  # OP_3
        return 'B'
    return 'unknown'
```

**Notes**:
- Style A is used for smaller payloads (≤100 KB) in OP_RETURN outputs
- Style B is used for larger payloads in scriptSig (reveal) or witness data
- Both styles use the same magic bytes (`gly`) for identification

---

## 8. Metadata Schema

### 8.1 Root Object

```json
{
  "v": 2,
  "type": "<token_type>",
  "p": [<protocol_ids>],
  "name": "<display_name>",
  "desc": "<description>",
  "created": "<ISO8601_timestamp>",
  "creator": "<pubkey_or_address>",
  "content": { ... },
  "preview": { ... },
  "policy": { ... },
  "rights": { ... },
  "rels": { ... },
  "mutable": { ... },
  "royalty": { ... },
  "app": { ... },
  "container": { ... },
  "crypto": { ... }
}
```

### 8.2 Field Specifications

| Field | Type | Max Size | Required | Description |
|-------|------|----------|----------|-------------|
| `v` | integer | - | Yes | Version (2) |
| `type` | string | 64 bytes | No | Human-readable token type hint (e.g., "nft", "fungible_token"). **Note**: The `p` array is authoritative for token type determination. Indexers MUST derive token type from `p`, not from this field. |
| `p` | array | 16 items | Yes | Protocol IDs |
| `name` | string | 256 bytes | No | Display name |
| `desc` | string | 4 KB | No | Description |
| `created` | string | 32 bytes | No | ISO8601 timestamp |
| `creator` | string | 66 bytes | No | Creator pubkey (hex) |
| `content` | object | - | No | Content files |
| `preview` | object | - | No | Preview/thumbnail |
| `policy` | object | - | No | Safety policy |
| `rights` | object | - | No | Licensing |
| `rels` | object | - | No | Relationships |
| `mutable` | object | - | No | Mutability config |
| `royalty` | object | - | No | Royalty settings |
| `app` | object | - | No | Application data |
| `container` | object | - | No | Collection info |
| `crypto` | object | - | No | Encryption params |
| `commit_outpoint` | string | 75 bytes | RECOMMENDED | Explicit commit tx reference (`txid:vout`). Enables deterministic commit↔reveal linking. Required when multiple commits share the same hash. |

### 8.3 Creator Signature

If `creator` is present as an object (not just a pubkey string), it enables provenance verification:

```json
{
  "creator": {
    "pubkey": "<33-byte compressed pubkey, hex>",
    "sig": "<signature, hex>",
    "algo": "ecdsa-secp256k1"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `pubkey` | string | 33-byte compressed public key, hex-encoded |
| `sig` | string | Signature over commit hash, hex-encoded |
| `algo` | string | `ecdsa-secp256k1` or `schnorr-secp256k1` |

**Signature Formats**:

| `algo` | Format | Size |
|--------|--------|------|
| `ecdsa-secp256k1` | DER-encoded | 70–72 bytes |
| `schnorr-secp256k1` | Raw (r \|\| s) | 64 bytes |

**Signed message**: `SHA256("glyph-v2-creator:" || commit_hash)`

Where `commit_hash` is computed with `creator.sig` set to empty string during signing.

**Signing Process (Pseudocode)**:
```
1. metadata_for_hash = metadata with creator.sig = ""
2. commit_hash = SHA256(SHA256(CBOR(metadata_for_hash)))
3. message = SHA256("glyph-v2-creator:" || commit_hash)
4. sig = SIGN(message, private_key)
5. metadata.creator.sig = hex(sig)
6. final_commit_hash = SHA256(SHA256(CBOR(metadata)))  // This goes in commit envelope
```

**Verification Process**:
```
1. Extract creator.sig from metadata
2. Set creator.sig = "" in metadata copy
3. temp_hash = SHA256(SHA256(CBOR(metadata_copy)))
4. message = SHA256("glyph-v2-creator:" || temp_hash)
5. VERIFY(sig, message, pubkey) using algo
```

**Note**: The `final_commit_hash` (signing step 6) differs from `temp_hash` (verification step 3). Indexers verify using the temp_hash approach.

### 8.4 Content Model

Glyph v2 supports two content formats: a full structured model and a compact shorthand for simple assets.

#### Full Content Model (Recommended for complex assets)

```json
{
  "content": {
    "primary": {
      "path": "image.png",
      "mime": "image/png",
      "size": 102400,
      "hash": { "algo": "sha256", "hex": "abc123..." },
      "encoding": "raw",
      "compression": "none",
      "storage": "inline"
    },
    "files": [
      {
        "path": "metadata.json",
        "mime": "application/json",
        "size": 1024,
        "hash": { "algo": "sha256", "hex": "def456..." },
        "storage": "inline"
      }
    ],
    "refs": [
      {
        "path": "video.mp4",
        "uri": "ipfs://QmVideo...",
        "mime": "video/mp4",
        "size": 50000000,
        "hash": { "algo": "sha256", "hex": "789abc..." }
      }
    ]
  }
}
```

#### Compact Shorthand (v1-compatible, for simple inline content)

For simple assets with a single inline file, the compact `main` shorthand saves ~100-150 bytes:

```json
{
  "main": {
    "t": "image/png",
    "b": "<base64_or_raw_bytes>"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `t` | string | MIME type |
| `b` | bytes | Raw content bytes (CBOR byte string) |

**Indexer Behavior**: Indexers MUST normalize shorthand to full model internally:
- `main.t` → `content.primary.mime`
- `main.b` → inline content with `storage: "inline"`
- Hash computed from `main.b` at index time

**When to use each format**:
| Use Case | Recommended Format |
|----------|--------------------|
| Simple NFT with single image | Shorthand (`main`) |
| Multi-file asset | Full (`content`) |
| External references (IPFS) | Full (`content.refs`) |
| Content requiring hash verification | Full (`content`) |

#### Full Model Field Specifications

| Field | Required | Description |
|-------|----------|-------------|
| `path` | Yes | File path within asset |
| `mime` | Yes | MIME type |
| `size` | Yes | Size in bytes |
| `hash` | Yes | `{ algo: "sha256", hex: "..." }` |
| `encoding` | No | `raw` (default) or `base64` |
| `compression` | No | `none` (default), `gzip`, or `zstd` |
| `storage` | Yes | `inline`, `ref`, or `ipfs` |
| `uri` | For refs | External URI for non-inline content |

**Storage types**:
- `inline`: Embedded in reveal transaction
- `ref`: External with integrity hash (URI required)
- `ipfs`: IPFS CID reference

### 8.5 Preview Model

```json
{
  "preview": {
    "thumb": {
      "path": "thumb.jpg",
      "mime": "image/jpeg",
      "width": 256,
      "height": 256
    },
    "blurhash": "LEHV6nWB2yk8pyo0adR*.7kCMdnj",
    "alt": "A digital artwork depicting..."
  }
}
```

### 8.6 Policy Model

```json
{
  "policy": {
    "renderable": true,
    "executable": false,
    "nsfw": false
  }
}
```

| Field | Default | Description |
|-------|---------|-------------|
| `renderable` | true | Safe to display in wallets |
| `executable` | false | Contains executable code |
| `nsfw` | false | Adult content flag |
| `transferable` | true | Token can be transferred |

### 8.7 Non-Transferable Tokens (Soulbound)

When `policy.transferable: false`:
- NFT is bound to the original owner (soulbound)
- Cannot be sold or transferred to another address
- Can still be burned by owner

**Use cases**: Certificates, badges, achievements, identity tokens

**Script Enforcement** (RECOMMENDED):
```
OP_PUSHINPUTREFSINGLETON <tokenRef>
OP_DROP
<original_owner_pubkeyhash> OP_DUP OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG
```

This script ensures only the original owner can spend the token (for burns only, since output must go back to same owner or be burned).

**Indexer Behavior**:
- Indexers SHOULD track `transferable` status
- Wallets SHOULD hide transfer options for soulbound tokens
- Marketplaces SHOULD reject listings for non-transferable tokens

---

## 9. Fungible Tokens

### 9.1 Contract Script

The FT contract enforces value-sum conservation:

```
<P2PKH>
OP_STATESEPARATOR
OP_PUSHINPUTREF <tokenRef>
OP_REFOUTPUTCOUNT_OUTPUTS
OP_INPUTINDEX OP_CODESCRIPTBYTECODE_UTXO OP_HASH256
OP_DUP OP_CODESCRIPTHASHVALUESUM_UTXOS
OP_OVER OP_CODESCRIPTHASHVALUESUM_OUTPUTS
OP_GREATERTHANOREQUAL OP_VERIFY
OP_CODESCRIPTHASHOUTPUTCOUNT_OUTPUTS OP_NUMEQUALVERIFY
```

**Validation rules**:
1. Input value sum >= Output value sum
2. All outputs with token ref use same code script

### 9.2 FT Metadata

```json
{
  "v": 2,
  "type": "fungible_token",
  "p": [1],
  "name": "Example Token",
  "ticker": "EXT",
  "decimals": 8,
  "supply": {
    "type": "fixed",
    "max": 21000000,
    "circulating": 0
  }
}
```

### 9.3 Transfer

FT transfers split value across outputs:

```
Input:  1000 tokens (tokenRef)
Output 0: 600 tokens to recipient (tokenRef)
Output 1: 400 tokens change to sender (tokenRef)
```

### 9.4 Token Decimals and Photon Backing

Fungible tokens specify display decimals via the `decimals` field (0-8, default 8).

**Photon-Backed Decimals** (RECOMMENDED for new tokens):

The `decimals` field determines the photon:token ratio:

| Decimals | Photons per Token | 1 RXD (100M photons) = |
|----------|-------------------|------------------------|
| 8 | 1 | 100,000,000 tokens |
| 6 | 100 | 1,000,000 tokens |
| 4 | 10,000 | 10,000 tokens |
| 2 | 1,000,000 | 100 tokens |
| 0 | 100,000,000 | 1 token |

**Formula**: `photons_per_token = 10^(8 - decimals)`

**Example**: Token with `decimals: 2` and `supply.max: 1000`:
- Total photons required: 1000 × 1,000,000 = 1,000,000,000 photons (10 RXD)
- Display: "500.50 TOKENS" = 500,500,000 photons on-chain

**Benefits**:
1. **Economic floor**: Tokens have intrinsic RXD value
2. **Spam prevention**: Lower decimal tokens cost more to mint
3. **Simpler indexing**: On-chain value directly maps to token amount

**Wallet Display**: `on_chain_photons / 10^(8 - decimals)` with `decimals` decimal places.

**Backwards Compatibility**: Tokens with `decimals: 8` behave identically to current implementation (1 photon = 1 token unit).

---

## 10. Non-Fungible Tokens

### 10.1 Contract Script

The NFT contract enforces singleton uniqueness:

```
OP_PUSHINPUTREFSINGLETON <tokenRef>
OP_DROP
<P2PKH>
```

**Validation rules**:
1. Singleton can only exist in one output
2. Transfer moves singleton to new owner

### 10.2 NFT Metadata

```json
{
  "v": 2,
  "type": "nft",
  "p": [2],
  "name": "Artwork #1",
  "desc": "A unique digital creation",
  "content": {
    "primary": {
      "path": "artwork.png",
      "mime": "image/png",
      "size": 2048000,
      "hash": { "algo": "sha256", "hex": "..." }
    }
  },
  "preview": {
    "thumb": { "path": "thumb.jpg", "mime": "image/jpeg" }
  },
  "policy": { "renderable": true }
}
```

### 10.3 Transfer

NFT transfers move the singleton:

```
Input:  NFT (singletonRef) owned by Alice
Output: NFT (singletonRef) owned by Bob
```

---

## 11. Decentralized Minting (dMint)

### 11.1 Overview

dMint enables proof-of-work token distribution:
- Miners solve hash puzzles to claim tokens
- Difficulty adjusts based on selected algorithm
- Multiple POW algorithms supported
- Dynamic difficulty adjustment (DAA)

### 11.2 POW Algorithms

| ID | Algorithm | Memory | GPU Support | Notes |
|----|-----------|--------|-------------|-------|
| 0x00 | SHA256d | ~1 KB | Excellent | Backward compatible |
| 0x01 | Blake3 | ~1 KB | Excellent | OP_BLAKE3 (0xee) — implemented |
| 0x02 | KangarooTwelve | ~200 B | Good | OP_K12 (0xef) — implemented |
| 0x03 | Argon2id-Light | 64 MB | Fair | Memory-hard, deferred |
| 0x04 | RandomX-Light | 256 MB | CPU only | CLI miner only, deferred |

### 11.3 DAA Modes

| ID | Mode | Description |
|----|------|-------------|
| 0x00 | Fixed | Static difficulty |
| 0x01 | Epoch | Bitcoin-style periodic adjustment |
| 0x02 | ASERT | Exponential moving average |
| 0x03 | LWMA | Linear weighted moving average |
| 0x04 | Schedule | Predetermined difficulty curve |

### 11.4 dMint Contract Variants

With the Radiant V2 hard fork, all supported PoW algorithms (SHA256d, Blake3, K12) are
validated fully on-chain via native opcodes. No indexer trust dependency is required.

**v1 Contract (SHA256d tokens):**
```
<height:4B>
OP_PUSHINPUTREFSINGLETON <contractRef:36B>
OP_PUSHINPUTREF <tokenRef:36B>
<maxHeight> <reward> <target>
OP_STATESEPARATOR
<v1_bytecode>  (validates SHA256d PoW + state transitions)
```

**v2 Contract (all algorithms — requires Hard Fork V2):**
```
<height:4B>
OP_PUSHINPUTREFSINGLETON <contractRef:36B>
OP_PUSHINPUTREF <tokenRef:36B>
<maxHeight> <reward> <target>
<algoId:1B>
<lastTime:4B>
<targetTime:minimal>
OP_STATESEPARATOR
<v2_bytecode>  (validates algorithm-specific PoW + state transitions + on-chain DAA)
```

| Token Config | Contract | On-Chain PoW | DAA |
|-------------|----------|-------------|-----|
| v1 token (no `dmint` in CBOR) | v1 contract | SHA256d (OP_HASH256) | Fixed |
| v2 + algo=SHA256d + fixed | v1 contract | SHA256d (OP_HASH256) | Fixed |
| v2 + algo=SHA256d + dynamic | v2-sha256d | SHA256d (OP_HASH256) | On-chain |
| v2 + algo=blake3 | v2-blake3 | Blake3 (OP_BLAKE3) | On-chain |
| v2 + algo=k12 | v2-k12 | K12 (OP_K12) | On-chain |

> **Note:** Argon2id (memory-hard) is deferred to a future fork pending DoS analysis
> for on-chain memory-bound computation.

> **See also:** [V2 dMint Design Specification](../Glyph-miner/docs/V2_DMINT_DESIGN.md)
> for exact byte layouts, buffer formats, DAA computation, and miner integration details.

### 11.4.1 On-Chain DAA Bytecode Specification

The v2 contract computes difficulty adjustment entirely in script using native opcodes.
The miner sets `nLockTime` to the current UNIX timestamp, which the contract reads via
`OP_TXLOCKTIME`. The activation height for these opcodes is **block 410,000** on mainnet.

#### ASERT-Lite Algorithm (Recommended Default)

The ASERT-lite DAA uses discrete power-of-2 adjustments via `OP_LSHIFT`/`OP_RSHIFT`:

```
// ── Read timestamps ──────────────────────────────────
//   Stack at entry: ... <old_target> <lastTime> <targetTime>
OP_TXLOCKTIME              // push current_time from nLockTime
                           // ... old_target lastTime targetTime current_time
OP_2 OP_PICK               // copy lastTime (index 2 from top)
                           // ... old_target lastTime targetTime current_time lastTime
OP_SUB                     // time_delta = current_time - lastTime
                           // ... old_target lastTime targetTime time_delta

// ── Compute drift ────────────────────────────────────
//   drift = (time_delta - targetTime) / halfLife
//   halfLife is embedded as an immediate in the bytecode
OP_SWAP                    // ... old_target lastTime time_delta targetTime
OP_SUB                     // excess = time_delta - targetTime
                           // ... old_target lastTime excess
<halfLife>                 // push halfLife constant (e.g., 3600)
OP_DIV                     // drift = excess / halfLife (integer, signed)
                           // ... old_target lastTime drift

// ── Clamp drift to [-4, +4] ─────────────────────────
OP_DUP OP_4 OP_GREATERTHAN OP_IF OP_DROP OP_4 OP_ENDIF
OP_DUP OP_4 OP_NEGATE OP_LESSTHAN OP_IF OP_DROP OP_4 OP_NEGATE OP_ENDIF
                           // ... old_target lastTime drift

// ── Rearrange stack for shift ────────────────────────
OP_ROT                     // ... lastTime drift old_target
OP_SWAP                    // ... lastTime old_target drift

// ── Apply shift to target ────────────────────────────
//   drift > 0: target <<= drift  (easier, larger target)
//   drift < 0: target >>= |drift| (harder, smaller target)
//   drift == 0: no change
//   OP_LSHIFT/OP_RSHIFT: (x n -- x<<n) / (x n -- x>>n)
OP_DUP OP_0 OP_GREATERTHAN
OP_IF
    OP_LSHIFT              // new_target = old_target << drift
OP_ELSE
    OP_DUP OP_0 OP_LESSTHAN
    OP_IF
        OP_NEGATE          // |drift|
        OP_RSHIFT          // new_target = old_target >> |drift|
    OP_ELSE
        OP_DROP            // drift == 0, target unchanged
    OP_ENDIF
OP_ENDIF
                           // ... lastTime new_target

// ── Clamp target to [MIN_TARGET, MAX_TARGET] ─────────
// MIN_TARGET = 1 (prevents zero/negative target)
// MAX_TARGET = contract-defined ceiling
OP_DUP OP_1 OP_LESSTHAN OP_IF OP_DROP OP_1 OP_ENDIF
```

**Opcode budget:** ~38 opcodes for the full DAA computation, well within the 32M op limit.

#### Linear DAA (Simpler Alternative)

For contracts that prefer simpler math without bitwise shifts.

> **Note:** The Linear DAA reuses the same timestamp extraction as ASERT-lite (reading
> `OP_TXLOCKTIME`, copying `lastTime` via `OP_PICK`, computing `time_delta`). The
> pseudocode below begins after `time_delta` is on the stack.

```
// new_target = old_target * time_delta / targetTime
//   Stack: ... <old_target> <time_delta> <targetTime>
OP_2 OP_PICK               // ... old_target time_delta targetTime old_target
OP_2 OP_PICK               // ... old_target time_delta targetTime old_target time_delta
OP_MUL                     // ... old_target time_delta targetTime (old_target * time_delta)
OP_SWAP                    // ... old_target time_delta (old_target * time_delta) targetTime
OP_DIV                     // new_target = old_target * time_delta / targetTime

// Clamp to [MIN_TARGET, MAX_TARGET]
OP_DUP OP_1 OP_LESSTHAN OP_IF OP_DROP OP_1 OP_ENDIF
```

#### Security Constraints

| Constraint | Value | Enforced By |
|-----------|-------|-------------|
| Max shift per mint | ±4 bits | Contract bytecode (clamp) |
| Timestamp source | `nLockTime` | `OP_TXLOCKTIME` |
| Timestamp bounds | Consensus MTP ≤ nLockTime ≤ MTP + 2hr | Radiant Core consensus |
| Min target | 1 | Contract bytecode (clamp) |
| Half-life | Immutable per contract | Embedded in bytecode |
| 64-bit overflow | Protected | Radiant uses 64-bit `CScriptNum` |

> **Additional arithmetic opcodes:** `OP_2MUL` (0x8d) and `OP_2DIV` (0x8e) are also
> re-enabled at block 410,000. These provide efficient multiply/divide by 2 as
> alternatives to single-bit shifts, useful for DAA arithmetic and contract logic.

### 11.5 Mining Process

1. Miner fetches current contract UTXO state (on-chain): height, target, algoId, lastTime
2. Target is read directly from the on-chain state (no indexer dependency)
3. Computes valid nonce: `algorithm_hash(preimage + nonce) < target`
4. Submits claim transaction with solution (nLockTime set to current time for DAA)
5. On-chain contract validates PoW using algorithm-specific opcode (OP_HASH256 / OP_BLAKE3 / OP_K12)
6. On-chain contract computes DAA target adjustment using OP_LSHIFT/OP_RSHIFT or OP_MUL/OP_DIV
7. State updates: height++, target adjusts, lastTime updates
8. Indexer indexes mint for analytics/discovery (optional, not trust-critical)

### 11.6 Collision Mitigation

When multiple miners work on the same low-difficulty contract, valid solutions are often found simultaneously. Since only one transaction can spend the contract UTXO, other miners' work is wasted.

#### Mitigation Strategies

1. **Dynamic DAA**: ASERT and LWMA automatically increase difficulty when solutions come faster than the target block time, naturally spacing out solutions.

2. **Recommended Minimum Difficulty**: To prevent excessive collisions:
   | Algorithm | Recommended Minimum | Approx. Time (RTX 4090) |
   |-----------|--------------------|-----------------------|
   | SHA256d | 500,000 | ~30 seconds |
   | Blake3 | 2,500,000 | ~30 seconds |
   | K12 | 2,000,000 | ~30 seconds |

3. **Wallet Warnings**: When difficulty is below recommended minimum, wallets should display:
   > "⚠️ Low difficulty may result in ~X% of solutions being orphaned due to collisions with other miners."

4. **Default to Dynamic DAA**: New contracts should default to ASERT with a 60-second target block time to automatically prevent collisions as more miners join.

### 11.7 dMint Metadata

```json
{
  "v": 2,
  "type": "fungible_token",
  "p": [1, 4],
  "name": "Mineable Token",
  "ticker": "MINE",
  "dmint": {
    "algorithm": "blake3",
    "daa": {
      "mode": "asert",
      "baseDifficulty": 10,
      "targetMintTime": 60,
      "halflife": 3600
    },
    "maxHeight": 10000,
    "reward": 100,
    "premine": 0
  }
}
```

**Algorithm Name Mapping**: CBOR metadata uses human-readable string names. On-chain contract
state uses integer byte IDs. The canonical mapping is:

| Metadata String | On-Chain ID | PoW Opcode |
|-----------------|-------------|------------|
| `"sha256d"` | `0x00` | `OP_HASH256` (0xaa) |
| `"blake3"` | `0x01` | `OP_BLAKE3` (0xee) |
| `"k12"` | `0x02` | `OP_K12` (0xef) |

Indexers and wallets MUST use this mapping when translating between metadata and contract state.

**DAA Mode Mapping**: The `daa.mode` string in CBOR metadata maps to on-chain DAA mode IDs:

| Metadata String | On-Chain ID | Description |
|-----------------|-------------|-------------|
| `"fixed"` | `0x00` | Static difficulty (no DAA bytecode) |
| `"epoch"` | `0x01` | Bitcoin-style periodic adjustment |
| `"asert"` | `0x02` | ASERT-lite (recommended, see §11.4.1) |
| `"lwma"` | `0x03` | Linear weighted moving average |
| `"schedule"` | `0x04` | Predetermined difficulty curve |

### 11.8 Induction Proofs

Radiant provides two independent mechanisms for mathematical induction proofs, enabling
contracts to verify provenance and enforce rules across transaction boundaries in O(1)
constant time and space.

#### 11.8.1 Method 1: Reference-Based Induction

The `OP_PUSHINPUTREF` family of opcodes creates an unbroken chain of provenance from a
genesis (mint) transaction to the current UTXO. This is the primary mechanism used by
all Glyph token contracts.

**How it works:**
- **Base case P(0):** At genesis, `OP_PUSHINPUTREF <ref>` is valid only if `<ref>` matches
  one of the input outpoints being spent (the mint transaction).
- **Inductive step P(k→k+1):** On every subsequent spend, the ref is valid only if at least
  one parent input's output script already contains that same `OP_PUSHINPUTREF` value.
- **Result:** Only the immediate parent inputs need to be checked — no history traversal.

**Available reference opcodes:**

| Opcode | Hex | Purpose |
|--------|-----|---------|
| `OP_PUSHINPUTREF` | 0xd0 | Push & propagate a 36-byte reference |
| `OP_REQUIREINPUTREF` | 0xd1 | Require a reference exists in inputs (don't propagate) |
| `OP_DISALLOWPUSHINPUTREF` | 0xd2 | Forbid a reference in this output |
| `OP_DISALLOWPUSHINPUTREFSIBLING` | 0xd3 | Forbid a reference in sibling outputs |
| `OP_PUSHINPUTREFSINGLETON` | 0xd8 | Push + disallow siblings (NFT shorthand) |

**Code-continuity induction** combines references with introspection to verify that the
parent UTXO used the same contract code. Since the parent also verified its parent, this
creates an inductive chain guaranteeing all ancestors followed the same rules:

```
// RadiantScript: code-continuity induction step
bytes myCodeScript = tx.inputs[this.activeInputIndex].codeScript;
bytes32 myCodeHash = hash256(myCodeScript);
require(tx.outputs.codeScriptCount(myCodeHash) >= 1);
```

#### 11.8.2 Method 2: TxId v3 Preimage Induction

For general-purpose mathematical induction beyond reference tracking, Radiant supports
**Transaction Identifier Version 3**. When `nVersion == 3`, the txid is computed from a
fixed 112-byte preimage instead of the full serialized transaction, preventing the
exponential size blowup that would otherwise occur when embedding parent transactions.

**TxId v3 Preimage Layout (112 bytes):**

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 4B | nVersion | Transaction version (LE) |
| 4 | 4B | nTotalInputs | Number of inputs (LE) |
| 8 | 32B | hashPrevoutInputs | SHA256 of all input outpoints |
| 40 | 32B | hashSequence | SHA256 of all input sequences |
| 72 | 4B | nTotalOutputs | Number of outputs (LE) |
| 76 | 32B | hashOutputHashes | SHA256 of concat of per-output SHA256 hashes |
| 108 | 4B | nLocktime | Transaction locktime (LE) |

**Usage pattern:** The caller provides the parent transaction's v3 preimage (112 bytes)
in the unlocking script. The contract verifies `hash256(preimage)` matches the parent's
txid via `tx.inputs[i].outpointTransactionHash`, then inspects individual fields:

```
// RadiantScript: v3 preimage induction
bytes32 derivedParentTxId = hash256(parentPreimage);
bytes32 actualParentTxId = tx.inputs[this.activeInputIndex].outpointTransactionHash;
require(derivedParentTxId == actualParentTxId);

// Extract and verify parent's version
bytes4 parentVersion, bytes108 rest = parentPreimage.split(4);
require(int(parentVersion) == 3);
```

This enables verification of arbitrary structural properties of ancestor transactions
without requiring full transaction data or exponential size growth.

#### 11.8.3 Transaction State Introspection (OP_PUSH_TX_STATE)

`OP_PUSH_TX_STATE` (0xed) provides access to transaction-level computed values that
complement induction proofs:

| Field | RadiantScript | Description |
|-------|---------------|-------------|
| 0 | `tx.state.txId` | Current transaction's txid (v3-aware, bytes32) |
| 1 | `tx.state.inputSum` | Total input value in photons (int) |
| 2 | `tx.state.outputSum` | Total output value in photons (int) |

These are useful for fee verification, value conservation checks, and self-referencing
contract logic:

```
// RadiantScript: fee-bounded transfer
int fee = tx.state.inputSum - tx.state.outputSum;
require(fee >= 0);
require(fee <= maxFee);
```

#### 11.8.4 Choosing the Right Method

| Criterion | Method 1 (References) | Method 2 (TxId v3) |
|-----------|----------------------|---------------------|
| **Use case** | Token identity, NFTs, FTs | Arbitrary ancestor verification |
| **Complexity** | Simple — single opcode | Complex — preimage parsing |
| **Data overhead** | 0 bytes in unlocking script | 112 bytes per ancestor level |
| **Verification** | Automatic by consensus | Explicit in contract logic |
| **Typical usage** | All Glyph tokens | Advanced state machines, provenance proofs |

For most token contracts, Method 1 with code-continuity introspection is sufficient and
recommended. Method 2 is available for advanced use cases that require verification of
specific structural properties of ancestor transactions.

---

## 12. Burn Mechanism

### 12.1 Overview

Glyph v2 introduces explicit burns with `GLYPH_BURN`:
- Destroys token reference permanently
- Returns photons to user's wallet
- Creates verifiable burn proof on-chain

### 12.2 Burn Transaction

```
Inputs:
  [0] Token UTXO (tokenRef, 1000 photons)
  [1] Fee funding UTXO (e.g., 6,000,000 photons)

Outputs:
  [0] Burn proof (OP_RETURN, 0 photons)
  [1] Recovered token photons to user (1000 photons)
  [2] Change from fee UTXO (remainder after tx fee)
```

**Note**: The transaction fee is paid from the fee funding input, not from the token UTXO. Token photons are returned to the user in a separate output.

### 12.3 Burn Proof Format

```
OP_RETURN
<PUSHDATA(MAGIC || V2 || BURN_MARKER)>
<PUSHDATA(burn_payload)>
```

**Burn payload (CBOR)**:
```json
{
  "v": 2,
  "p": [6],
  "action": "burn",
  "token_ref": "<ref_being_burned>",
  "amount": 500,
  "reason": "optional burn reason"
}
```

### 12.4 Burn Validation

Indexers validate burns by:
1. Confirming token ref exists in input
2. Verifying ref does not exist in any output
3. Recording burn with timestamp and amount

### 12.5 FT Partial Burns

Fungible tokens support partial burns:

```
Input:  1000 tokens
Output 0: Burn proof (500 tokens burned)
Output 1: 500 tokens remaining to sender
```

### 12.6 Burn Confirmation and Reorg Handling

Burns MUST follow standard confirmation rules:

1. **Pending State**: Burns are considered "pending" until 6 confirmations
2. **Confirmed State**: After 6 confirmations, burns are considered final and locked in
3. **Reorg Handling**: On chain reorganization:
   - Mark all burns in orphaned blocks as "invalid"
   - Re-validate burns in the new chain
   - Update burn records accordingly

Indexers SHOULD expose confirmation count in burn queries:

```json
{
  "burn_ref": "abc123...:0",
  "token_ref": "def456...:0",
  "amount": 500,
  "confirmations": 3,
  "status": "pending",
  "block_height": 500000
}
```

---

## 13. On-Chain Royalties

### 13.1 Overview

Glyph v2 supports optional on-chain royalty enforcement:
- Creator specifies royalty terms at mint
- Enforced royalties are validated by the NFT locking script
- Non-compliant sales fail script validation

See REP-3012 for complete royalty enforcement specification.

### 13.2 Royalty Metadata

```json
{
  "royalty": {
    "enforced": true,
    "bps": 500,
    "address": "1CreatorAddress...",
    "minimum": 1000,
    "splits": [
      { "address": "1Artist...", "bps": 400 },
      { "address": "1Platform...", "bps": 100 }
    ]
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `enforced` | boolean | On-chain enforcement enabled |
| `bps` | integer | Basis points (500 = 5%) |
| `address` | string | Primary royalty recipient |
| `minimum` | integer | Minimum royalty in photons |
| `splits` | array | Optional royalty splits |

### 13.3 Enforcement Mechanism

When `enforced: true`, royalties are enforced at the script level by requiring specific output values and recipients.

For standardized RXD-priced sales (including swap orderbook offers), the sale price is the photon value of the seller payment output.

**Canonical output ordering (enforced royalties)**:

1. **Output 0**: NFT output to buyer (new owner script)
2. **Output 1**: Seller payment output (RXD)
3. **Output 2..N**: Royalty output(s) (single recipient or splits)
4. **Remaining outputs**: change and other outputs

### 13.4 Royalty Calculation

```
royalty_amount = MAX(
  sale_price * royalty_bps / 10000,
  royalty_minimum
)
```

### 13.5 Non-Enforced Royalties

When `enforced: false`:
- Royalty terms are advisory
- Indexers track sales and calculate owed amounts
- Marketplaces can enforce voluntarily
- Community reputation for compliance

---

## 14. Containers and Collections

### 14.1 Overview

`GLYPH_CONTAINER` enables hierarchical token groupings:
- Parent container token
- Child token references
- Collection-level metadata
- Shared attributes inheritance

### 14.2 Container Metadata

```json
{
  "v": 2,
  "type": "container",
  "p": [2, 7],
  "name": "My NFT Collection",
  "desc": "A curated collection of 100 unique NFTs",
  "container": {
    "type": "collection",
    "max_items": 100,
    "minted": 50,
    "items": []
  },
  "preview": {
    "thumb": { "path": "collection-banner.jpg" }
  }
}
```

### 14.3 Child Token Reference

Child tokens reference their parent. Children use their own protocol IDs (e.g., `[2]` for NFT) — the `GLYPH_CONTAINER` (7) protocol is only for the parent container token itself:

```json
{
  "v": 2,
  "type": "nft",
  "p": [2],
  "name": "Collection Item #42",
  "rels": {
    "container": {
      "ref": "<container_token_ref>",
      "index": 42
    }
  }
}
```

### 14.4 Container Types

| Type | Description |
|------|-------------|
| `collection` | NFT collection with numbered items |
| `album` | Media album (images, audio, video) |
| `bundle` | Package of related assets |
| `series` | Sequential releases |

### 14.5 Indexer Queries

Containers enable efficient queries:
- `GET /container/:ref/items` — List all items
- `GET /container/:ref/stats` — Collection statistics
- `GET /token/:ref/container` — Get parent container

---

## 15. Mutable State

### 15.1 Overview

`GLYPH_MUT` enables updateable token metadata:
- Controller authorizes changes
- Update history preserved on-chain
- Specified fields can be modified

### 15.2 Mutable Configuration

```json
{
  "mutable": {
    "allowed": true,
    "controller": "<controller_ref>",
    "fields": [
      "/app/data/level",
      "/app/data/stats",
      "/preview/thumb"
    ]
  }
}
```

### 15.3 Mutable Contract Script

```
<payloadHash> OP_DROP
OP_STATESEPARATOR
OP_PUSHINPUTREFSINGLETON <mutableRef>
<validation_bytecode>
```

### 15.4 Update Transaction

**Ordering**: When multiple updates appear in the same block, indexers MUST process them in **transaction index order** within the block. If two updates reference the same `prev`, only the first (by tx index) is valid; subsequent conflicting updates MUST be rejected.

```json
{
  "v": 2,
  "schema": "glyph_update_v1",
  "target": "<token_ref>",
  "prev": "<previous_update_ref_or_null>",
  "update": {
    "op": "replace",
    "path": "/app/data/level",
    "value": 5
  },
  "auth": {
    "controller": "<controller_ref>",
    "sig": "<signature>"
  }
}
```

### 15.5 Update Operations

| Operation | Description |
|-----------|-------------|
| `replace` | Replace value at path |
| `merge` | Merge object at path |
| `append` | Append to array at path |
| `remove` | Remove value at path |

---

## 16. Encrypted Content

### 16.1 Overview

`GLYPH_ENCRYPTED` enables private content:
- Ciphertext stored on-chain
- Metadata describes encryption
- Key delivery via passphrase or recipient wrapping

### 16.2 Crypto Metadata

```json
{
  "crypto": {
    "mode": "A",
    "aead": "xchacha20poly1305",
    "nonce": "<hex>",
    "aad": { "mode": "fields", "paths": ["/content/primary/path"] },
    "key": {
      "kdf": "scrypt",
      "salt": "<hex>",
      "n": 1048576,
      "r": 8,
      "p": 1
    }
  }
}
```

### 16.3 Supported Algorithms

**AEAD**:
- `xchacha20poly1305` (recommended)
- `aes-256-gcm`
- `chacha20poly1305`

**KDF**:
- `scrypt` (passphrase)
- `hkdf-sha256` (key derivation)

### 16.4 Multi-Recipient

For multiple recipients, wrap the CEK per recipient:

```json
{
  "crypto": {
    "key": {
      "wrap": {
        "alg": "x25519-hkdf-aes256gcm",
        "recipients": [
          {
            "kid": "alice",
            "pubkey": "<x25519_pubkey>",
            "eph_pubkey": "<ephemeral>",
            "wrapped_cek": "<encrypted_cek>"
          }
        ]
      }
    }
  }
}
```

### 16.5 Minimum Security Parameters

Implementations MUST reject encrypted glyphs with parameters below these thresholds:

**scrypt (KDF)**:
| Parameter | Minimum | Recommended |
|-----------|---------|-------------|
| `n` | 2^17 (131072) | 2^20 (1048576) |
| `r` | 8 | 8 |
| `p` | 1 | 1 |
| `salt` length | 16 bytes | 32 bytes |

**AEAD Nonces** (MUST be unique per encryption):
| Algorithm | Nonce Length |
|-----------|--------------|
| `xchacha20poly1305` | 24 bytes |
| `aes-256-gcm` | 12 bytes |
| `chacha20poly1305` | 12 bytes |

**Indexer Behavior**:
- Indexers MUST reject glyphs with parameters below minimum thresholds
- Indexers SHOULD flag glyphs with parameters below recommended thresholds
- Wallets SHOULD warn users when decrypting content with weak parameters

---

## 17. Timelocked Reveals

### 17.1 Overview

`GLYPH_TIMELOCK` enables scheduled key reveals:
- Content encrypted until unlock time
- Key hash committed at creation
- Key revealed at or after block height/time

### 17.2 Timelock Metadata

```json
{
  "crypto": {
    "reveal": {
      "scheme": "hash-commit",
      "key_hash": {
        "algo": "sha256",
        "hex": "<hash_of_cek>"
      },
      "unlock": {
        "min_height": 1000000,
        "min_time": "2026-06-01T00:00:00Z"
      }
    }
  }
}
```

### 17.3 Key Reveal Transaction

```json
{
  "v": 2,
  "schema": "key_reveal_v1",
  "target": "<encrypted_glyph_ref>",
  "material": {
    "kind": "cek",
    "bytes": "<cek_hex>",
    "note": "Unlocked as promised"
  }
}
```

### 17.4 Validation

Indexers verify:
1. `SHA256(revealed_key) == key_hash`
2. Current block height >= `min_height` (if specified)
3. Current block's Median Time Past (MTP) >= `min_time` (if specified)

**If both `min_height` and `min_time` are specified, BOTH conditions MUST be satisfied (AND logic).**

**Note**: MTP is the median of the timestamps of the previous 11 blocks, not wall clock time. This provides manipulation-resistant time references.

---

## 18. Authority Tokens

### 18.1 Overview

`GLYPH_AUTHORITY` (protocol ID 10) represents transferable project authority, minting rights, and creator verification. Authority tokens enable governance structures for Glyph projects.

### 18.2 Use Cases

| Use Case | Description |
|----------|-------------|
| **Transferable Project Ownership** | Sell/transfer the right to issue new tokens in a series |
| **Multi-Sig Minting** | Require multiple authority holders to approve new mints |
| **Revocable Delegation** | Grant temporary minting rights that can be revoked |
| **Verified Creator Badge** | Proof of being an authorized issuer for a project |

### 18.3 Authority Types

| Type | Description |
|------|-------------|
| `issuer` | Right to mint new tokens |
| `manager` | Right to manage existing tokens |
| `delegate` | Temporary/limited authority |
| `badge` | Verification only, no permissions |

### 18.4 Authority Metadata

```json
{
  "v": 2,
  "type": "authority",
  "p": [2, 10],
  "name": "Project XYZ Issuer Authority",
  "authority": {
    "type": "issuer",
    "scope": {
      "project": "Project XYZ",
      "containers": ["<container_ref>"],
      "token_types": ["nft", "ft"]
    },
    "permissions": ["mint", "update", "burn", "delegate"],
    "constraints": {
      "max_mints": 10000,
      "expires": null,
      "revocable": true,
      "transferable": true
    },
    "issuer": {
      "ref": null,
      "pubkey": "<creator_pubkey>"
    }
  }
}
```

### 18.5 Authority Hierarchy

**Root Authority**: `issuer.ref: null` indicates original project authority.

**Delegated Authority**: References parent authority:

```json
{
  "authority": {
    "type": "delegate",
    "issuer": {
      "ref": "<parent_authority_ref>",
      "pubkey": "<delegator_pubkey>"
    },
    "permissions": ["mint"],
    "constraints": {
      "max_mints": 100,
      "expires": "2027-01-01T00:00:00Z",
      "revocable": true
    }
  }
}
```

### 18.6 Multi-Sig Authority

For DAO governance:

```json
{
  "authority": {
    "type": "issuer",
    "multisig": {
      "threshold": 3,
      "signers": [
        { "pubkey": "<member1>", "weight": 1 },
        { "pubkey": "<member2>", "weight": 1 },
        { "pubkey": "<member3>", "weight": 1 },
        { "pubkey": "<member4>", "weight": 1 },
        { "pubkey": "<member5>", "weight": 1 }
      ]
    },
    "permissions": ["mint", "update", "delegate"]
  }
}
```

Any 3 of 5 members can authorize actions.

### 18.7 Revocation

Authorities with `revocable: true` can be revoked by their issuer:

```json
{
  "v": 2,
  "p": [10],
  "action": "revoke",
  "target": "<authority_ref_to_revoke>",
  "reason": "Contract ended",
  "auth": {
    "authority": "<issuer_authority_ref>",
    "sig": "<signature>"
  }
}
```

> **Note:** Revocations use `p: [10]` as an **action marker** (similar to burn's `p: [6]`),
> not as a token creation. The `GLYPH_AUTHORITY` (10) requires `GLYPH_NFT` (2) rule
> from §3.5 applies only to authority **token creation**, not to action envelopes.

### 18.8 Container Integration

Containers can require authority for item addition:

```json
{
  "container": {
    "authority": {
      "required": true,
      "accepted": ["<authority_ref_1>", "<authority_ref_2>"]
    }
  }
}
```

### 18.9 Authority Validation

Authority validation uses a two-layer approach:

**Layer 1: Script-Enforced (RECOMMENDED for high-value operations)**

For on-chain enforcement, the container/project contract must reference the authority token:

```
// Container that requires authority for item addition
OP_PUSHINPUTREF <authority_ref>     // Authority token must be in inputs
OP_REFTYPE_UTXO OP_1 OP_EQUALVERIFY // Verify it's a valid ref
OP_PUSHINPUTREFSINGLETON <container_ref>
<container_logic>
```

This ensures the authority holder must spend their authority token alongside the action, providing consensus-level guarantee.

**Layer 2: Indexer-Enforced (for metadata-level permissions)**

For indexer-level validation (lighter weight, off-chain verification):

1. When processing a mint/update that claims authority, indexer checks:
   - Does the referenced authority token exist and is valid?
   - Does the authority have the required `permissions`?
   - Is the authority `expired`?
   - Is the authority `revoked`?
2. Signature verification: transaction signer must match authority holder

**Recommendations**:
- Projects SHOULD use script-enforcement for minting and critical operations
- Projects MAY use indexer-enforcement for metadata updates
- Wallets SHOULD verify authority chain before displaying "verified" badges

See REP-3015 for complete Authority Token specification.

---

## 19. WAVE Naming Integration

### 19.1 Overview

WAVE names are implemented as `GLYPH_NFT` + `GLYPH_WAVE`:
- Claim Tokens are Glyph NFTs
- Zone records stored in mutable metadata
- Prefix tree for name resolution

### 19.2 WAVE Metadata

```json
{
  "v": 2,
  "type": "wave_name",
  "p": [2, 5, 11],
  "name": "alice",
  "app": {
    "namespace": "rxd.wave",
    "schema": "wave_name_v1",
    "data": {
      "name": "alice",
      "parent": null,
      "zone": {
        "address": "1AliceAddress...",
        "avatar": "ipfs://...",
        "A": "192.168.1.1"
      }
    }
  },
  "mutable": {
    "allowed": true,
    "fields": ["/app/data/zone"]
  }
}
```

### 19.3 Resolution

See REP-3011 for complete WAVE specification.

### 19.4 Subdomain Resolution

For subdomains, `parent` references the parent name's GlyphID:

```json
{
  "app": {
    "data": {
      "name": "shop",
      "parent": "<alice_glyph_id>"
    }
  }
}
```

This results in the subdomain "shop.alice".

**Resolution Algorithm**:
1. Find name token by exact `name` match
2. If `parent` is non-null, recursively resolve parent
3. Concatenate: `child.parent` (dot notation)
4. Continue until `parent` is null (root name)

**Example Resolution Chain**:
```
store.shop.alice:
  └─ store (parent: <shop_glyph_id>)
       └─ shop (parent: <alice_glyph_id>)
            └─ alice (parent: null) ← root
```

**Validation**:
- Indexers MUST verify parent name exists and is valid
- Indexers MUST reject circular parent references
- Maximum subdomain depth: 5 levels (RECOMMENDED)

---

## 20. Indexer Implementation

### 20.1 Pipeline

```
1. Scan blocks for Glyph magic bytes
2. Parse envelope (commit or reveal)
3. Validate commit ↔ reveal linkage
4. Decode metadata (CBOR/JSON)
5. Verify content hashes
6. Store in database
7. Process updates
```

### 20.2 Database Schema

Core tables required for Glyph indexing:

| Table | Primary Key | Purpose |
|-------|-------------|---------|
| `glyphs` | `ref` | Token metadata, type, owner |
| `glyph_content` | `(ref, path)` | Content files and hashes |
| `glyph_updates` | `id` | Mutable state change history |
| `burns` | `ref` | Burn records with amounts |
| `containers` | `(container_ref, item_ref)` | Collection membership |
| `authorities` | `ref` | Authority tokens and permissions |

See REP-3004 for complete schema definitions, indexes, and query optimization guidelines.

### 20.3 API Endpoints

```
GET  /glyphs/:ref              — Get glyph by ref
GET  /glyphs?type=nft          — List by type
GET  /glyphs?owner=:address    — List by owner
GET  /glyphs/:ref/history      — Update history
GET  /glyphs/:ref/content/:path — Get content file
GET  /containers/:ref/items    — List container items
GET  /burns?token=:ref         — List burns for token
POST /glyphs/validate          — Validate metadata
```

### 20.4 Content Verification

Indexers MUST verify content hashes according to storage type:

**Inline Content**:
- Verify hash at index time (REQUIRED)
- Reject glyph if hash mismatch

**External Refs (IPFS, HTTP)**:
- Verify on first access (RECOMMENDED)
- Cache verification results
- Re-verify if cache is stale (>24 hours)
- Flag as "unverified" if external content unavailable

**Verification Status**:
```json
{
  "content": {
    "primary": {
      "path": "image.png",
      "verified": true,
      "verified_at": "2026-01-26T12:00:00Z"
    },
    "refs": [
      {
        "path": "video.mp4",
        "verified": false,
        "error": "external_unavailable"
      }
    ]
  }
}
```

---

## 21. Backwards Compatibility

### 21.1 v1 Support Requirements

v2 indexers MUST:
- Recognize v1 magic bytes and version
- Parse v1 metadata format
- Support v1 contract scripts
- Display v1 tokens in queries

### 21.2 v1 to v2 Migration

Existing v1 tokens can be "upgraded" to v2 by:
1. Creating an update transaction (if mutable)
2. Adding v2 metadata fields
3. Maintaining original ref

### 21.3 Wallet Compatibility

| Wallet | v1 Tokens | v2 Tokens |
|--------|-----------|-----------|
| v1 wallet | Full support | Display as basic NFT/FT |
| v2 wallet | Full support | Full support |

### 21.4 Version Detection

Indexers MUST detect Glyph version by checking the `v` field in metadata:

```python
def detect_glyph_version(metadata: dict) -> int:
    """Detect Glyph version from metadata."""
    if 'v' in metadata and metadata['v'] == 2:
        return 2
    return 1  # Legacy v1 (no version field or v=1)
```

**Version-Specific Behavior**:
| Aspect | v1 | v2 |
|--------|-----|-----|
| `v` field | Optional/missing | Required (must be 2) |
| `type` field | Content type ("object") | Optional token type hint |
| `p` array | Protocol IDs | Protocol IDs (authoritative) |
| Content model | `main` shorthand only | `main` or `content` |
| Commit hash | `SHA256(SHA256(cbor))` | `SHA256(SHA256(cbor))` |

### 21.5 Field Migration Guide

Mapping between v1 and v2 field names:

| v1 Field | v2 Equivalent | Notes |
|----------|---------------|-------|
| `main` | `content.primary` or `main` | v2 supports both; `main` is shorthand |
| `main.t` | `content.primary.mime` | MIME type |
| `main.b` | Inline content bytes | Raw bytes in CBOR |
| `in` | `rels.container.ref` | Container reference |
| `by` | `creator.pubkey` | Creator identification |
| `attrs` | `app.data` or root `attrs` | Custom attributes |
| `type` | Derived from `p` array | v2 uses protocols as authoritative |
| `desc` or `description` | `desc` | Description field |
| `image` or `icon` | `content.primary` or `preview.thumb` | Image content |

**Migration Example**:

*v1 Metadata*:
```json
{
  "p": [2],
  "name": "My NFT",
  "type": "object",
  "main": { "t": "image/png", "b": "<bytes>" },
  "in": ["<container_ref_bytes>"],
  "by": ["<author_ref_bytes>"]
}
```

*v2 Equivalent*:
```json
{
  "v": 2,
  "p": [2],
  "name": "My NFT",
  "main": { "t": "image/png", "b": "<bytes>" },
  "rels": {
    "container": { "ref": "<container_token_ref>" }
  },
  "creator": { "pubkey": "<author_pubkey_hex>" }
}
```

### 21.6 Container Reference Format

**v1 Format**: Container references stored as byte arrays in `in` field:
```json
{ "in": [Uint8Array([...])] }
```

**v2 Format**: Structured object with optional index:
```json
{
  "rels": {
    "container": {
      "ref": "<txid>:<vout>",
      "index": 42
    }
  }
}
```

**Indexer Normalization**: When processing v1 tokens, indexers SHOULD normalize container references:
1. Convert byte array to `txid:vout` string format
2. Store in `rels.container.ref` internally
3. Set `index` to null if not provided

---

## 22. Security Considerations

### 22.1 Metadata Validation

- Enforce max sizes for all fields
- Sanitize strings before display
- Validate content hashes
- Reject malformed CBOR

### 22.2 Executable Content

- Require `policy.executable: true` for code
- Warn users before execution
- Sandbox execution environments
- Default to non-executable

### 22.3 Royalty Bypass

On-chain enforcement prevents bypass for compliant contracts. Non-enforced royalties rely on:
- Marketplace enforcement
- Community reputation
- Economic incentives

### 22.4 Front-Running

Commit-reveal pattern mitigates front-running:
- Commit publishes hash only
- Reveal exposes content
- Miners cannot preview content

### 22.5 Key Management

For encrypted content:
- Never store plaintext keys on-chain (except for reveals)
- Use strong passphrases for KDF
- Ephemeral keys for recipient wrapping
- Nonce uniqueness is critical

### 22.6 Error Handling

Indexers MUST reject and not index a Glyph if:
- `v` is missing or not equal to `2`
- `p` array is missing or empty
- `name` exceeds 256 bytes
- Protocol combination is invalid (see Section 3.5)
- Any `content.*.hash.hex` is malformed (invalid hex, wrong length)
- Total canonical metadata exceeds 256 KB

**Type-Specific Content Requirements**:

| Token Type | Content Requirement |
|------------|---------------------|
| NFT (`p` includes 2) | `content` with `primary` REQUIRED, **or** `main` shorthand present (see §8.4) |
| FT (`p` includes 1) | `content` OPTIONAL; `ticker` REQUIRED if no content |
| DAT (`p` includes 3) | `content` OPTIONAL |
| Container (`p` includes 7) | `content` OPTIONAL; `container` object REQUIRED |

Indexers MUST validate content requirements based on token type.

Indexers SHOULD index but flag as "malformed" if:
- Unknown compression or encoding values are used
- `created` timestamp is in the future (>24h)
- `content.*.size` doesn't match actual payload size

Indexers MUST ignore unknown top-level keys and preserve them when re-serializing for updates.

---

## 23. Reference Implementation

### 23.1 Repositories

- **Radiant Core**: Consensus node — [github.com/AustEcon/Radiant-Core](https://github.com/AustEcon/Radiant-Core)
- **Photonic Wallet**: Token creation and management — [github.com/AustEcon/Photonic-Wallet](https://github.com/AustEcon/Photonic-Wallet)
- **Glyph Miner**: dMint mining client — [github.com/AustEcon/Glyph-miner](https://github.com/AustEcon/Glyph-miner)
- **RXinDexer**: Reference indexer implementation — [github.com/AustEcon/RXinDexer](https://github.com/AustEcon/RXinDexer)
- **radiantjs**: JavaScript library — [github.com/AustEcon/radiantjs](https://github.com/AustEcon/radiantjs)
- **RadiantScript**: Smart contract compiler — [github.com/AustEcon/RadiantScript](https://github.com/AustEcon/RadiantScript)

### 23.2 Libraries

**TypeScript/JavaScript**:
```typescript
import { encodeGlyph, decodeGlyph } from '@radiant/glyph';
import { GLYPH_NFT, GLYPH_FT, GLYPH_BURN } from '@radiant/glyph/protocols';

const metadata = {
  v: 2,
  type: 'nft',
  p: [GLYPH_NFT],
  name: 'My NFT',
  content: {
    primary: {
      path: 'image.png',
      mime: 'image/png',
      storage: 'inline'
    }
  }
};

const { payloadHash, revealScriptSig } = encodeGlyph(metadata);
```

### 23.3 Test Vectors

See REP-3003 for comprehensive test vectors covering:
- CBOR encoding
- Commit hash calculation
- Envelope construction
- Validation cases

---

## 24. Appendices

### Appendix A: Protocol ID Registry

| ID | Name | REP | Status |
|----|------|-----|--------|
| 1 | GLYPH_FT | 3001 | Active |
| 2 | GLYPH_NFT | 3001 | Active |
| 3 | GLYPH_DAT | 3001 | Active |
| 4 | GLYPH_DMINT | 3010 | Active |
| 5 | GLYPH_MUT | 3001 | Active |
| 6 | GLYPH_BURN | 3014 | Active |
| 7 | GLYPH_CONTAINER | 3013 | Active |
| 8 | GLYPH_ENCRYPTED | 3006 | Active |
| 9 | GLYPH_TIMELOCK | 3009 | Active |
| 10 | GLYPH_AUTHORITY | 3015 | Active |
| 11 | GLYPH_WAVE | 3011 | Active |

### Appendix B: CBOR Encoding Rules

1. Map keys sorted lexicographically (UTF-8 bytes)
2. Integers use minimal encoding
3. No indefinite-length arrays/maps/strings
4. No duplicate map keys
5. No NaN or Infinity floats

### Appendix C: Size Limits Summary

| Component | Limit | Notes |
|-----------|-------|-------|
| `name` | 256 bytes | UTF-8 encoded |
| `desc` | 4 KB | UTF-8 encoded |
| `path` | 512 bytes | UTF-8 encoded |
| `mime` | 128 bytes | ASCII |
| Total metadata | 256 KB | Canonical CBOR bytes |
| Commit envelope | 100 KB | Style A or B |
| Reveal envelope (A) | 100 KB | OP_RETURN style |
| Reveal envelope (B) | 12 MB | Limited by MAX_TX_SIZE |
| Update envelope | 64 KB | Incremental changes |
| Inline file | 1 MB | Per file |
| Total inline | 10 MB | All files combined |

**Consensus Limits** (from Radiant-Core):
| Limit | Value |
|-------|-------|
| `MAX_TX_SIZE` | 12 MB |
| `MAX_STANDARD_TX_SIZE` | 20 MB |

### Appendix D: Related REPs

| REP | Title |
|-----|-------|
| 3001 | Glyph Protocol v2 Core |
| 3002 | Glyph v2 Envelope Formats |
| 3003 | Glyph v2 Test Vectors |
| 3004 | Glyph v2 Indexer Guide |
| 3005 | Game Item Profile |
| 3006 | Encrypted Glyphs |
| 3007 | Encryption Test Vectors |
| 3008 | Multi-Recipient Key Wrap |
| 3009 | Timelocked Key Reveals |
| 3010 | dMint Enhancements |
| 3011 | WAVE Naming System |
| 3012 | On-Chain Royalties |
| 3013 | Containers and Collections |
| 3014 | Burn Mechanism |
| 3015 | Authority Tokens |

### Appendix E: V2 dMint Contract Bytecodes

#### E.1 V1 Contract Bytecode (SHA256d, Fixed Difficulty)

Used for: v1 tokens and v2 tokens with `algo=SHA256d` + `daa=fixed`.

> **Note:** This hex represents the bytecode **after** `OP_STATESEPARATOR` (0xbd).
> The full script includes the state data prefix and `bd` separator before this bytecode.

```
5175c0c855797ea8597959797ea87e5a7a7eaabc01147f77587f
040000000088817600a269a269577ae500a069567ae600a06901
d053797e0cdec0e9aa76e378e4a269e69d7eaa76e47b9d547a81
8b76537a9c537ade789181547ae6939d635279cd01d853797e01
6a7e886778de519d547854807ec0eb557f777e5379ec78885379
eac0e9885379cc519d75686d7551
```

#### E.2 V2 Contract Bytecode Template (Pseudocode)

All v2 contracts share the same structure, differing only in the hash opcode used for PoW
validation. The `<powHashOp>` placeholder is replaced per algorithm:

| Algorithm | `<powHashOp>` | Opcode Byte |
|-----------|--------------|-------------|
| SHA256d | `OP_HASH256` | `0xaa` |
| Blake3 | `OP_BLAKE3` | `0xee` |
| K12 | `OP_K12` | `0xef` |

**V2 contract pseudocode (with on-chain DAA):**

```
// ═══ PART A: State validation + PoW check ═══════════
// Stack from scriptSig: <nonce> <preimage>

// 1. Verify height < maxHeight
<old_height> OP_1ADD                    // new_height
OP_DUP <maxHeight> OP_LESSTHAN OP_VERIFY

// 2. Validate PoW
OP_OVER OP_OVER OP_CAT                 // preimage || nonce
<powHashOp>                             // hash(preimage || nonce)
OP_DUP                                  // keep hash for target check

// 3. Check hash < target (256-bit comparison for blake3/k12)
//    For SHA256d: first 4 bytes == 0, next 8 bytes < target
<target> OP_LESSTHAN OP_VERIFY

// 4. Verify contractRef singleton preserved
OP_PUSHINPUTREFSINGLETON <contractRef>
OP_DROP

// 5. Verify tokenRef output with correct reward
OP_PUSHINPUTREF <tokenRef>
OP_DROP

// ═══ PART B: DAA computation (see §11.4.1) ══════════
// Read old state: lastTime, targetTime
// Compute new target via ASERT-lite or Linear DAA
// (exact bytecode per §11.4.1)

// ═══ PART C: Output script continuity ════════════════
// Verify output script matches input (except mutable state)
// Store: new_height, new_target, current_time as lastTime
```

**Activation:** Block **410,000** on mainnet (Radiant Core `radiantCore2UpgradeHeight`).

#### E.3 Contract Script Layout Summary

```
┌──── State Data (mutable per mint) ────────────────────┐
│ <height:4B>                                            │
│ OP_PUSHINPUTREFSINGLETON(d8) <contractRef:36B>         │
│ OP_PUSHINPUTREF(d0) <tokenRef:36B>                     │
│ <maxHeight:minimal>  <reward:minimal>  <target:minimal>│
│ <algoId:1B>  <lastTime:4B>  <targetTime:minimal>       │  ← v2 only
├──── Separator ─────────────────────────────────────────┤
│ OP_STATESEPARATOR(bd)                                  │
├──── Bytecode (immutable) ──────────────────────────────┤
│ PART_A: state validation + PoW (<powHashOp>) check     │
│ PART_B: on-chain DAA (OP_LSHIFT/OP_RSHIFT or MUL/DIV) │  ← v2 only
│ PART_C: output script continuity + token distribution  │
└────────────────────────────────────────────────────────┘
```

> **Note:** The v2 contract bytecodes have been verified on 2-node regtest (all 7 integration
> scenarios A–G passed, Feb 2026). The Photonic Wallet contains the production contract
> bytecode templates. See the [V2 dMint Design Specification](../Glyph-miner/docs/V2_DMINT_DESIGN.md)
> for buffer formats and miner integration details.

---

## References

1. Satoshi Nakamoto, "Bitcoin: A Peer-to-Peer Electronic Cash System," 2008
2. "Radiant: A Peer-to-Peer Digital Asset System," 2022
3. RFC 8949: Concise Binary Object Representation (CBOR)
4. RFC 8785: JSON Canonicalization Scheme (JCS)
5. RFC 7693: The BLAKE2 Cryptographic Hash
6. RFC 9106: Argon2 Memory-Hard Function
7. BLAKE3 — https://github.com/BLAKE3-team/BLAKE3
8. KangarooTwelve — https://keccak.team/kangarootwelve.html

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 2.0-draft-8 | 2026-02-12 | **Induction Proofs**: Added §11.8 Induction Proofs section covering Method 1 (reference-based, OP_PUSHINPUTREF family), Method 2 (TxId v3 preimage embedding with 112-byte layout), OP_PUSH_TX_STATE transaction state introspection (tx.state.txId, tx.state.inputSum, tx.state.outputSum), code-continuity induction pattern, and method comparison table. Added RadiantScript compiler support for tx.state.* nullary operators. Added InductionProof.rxd example contract. |
| 2.0-draft-7 | 2026-02-12 | **Production Ready**: Updated status to Release Candidate. Added V2 hard fork summary to Executive Summary. Fixed §12.2 burn example (clarified fee source, added fee funding UTXO). Fixed §22.6 NFT content requirement to accept `main` shorthand (§8.4). Swapped §5 table rows (consensus MAX_TX_SIZE first). Added DAA mode string↔integer mapping table (§11.7). Added Linear DAA timestamp extraction note (§11.4.1). Added signature format/size table to §8.3. Clarified §14.3 child `p` array (CONTAINER is parent-only). Added §15.4 update ordering rules (tx index within block). Clarified §18.7 revocation `p:[10]` as action marker. Added GitHub repository URLs to §23.1. Updated Appendix E note (Phase 10 complete, bytecodes verified on regtest). |
| 2.0-draft-6 | 2026-02-10 | **Audit Fixes**: Fixed §22.6 error handling (removed `type`/`policy` mandatory rejection; `p` array is authoritative per §8.2). Fixed §8.3 creator signature to use double SHA256 matching §6.2 commit hash. Fixed §11.4.1 ASERT-lite stack tracking bug (incorrect OP_SUB operand order). Added OP_ROT/OP_SWAP for correct stack rearrangement before shift. Updated §1.3 to list all 6 fork-gated opcodes. Added OP_2MUL/OP_2DIV note in §11.4.1. Added algorithm name mapping table in §11.7. Removed deferred Argon2id from §11.6 collision table. Added §11.4.1 and Appendix E to ToC. Fixed §7.2 MAX_STANDARD_TX_SIZE wording. Updated header to draft-6/Feb 2026. |
| 2.0-draft-5 | 2026-02-10 | **DAA Bytecode Spec & Contract Appendix**: Added §11.4.1 On-Chain DAA Bytecode Specification (ASERT-lite pseudocode with OP_LSHIFT/OP_RSHIFT, linear DAA alternative, security constraints table). Added Appendix E: V2 dMint Contract Bytecodes (v1 hex reference, v2 pseudocode template with `<powHashOp>` per algorithm, script layout summary). Activation height: block 410,000 on mainnet. |
| 2.0-draft-4 | 2026-02-10 | **Hard Fork Integration**: Updated §1.3 to acknowledge V2 consensus changes. Updated §11.2 POW algorithm table with OP_BLAKE3/OP_K12 implementation status. Revised §11.4 intro to reflect fully on-chain PoW validation (no indexer trust). Added BLAKE3 and KangarooTwelve references. |
| 2.0-draft-3 | 2026-01-30 | **Compatibility Updates**: Changed commit hash to double SHA256 for v1 compatibility (Section 6.2). Added hybrid content model with v1-compatible `main` shorthand (Section 8.4). Made `type` field optional with `p` array as authoritative (Section 8.2). Added envelope style detection algorithm (Section 7.4). Added TIMELOCK requires ENCRYPTED validation (Section 3.5). Clarified BURN as action marker. Added version detection, field migration guide, and container reference format sections (Sections 21.4-21.6). |
| 2.0-draft-2 | 2026-01-26 | Added: Section 3.5 Protocol Combination Rules, Section 8.7 Soulbound Tokens, Section 9.4 Photon-Backed Decimals, Section 12.6 Burn Reorg Handling, Section 16.5 Minimum Encryption Parameters, Section 18.9 Authority Validation, Section 19.4 Subdomain Resolution, Section 20.4 Content Verification. Updated: Size limits to match Radiant-Core consensus (12 MB MAX_TX_SIZE), commit_outpoint description, creator signature pseudocode, timelock AND logic, type-specific content requirements, policy.transferable field. |
| 2.0-draft | 2026-01-25 | Initial Glyph v2 specification |

---

## License

This specification is released under the MIT License.

---

*Glyph v2: Powering the next generation of digital assets on Radiant.*
