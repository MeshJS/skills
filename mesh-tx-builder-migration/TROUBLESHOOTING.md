# Migration Troubleshooting

Errors a migrated build may throw, and things that behave differently from the old builder.

Every error is a `TxError` with a `code`. Check the code, not the message. Old message text is gone. Codes are `TxBuildError` unless the heading says `TxPartError` or `TxPlanError`.

```typescript
import { TxError } from '@meshsdk/core';

try {
  const built = await builder.build(parts, { payer: wallet });
  await built.sign();
  await built.submit();
} catch (error) {
  if (TxError.is(error, 'INSUFFICIENT_FUNDS')) {
    const { unit, missing } = error.details; // typed by the code
  } else throw error;
}
```

The examples use `builder` (a `TxBuilder`), `built` (a `BuiltTx`), `provider`, `wallet`, `parts` (a list of parts) and `recipient`, which your app defines. The other names are clear from context. `key` is the text of a signing key.

## Table of Contents

- [Old Message to New Code](#old-message-to-new-code)
- [Setup Errors](#setup-errors)
- [Build Errors](#build-errors)
- [Signing and Sending](#signing-and-sending)
- [Other Codes](#other-codes)
- [Behaves Differently](#behaves-differently)
- [Where the Old Builder Keeps Its Own Way](#where-the-old-builder-keeps-its-own-way)
- [Debug Checklist](#debug-checklist)

---

## Old Message to New Code

| Old message | New code | Note |
|-------------|----------|------|
| "Change address is not set, utxo selection cannot be done without this" | `MISSING_PAYER` | Pass `{ payer }` |
| "Transaction information is incomplete while no fetcher instance is provided" | `MISSING_FETCHER` | Give a `provider`, or pass UTxOs in full |
| "Couldn't find value information for txHash#txIndex" | `UTXO_NOT_FOUND` | Pass the UTxO in full, or check the out ref |
| "Script input does not contain datum information" | none | The builder reads the datum from the output |
| "Script input does not contain redeemer information", "... script information" | none | `script` and `redeemer` are required fields of `spend.plutus` |
| "Evaluate redeemers failed", "Tx evaluation failed" | `EVALUATION_FAILED` | `details.failures` names the part and label. On chain: `SCRIPT_FAILED` |
| "Insufficient funds" | `INSUFFICIENT_FUNDS` | `details` gives `unit`, `missing`, `required`, `available` |

Code that matched on old message text must match on `error.code` now.

---

## Setup Errors

### `MISSING_PAYER`

**Cause:** `build()` or `plan()` was called without a payer. The builder has no default payer and no `changeAddress` option.

**Solution:**
```typescript
const built = await builder.build(parts, { payer: wallet });
// or { payer: address }, or { payer: { changeAddress, utxos } }
```

---

### `INVALID_OPTION`

**Cause:** An option is not accepted. Common ones: `network: 'testnet'`, `signers: key`, a payer of no known form. On the old builder, also a serializer or selector that is not Mesh's.

**Solution:**
```typescript
import { keySigner, TxBuilder } from '@meshsdk/core';

const builder = new TxBuilder({ network: 'preprod', provider }); // 'mainnet', 'preprod' or 'preview'
await built.sign({ signers: keySigner(key) }); // a key goes through keySigner
```

`error.details` gives `option`, `value` and `allowed`.

---

### `MISSING_PARAMETER`

**Cause:** The builder has no protocol parameters, no cost models, or no governance deposit. There are no built-in values.

**Solution:**
```typescript
import { TxBuilder } from '@meshsdk/core';

const online = new TxBuilder({ network: 'preprod', provider }); // reads them
const offline = new TxBuilder({ network: 'preprod', params, costModels });
const gov = new TxBuilder({
  network: 'preprod',
  provider,
  deposits: { drepDeposit, govActionDeposit },
});
```

`error.details.parameter` is `'params'`, `'costModels'` or `'deposits'`.

---

## Build Errors

### `INVALID_VALUE` (`TxPartError`)

**Cause:** An amount is a string or a number. Amounts are `bigint`. The error comes at the part call, before `build()`.

**Solution:**
```typescript
import { output } from '@meshsdk/core';

output({ address: recipient, value: { lovelace: 5_000_000n } }); // not '5000000'
```

---

### `OUTPUT_BELOW_MIN_COIN`

**Cause:** An output holds less lovelace than the ledger requires. The old builder returned a transaction the ledger then refused.

**Solution:**
```typescript
import { output } from '@meshsdk/core';

output({ address: recipient, value: { [unit]: 1n } }); // no lovelace: the output holds its minimum
```

`error.details` gives `coin` and `minCoin`.

---

### `INSUFFICIENT_FUNDS`

**Cause:** The payer's UTxOs cannot pay for the outputs, fee and deposits. A `canSelect` filter may exclude too much.

**Solution:** Fund the payer, or loosen `canSelect`. `error.details` gives `unit`, `missing`, `required` and `available`.

A Plutus build whose redeemers have no `exUnits` is balanced with an upper estimate of the fee. A payer short by less than that gets this error. Stating the units avoids it.

---

### `INSUFFICIENT_COLLATERAL`

**Cause:** No collateral can be put up. Usually `topUpCollateral: false` with collateral that is too small, or a payer with no suitable UTxO.

**Solution:** Drop `topUpCollateral: false`, or add a larger `collateral({ utxo })` part. Otherwise leave collateral to the builder.

---

### `DUPLICATE_PART`

**Cause:** The same input, policy, reward address, voter or collateral input twice. Or a once-only part twice: `validity`, `donation`, `collateralReturn`, `message`.

**Solution:**
```typescript
import { mint } from '@meshsdk/core';

// One part per policy, every name in `assets`
mint.native({ script: forgingScript, assets: { [nameA]: 1n, [nameB]: 1n } });
```

`error.details.item` names the repeat.

---

### `EVALUATION_FAILED`

**Cause:** A script failed. The datum, the redeemer, the validity interval or a missing signer may not fit what the script expects.

**Solution:** Give the part a `label`. The error repeats it, and `error.details.failures` lists each failure with its part.

```typescript
import { mConStr0 } from '@meshsdk/core';
import { spend } from '@meshsdk/core';

spend.plutus({
  utxo: scriptUtxo,
  script: { code: policyCode, version: 'V2' },
  redeemer: { data: mConStr0([]) },
  label: 'unlock',
});
```

---

## Signing and Sending

### `MISSING_SIGNATURES` (`TxPlanError`)

**Cause:** Nobody can sign, or keys are still missing. Nothing was sent. Typical after migrating: a stake or DRep key is now required, or a second wallet was not named.

**Solution:**
```typescript
await built.sign({ signers: [wallet, otherWallet] });
console.log(built.missing); // each { keyHash, roles }
await built.submit();
```

`submit()` throws this while `built.missing` is not empty.

---

### `SUBMIT_FAILED` and `CONFIRM_TIMEOUT` (`TxPlanError`)

**Cause:** The outcome is unknown. `CONFIRM_TIMEOUT` does not mean "not paid".

**Solution:** Repeat the same call on the same object. Never build again.

```typescript
await built.submit(); // the same bytes, the same hash
const { slot } = await built.awaitInclusion();
```

A new build may pick other inputs, and both transactions could land.

---

### `CANNOT_OBSERVE` (`TxPlanError`)

**Cause:** The provider does not return what the call needs. `plan()` needs block heights. `awaitInclusion()` of a Plutus transaction needs the validity flag. The plain `BlockfrostProvider` returns neither.

**Solution:** For a straight port use `build()`, `sign()` and `submit()`. Those need neither. To use `plan()`, or to wait on a Plutus transaction, use the temporary `BlockfrostProvider` subclass in `node_modules/@meshsdk/transaction/README.md`.

`error.details.reason` is `'noTip'`, `'noSearch'`, `'noDepth'`, `'noValidity'` or `'unconfirmed'`. Only `'unconfirmed'` is retryable: call `run()` again after `error.retryAfterMs`.

---

## Other Codes

| Code | Cause | Fix |
|------|-------|-----|
| `NETWORK_MISMATCH` | An address, UTxO, wallet or provider is of another network than the builder's | Use addresses of the builder's network. `details.found` names each mismatch |
| `INVALID_DATA` | Metadata text or bytes over 64 bytes, a validity bound given in two ways, or bytes after an inline datum | Use `message({ text })`, which cuts long text. Give each bound one way |
| `WITNESS_MISMATCH` | A key spend of a script-locked output, or the reverse | `spend.plutus` or `spend.native` for a script output, `spend.pubKey` for a key output |
| `DATUM_MISMATCH` | `datum: { data }` does not hash to the output's datum hash | Pass the datum the output was locked with. For an inline datum leave `datum` out |
| `EXUNITS_TOO_LOW` | Stated `exUnits` are below what the script needs | Leave `exUnits` out. The builder evaluates them |
| `UTXO_NOT_FOUND` | An out ref does not exist, is spent, or is not known to the provider yet | Check the ref. Use `knownUtxos` for new outputs and `after` for earlier built transactions |
| `MISSING_FETCHER` | Something must be looked up and the builder has no `provider` | Give a `provider`, or pass UTxOs in full |
| `TX_TOO_LARGE` | The transaction is over the size limit. A long mint list is a common cause | Split the parts. `builder.plan({ first: partsA, second: partsB }, { payer })` sends several in order |
| `FEE_TOO_LOW` | The `fee` option is below the minimum. A given fee is never raised | Drop `fee`, or give a higher one |
| `INVALID_GUARD` | A `guard.pubKey` whose hash is a script hash of the transaction | Give a key hash or an address |
| `MISSING_SUBMITTER` | No submitter in the call, the builder or the provider. Nothing was sent | `built.submit({ submitter: wallet })` |
| `SIGN_DECLINED`, `SIGNER_FAILED` | The user declined, or the signer failed. A wallet that changed account is `SIGNER_FAILED` with `details.reason` `'accountChanged'` | Asking again is your choice. After an account change, connect again and pass the new api as `signers` |

---

## Behaves Differently

No error. The result differs from the old builder.

- **The result is a `BuiltTx`.** The old string is `built.tx`.
- **`build()` signs and sends nothing.** `submit()` returns the same `BuiltTx`. The hash is `built.txHash`.
- **`submit()` uses the provider.** `wallet.submitTx` runs only when the wallet is the `submitter`.
- **Signing is partial and has no flag.** A key that did not sign is in `built.missing`.
- **Collateral is automatic.** Every Plutus transaction gets collateral, a total collateral and a collateral return.
- **Execution units are evaluated.** The default evaluator is local. `evaluator: provider` is not needed.
- **The datum of a spent script output is worked out.** An output with a datum hash needs `datum: { data }`.
- **Stake registration needs the stake key and states the deposit.** `registerStakeCertificate` did neither.
- **A withdrawal must be the whole balance.** `amount: 'all'` looks it up.
- **`build()` sets no `until`.** A plan sets one. Add `validity({ until })` when the transaction must expire.
- **Selection never spends an output a part only reads,** and never a script-locked output.
- **Chain state is not checked.** An unregistered stake address, an unknown pool or DRep, or an unknown governance action is refused by the ledger on submit.
- **`plan().run()` returns at `included`.** A rollback can still undo that. Release goods at `final`.

---

## Where the Old Builder Keeps Its Own Way

The old `MeshTxBuilder` is now an adapter over `TxBuilder`. It keeps some old behaviour so that old code works. New code does not copy it.

| Old builder | Direct `TxBuilder` |
|-------------|--------------------|
| Collateral given with `txInCollateral` is not topped up | Topped up by default |
| `registerStakeCertificate` is certificate 0 | `stakeRegistration` is certificate 7 and needs a witness |
| Deposits default to 500 ADA and 100,000 ADA | No default. `deposits`, or `deposit` on the part |
| Built-in protocol parameters when nobody can be asked | `MISSING_PARAMETER` |
| The network is guessed from the fetcher, `setNetwork` or the change address | `network` is required |
| Exact repeats of inputs and withdrawals are merged, and so are `.mint(...)` calls of one policy | `DUPLICATE_PART` |
| Redeemer units are kept when there is no evaluator | Taken unchecked only with `evaluator: null` |

Old code that still runs on the adapter also meets the new codes: `OUTPUT_BELOW_MIN_COIN`, `FEE_TOO_LOW`, `NETWORK_MISMATCH`, `DATUM_MISMATCH`.

A script transaction with a validity interval on a test network that nothing names is `EVALUATION_FAILED`. Pass `network` to the old builder.

---

## Debug Checklist

1. **Read the code and the details.** Log `error.code`, `error.label` and `error.details`.

2. **Name your parts.** Add a `label` to each. Errors repeat it.

3. **Read the result before signing:**
   ```typescript
   console.log(built.fee, built.warnings, built.missing, built.outputs);
   ```

4. **Trace the build.** The build option `trace` replaces `verbose: true`:
   ```typescript
   const built = await builder.build(parts, { payer: wallet, trace: (event) => console.log(event) });
   ```
