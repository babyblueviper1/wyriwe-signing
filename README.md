# WYRIWE for signing: reviewed = signed = executed

**Status:** idea, pre-draft (30 Oct 2026). Proposed in the trustless-ai working group by **Tiago Merlini** and
**Jimmy Shi**; leaning toward a **signing-input profile of [ERC-8299 (WYRIWE)](https://github.com/ethereum/ERCs/pull/1810)**
(the chain intent → pinned encoding → hash is already 8299's). Nothing here is normative yet.

**Co-authors:** Tiago Merlini (@TMerlini), Jimmy Shi (@JimmyShi22), babyblueviper1 (@babyblueviper1).
**Contributors:** Pavlo (review, WG 5957), OnThe_Stars (review trust root, WG 5959).
**Lanes:** babyblueviper1 carries step 6, an independent stdlib checker (intent + pinned encoder + signed tx →
reviewed = signed = executed or not) with vectors, including a swapped-payload case that must fail. Built separately from
the reference encoder, it is also the second independent implementation.

**Independent checker:** [babyblueviper1/invinoveritas `examples/wyriwe-signing`](https://github.com/babyblueviper1/invinoveritas/tree/2aaec8ed898faf5ff3d2048a05bd16ba65606760/examples/wyriwe-signing) (stdlib, vectors for every test case).

**Draft spec:** [`PROFILE.md`](PROFILE.md), the ERC-8299 signing-input profile, draft 0 (mirrors 8299's sections).

## The failure
The approval flow reviews one thing and the signers sign another, because something compromised sits between them.
Bybit lost about $1.5B this way in 2025: the reviewers saw a legitimate-looking transaction, and the signers signed a
different one. Multi-stage approval doesn't help when every stage trusts the same rendered view or the same handed-over
hash. Account upgrades (EIP-7702) widen the blast radius: one bad signature can now replace the whole account's code.

## The principle: derive, don't receive (Jimmy)
A signer never signs what it is handed. It **builds the thing it signs from data it can verify itself**, and signs only
its own build. If every signer derives independently and they agree, the payload is what everyone meant. If anyone in
the chain swapped it, the independent builds disagree, the signatures don't form a quorum, and the contract refuses.

## Construction sketch
1. **Intent.** The agreed action as canonical structured data (RFC 8785 JSON): chain id, account, target, value,
   function + typed arguments, nonce policy. `intent_hash = keccak256(JCS(intent))` (Keccak-256, as ERC-8299).
2. **Pinned encoder.** The exact code that turns an intent into the transaction to sign, identified by its digest
   (`encoder_hash`). Pinning it is what makes independent builds comparable.
3. **Independent derivation.** Each signer runs the pinned encoder on the intent locally, computes the digest it would
   sign (for example the EIP-712 / Safe transaction hash), and signs **only** that. It never accepts a digest from
   anyone else.
4. **Review binding.** Reviewers approve `(intent_hash, encoder_hash)`. Anything shown to a human is rendered from the
   intent, so what they read is what gets encoded.
5. **On-chain.** The account's existing signature check rejects mismatched digests. Optionally the executed transaction
   carries or commits to `intent_hash` and `encoder_hash`, so the link is on-chain.
6. **Recompute from outside.** Anyone holding the published intent and encoder can recompute the digest and compare it
   with the executed transaction, without asking the builder.

## How it maps to ERC-8299
| ERC-8299 (AI inference input) | Signing |
|---|---|
| `raw_input_hash` | `intent_hash` |
| `sanitization_pipeline_hash` | `encoder_hash` |
| `input_hash` | the signed transaction digest |

The same three-step chain, with the same rule: each step is recomputable, and the middle one is pinned.

## What already exists (not reinvented here)
- **ERC-7730** clear-signing descriptors: what calldata *means*, for wallet display.
- **EIP-712** typed structured data.
- **Post-Bybit Safe transaction-hash recompute tools**: check a hash on a second machine before signing.
- Hardware-wallet clear signing.

The gap this addresses is the **binding across stages** (intent → review → every signer → execution), made by
independent derivation and **checkable by a stranger** afterwards, not display alone.

## Open questions
- Extension of ERC-8299 (a signing-input profile; current lean) or a companion ERC that consumes it?
- The intent format: reuse ERC-7730 descriptors or EIP-712 types, or define a minimal canonical intent?
- How is the encoder pinned and distributed (reference implementation digest, a WASM build, several independent
  implementations that must agree)?
- Threat model: what if a signer's machine is compromised *and* runs the pinned encoder? (Independent devices, quorum
  across different implementations.)
- Where `intent_hash` lives on-chain, if at all, and what it costs.
- EIP-7702 delegations: should a delegation (code replacement) need a stricter intent class?

## Provenance
Proposed in the trustless-ai working group on 30 Oct 2026 (messages 5945–5955). The record of those messages is
`provenance/2026-10-30-wyriwe-for-signing-origination.json` (published with the agreement of those quoted), and
`provenance/…json.ots` is its OpenTimestamps proof, committing to sha256
`d4a811ffb2818da800d90382fd6fd1cbce62f021b168164b6cff54a93d36055d`. Check: `shasum -a 256` the JSON, then verify the
`.ots` against a Bitcoin block header. **Confirmed in Bitcoin block 973827** (hash `00000000000000000000b8a136d8181df8a281b3ce27d358b1cb14de4d9a475d`, block time 2026-10-30 13:35:33 UTC): the proof's computed merkle root equals that block's, with the header read from mempool.space and blockstream.info (identical). To check yourself: `ots verify provenance/2026-10-30-wyriwe-for-signing-origination.json.ots` against your own node or two explorers.

## License
CC0 1.0 Universal. See `LICENSE`.
