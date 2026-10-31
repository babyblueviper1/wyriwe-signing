# ERC-8299 signing-input profile (draft 0)

**Status:** working draft 4, 31 Oct 2026 (draft 1: Pavlo's review, WG 5957; draft 2: the review trust root, OnThe_Stars, WG 5959; draft 3: authenticated trust roots and execution, Pavlo and Echo, WG 5962-5967; draft 4: the reviewer-list pin recorded on-chain before execution, Echo, PR #1 review 31 Oct). Not normative. Laid out to mirror ERC-8299's own sections so the two read side by
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

**v0 scope: Safe only.** Version 0 of this profile defines exactly one account binding, the Safe (SafeTx EIP-712 hash,
`execTransaction`). ERC-4337 and EIP-7702 bindings are listed for direction only and are not part of v0; each gets its own
binding section, vectors and negative controls before it is added.

### 5. Signer rule ("derive, don't receive")
A conforming signer MUST:
1. obtain the `intent` (not a payload, not a digest);
2. run the pinned encoder itself to produce `payload`, then derive the signing digest;
3. check that `encoder_pipeline_hash` matches the one the reviewers approved (§6);
4. sign only the digest it derived. It MUST NOT sign a digest or payload supplied by anyone else.

If any signer's machine produces a different payload, its signature will not match the others' and the account's own
threshold check refuses execution. That refusal is the safety property.

### 6. Review binding (authenticated)
Reviewers approve the pair `(intent_hash, encoder_pipeline_hash)`. Any rendering shown to a human MUST be produced from the
`intent` (for example through ERC-7730 descriptors), never from a payload.

An approval MUST be an EIP-712 signature over the `WyriweSigningInput` struct (§7) by a reviewer key, and the set of
reviewer keys and the approval threshold MUST be pinned in advance, independently of the flow being checked (the **review
trust root**: for example the Safe's owner set at the intent's block, or a reviewer list committed on-chain or in a pinned
document). Recomputing a valid transaction does not establish what a human approved; only an authenticated approval against
the review trust root does. Without it, the `reviewed` verdict (§9) is `NOT_ESTABLISHED`, never passed.

**The review trust root must itself be recomputable** (the rule ERC-8275 applies to every reputation input: no scorer or
settler to trust). Two kinds, and the verdict always says which one was used:
- **`owners@block` (v0 default).** The Safe's owner set and threshold at a pinned block, read from the chain. It adds no new
  trust, but the reviewers *are* the signers, so a `reviewed: PASS` here proves **fidelity** (what the owners approved is
  what they signed), **not independent review**: an owner set compromised past its threshold approves and signs the same
  bad payload.
- **`reviewer-list@commitment` (extension).** A separate reviewer key set and threshold, valid only if **committed on-chain**
  (a contract, at a pinned block) **or content-addressed** (its hash pinned in advance). A list someone maintains off-chain
  is not a trust root: it reintroduces the trusted party this profile exists to remove. Only this kind can let `reviewed`
  catch a compromised signer.
- **Both together** only once their composition is defined (open question 0a); until then a flow declares exactly one.

The checker MUST report, with the `reviewed` verdict: the trust-root kind and its identifier (block number, or contract and
block, or content hash), and the **overlap between the approving reviewers and the signers**, so a consumer can tell
fidelity from independent review without assuming either.

**Authority, not only integrity (Pavlo, WG 5962).** A trust root counts only when the checker authenticates it itself:
- `owners@block`: the owner set and threshold are **read from proofs** (the account proof and the storage proofs of the
  Safe's owners list and threshold) against a block header the checker hashes, which must hash to the stated block hash and
  carry the stated block number. A supplied owner list is never trusted: without the proofs, `reviewed: NOT_ESTABLISHED`.
- `reviewer-list@commitment`: a content hash proves the list is unchanged, not that anyone with authority chose it. The list
  counts only when its pin is **recorded on-chain before the execution block** and the checker proves that record itself
  (Echo, PR #1 review 2026-10-31). The pin is
  `reviewerListPin = keccak256(abi.encode(keccak256("WyriweReviewerList(uint256 chainId,address account,bytes32 listHash)"), chainId, account, listHash))`,
  with `listHash = keccak256(JCS(list))`. The checker reads account and storage proofs against the header whose hash equals
  the authenticated execution header's `parentHash` (block E−1, so the state before the execution block). It hashes that
  header itself, and its link to the execution is that `parentHash`, so it needs no separate canonicity check beyond block E's.
  Either record counts:
  - **Safe message (v0):** `signedMessages[getMessageHash(abi.encode(reviewerListPin))] == 1`, at storage slot
    `keccak256(abi.encode(msgHash, 7))`, where `getMessageHash(m) = keccak256(0x19 ‖ 0x01 ‖ domainSeparator ‖ keccak256(abi.encode(SAFE_MSG_TYPEHASH, keccak256(m))))`.
    It is set by a Safe transaction delegatecalling `SignMessageLib.signMessage`. An enabled module can execute without owner
    signatures, so this record counts only if the same proof shows the module list empty (slot 1 linked list: sentinel →
    sentinel). Otherwise `reviewed: NOT_ESTABLISHED` (`reviewer_list_pin_module_possible`).
  - **Owner approvals:** `approvedHashes[owner][reviewerListPin] == 1`, at slot
    `keccak256(abi.encode(reviewerListPin, keccak256(abi.encode(owner, 8))))`, for at least `threshold` of the owners proven in
    the same state.

  Signatures over a message that *states* a block number don't count as a prior pin, because the signers choose that number:
  `reviewed: NOT_ESTABLISHED` (`reviewer_list_prior_binding_unproven`). Without the record, `reviewed: NOT_ESTABLISHED`.

**v0 account binding (Safe v1.4.1 storage, `SafeStorage.sol`):** slot 1 `modules`, 2 `owners` (linked list from the sentinel
`0x…01`), 3 `ownerCount`, 4 `threshold`, 5 `nonce`, 7 `signedMessages`, 8 `approvedHashes`.

An offline checker cannot know whether a header's block is canonical; it reports the block hash for the consumer to confirm.

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
The result is **four separate verdicts**, never one combined pass. Each is `PASS`, `FAIL` (with the step that failed) or
`NOT_ESTABLISHED` (its evidence was not supplied). A verdict is never inferred from another one.

| verdict | means | established by |
|---|---|---|
| `encoded` | the payload is exactly what the pinned encoder makes from the intent | steps 1–3 |
| `reviewed` | an authenticated approval of this `(intent_hash, encoder_pipeline_hash)` meets the review trust root's threshold; reported with the trust-root kind and the reviewer/signer overlap (§6) | step 4 |
| `signed` | the signatures on the chain are over the digest derived from that payload, by enough account owners | steps 5–6 |
| `executed` | the transaction carrying that payload was included **and succeeded**, proven against a header the checker hashes | step 7 |

1. Recompute `intent_hash` from the intent.
2. Recompute `encoder_pipeline_hash` from `encoder_spec_cid` and `intent_hash`.
3. Run the pinned encoder on the intent; check `keccak256(payload) == payloadHash`.
4. Resolve and authenticate the review trust root (§6: owners@block from proofs against a hashed header, or
   reviewer-list@commitment from its content hash plus its pin recorded on-chain in the state of the execution block's
   parent; anything else:
   `reviewed: NOT_ESTABLISHED`). Verify each approval signature over the struct against
   it; count distinct approved reviewers against its threshold. Report the trust-root kind and identifier, and which
   approving reviewers are also signers in step 6.
5. Derive the signing digest with the stated account binding (v0: Safe); check it equals `signingDigest`.
6. Check the on-chain transaction's call equals `payload`, and its signatures are over `signingDigest` by enough current
   owners.
7. Authenticate the execution (Echo, WG 5967): the header hashes to the stated block hash and carries the stated block
   number; the transaction's canonical bytes and its receipt are proven, by Merkle-Patricia proofs at the same index, under
   the header's `transactionsRoot` and `receiptsRoot`; and the proven transaction is the one checked in step 6 (`keccak256`
   of its bytes equals the supplied `tx.hash` and `receipt.transactionHash`, and its `to` and calldata equal step 6's). Then,
   from the **proven** receipt: `status = 1` and, for a Safe, the `ExecutionSuccess` event for this `signingDigest` (an
   `ExecutionFailure` event, or a revert, is `executed: FAIL` even though steps 1–6 pass). Anything less is
   `executed: NOT_ESTABLISHED`, never PASS: receipt data without these proofs (reason `rpc_data_consistent_unauthenticated`),
   no transaction, a hash mismatch, or a proof of another transaction. The block hash is reported for the consumer to confirm
   as canonical.

**No false green.** Removing or skipping any step MUST turn its verdict into `NOT_ESTABLISHED` or `FAIL`, never `PASS`; each
verdict's negative vectors (below) and a mutation run that deletes each check prove it. Consumers that need "what was
reviewed is what ran" MUST require all four verdicts to be `PASS`.

## Rationale (draft)
- **Why the encoder is pinned by CID:** as in ERC-8299, the pipeline must be public and frozen for "anyone can
  recompute" to mean anything.
- **Why signers take the intent and not the payload:** a payload or digest handed over is exactly what a compromised step
  can swap. Independent derivation turns a swap into a visible disagreement.
- **Why not only display (ERC-7730):** a correct display on one machine says nothing about what another machine signed.

## Relationship to ERC-8419 (Know-Your-Agent)
ERC-8419 concludes about an **agent** (a level, a validity window, revocation). This profile concludes about **one
execution**. The boundary agreed on the KYA PR holds here unchanged: one verified transaction never becomes a trust level.
A **history** of verdicts can feed an agent-level assertion through an adapter (the planned WYRIWE → KYA adapter, built
like the ERC-8294 one in the KYA reference repo):
- Signing verdicts arrive **already bound to the invocation**: §9 step 5 checks the executed transaction itself. For
  inference attestations that binding needs a separate receipt naming `input_hash`; for signing, the chain is the receipt.
- They are `ReExecution`-class, so the adapter can run in **proof-checked** mode (rerun the encoder), not only attested.
- The adapter's output should carry the window, the number of flows and the number of failures (for example "transaction
  fidelity verified: N of N signing flows recompute over [window]"), never a bare "trusted".

## Test cases
The independent checker carries all of them as vectors, with the mutation controls below:
[babyblueviper1/invinoveritas `examples/wyriwe-signing`](https://github.com/babyblueviper1/invinoveritas/tree/2aaec8ed898faf5ff3d2048a05bd16ba65606760/examples/wyriwe-signing) (Python standard library only; 55 vectors, every check
load-bearing, plus live runs against an Ethereum mainnet Safe v1.4.1).

Positive: an honest Safe flow, all four verdicts `PASS`.

Negative, one per guarantee (each must FAIL or be NOT_ESTABLISHED, never PASS):
1. **Swapped payload:** differs from the encoder's output: `encoded: FAIL` even with valid signatures.
2. Wrong encoder: a different `encoder_spec_cid`: `encoded: FAIL`.
3. No approval supplied: `reviewed: NOT_ESTABLISHED` (not PASS), even when everything else passes.
4. Approval by a key outside the review trust root, or below threshold: `reviewed: FAIL`.
4a. A reviewer list that is neither on-chain nor content-addressed: `reviewed: NOT_ESTABLISHED`, never PASS.
4b. owners@block with every approver also a signer: `reviewed: PASS` **reported as owners@block with full overlap**; a
    consumer requiring independent review must read it as not independent.
5. Approval of a different `intent_hash`: `reviewed: FAIL`.
6. Digest substitution: a valid payload with a different signing digest: `signed: FAIL`.
7. Signatures from non-owners or below the Safe threshold: `signed: FAIL`.
8. Included but reverted, or a Safe `ExecutionFailure`: `executed: FAIL` while the other three PASS.
4c. Trust-root authority (§6), each `reviewed: NOT_ESTABLISHED`, never PASS: a content-addressed list with valid approvals
    but no on-chain pin; the backdating case (off-chain owner signatures on a pin that states an earlier block, no on-chain
    record); a pin recorded in the execution block itself (absent from the E−1 state); a pin proven against a header that
    isn't block E's parent; a valid pin with no execution evidence; a Safe-message record with a module enabled; owner approvals
    from fewer than `threshold` proven owners, or from a non-owner; a pin for another `listHash`, another account or another
    chain; the pin state supplied without proofs; an owner set supplied without proofs, a header that does not hash to the
    block hash, or a proof for another account. Positives: an honest Safe-message record and honest owner approvals
    (`reviewed: PASS`, independent review). Mutation controls: deleting the parent-hash binding, the module check, or either
    record check must each flip at least one vector.
9. No transaction receipt supplied: `executed: NOT_ESTABLISHED`.
9a. Execution binding (Pavlo, Echo, WG 5966-5967), each `executed: NOT_ESTABLISHED`, never PASS: `tx.hash` differs from
    `receipt.transactionHash`; a receipt with an arbitrary `blockHash` and no inclusion proof; the transaction removed while a
    matching success receipt is kept; a header that does not hash to the block hash; a proof with a node removed; a proof of
    another transaction in the same block.
10. Two independent encoder implementations: identical payloads for every vector, or the vector fails.

Mutation controls: a run that deletes each check in turn must change at least one vector's verdict, for example from FAIL or
NOT_ESTABLISHED to PASS (so every check is load-bearing; the 9a binding checks included), and a crash must be counted as a crash, not a kill (the harness rule from DEILS Review 5).

## Security considerations (draft)
- A signer whose machine is compromised *and* runs the pinned encoder can still sign a different digest. The protection is
  the threshold: an attacker must control enough signers' machines independently. Signers SHOULD derive on separate
  devices, ideally with different encoder implementations.
- `delegatecall` and EIP-7702 delegations replace or borrow code; an intent class for them SHOULD require stricter review.
- The intent itself can be wrong (the right encoding of a bad decision). This profile proves fidelity, not judgment.

## Open questions
0. ~~The review trust root's form.~~ Settled for v0 (WG 5959–5960): owners@block default, reviewer-list@commitment as the extension, the kind always reported (§6).
0a. How owners@block and a reviewer list compose when a flow wants both (e.g. reviewers AND owners, each with its own threshold).
1. Intent format: this minimal object, ERC-7730 descriptors, or EIP-712 types as the intent?
2. Encoder distribution: reference digest only, a WASM build, or several independent implementations required?
3. Where `intentHash` lives on-chain, if at all (calldata suffix, event, registry), and its gas cost.
4. Batched actions (MultiSend, 4337 batches): one intent per call, or an ordered list?
5. Should `delegatecall` and 7702 delegations be a separate intent class with extra review requirements?
