# Old to New Mapping

Every `MeshTxBuilder` call, with its `TxBuilder` counterpart. Read a row as: replace the old call with the new part.

All new names come from `@meshsdk/core`. `@meshsdk/transaction` exports them too. Parts go in one list: `builder.build([partA, partB], { payer })`.

Amounts, fees and deposits are `bigint`: `5_000_000n`. The exceptions are an `Asset[]` value in `output`, which keeps string quantities, and the `pledge` and `cost` of `poolRegistration({ params })`, which keep the old `PoolParams` strings.

## Table of Contents

- [How a Chain Splits Into Parts](#how-a-chain-splits-into-parts)
- [Builder and Options](#builder-and-options)
- [Payer, Selection and Change](#payer-selection-and-change)
- [Outputs](#outputs)
- [Spending](#spending)
- [Collateral](#collateral)
- [Mint and Burn](#mint-and-burn)
- [Withdrawals](#withdrawals)
- [Certificates](#certificates)
- [Votes and Proposals](#votes-and-proposals)
- [Shapes That Changed](#shapes-that-changed)
- [Time, Metadata, Signers and the Rest](#time-metadata-signers-and-the-rest)
- [Finish: Build, Sign, Submit](#finish-build-sign-submit)
- [Removed From the Old Builder](#removed-from-the-old-builder)
- [What Stays the Same](#what-stays-the-same)

---

## How a Chain Splits Into Parts

An old chain is a run of calls that all describe one item. A new part is one call with every argument.

| Old calls that describe one item | One new part |
|----------------------------------|--------------|
| `.txOut(...)` and its `.txOut*Datum*()` or `.txOutReferenceScript()` | `output({ address, value, datum?, scriptRef? })` |
| `.txIn(...)` and its script, datum and redeemer calls | `spend.pubKey`, `spend.native` or `spend.plutus` |
| `.mint(...)`, `.mintingScript(...)`, `.mintRedeemerValue(...)` | `mint.native` or `mint.plutus` |
| `.withdrawal(...)` and its script calls | `withdrawal.pubKey`, `.native` or `.plutus` |
| a certificate call and its `.certificate*()` script calls | `certificate.*` with `witness` |
| `.vote(...)` and its script calls | `vote({ voter, votes, witness? })` |
| `.proposal(...)` and its script calls | `proposal.*` with `guardrail` |

There is no call order to keep. Parts may come in any order, except certificates, proposals and outputs, which keep the order you give.

Parts may be one part, a list, nested lists, or `false`, `null`, `undefined`. A skipped value lets you write `condition && part`.

Every part takes an optional `label`. Errors repeat it.

---

## Builder and Options

| Old | New | Note |
|-----|-----|------|
| `new MeshTxBuilder({ fetcher })` | `new TxBuilder({ network, provider })` | `network` is required: `'mainnet'`, `'preprod'`, `'preview'` or `customNetwork(...)` |
| `submitter` | `submitter` | Same. Without it the provider submits |
| `evaluator: provider` | `evaluator: provider` | Not needed. The default is local. `null` turns evaluation off: give `exUnits` |
| `serializer` | `serializer` | Leave it out. An own one needs `serializeTxPrototype` |
| `params` (partial), `.protocolParams(p)` | `params` (a whole `Protocol`) | Read from the `provider`. With neither: `MISSING_PARAMETER` |
| `.setNetwork('preprod')` | `network: 'preprod'` | A constructor option, not a call |
| `.setCostModels(m)` | `costModels` | `number[][]`, V1 first. Else read from the provider |
| `isHydra: true` | `params` of the head, or the head's provider | No flag |
| `verbose: true` | build option `trace` | The builder never logs by itself |
| `.txEvaluationMultiplier` | none | It multiplies nothing in the old builder either |
| `selector` | none | Leave it out. `coinSelection` replaces a custom rule |
| `.reset()` | none | The builder holds no transaction state |
| `new MeshTxBuilder()` per transaction | one `TxBuilder` for all | `builder.add(...parts)` returns a new builder with parts held |
| deposits (defaults inside) | `deposits: { drepDeposit, govActionDeposit }` | New option. See [Certificates](#certificates) |
| the emulator as `fetcher`, `evaluator` and `submitter` | `provider: emulator` | One option |

---

## Payer, Selection and Change

| Old | New | Note |
|-----|-----|------|
| `wallet.getUtxos()`, `wallet.getChangeAddress()`, `.changeAddress(a)`, `.selectUtxosFrom(u)` | `build(parts, { payer: wallet })` | A `MeshWallet`, a `BrowserWallet` or a CIP-30 api, read at the build |
| the same, with the data in hand | `{ payer: { changeAddress, utxos } }` | Nothing is read. `utxos` are Mesh `UTxO`s or CIP-30 CBOR |
| `.changeAddress(a)` with no `selectUtxosFrom` | `{ payer: address }` | Its UTxOs are read through the provider |
| the fixed rule behind `.selectUtxosFrom(u)` | build options `canSelect`, `coinSelection` | Built-in: each token from its largest holders, then lovelace. `canSelect(utxo)` narrows the candidates. `coinSelection(need, candidates, context)` replaces the rule |
| none | build option `sendAllTo` | Spends every selectable UTxO of the payer and sends the rest to one address |
| `.setFee(f)` | build option `fee: bigint` | `FEE_TOO_LOW` when the minimum is higher. A given fee is never raised |
| `.chainTx(txHex)` | build option `after` | A `BuiltTx`, a plan, or a list. For a hex made elsewhere: `builder.fromCbor(txHex)` first |
| `.inputForEvaluation(utxo)` | build option `knownUtxos` | Outputs the provider may not know yet |
| `.signingKey(k)`, `.completeSigning()` | `built.sign({ signers: keySigner(k) })` | `k` is the text of a signing key, such as a key file's `cborHex` |

The change address is the payer's. There is no separate option.

Script-locked UTxOs and UTxOs that a part only reads are never selected for payment.

---

## Outputs

| Old | New | Note |
|-----|-----|------|
| `.txOut(address, amount)` | `output({ address, value })` | `value` is `{ lovelace: 5_000_000n, [unit]: 1n }` or the old `Asset[]`. No `lovelace`: the output holds its minimum |
| `.txOutInlineDatumValue(d, type?)` | `output({ ..., datum: { type: 'inline', data } })` | `data` is Mesh `Data`, or `{ type: 'Mesh' \| 'JSON' \| 'CBOR', content }` |
| `.txOutDatumHashValue(d, type?)` | `output({ ..., datum: { type: 'hash', data } })` | For a known hash with no datum: `{ type: 'hash', hash }` |
| `.txOutDatumEmbedValue(d, type?)` | `output({ ..., datum: { type: 'hash', data, embed: true } })` | The datum goes in the witness set |
| `.txOutReferenceScript(cbor, version = 'V3')` | `output({ ..., scriptRef: { code, version } })` | Write the version yourself. A native script is a `NativeScript` object, not CBOR |

A reward address as an output address is `INVALID_ADDRESS`. A script address for a lock output: `builder.scriptAddress(script, stake?)`.

---

## Spending

| Old | New | Note |
|-----|-----|------|
| `.txIn(hash, index, amount?, address?)` | `spend.pubKey({ utxo })` | `utxo` is `{ txHash, outputIndex }` (looked up), or a full UTxO. Pass the one you hold |
| `.txIn(...).txInScript(nativeCbor)` | `spend.native({ utxo, script })` | `script` is a `NativeScript`, its CBOR, or `{ reference }` |
| `.spendingPlutusScriptV2().txIn(...).txInScript(code).txInRedeemerValue(r)` | `spend.plutus({ utxo, script: { code, version: 'V2' }, redeemer: { data: r } })` | One call. No marker call, no call order |
| `.txInInlineDatumPresent()` | leave `datum` out | The datum is worked out from the output. `datum: 'inline'` says it |
| `.txInDatumValue(d)` | `datum: { data: d }` on `spend.plutus` | For an output with a datum hash |
| `.txInRedeemerValue(r, type, exUnits)` | `redeemer: { data, exUnits? }` | No `exUnits`: they are evaluated. Stated units are checked |
| `.spendingTxInReference(hash, index, size?, scriptHash?)` | `script: { reference: { txHash, outputIndex } }` | Size, hash and version are read from the output |
| `.simpleScriptTxInReference(hash, index, scriptHash?, size?)` | `spend.native({ utxo, script: { reference } })` | |
| `.spendingReferenceTxInInlineDatumPresent()`, `.spendingReferenceTxInRedeemerValue(...)` | as the plain forms above | Aliases |
| `.readOnlyTxInReference(hash, index, size?)` | `referenceInput({ utxo })` | An output that is also spent is not a reference input |

`redeemer.data` may be a function of the final transaction: `(tx) => mConStr0([tx.self, tx.output('payout')])`.

A key spend of a script-locked output, or the reverse, is `WITNESS_MISMATCH`.

---

## Collateral

| Old | New | Note |
|-----|-----|------|
| `wallet.getCollateral()` and `.txInCollateral(hash, index, amount, address)` | nothing | Collateral is chosen from the payer's UTxOs for any Plutus transaction |
| the same, when the wallet's collateral must be used | `collateral({ utxo })` | Topped up from the payer when short. Build option `topUpCollateral: false` turns that off |
| `.setTotalCollateral(s)`, `.setCollateralReturnAddress(a)` | `collateralReturn({ address, totalCollateral })` | Build option `collateralReturn: false` means no return and no total |

`collateralReturn: false` puts all the lovelace of the collateral inputs at stake. It adds the warning `COLLATERAL_FORFEITED`.

---

## Mint and Burn

| Old | New | Note |
|-----|-----|------|
| `.mint('1', policyId, nameHex).mintingScript(nativeCbor)` | `mint.native({ script, assets: { [nameHex]: 1n } })` | A `ForgeScript` result goes in as it is. The policy id comes from the script |
| `.mint('-1', ...)` | `assets: { [nameHex]: -1n }` | A negative quantity burns |
| `.mintPlutusScriptV2().mint(...).mintingScript(code).mintRedeemerValue(r)` | `mint.plutus({ script: { code, version: 'V2' }, redeemer: { data: r }, assets })` | |
| `.mintTxInReference(hash, index, size?, scriptHash?)` | `mint.plutus({ script: { reference: { txHash, outputIndex } }, redeemer, assets, policyId? })` | `policyId` only for a script given by out ref alone |
| `.mintReferenceTxInRedeemerValue(...)` | as `.mintRedeemerValue(...)` | Alias |
| several `.mint(...)` of one policy | one `mint.*` part with all names in `assets` | A second part of one policy is `DUPLICATE_PART` |

---

## Withdrawals

| Old | New | Note |
|-----|-----|------|
| `.withdrawal(rewardAddress, coin)` | `withdrawal.pubKey({ rewardAddress, amount })` | `amount` is `bigint`, the whole balance, or `'all'` to look it up |
| `.withdrawalPlutusScriptV2().withdrawal(a, c).withdrawalScript(code).withdrawalRedeemerValue(r)` | `withdrawal.plutus({ rewardAddress?, amount, script: { code, version: 'V2' }, redeemer: { data: r } })` | `amount: 0n` is withdraw-zero. No `rewardAddress`: the script's own |
| `.withdrawal(a, c).withdrawalScript(nativeCbor)` | `withdrawal.native({ rewardAddress?, amount, script })` | |
| `.withdrawalTxInReference(...)` | `script: { reference }` on `withdrawal.plutus` or `.native` | |

---

## Certificates

| Old | New | Note |
|-----|-----|------|
| `.registerStakeCertificate(a)` | `certificate.stakeRegistration({ stake })` | Not the same certificate. It states the deposit and needs the stake key's signature |
| `.deregisterStakeCertificate(a)` | `certificate.stakeDeregistration({ stake })` | The refund defaults to `keyDeposit`. `refund?` overrides it |
| `.delegateStakeCertificate(a, pool)` | `certificate.stakeDelegation({ stake, pool })` | `pool` is the bech32 id or the hex hash. No `deserializePoolId` |
| `.voteDelegationCertificate(drep, a)` | `certificate.voteDelegation({ stake, drep })` | `drep` is a DRep id, `'alwaysAbstain'` or `'alwaysNoConfidence'` |
| `.drepRegistrationCertificate(id, anchor?, coin?)` | `certificate.drepRegistration({ drep, anchor?, deposit? })` | The anchor is `{ url, dataHash }` |
| `.drepUpdateCertificate(id, anchor?)` | `certificate.drepUpdate({ drep, anchor? })` | |
| `.drepDeregistrationCertificate(id, coin?)` | `certificate.drepDeregistration({ drep, refund? })` | |
| `.registerPoolCertificate(params)` | `certificate.poolRegistration({ params })` | `params` is the same Mesh `PoolParams` object. It owes `poolDeposit` unless the operator is in build option `registeredPools` |
| `.retirePoolCertificate(pool, epoch)` | `certificate.poolRetirement({ pool, epoch })` | |
| `.certificateScript(...)`, `.certificateTxInReference(...)`, `.certificateRedeemerValue(...)` | `witness: { native } \| { plutus, redeemer }` on the certificate part | For a script credential. Not on pool certificates |
| none | `stakeAndVoteDelegation`, `stakeRegistrationAndDelegation`, `voteRegistrationAndDelegation`, `stakeVoteRegistrationAndDelegation`, `committeeHotAuth`, `committeeColdResign` | New one-part forms. `stake`, `pool`, `drep`, `deposit?` as in the rows above |

- `registerStakeCertificate` and `deregisterStakeCertificate` made certificates 0 and 1. `certificate.legacyStakeRegistration` and `legacyStakeDeregistration` are those, `@deprecated`. New code uses `stakeRegistration` and `stakeDeregistration`.
- There is no default `drepDeposit` or `govActionDeposit`. Give `deposits` to the builder, or `deposit` on the part. With neither: `MISSING_PARAMETER`. `keyDeposit` comes from `params`.

---

## Votes and Proposals

| Old | New | Note |
|-----|-----|------|
| `.vote(voter, govActionId, procedure)` | `vote({ voter, votes: [{ action, vote, anchor? }] })` | One part per voter holds all of that voter's votes |
| `.votePlutusScriptV3().vote(...).voteScript(code).voteRedeemerValue(r)` | `vote({ ..., witness: { plutus: { code, version: 'V3' }, redeemer: { data: r } } })` | `voteTxInReference` becomes `plutus: { reference }`. A script DRep takes V3 only |
| `.vote(...).voteScript(nativeCbor)` | `vote({ ..., witness: { native: script } })` | |
| `.proposal({ kind: 'InfoAction', action: {} }, anchor, rewardAccount, deposit?)` | `proposal.info({ rewardAccount, anchor, deposit? })` | |
| `TreasuryWithdrawalsAction` | `proposal.treasuryWithdrawals({ rewardAccount, anchor, withdrawals })` | `withdrawals` maps a reward address to a `bigint` |
| `NoConfidenceAction` | `proposal.noConfidence({ rewardAccount, anchor, previous? })` | |
| `ParameterChangeAction` | `proposal.parameterChange({ rewardAccount, anchor, updates, previous?, guardrail?, policyHash? })` | `updates` uses the ledger's snake_case names. The old camelCase names are renamed |
| `UpdateCommitteeAction` | `proposal.updateCommittee({ rewardAccount, anchor, remove?, add?, quorum, previous? })` | `add` is `[{ credential, termLimit }]`. `quorum` is `{ numerator, denominator }` |
| `NewConstitutionAction` | `proposal.newConstitution({ rewardAccount, anchor, constitution: { anchor, scriptHash? }, previous? })` | |
| `HardForkInitiationAction` | `proposal.hardForkInitiation({ rewardAccount, anchor, protocolVersion: { major, minor }, previous? })` | |
| `.proposalScript(...)`, `.proposalTxInReference(...)`, `.proposalRedeemerValue(...)` | `guardrail: { native } \| { plutus, redeemer }`, `policyHash?` | Only on `treasuryWithdrawals` and `parameterChange` |

`deposit` on a proposal defaults to `govActionDeposit` from the builder's `deposits` option.

A DRep key vote is signed by a `MeshWallet`, or by a browser wallet connected with the CIP-95 extension.

---

## Shapes That Changed

| Old | New |
|-----|-----|
| `[{ unit: 'lovelace', quantity: '5000000' }]` | `{ lovelace: 5_000_000n }` (the `Asset[]` form is also accepted by `output`) |
| quantity `'1'`, `'-1'` | `1n`, `-1n` |
| script CBOR and a `spendingPlutusScriptV2()` marker | `{ code, version: 'V2' }` |
| out ref `{ txHash, txIndex }` | `{ txHash, outputIndex }` |
| gov action id `{ txHash, txIndex }` or `{ transactionId, govActionIndex }` | `{ txHash, index }` |
| `{ dRepId }`, `{ alwaysAbstain: null }`, `{ alwaysNoConfidence: null }` | `drepId`, `'alwaysAbstain'`, `'alwaysNoConfidence'` |
| voter `{ type: 'DRep', drepId }` | `{ drep: drepId }` |
| voter `{ type: 'StakingPool', keyHash }` | `{ pool: keyHash }` |
| voter `{ type: 'ConstitutionalCommittee', hotCred }` | `{ committee: { type: 'KEY', value: keyHash } }` |
| credential `{ type: 'KeyHash', keyHash }` or `{ type: 'ScriptHash', scriptHash }` | `{ type: 'KEY', value: keyHash }` or `{ type: 'SCRIPT', value: scriptHash }`, or a reward address. Any other shape is `INVALID_DATA` |
| `{ voteKind: 'Yes' }` | `vote: 'yes'` (`'no'`, `'abstain'`) |
| anchor `{ anchorUrl, anchorDataHash }` | `{ url, dataHash }` |
| pool id from `deserializePoolId(...)` | the bech32 pool id, or the hex hash |

---

## Time, Metadata, Signers and the Rest

| Old | New | Note |
|-----|-----|------|
| `.invalidBefore(slot)`, `.invalidHereafter(slot)` | `validity({ invalidBefore, invalidHereafter })` | Slots, as before. Two calls become one part. A second `validity` is `DUPLICATE_PART` |
| `resolveSlotNo(network, ms)`, then `.invalidHereafter(slot)` | `validity({ from, until })` | POSIX milliseconds as `number`, `bigint` or `Date`. Converted on the builder's network |
| `.metadataValue(label, obj)` | `metadata({ metadataLabel, value })` | `metadataLabel` is the key (`721n`). Text and bytes are at most 64 bytes each, else `INVALID_DATA` |
| `.metadataValue(674, { msg: [...] })` | `message({ text })` | `text` is a string or a list. Long text is cut into 64-byte lines |
| `.metadataScript(cbor, 'Native' \| 'PlutusV1' \| 'PlutusV2' \| 'PlutusV3')` | `auxiliaryScript({ script })` | `script` is a `NativeScript` object or `{ code, version }`. `'PlutusV2'` is `{ code, version: 'V2' }`. A native script in CBOR must be given as an object |
| `.requiredSignerHash(hash)` | `guard.pubKey({ keyHash })` | Or `guard.pubKey({ address })` for the payment key of an address |
| `.setDonation(s)` | `donation({ amount })` | `bigint` |
| none | `treasuryValue({ amount })`, `networkId({ network })` | New parts |

Rules:

- Give each validity bound in one way only: `from` or `invalidBefore`, `until` or `invalidHereafter`. Both ways for one bound is `INVALID_DATA`.
- `from: Date.now()` is valid from the next slot and warns `VALID_FROM_FUTURE`. For "valid from now" leave `from` out.
- `message` and `metadata` with label 674 together are `DUPLICATE_PART`.
- A `guard.pubKey` whose hash is one of the transaction's script hashes is `INVALID_GUARD`.
- Keys that must sign for inputs, certificates, withdrawals or votes need no `guard`. The builder lists them in `built.signers`.

---

## Finish: Build, Sign, Submit

| Old | New | Note |
|-----|-----|------|
| `.complete()` | `await builder.build(parts, { payer })` | Returns a `BuiltTx`, not a string. The old string is `built.tx` |
| `wallet.signTx(tx)`, `wallet.signTx(tx, true)` | `await built.sign()` | The payer signs. Partial by default, with no flag. For others: `built.sign({ signers: [a, b] })` |
| `wallet.submitTx(signed)` | `await built.submit()` | Through the provider. Through the wallet: `built.submit({ submitter: wallet })` |
| `txBuilder.submitTx(signedTx)` | `await built.submit()` | Through the builder's `submitter`, else the provider |
| the hash `submitTx` returned | `built.txHash` | Known before signing |
| sign, submit, then poll | `await built.awaitInclusion()` | Waits for a block. Returns `{ txHash, slot, block, blockHeight }` as far as the provider gives them |
| the whole build, sign, submit, wait | `const plan = await builder.plan(parts, { payer })`, `await plan.run()`, `plan.step('main').txHash` | A plan of one step. Needs a provider that returns block heights |
| `.completeSync()`, `.completeUnbalanced()`, `.completeUnbalancedSync()` | `assembleTxPrototype(parts, { fee, costModels })`, then the serializer, then `builder.fromCbor(tx)` | No balancing and no reads |
| `.calculateMinLovelaceForOutput(o)` | `minCoin(output(...), params)` | |
| `.calculateFee()`, `.getActualFee()` | `built.fee`, or `minFee(prototype, { params, vkeyWitnesses })` | `bigint` |
| `.getTotalExecutionUnits()` | `built.exUnits` | `{ mem, steps }` |
| `.complete(meshTxBody)`, `MeshTxBuilderBody` | a list of parts | The body is `@deprecated` |
| `txBuilder.meshTxBuilderBody` read for debugging | `built.parts`, `built.inputs`, `built.outputs`, `built.redeemers` | The result is plain data |

The signers a call accepts:

| Signer | Given as |
|--------|----------|
| a wallet | a CIP-30 api, a `MeshWallet`, a `BrowserWallet`, or any object with `signTxWitnesses(tx, keyHashes?)` or `signTx(tx, partial?)` |
| a private key | `keySigner(key)`. The key is hex, a key file's `cborHex`, or bech32 |
| your own code | an object with `signTxHash({ txHash, keys, tx, signal })`, or that function. It answers `{ vkey, signature }[]` |

A key goes to `keySigner(key)` and nowhere else. `signers: key` is `INVALID_OPTION`.

---

## Removed From the Old Builder

The old `MeshTxBuilder` is `@deprecated` and still works. It now runs on `TxBuilder`. These members of it are gone.

- `selectUtxos`, `getUtxosForSelection`, `evaluateRedeemers`, `completeCostModels`, `updateByTxPrototype`: use `build(parts, { payer })`.
- `sortTxParts`, `sortInputs`, `sortMints`, `sortWithdrawals`, `sortVotes`, `sortProposals`: the builder orders every item by the ledger's rules.
- The protected members (`completeTxParts`, `sanitizeOutputs`, ...): none.

---

## What Stays the Same

- Mesh `Data` helpers: `mConStr0` and the rest, `conStr0`, `integer`, `byteString`.
- `ForgeScript`, `resolveScriptHash`, `serializePlutusScript`, `deserializeAddress`, `hashDrepAnchor`, `metadataToCip68`, `CIP68_100`, `CIP68_222`.
- Plutus scripts are `{ code, version }`. A native script is a `NativeScript` object, or its CBOR in `spend.native`, `mint.native`, `withdrawal.native` and `witness: { native }`. `output({ scriptRef })` and `auxiliaryScript` take only the `NativeScript` object.
- A Mesh `UTxO` (`{ input, output }`) from a provider or a wallet is accepted wherever a part takes a UTxO.
- `TxParser` and `TxTester` still work on a body. They are not deprecated.
- `Transaction` and `MeshTxBuilderV2` keep working through the old builder. This skill does not map them.
