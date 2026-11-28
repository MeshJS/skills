---
name: mesh-tx-builder-migration
description: Use when migrating, converting, porting or upgrading Cardano transaction code from the deprecated fluent MeshTxBuilder (txIn, txOut, complete(), signTx, submitTx) to the new TxBuilder, or when writing new transaction code with TxBuilder parts (output, spend, mint, certificate). Covers old call to part mapping, the payer, build() or plan(), signing, and new errors. Includes recipes and troubleshooting.
license: Apache-2.0
metadata:
  author: MeshJS
  version: "1.0"
---

# Mesh SDK Transaction Builder Migration Skill

Move code from the deprecated `MeshTxBuilder` to `TxBuilder` from `@meshsdk/transaction`.

The old builder still works. Move code when asked, or when you write new code. Leave `Transaction` and `MeshTxBuilderV2` code unchanged.

## Package Info

```bash
npm install @meshsdk/transaction
# or
npm install @meshsdk/core  # includes transaction + wallet + provider
```

All new names come from `@meshsdk/core`. `@meshsdk/transaction` exports them too.

## The Change

The old builder is one fluent chain. The new builder takes a list of parts and a payer. The payer is a wallet, an address, or `{ changeAddress, utxos }`. It replaces `changeAddress` and `selectUtxosFrom`. A build without a payer is `MISSING_PAYER`.

The examples use these names, which your app defines: `txBuilder` (the old builder), `builder` (a `TxBuilder`), `wallet`, `utxos`, `changeAddress` and `recipient`.

```typescript
import { output } from '@meshsdk/core';

// Before
const unsignedTx = await txBuilder
  .txOut(recipient, [{ unit: 'lovelace', quantity: '5000000' }])
  .changeAddress(changeAddress)
  .selectUtxosFrom(utxos)
  .complete();

// After
const built = await builder.build(
  output({ address: recipient, value: { lovelace: 5_000_000n } }),
  { payer: wallet },
);
await built.sign();
await built.submit();
```

## Workflow

1. **Find the chain.** Look for each chain from `new MeshTxBuilder` to `.complete()`, and the lines after it that sign and submit. A chain built over several statements, in a loop or behind an `if` is one chain.
2. **Create the builder once.** `new TxBuilder({ network, provider })`. Take `network` from `setNetwork`, `resolveSlotNo` or the provider. If none is visible, ask the user.
3. **Map each call.** Look it up in [MAPPING.md](./MAPPING.md). Its table of contents lists Spending, Mint and Burn, Certificates, Votes and Proposals and Time. One old item is one part: an input with its script, datum and redeemer calls becomes one `spend.*` call.
4. **Build the parts list.** Put the parts in one list. A loop or `if` becomes `parts.push(...)` or `condition && part`. Drop what the builder does itself: collateral, execution units, `changeAddress`, `selectUtxosFrom`, `resolveSlotNo`, `deserializePoolId`.
5. **Choose `build()` or `plan()`.** See below.
6. **Replace signing and submitting.** `built.sign()` and `built.submit()` replace `wallet.signTx` and `wallet.submitTx`.
7. **Read the behaviour notes.** See "Behaves Differently" in [TROUBLESHOOTING.md](./TROUBLESHOOTING.md).
8. **Type-check and test.** Fix the `bigint` errors first. Tests that compare CBOR, hash, fee or chosen inputs will differ. Update the expected values, not the migrated code.

Never invent a name. A call that is not in [MAPPING.md](./MAPPING.md) or the package README stays as it is. Ask the user.

## Quick Reference

| Old | New |
|-----|-----|
| `new MeshTxBuilder({ fetcher })` | `new TxBuilder({ network, provider })` |
| `.txOut(address, assets)` | `output({ address, value })` |
| `.txIn(hash, index, ...)` | `spend.pubKey({ utxo })` |
| `.spendingPlutusScriptV2().txIn(...).txInScript(...).txInRedeemerValue(...)` | `spend.plutus({ utxo, script, redeemer })` |
| `.mint('1', policyId, name).mintingScript(script)` | `mint.native({ script, assets })` |
| `.mintPlutusScriptV2().mint(...).mintingScript(...).mintRedeemerValue(...)` | `mint.plutus({ script, redeemer, assets })` |
| `.changeAddress(a).selectUtxosFrom(u)` | `{ payer: wallet }`, or an address, or `{ changeAddress, utxos }` |
| `.metadataValue(label, value)` | `metadata({ metadataLabel, value })` |
| `.complete()` | `await builder.build(parts, { payer })`. The old string is `built.tx` |
| `wallet.signTx(tx)`, `wallet.submitTx(signed)` | `built.sign()`, `built.submit()` |

Everything else is in [MAPPING.md](./MAPPING.md).

## Files

- [MAPPING.md](./MAPPING.md) - Every old call and its new part or option
- [PATTERNS.md](./PATTERNS.md) - Before and after recipes
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) - Errors a migrated build may throw, and what behaves differently
- `node_modules/@meshsdk/transaction/README.md` - The full `TxBuilder` guide

## Key Concepts

### build() or plan()

| | `build()` | `plan()` |
|---|-----------|----------|
| Returns | a `BuiltTx` | a `TxPlan` |
| Finish | `built.sign()`, `built.submit()`, `built.awaitInclusion()` | `plan.run()` |
| Provider | any | must return block heights, else `CANNOT_OBSERVE` |
| Use for | a straight port of `complete()` | waiting for a block, retries, several transactions in order |

Default to `build()` for a port.

## Important Notes

1. **Amounts are `bigint`** - `5_000_000n`, not `'5000000'`. The exceptions are an `Asset[]` value in `output`, and the `pledge` and `cost` of `poolRegistration({ params })`, which keep the old strings.
2. **`build()` signs and sends nothing** - the old CBOR string is `built.tx`, the hash is `built.txHash`. `submit()` goes through the provider. Use `submit({ submitter: wallet })` to submit through the wallet.
3. **`network` is required** - `'mainnet'`, `'preprod'` or `'preview'`.
4. **One mint part per policy** - put every asset name in one `assets` record.
5. **Stake registration changed** - `certificate.stakeRegistration` states a deposit and needs the stake key.
6. **No built-in values** - there are no protocol parameters and no governance deposits. Give a `provider` (or `params` and `costModels`), and `deposits` to the builder or `deposit` to the part.
