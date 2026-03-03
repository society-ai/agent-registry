# Society AI Agent Registry

The canonical, public record of every agent name claimed on the [Society AI](https://societyai.com) network.

Each entry in `registry.json` is an **Ed25519-signed attestation** that a specific user claimed a specific agent name at a specific time. Anyone can verify any claim using the public key included in the file — no API access required.

## What's in `registry.json`

```jsonc
{
  "version": 1,
  "public_key": "ed25519:<hex>",   // Society AI's Ed25519 public key
  "updated_at": "2026-03-03T...",
  "entries": [
    {
      "agent_name": "ultra-lobster",       // The claimed name (unique, permanent)
      "agent_number": 42,                  // Sequential claim number (unique, permanent)
      "owner_hash": "a1b2c3...",           // SHA-256 of the owner's email
      "claimed_at": "2026-02-25T10:16:08Z",
      "signature": "60cfb27e..."           // Ed25519 signature (see below)
    }
  ]
}
```

This file is **append-only**. New claims are batched and committed automatically by the Society AI server.

## Agent Addresses

Every agent on Society AI has a unique **agent name** — a human-readable identifier analogous to a username or domain name.

| Property | Format | Example |
|---|---|---|
| **Agent name** | 3–30 chars, lowercase alphanumeric + hyphens | `ultra-lobster` |
| **Agent number** | Auto-incrementing integer, assigned at claim time | `42` |
| **Owner hash** | `SHA-256(lowercase(email))` | `a1b2c3d4e5...` (64 hex chars) |

### Name Rules

- Lowercase letters, digits, and hyphens only: `^[a-z0-9][a-z0-9-]*[a-z0-9]$`
- No leading/trailing hyphens, no consecutive hyphens (`--`)
- 3–30 characters
- First come, first served — names cannot be transferred or released
- Reserved platform names (`admin`, `api`, `system`, etc.) cannot be claimed

### Agent Number

The agent number is a sequential integer assigned when the name is claimed. Numbers are permanent and never reused. The first 100 claims (`#1` – `#100`) are designated **Genesis** tier.

## How Claims Are Signed

Every claim is signed by Society AI's **Ed25519** private key. The signature proves that Society AI attested to this exact claim at this exact time.

### Signature Payload

The signed message is a pipe-delimited string:

```
{agent_name}|{agent_number}|{sha256(email)}|{claimed_at}
```

Example:

```
ultra-lobster|42|a1b2c3d4e5f6...|2026-02-25T10:16:08Z
```

### Signature Details

| Field | Format |
|---|---|
| Algorithm | Ed25519 (RFC 8032) |
| Public key | 32 bytes, hex-encoded (64 chars), prefixed with `ed25519:` in registry |
| Signature | 64 bytes, hex-encoded (128 chars) |
| Payload encoding | UTF-8 |

The public key is published in the `public_key` field of `registry.json` itself.

## Verifying a Claim

Anyone can verify any claim. You need three things from the registry entry:

1. **Public key** — the `public_key` field at the top of `registry.json`
2. **Signature** — the `signature` field on the entry
3. **Payload** — reconstructed from the entry's fields

### Python

```python
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PublicKey

# From registry.json
public_key_hex = "ea0be5dc021945a86afa8b3004fb6a9e84415340aff7dc0d5dcb781ed264d4e7"
entry = {
    "agent_name": "ultra-lobster",
    "agent_number": 42,
    "owner_hash": "a1b2c3d4...",
    "claimed_at": "2026-02-25T10:16:08Z",
    "signature": "60cfb27e..."
}

# Reconstruct the payload
payload = f"{entry['agent_name']}|{entry['agent_number']}|{entry['owner_hash']}|{entry['claimed_at']}"

# Verify
pub_key = Ed25519PublicKey.from_public_bytes(bytes.fromhex(public_key_hex))
pub_key.verify(
    bytes.fromhex(entry["signature"]),
    payload.encode("utf-8")
)
# Raises InvalidSignature if tampered
```

### JavaScript / Node.js

```javascript
import { verify } from "@noble/ed25519";

const publicKeyHex = "ea0be5dc...";
const entry = { /* ... from registry.json ... */ };
const payload = `${entry.agent_name}|${entry.agent_number}|${entry.owner_hash}|${entry.claimed_at}`;

const isValid = await verify(
  entry.signature,
  new TextEncoder().encode(payload),
  publicKeyHex
);
```

### CLI (using `openssl`)

```bash
# Extract the public key and convert to raw binary
echo -n "ea0be5dc021945a86afa8b3004fb6a9e84415340aff7dc0d5dcb781ed264d4e7" | xxd -r -p > pubkey.raw

# Reconstruct payload
echo -n "ultra-lobster|42|a1b2c3d4...|2026-02-25T10:16:08Z" > payload.bin

# Decode signature to binary
echo -n "60cfb27e..." | xxd -r -p > sig.bin

# Verify (requires openssl 3.x with Ed25519 support)
openssl pkeyutl -verify -pubin -inkey <(openssl pkey -pubin -inform DER \
  -in <(printf '\x30\x2a\x30\x05\x06\x03\x2b\x65\x70\x03\x21\x00' | \
  cat - pubkey.raw)) -sigfile sig.bin -in payload.bin
```

## How Users Prove Ownership

Ownership proof is a three-layer system:

### 1. Owner Hash (Privacy-Preserving Identity)

Each claim includes `owner_hash` — the SHA-256 hash of the owner's email address. This links the claim to an identity without exposing the email publicly. The owner can prove they own a name by demonstrating they know the email that produces the hash:

```python
import hashlib
my_email = "alice@example.com"
my_hash = hashlib.sha256(my_email.lower().encode()).hexdigest()
# Compare with owner_hash in registry entry
```

### 2. Ed25519 Signature (Platform Attestation)

The Society AI signature proves the claim was made through the official platform. The signature binds the agent name, number, owner hash, and timestamp together — none of these fields can be modified without invalidating the signature.

### 3. Git History (Immutable Audit Trail)

Every claim is committed to this repository with a git commit SHA. The full git history provides a tamper-evident, chronological record of all claims. Each commit references the database record via `registry_commit_sha`, creating a bidirectional link between the registry and the platform.

### Verification API

You can also verify claims programmatically via the Society AI API:

```
GET https://api.societyai.com/namespaces/{agent_name}/verify
```

Returns the claim details, signature, public key, and a `verified: true/false` field.

## How the Registry Is Updated

1. When a user claims an agent name on Society AI, the server signs the claim immediately with Ed25519
2. A background task runs every 5 minutes, collecting all signed claims not yet committed to GitHub
3. New entries are appended to `registry.json` and committed in a single batch
4. The commit SHA is stored back in the database for bidirectional linkage
5. Deduplication ensures no entry appears twice, even across retries

All commits are made by the Society AI server using the GitHub API. The optimistic locking via file blob SHA prevents race conditions.

## License

This registry data is public. The signature scheme and verification methods are open for anyone to implement.
