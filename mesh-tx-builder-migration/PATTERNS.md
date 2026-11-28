# Migration Patterns

Before and after code for the recipes you meet most. Each pair stops at the build. Signing and submitting are in [The Frame](#the-frame).

The examples use `txBuilder` (a `MeshTxBuilder`), `builder` (a `TxBuilder`), `wallet`, `utxos` and `changeAddress`, which your app defines. The other names are clear from context. These are not: `unit` (policy id plus asset name in hex), `policyCode`, `scriptUtxo`, `collateralUtxo` and `anchor` (`{ url, dataHash }`).

## Table of Contents

- [The Frame](#the-frame)
- [Several Outputs and Metadata](#several-outputs-and-metadata)
- [Native Mint and Burn](#native-mint-and-burn)
- [Plutus Mint](#plutus-mint)
- [Lock at a Script Address](#lock-at-a-script-address)
- [Plutus Spend With a Reference Script](#plutus-spend-with-a-reference-script)
- [Staking](#staking)
- [Governance](#governance)
- [Required Signers and Validity](#required-signers-and-validity)
- [Several Signers](#several-signers)
- [A Browser Wallet Flow](#a-browser-wallet-flow)

---

## The Frame

Every old chain ends the same way: `changeAddress`, `selectUtxosFrom`, `complete()`, then the wallet signs and submits. The new code has the same three steps.

```typescript
import { MeshTxBuilder, output, TxBuilder } from '@meshsdk/core';

// Before
const oldBuilder = new MeshTxBuilder({ fetcher: provider });
const utxos = await wallet.getUtxos();
const changeAddress = await wallet.getChangeAddress();
const unsignedTx = await oldBuilder
  .txOut(recipient, [{ unit: 'lovelace', quantity: '5000000' }])
  .changeAddress(changeAddress)
  .selectUtxosFrom(utxos)
  .complete();
const signedTx = await wallet.signTx(unsignedTx);
const txHash = await wallet.submitTx(signedTx);

// After
const newBuilder = new TxBuilder({ network: 'preprod', provider });
const built = await newBuilder.build(
  output({ address: recipient, value: { lovelace: 5_000_000n } }),
  { payer: wallet },
);
await built.sign();
await built.submit();
console.log(built.txHash);
```

Create the `TxBuilder` once and reuse it.

`build()` signs and sends nothing. `built.tx` is the unsigned CBOR that `complete()` returned.

`submit()` goes through the provider and returns the same `BuiltTx`. The hash is `built.txHash`.

The later recipes use the names `txBuilder` and `builder` and show only the part that changes.

---

## Several Outputs and Metadata

```typescript
import { message, output } from '@meshsdk/core';

// Before
const unsignedTx = await txBuilder
  .txOut(recipient, [{ unit: 'lovelace', quantity: '5000000' }])
  .txOut(partner, [
    { unit: 'lovelace', quantity: '2000000' },
    { unit, quantity: '1' },
  ])
  .metadataValue(674, { msg: ['Invoice 42'] })
  .changeAddress(changeAddress)
  .selectUtxosFrom(utxos)
  .complete();

// After
const built = await builder.build(
  [
    output({ address: recipient, value: { lovelace: 5_000_000n } }),
    output({
      address: partner,
      value: { lovelace: 2_000_000n, [unit]: 1n },
    }),
    message({ text: 'Invoice 42' }),
  ],
  { payer: wallet },
);
```

`message` is metadata label 674. For another label use `metadata({ metadataLabel, value })`.

Leave `lovelace` out and the output holds its minimum.

---

## Native Mint and Burn

```typescript
import { ForgeScript, metadata, mint, resolveScriptHash, stringToHex } from '@meshsdk/core';

const forgingScript = ForgeScript.withOneSignature(changeAddress);
const policyId = resolveScriptHash(forgingScript);
const assetNameHex = stringToHex('MeshToken');
const nft = {
  [policyId]: { MeshToken: { name: 'Mesh Token', image: 'ipfs://...' } },
};

// Before: mint
const unsignedMint = await txBuilder
  .mint('1', policyId, assetNameHex)
  .mintingScript(forgingScript)
  .metadataValue(721, nft)
  .changeAddress(changeAddress)
  .selectUtxosFrom(utxos)
  .complete();

// After: mint
const builtMint = await builder.build(
  [
    mint.native({ script: forgingScript, assets: { [assetNameHex]: 1n } }),
    metadata({ metadataLabel: 721n, value: nft }),
  ],
  { payer: wallet },
);

// Burn: the same calls with a negative quantity
// Before: .mint('-1', policyId, assetNameHex).mintingScript(forgingScript)
// After:
const burn = mint.native({
  script: forgingScript,
  assets: { [assetNameHex]: -1n },
});
```

Put every asset name of one policy in one `assets` record. A second `mint` part for the same policy is `DUPLICATE_PART`.

---

## Plutus Mint

```typescript
import { mConStr0, mint, resolveScriptHash, stringToHex } from '@meshsdk/core';

const policyId = resolveScriptHash(policyCode, 'V2');
const assetNameHex = stringToHex('MeshToken');

// Before
const unsignedTx = await txBuilder
  .mintPlutusScriptV2()
  .mint('1', policyId, assetNameHex)
  .mintingScript(policyCode)
  .mintRedeemerValue(mConStr0([]))
  .txInCollateral(
    collateralUtxo.input.txHash,
    collateralUtxo.input.outputIndex,
    collateralUtxo.output.amount,
    collateralUtxo.output.address,
  )
  .changeAddress(changeAddress)
  .selectUtxosFrom(utxos)
  .complete();

// After
const built = await builder.build(
  mint.plutus({
    script: { code: policyCode, version: 'V2' },
    redeemer: { data: mConStr0([]) },
    assets: { [assetNameHex]: 1n },
  }),
  { payer: wallet },
);
```

The script version moves into `script`.

Collateral is chosen for you. Leave the collateral call out.

Leave `exUnits` out of the redeemer. The builder evaluates the script.

---

## Lock at a Script Address

```typescript
import { mConStr0, output, serializePlutusScript } from '@meshsdk/core';

const { address: scriptAddress } = serializePlutusScript(script);

// Before
const unsignedTx = await txBuilder
  .txOut(scriptAddress, [{ unit: 'lovelace', quantity: '10000000' }])
  .txOutInlineDatumValue(mConStr0([]))
  .changeAddress(changeAddress)
  .selectUtxosFrom(utxos)
  .complete();

// After
const built = await builder.build(
  output({
    address: scriptAddress,
    value: { lovelace: 10_000_000n },
    datum: { type: 'inline', data: mConStr0([]) },
  }),
  { payer: wallet },
);
```

`builder.scriptAddress(script)` gives the same address on the builder's network.

A datum hash is `{ type: 'hash', data }`. Add `embed: true` to put the datum in the witness set.

---

## Plutus Spend With a Reference Script

```typescript
import { mConStr0, output, spend } from '@meshsdk/core';

// Before
const unsignedTx = await txBuilder
  .spendingPlutusScriptV2()
  .txIn(
    scriptUtxo.input.txHash,
    scriptUtxo.input.outputIndex,
    scriptUtxo.output.amount,
    scriptUtxo.output.address,
  )
  .txInInlineDatumPresent()
  .txInRedeemerValue(mConStr0([]))
  .spendingTxInReference(refTxHash, 0)
  .txInCollateral(
    collateralUtxo.input.txHash,
    collateralUtxo.input.outputIndex,
    collateralUtxo.output.amount,
    collateralUtxo.output.address,
  )
  .txOut(recipient, [{ unit: 'lovelace', quantity: '5000000' }])
  .changeAddress(changeAddress)
  .selectUtxosFrom(utxos)
  .complete();

// After
const built = await builder.build(
  [
    spend.plutus({
      utxo: scriptUtxo,
      script: { reference: { txHash: refTxHash, outputIndex: 0 } },
      redeemer: { data: mConStr0([]) },
    }),
    output({ address: recipient, value: { lovelace: 5_000_000n } }),
  ],
  { payer: wallet },
);
```

One `spend.plutus` replaces the whole chain from `spendingPlutusScriptV2()` to `spendingTxInReference()`.

Pass the UTxO you already have. A bare `{ txHash, outputIndex }` is looked up through the provider.

The datum is worked out from the output. Leave `txInInlineDatumPresent()` out.

The script's version and size are read from the reference output.

For a script given inline, use `script: { code, version: 'V2' }`.

---

## Staking

```typescript
import { certificate, deserializePoolId, withdrawal } from '@meshsdk/core';

// Before: register and delegate
const unsignedTx = await txBuilder
  .registerStakeCertificate(rewardAddress)
  .delegateStakeCertificate(rewardAddress, deserializePoolId(poolId))
  .changeAddress(changeAddress)
  .selectUtxosFrom(utxos)
  .complete();

// After: register and delegate
const built = await builder.build(
  [
    certificate.stakeRegistration({ stake: rewardAddress }),
    certificate.stakeDelegation({ stake: rewardAddress, pool: poolId }),
  ],
  { payer: wallet },
);

// Before: withdraw
const unsignedWithdraw = await txBuilder
  .withdrawal(rewardAddress, '5000000')
  .changeAddress(changeAddress)
  .selectUtxosFrom(utxos)
  .complete();

// After: withdraw
const builtWithdraw = await builder.build(
  withdrawal.pubKey({ rewardAddress, amount: 5_000_000n }),
  { payer: wallet },
);
```

`poolId` is the bech32 pool id. Drop `deserializePoolId`.

Certificates keep the order you give them.

`stakeRegistration` states the deposit and needs the stake key's signature. `registerStakeCertificate` did neither.

A withdrawal must be the whole reward balance. `amount: 'all'` looks it up.

---

## Governance

```typescript
import { certificate, vote } from '@meshsdk/core';

// Before
const unsignedTx = await txBuilder
  .voteDelegationCertificate({ dRepId: drepId }, rewardAddress)
  .vote(
    { type: 'DRep', drepId },
    { txHash: actionTx, txIndex: 0 },
    { voteKind: 'Yes' },
  )
  .changeAddress(changeAddress)
  .selectUtxosFrom(utxos)
  .complete();

// After
const built = await builder.build(
  [
    certificate.voteDelegation({ stake: rewardAddress, drep: drepId }),
    vote({
      voter: { drep: drepId },
      votes: [{ action: { txHash: actionTx, index: 0 }, vote: 'yes' }],
    }),
  ],
  { payer: wallet },
);

// Register as a DRep. The anchor is { url, dataHash }
// Before: .drepRegistrationCertificate(drepId, { anchorUrl, anchorDataHash })
// After:
const register = certificate.drepRegistration({ drep: drepId, anchor });
```

`drep` is a DRep id, `'alwaysAbstain'` or `'alwaysNoConfidence'`.

One `vote` part holds all the votes of one voter.

A DRep key vote is signed by a `MeshWallet`, or by a browser wallet connected with CIP-95.

A DRep registration or a proposal pays a deposit. There is no default. Give `deposits: { drepDeposit, govActionDeposit }` to the `TxBuilder`, or `deposit` to the part. Without one, the build throws `MISSING_PARAMETER`.

---

## Required Signers and Validity

```typescript
import { deserializeAddress, guard, output, resolveSlotNo, validity } from '@meshsdk/core';

const { pubKeyHash } = deserializeAddress(owner);
const slot = resolveSlotNo('preprod', Date.now() + 3_600_000);

// Before
const unsignedTx = await txBuilder
  .txOut(recipient, [{ unit: 'lovelace', quantity: '5000000' }])
  .requiredSignerHash(pubKeyHash)
  .invalidHereafter(Number(slot))
  .changeAddress(changeAddress)
  .selectUtxosFrom(utxos)
  .complete();

// After
const built = await builder.build(
  [
    output({ address: recipient, value: { lovelace: 5_000_000n } }),
    guard.pubKey({ address: owner }),
    validity({ until: Date.now() + 3_600_000 }),
  ],
  { payer: wallet },
);
```

`guard.pubKey` takes `{ address }` or `{ keyHash }`.

`validity` takes times in milliseconds. Drop `resolveSlotNo`.

Both bounds go in one part: `validity({ from, until })`. Slots still work: `validity({ invalidBefore, invalidHereafter })`.

`build()` sets no `until` for you. Add one when the transaction must expire.

---

## Several Signers

```typescript
// Before: unsignedTx comes from complete(). After: built comes from build().

// Before
const signed1 = await wallet.signTx(unsignedTx, true);
const signed2 = await mintingWallet.signTx(signed1, true);
const txHash = await wallet.submitTx(signed2);

// After
await built.sign({ signers: [wallet, mintingWallet] });
await built.submit();
```

Each signer is asked once, in order, for the keys still missing.

`built.missing` lists the keys that have not signed. `submit()` throws `MISSING_SIGNATURES` while any is missing, and sends nothing.

A key from a file goes through `keySigner(key)`.

---

## A Browser Wallet Flow

```typescript
import { BrowserWallet, output } from '@meshsdk/core';

const browserWallet = await BrowserWallet.enable(walletName);

// Before
const browserUtxos = await browserWallet.getUtxos();
const browserChange = await browserWallet.getChangeAddress();
const unsignedTx = await txBuilder
  .txOut(recipient, [{ unit: 'lovelace', quantity: '2000000' }])
  .changeAddress(browserChange)
  .selectUtxosFrom(browserUtxos)
  .complete();
const signedTx = await browserWallet.signTx(unsignedTx);
const txHash = await browserWallet.submitTx(signedTx);

// After
const built = await builder.build(
  output({ address: recipient, value: { lovelace: 2_000_000n } }),
  { payer: browserWallet },
);
await built.sign();
await built.submit({ submitter: browserWallet });
console.log(built.txHash);
```

The wallet is the payer. It gives its UTxOs and change address, and it signs.

`submit()` uses the provider by default. Pass `{ submitter: browserWallet }` to submit through the wallet.

To follow the transaction to a block, call `await built.awaitInclusion()`.

A raw CIP-30 api from `enable()` also works as `payer` and as a signer.
