# Action Code Generation

## Problem

Adding support for a new protocol (DEX, lending, etc.) requires:
1. Writing a new Move action module
2. Writing staging functions
3. Writing TypeScript SDK handlers
4. Auditing all of the above

This doesn't scale. Each integration costs audit fees and dev time.

## Solution

**Audit a template once. Generate all action modules.**

```
action-definitions.ts  →  codegen script  →  generated .move files
       (data)                (audited)            (free)
```

## How It Works

### 1. Define actions as data

```typescript
// action-definitions.ts
{
  id: 'cetus_swap',
  package: '0xCETUS',
  module: 'router',
  function: 'swap',
  typeParams: ['CoinIn', 'CoinOut'],
  inputs: [{ name: 'coin_in', type: 'Coin', resourceName: true }],
  outputs: [{ name: 'coin_out', type: 'Coin', resourceName: true }],
}
```

### 2. Template generates Move code

```move
// GENERATED - DO NOT EDIT
module account_actions::cetus_swap;

public struct CetusSwap<phantom CoinIn, phantom CoinOut> has drop {}

public fun do_cetus_swap<Config, Outcome, CoinIn, CoinOut, IW: drop>(
    executable: &mut Executable<Outcome>,
    ...
) {
    action_validation::assert_action_type<CetusSwap<CoinIn, CoinOut>>(spec);

    let coin_in = executable_resources::take_coin(...);
    let coin_out = 0xCETUS::router::swap<CoinIn, CoinOut>(coin_in, ...);
    executable_resources::provide_coin(..., coin_out);

    executable::increment_action_idx(executable);
}
```

### 3. Same template generates TypeScript

```typescript
// GENERATED
registerAction('cetus_swap', (ctx, action) => {
  ctx.tx.moveCall({
    target: `${pkg}::cetus_swap::do_cetus_swap`,
    typeArguments: [configType, outcomeType, action.coinIn, action.coinOut, witnessType],
    arguments: [executable, ...],
  });
});
```

## Audit Model

| Component | Audit Status |
|-----------|--------------|
| Template (~100 lines) | Audit once |
| Generator script | Audit once |
| Core framework | Already audited |
| Generated modules (unlimited) | Inherited from template |

### Proving correctness

- CI regenerates files and diffs against committed versions
- Any manual edit to generated files fails CI
- `hash(generator + definitions) = deterministic output`

## Adding a New Protocol

```bash
# 1. Add definition
echo '{id: "aftermath_swap", package: "0xAFTERMATH", ...}' >> definitions.json

# 2. Regenerate
./scripts/generate-actions.sh

# 3. Commit
git add sources/generated/
git commit -m "Add Aftermath swap support"
```

No new audit required.

## Why This Works

Move requires explicit function calls - you can't dynamically dispatch. But you CAN generate the explicit calls from data.

The guarantee comes from:
1. Template correctness (audited)
2. Generator faithfulness (audited)
3. Deterministic output (CI-verified)

## Third-Party Verification

Anyone can verify generated on-chain code matches the generator:

### How It Works

1. **Generator + definitions are public** (open source repo)
2. **Generation is deterministic** (same input → same bytecode)
3. **Published package is immutable** (or additive-only policy)

### Verification Steps

```bash
# 1. Clone the generator repo
git clone https://github.com/govex/action-codegen

# 2. Run generator with same definitions
./generate.sh --definitions definitions.json

# 3. Build Move package
sui move build

# 4. Compare against on-chain bytecode
sui client verify-bytecode-meter --package <PACKAGE_ID>

# Or manually compare:
sui client object <PACKAGE_ID> --json | jq '.content.disassembled'
```

### Trust Model

| Component | Trust |
|-----------|-------|
| Generator source | Audited, open source |
| Definitions file | Data only, human-readable |
| On-chain bytecode | Verifiable via rebuild |
| Sui bytecode verifier | Part of Sui protocol |

**No new Sui primitives needed.** This is standard practice for audited contracts - publish source, anyone rebuilds and verifies bytecode hash matches.

## Implementation

**Language:** TypeScript (matches existing SDK, team expertise)

```
codegen/
├── templates/
│   ├── action.move.tmpl
│   ├── staging.move.tmpl
│   └── sdk-handler.ts.tmpl
├── generator.ts
└── definitions.json
```

Run with `npx tsx generator.ts`

### Estimate

| Task | Time |
|------|------|
| Move template | 2-3 hours |
| Staging template | 1-2 hours |
| TypeScript template | 1-2 hours |
| Generator script | 3-4 hours |
| CI integration | 1-2 hours |
| **Total** | **~2 days** |

## Status

Planning. Scheduled for Feb 2026.
