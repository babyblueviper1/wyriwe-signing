# ERC-8299 signing-input profile (draft 0)

**Status:** working draft, 30 Oct 2026. Not normative. Laid out to mirror ERC-8299's own sections so the two read side by
side. Authors: Tiago Merlini, Jimmy Shi, babyblueviper1. Comments welcome as issues or PRs.

The key words "MUST", "MUST NOT", "SHOULD" and "MAY" are to be interpreted as described in RFC 2119.

## Abstract
ERC-8299 binds an AI model's input to what the user supplied: raw input, a pinned public pipeline, and the exact bytes the
model received. This profile applies the same triple-hash construction to **transaction signing**: the agreed *intent*, a
pinned public *encoder*, and the exact *payload* each signer signs. Every signer derives the payload itself from the intent
with the pinned encoder ("derive, don't receive") and signs only its own build. Anyone can later recompute that the
reviewed intent, the signed payload and the executed transaction are one and the same.

## Motivation
Multi-party approval fails when reviewers and signers trust the same rendered view or the same handed-over hash: a
compromised step between them can present one transaction for review and another for signature (the 2025 Bybit theft).
Display standards (ERC-7730, EIP-712) make calldata readable; they do not prove that every signer signed what was
reviewed. This profile makes that link deterministic and recomputable by a stranger.

## Specification

### 1. Triple-hash construction (mapping to ERC-8299 §1)
A conforming signing flow MUST produce:

```
intent_hash             = keccak256(JCS(intent))                       // ERC-8299 raw_input_hash
encoder_pipeline_hash   = keccak256(encoder_spec_cid || intent_hash)   // ERC-8299 sanitization_pipeline_hash
payload_hash            = keccak256(payload)                           // ERC-8299 input_hash
```

- `intent` is the agreed action as a JSON object, serialized with RFC 8785 (JCS) as UTF-8 bytes (§2).
- `encoder_spec_cid` is the full `ipfs://` URI string of the pinned encoder specification and reference implementation
  (§3), as UTF-8 bytes, `ipfs://` prefix included, exactly as in ERC-8299 §1.
- `payload` is the exact bytes the encoder outputs for this intent: the call the account will execute (§4).
- `||` is byte concatenation.

**Verification invariant:** given `intent`, `encoder_spec_cid` and `payload`, anyone MUST be able to rerun the pinned
encoder on the intent and obtain `payload` byte for byte. No party needs to be trusted for this.

### 2. Intent
A JSON object with exactly these members (draft; see open question 1):

| member | type | meaning |
|---|---|---|
| `chainId` | integer | the chain the action runs on |
| `account` | address (lowercase hex) | the account that executes (Safe, 4337 account, 7702 account, EOA) |
| `target` | address | the contract or recipient called |
| `value` | decimal string (wei) | native value sent |
| `function` | string | canonical signature, e.g. `transfer(address,uint256)`; `""` for a plain value transfer |
| `args` | array | arguments in ABI order, each `{ "type": <ABI type>, "value": <canonical JSON value> }` |
| `operation` | `"call"` or `"delegatecall"` | `delegatecall` MUST be explicit (open question 5) |
| `nonce` | decimal string or `null` | the account nonce the action is pinned to, if the account has one |

Integers are decimal strings (no floats). Addresses and bytes are lowercase `0x` hex. Unknown members MUST be rejected.

### 3. Pinned encoder
The encoder specification at `encoder_spec_cid` MUST define, completely and deterministically:
- how each intent member maps to the account's call (`to`, `value`, `data`, `operation`, nonce handling);
- ABI encoding of `function` + `args` (standard Solidity ABI);
- which account bindings it supports (§4) and how the signing digest is derived for each;
- a reference implementation, identified by its own content hash inside the specification.

An encoder is a pure function: same intent, same payload, on any machine. Two independent implementations of the same
specification MUST agree byte for byte; disagreement is a conformance failure, not a tie to break.

### 4. Payload and signing digest (account bindings)
`payload` is the account-level call the signers authorize. The signing digest is derived from it by the account's own
rules, so this profile does not change what signers sign:
- **Safe:** `payload` = ABI encoding of the SafeTx fields; signing digest = the Safe's EIP-712 `SafeTx` hash.
- **ERC-4337:** `payload` = the packed UserOperation call fields; signing digest = `userOpHash`.
- **EOA / EIP-7702 account:** `payload` = the unsigned transaction fields; signing digest = the transaction signing hash.

The encoder specification MUST state the binding it uses; a verifier MUST recompute the digest with that binding.

### 5. Signer rule ("derive, don't receive")
A conforming signer MUST:
1. obtain the `intent` (not a payload, not a digest);
2. run the pinned encoder itself to produce `payload`, then derive the signing digest;
3. check that `encoder_pipeline_hash` matches the one the reviewers approved (§6);
4. sign only the digest it derived. It MUST NOT sign a digest or payload supplied by anyone else.

If any signer's machine produces a different payload, its signature will not match the others' and the account's own
threshold check refuses execution. That refusal is the safety property.

### 6. Review binding
Reviewers approve the pair `(intent_hash, encoder_pipeline_hash)`. Any rendering shown to a human MUST be produced from the
`intent` (for example through ERC-7730 descriptors), never from a payload. A reviewer's approval MAY be recorded as the
EIP-712 struct in §7.

### 7. WyriweSigningInput struct (mapping to ERC-8299 §3)
```solidity
struct WyriweSigningInput {
    uint256 chainId;
    address account;
    bytes32 intentHash;
    bytes32 encoderPipelineHash;
    bytes32 payloadHash;
    bytes32 signingDigest;
    uint256 timestamp;
}
```
Type string (field order normative):
`WyriweSigningInput(uint256 chainId,address account,bytes32 intentHash,bytes32 encoderPipelineHash,bytes32 payloadHash,bytes32 signingDigest,uint256 timestamp)`

Domain: as ERC-8299 §4 (`chainId` from the chain the record is produced on, never hardcoded).

### 8. ClaimType
The intent → payload → digest binding is deterministic, so a `WyriweSigningInput` is a **`ReExecution`-class** claim
(ERC-8299 §7): objectively disputable by rerunning the pinned encoder. A reviewer's signature over the struct adds an
`Attestation` (who approved it), carried separately.

### 9. Verification procedure (the outside recompute; independent checker lane)
A verifier holding `intent`, `encoder_spec_cid`, the executed transaction and the struct MUST:
1. recompute `intent_hash` from the intent;
2. recompute `encoder_pipeline_hash` from `encoder_spec_cid` and `intent_hash`;
3. run the pinned encoder on the intent and check `keccak256(payload) == payloadHash`;
4. derive the signing digest with the stated account binding and check it equals `signingDigest`;
5. check the executed transaction's call equals `payload` and its signatures are over `signingDigest`;
6. accept only if every step passes. Reported result: `reviewed = signed = executed`, or the first step that failed.

## Rationale (draft)
- **Why the encoder is pinned by CID:** as in ERC-8299, the pipeline must be public and frozen for "anyone can
  recompute" to mean anything.
- **Why signers take the intent and not the payload:** a payload or digest handed over is exactly what a compromised step
  can swap. Independent derivation turns a swap into a visible disagreement.
- **Why not only display (ERC-7730):** a correct display on one machine says nothing about what another machine signed.

## Test cases (to come with the independent checker)
1. Honest flow: all steps pass.
2. **Swapped payload:** a payload that differs from the encoder's output must fail step 3, even with valid signatures.
3. Wrong encoder: a different `encoder_spec_cid` must fail step 2.
4. Digest substitution: a valid payload with a different signing digest must fail step 4.
5. Two independent encoder implementations: identical payloads for every vector, or the vector fails.

## Security considerations (draft)
- A signer whose machine is compromised *and* runs the pinned encoder can still sign a different digest. The protection is
  the threshold: an attacker must control enough signers' machines independently. Signers SHOULD derive on separate
  devices, ideally with different encoder implementations.
- `delegatecall` and EIP-7702 delegations replace or borrow code; an intent class for them SHOULD require stricter review.
- The intent itself can be wrong (the right encoding of a bad decision). This profile proves fidelity, not judgment.

## Open questions
1. Intent format: this minimal object, ERC-7730 descriptors, or EIP-712 types as the intent?
2. Encoder distribution: reference digest only, a WASM build, or several independent implementations required?
3. Where `intentHash` lives on-chain, if at all (calldata suffix, event, registry), and its gas cost.
4. Batched actions (MultiSend, 4337 batches): one intent per call, or an ordered list?
5. Should `delegatecall` and 7702 delegations be a separate intent class with extra review requirements?
