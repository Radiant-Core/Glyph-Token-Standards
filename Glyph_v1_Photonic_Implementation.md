# Glyph v1 Token Standard - Photonic Wallet Implementation

This document describes the **original Glyph v1** token standard as implemented in Photonic Wallet v1. This serves as the reference implementation for Glyph v1 tokens on the Radiant blockchain.

> **Note:** This documents Glyph **v1**. For the enhanced Glyph v2 standard with additional protocols (BURN, CONTAINER, ENCRYPTED, TIMELOCK, AUTHORITY, WAVE), see the [Glyph v2 Token Standard Whitepaper](./Glyph_v2_Token_Standard_Whitepaper.md).

---

## Overview

Photonic Wallet v1 implements Glyph v1 token support through these modules:
- **Token parsing** (`packages/lib/src/token.ts`) - Glyph envelope detection and CBOR decoding
- **Script handling** (`packages/lib/src/script.ts`) - Script patterns for NFT, FT, and mutable tokens
- **Protocol definitions** (`packages/lib/src/protocols.ts`) - Protocol ID constants
- **Token discovery** (`packages/app/src/electrum/worker/NFT.ts`) - Blockchain querying and indexing

**Source:** `/Users/main/Downloads/Photonic-Wallet-v1`

---

## 1. Glyph Magic Bytes

Glyph tokens are identified by magic bytes in the transaction script:

```typescript
// ASCII 'gly' = 0x67 0x6c 0x79
export const glyphMagicBytesHex = "676c79";
export const glyphMagicBytesBuffer = Buffer.from(glyphMagicBytesHex, "hex");
```

The magic bytes `gly` (hex: `676c79`) mark the beginning of a Glyph envelope in a script.

---

## 2. Glyph v1 Protocol IDs

Glyph v1 defines **5 protocol IDs**:

```typescript
// From protocols.ts
export const GLYPH_FT = 1;      // Fungible Token
export const GLYPH_NFT = 2;     // Non-Fungible Token
export const GLYPH_DAT = 3;     // Data Storage
export const GLYPH_DMINT = 4;   // Decentralized Minting
export const GLYPH_MUT = 5;     // Mutable State
```

| ID | Constant | Description |
|----|----------|-------------|
| 1 | `GLYPH_FT` | Fungible Token - divisible, value-conserving |
| 2 | `GLYPH_NFT` | Non-Fungible Token - unique singleton |
| 3 | `GLYPH_DAT` | Data Storage - no ref created |
| 4 | `GLYPH_DMINT` | Decentralized Minting (PoW mining) |
| 5 | `GLYPH_MUT` | Mutable State - updateable NFT |

### Protocol Combinations

- **FT + DMINT** = Mineable fungible token
- **NFT + MUT** = Mutable/updateable NFT
- **NFT alone** = Immutable NFT

---

## 3. Glyph Envelope Format

### 3.1 Script Structure

A Glyph envelope in the reveal transaction input script:

```
[OP_PUSHBYTES_3] [0x67 0x6c 0x79] [OP_PUSHDATA...] [CBOR_PAYLOAD]
```

1. **Magic bytes** (`gly`) - 3 bytes via opcode 0x03 (OP_PUSHBYTES_3)
2. **CBOR payload** - Variable length, CBOR-encoded metadata

### 3.2 Decoding Process

The `decodeGlyph()` function extracts the Glyph from a script:

```typescript
export function decodeGlyph(script: Script): undefined | DecodedGlyph {
  const result: { payload: object } = { payload: {} };
  
  (script.chunks as { opcodenum: number; buf?: Uint8Array }[])
    .some(({ opcodenum, buf }, index) => {
      // Look for 'gly' magic (opcode 3 = 3-byte push)
      if (
        !buf ||
        opcodenum !== 3 ||
        Buffer.from(buf).toString("hex") !== glyphMagicBytesHex ||
        script.chunks.length <= index + 1
      ) {
        return false;
      }

      // Next chunk is the CBOR payload
      const payload = script.chunks[index + 1];
      if (!payload.buf) return false;
      
      const decoded = decode(Buffer.from(payload.buf));
      if (!decoded) return false;

      result.payload = decoded;
      return true;
    });

  // Separate files from metadata
  const { p, attrs, ...rest } = result.payload as { [key: string]: unknown };
  
  const { meta, embeds, remotes } = Object.entries(rest).reduce(
    (a, [k, v]) => {
      const { embed, remote } = filterFileObj(v);
      if (embed) a.embeds.push([k, embed]);
      else if (remote) a.remotes.push([k, remote]);
      else a.meta.push([k, v]);
      return a;
    },
    { meta: [], embeds: [], remotes: [] }
  );

  return {
    payload: {
      p: Array.isArray(p) ? p.filter(v => ["string", "number"].includes(typeof v)) : [],
      attrs: toObject(attrs),
      ...Object.fromEntries(meta),
    },
    embeddedFiles: Object.fromEntries(embeds),
    remoteFiles: Object.fromEntries(remotes),
  };
}
```

### 3.3 Decoded Result Structure

```typescript
export type DecodedGlyph = {
  payload: SmartTokenPayload;
  embeddedFiles: { [key: string]: SmartTokenEmbeddedFile };
  remoteFiles: { [key: string]: SmartTokenRemoteFile };
};
```

---

## 4. CBOR Payload Structure

### 4.1 SmartTokenPayload Type

```typescript
export type SmartTokenPayload = {
  p: (string | number)[];  // Protocol IDs array (required)
  in?: Uint8Array[];       // Container references
  by?: Uint8Array[];       // Author references  
  attrs?: {                // Custom attributes
    [key: string]: unknown;
  };
  [key: string]: unknown;  // Additional properties
};
```

### 4.2 Standard Metadata Fields

| Field | Type | Description |
|-------|------|-------------|
| `p` | `number[]` | **Required.** Protocol IDs (e.g., `[2]` for NFT) |
| `name` | `string` | Token name (max 80 chars displayed) |
| `desc` | `string` | Description (max 1000 chars displayed) |
| `ticker` | `string` | Ticker symbol for FT (max 20 chars) |
| `type` | `string` | Content type (default: "object") |
| `loc` | `number` | Location vout for linked tokens |
| `in` | `Uint8Array[]` | Container ref(s) this token belongs to |
| `by` | `Uint8Array[]` | Author/creator ref(s) |
| `attrs` | `object` | Custom key-value attributes |

### 4.3 Embedded Files

Files embedded directly in the CBOR payload:

```typescript
export type SmartTokenEmbeddedFile = {
  t: string;      // MIME type (e.g., "image/png")
  b: Uint8Array;  // Binary content
};
```

Example in payload:
```javascript
{
  p: [2],
  name: "My NFT",
  main: {           // 'main' is the primary file
    t: "image/png",
    b: Uint8Array([0x89, 0x50, 0x4e, 0x47, ...])
  }
}
```

### 4.4 Remote Files

Files referenced by URL:

```typescript
export type SmartTokenRemoteFile = {
  t: string;       // MIME type
  u: string;       // URL
  h?: Uint8Array;  // Content hash (optional)
  hs?: Uint8Array; // Hash of hash (optional)
};
```

---

## 5. Token Script Patterns

### 5.1 NFT Script (Singleton)

NFTs use `OP_PUSHINPUTREFSINGLETON` ensuring uniqueness:

```typescript
export function nftScript(address: string, ref: string) {
  const script = Script.fromASM(
    `OP_PUSHINPUTREFSINGLETON ${ref} OP_DROP`
  ).add(Script.buildPublicKeyHashOut(address));
  return script.toHex();
}
```

**Hex Pattern:** `d8[72-char ref]7576a914[40-char pubkeyhash]88ac`

**Parsing:**
```typescript
export function parseNftScript(script: string): { ref?: string; address?: string } {
  const pattern = /^d8([0-9a-f]{72})7576a914([0-9a-f]{40})88ac$/;
  const [, ref, address] = script.match(pattern) || [];
  return { ref, address };
}
```

### 5.2 FT Script (Normal Ref)

FTs use `OP_PUSHINPUTREF` with value conservation rules:

```typescript
export function ftScript(address: string, ref: string) {
  const script = Script.buildPublicKeyHashOut(address).add(
    Script.fromASM(
      `OP_STATESEPARATOR OP_PUSHINPUTREF ${ref} ` +
      `OP_REFOUTPUTCOUNT_OUTPUTS OP_INPUTINDEX OP_CODESCRIPTBYTECODE_UTXO OP_HASH256 ` +
      `OP_DUP OP_CODESCRIPTHASHVALUESUM_UTXOS OP_OVER OP_CODESCRIPTHASHVALUESUM_OUTPUTS ` +
      `OP_GREATERTHANOREQUAL OP_VERIFY OP_CODESCRIPTHASHOUTPUTCOUNT_OUTPUTS OP_NUMEQUALVERIFY`
    )
  );
  return script.toHex();
}
```

**Hex Pattern:** `76a914[40-char address]88acbdd0[72-char ref]dec0e9aa76e378e4a269e69d`

**Parsing:**
```typescript
export function parseFtScript(script: string): { ref?: string; address?: string } {
  const pattern = /^76a914([0-9a-f]{40})88acbdd0([0-9a-f]{72})dec0e9aa76e378e4a269e69d$/;
  const [, address, ref] = script.match(pattern) || [];
  return { ref, address };
}
```

### 5.3 NFT Commit Script

The commit transaction validates the payload hash before allowing reveal:

```typescript
export function nftCommitScript(
  address: string,
  payloadHash: string,
  delegateRef: string | undefined
) {
  const script = new Script();

  if (delegateRef) {
    addDelegateRefScript(script, delegateRef);
  }

  // Verify payload hash
  script
    .add(Opcode.OP_HASH256)
    .add(Buffer.from(payloadHash, "hex"))
    .add(Opcode.OP_EQUALVERIFY);
  
  // Verify 'gly' magic
  script.add(glyphMagicBytesBuffer).add(Opcode.OP_EQUALVERIFY);
  
  // Ensure singleton ref created in output (reftype 2)
  script.add(Script.fromASM(
    "OP_INPUTINDEX OP_OUTPOINTTXHASH OP_INPUTINDEX OP_OUTPOINTINDEX " +
    "OP_4 OP_NUM2BIN OP_CAT OP_REFTYPE_OUTPUT OP_2 OP_NUMEQUALVERIFY"
  ));

  // P2PKH spending condition
  script.add(Script.buildPublicKeyHashOut(Address.fromString(address)));
  
  return script.toHex();
}
```

### 5.4 Mutable NFT Contract

Mutable tokens use a contract that validates state updates:

```typescript
export function mutableNftScript(mutableRef: string, payloadHash: string) {
  /* ScriptSig format:
   * gly
   * <cbor payload>
   * mod | sl (operation: modify or seal)
   * <contract output index>
   * <ref+hash index in token output>
   * <ref index in token output data summary>
   * <token output index>
   */
  return Script.fromASM([
    `${payloadHash} OP_DROP`,                              // State (current payload hash)
    `OP_STATESEPARATOR OP_PUSHINPUTREFSINGLETON ${mutableRef}`,
    `OP_DUP 20 OP_SPLIT OP_BIN2NUM OP_1SUB OP_4 OP_NUM2BIN OP_CAT`,
    // ... validation logic for mod/seal operations
    `OP_4 OP_ROLL ${glyphMagicBytesHex} OP_EQUALVERIFY OP_2DROP OP_2DROP OP_1`,
  ].join(" ")).toHex();
}
```

### 5.5 dMint Script (v1)

Decentralized minting with SHA256d PoW (v1 only supports SHA256d):

```typescript
export function dMintScript(
  height: number,
  contractRef: string,
  tokenRef: string,
  maxHeight: number,
  reward: number,
  target: bigint
) {
  return `${push4bytes(height)}d8${contractRef}d0${tokenRef}${pushMinimal(
    maxHeight
  )}${pushMinimal(reward)}${pushMinimal(
    target
  )}bd5175c0c855797ea8597959797ea87e5a7a7eaabc01147f77587f040000000088817600a269a269577ae500a069567ae600a06901d053797e0cdec0e9aa76e378e4a269e69d7eaa76e47b9d547a818b76537a9c537ade789181547ae6939d635279cd01d853797e016a7e886778de519d547854807ec0eb557f777e5379ec78885379eac0e9885379cc519d75686d7551`;
}
```

**Note:** v1 dMint only supports SHA256d algorithm with fixed difficulty. v2 adds Blake3, K12, Argon2 algorithms and dynamic difficulty adjustment.

---

## 6. Token Discovery Flow

### 6.1 Process Overview

```
1. Wallet subscribes to NFT script hash pattern via ElectrumX
2. ElectrumX notifies of UTXOs matching the pattern
3. Wallet parses output script to extract singleton ref
4. Wallet calls blockchain.ref.get to find reveal tx
5. Wallet fetches reveal transaction
6. Wallet finds input spending the commit UTXO
7. Glyph payload extracted from that input's scriptSig
8. CBOR decoded to get token metadata
9. Token stored in local IndexedDB
```

### 6.2 Reveal Payload Extraction

```typescript
export function extractRevealPayload(
  ref: string,
  inputs: rjs.Transaction.Input[]
) {
  const refTxId = ref.substring(0, 64);
  const refVout = parseInt(ref.substring(64), 16);

  // Find input that spends the commit UTXO
  const revealIndex = inputs.findIndex((input) => {
    return (
      input.prevTxId.toString("hex") === refTxId &&
      input.outputIndex === refVout
    );
  });
  
  const script = revealIndex >= 0 && inputs[revealIndex].script;
  if (!script) {
    return { revealIndex: -1 };
  }

  return { revealIndex, glyph: decodeGlyph(script) };
}
```

### 6.3 ElectrumX API Calls

| Method | Purpose |
|--------|---------|
| `blockchain.scripthash.subscribe` | Watch for new UTXOs |
| `blockchain.ref.get` | Get reveal tx for a ref |
| `blockchain.transaction.get` | Fetch full transaction |

---

## 7. Reference Format

### 7.1 Outpoint Reference

A ref is 36 bytes: `txid (32 bytes) + vout (4 bytes big-endian)`

```typescript
// Big-endian format for display/storage
const ref = "abc123...def45600000001"; // txid + vout=1

// Little-endian for on-chain use
const refLE = Outpoint.fromString(ref).reverse().toString();
```

### 7.2 Ref Types

| OP_REFTYPE | Value | Description |
|------------|-------|-------------|
| Normal | 1 | FT refs - can exist in multiple outputs |
| Singleton | 2 | NFT refs - unique, only one can exist |

---

## 8. Encoding a New Glyph

```typescript
export function encodeGlyph(payload: unknown) {
  const encodedPayload = encode(payload);  // CBOR encode
  return {
    revealScriptSig: new Script()
      .add(glyphMagicBytesBuffer)
      .add(encodedPayload)
      .toHex(),
    payloadHash: bytesToHex(sha256(sha256(Buffer.from(encodedPayload)))),
  };
}
```

The `payloadHash` (double SHA256 of CBOR) is committed in the commit script.

---

## 9. Token Type Detection

```typescript
export function isImmutableToken({ p }: SmartTokenPayload) {
  // Mutable tokens require both NFT and MUT protocols
  return !(p.includes(GLYPH_NFT) && p.includes(GLYPH_MUT));
}
```

Token type determination in v1:
```typescript
function getTokenType(protocols: number[]): string {
  if (protocols.includes(GLYPH_FT)) {
    if (protocols.includes(GLYPH_DMINT)) return 'dMint FT';
    return 'Fungible Token';
  }
  if (protocols.includes(GLYPH_NFT)) {
    if (protocols.includes(GLYPH_MUT)) return 'Mutable NFT';
    return 'NFT';
  }
  if (protocols.includes(GLYPH_DAT)) return 'Data';
  return 'Unknown';
}
```

---

## 10. Example Payloads

### Simple NFT

```javascript
{
  p: [2],                    // GLYPH_NFT
  name: "My NFT",
  desc: "A sample NFT token"
}
```

### NFT with Embedded Image

```javascript
{
  p: [2],                    // GLYPH_NFT
  name: "Art NFT",
  desc: "Digital artwork",
  attrs: {
    rarity: "legendary",
    edition: 1
  },
  main: {
    t: "image/png",
    b: Uint8Array([...])     // PNG binary
  }
}
```

### Fungible Token

```javascript
{
  p: [1],                    // GLYPH_FT
  name: "My Token",
  ticker: "MYT",
  desc: "A sample fungible token"
}
```

### Mutable NFT

```javascript
{
  p: [2, 5],                 // GLYPH_NFT + GLYPH_MUT
  name: "Upgradeable NFT",
  desc: "This NFT can be updated",
  level: 1,
  attrs: { power: 100 }
}
```

### dMint Token (Mineable FT)

```javascript
{
  p: [1, 4],                 // GLYPH_FT + GLYPH_DMINT
  name: "Mineable Token",
  ticker: "MINE",
  desc: "Proof-of-work mineable token"
}
```

---

## 11. v1 vs v2 Comparison

| Feature | Glyph v1 | Glyph v2 |
|---------|----------|----------|
| Protocol IDs | 5 (FT, NFT, DAT, DMINT, MUT) | 11 (adds BURN, CONTAINER, ENCRYPTED, TIMELOCK, AUTHORITY, WAVE) |
| dMint Algorithms | SHA256d only | SHA256d, Blake3, K12, Argon2id-Light |
| dMint DAA | Fixed difficulty | Fixed, Epoch, ASERT, LWMA, Schedule |
| Magic bytes | `gly` (676c79) | `gly` (676c79) - same |
| Encoding | CBOR | CBOR - same |
| Script patterns | Same | Same base patterns |

---

## 12. Dependencies

- **cbor-x** - CBOR encoding/decoding
- **@noble/hashes** - SHA256 hashing
- **@radiantblockchain/radiantjs** - Script parsing, transaction handling
- **@bitauth/libauth** - Data push encoding

---

## References

- **Source:** `Photonic-Wallet-v1` at `/Users/main/Downloads/Photonic-Wallet-v1`
- [Glyph v2 Token Standard Whitepaper](./Glyph_v2_Token_Standard_Whitepaper.md)
- [Radiant Blockchain](https://radiantblockchain.org)
