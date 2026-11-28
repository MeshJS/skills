# Transaction Builder Migration Skill

Skill for moving code from the deprecated `MeshTxBuilder` to `TxBuilder` in `@meshsdk/transaction`.

Part of [@meshsdk/ai-skills](../README.md).

## Coverage

- Mapping of every old call to a part or an option
- Payer, build or plan, signing and submitting
- Before and after recipes for common transactions
- Errors a migrated build may throw, and what behaves differently

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Main entry - workflow, quick reference |
| `MAPPING.md` | Old call to new part, by task |
| `PATTERNS.md` | Before and after recipes |
| `TROUBLESHOOTING.md` | Errors a migrated build may throw |

## Example Prompts

- "Migrate this file from MeshTxBuilder to TxBuilder"
- "What replaces `txIn` and `txInScript` in the new builder?"
- "Rewrite this mint transaction with parts"
- "Replace `complete()`, `signTx` and `submitTx` in this code"
- "Why does my migrated build throw `MISSING_SIGNATURES`?"

## Related Packages

- `@meshsdk/transaction` - The SDK package this skill documents
- `@meshsdk/core` - Full SDK (includes transaction)

## License

Apache-2.0
