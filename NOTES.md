# Codama challenge — notes

## Versions

| Tool | Version |
|---|---|
| anchor-cli | 1.1.2 |
| solana-cli | 3.1.10 |
| node | v24.14.1 |
| surfpool | 1.5.0 |
| @codama/cli | 1.6.3 |
| @codama/renderers-js | 2.5.0 |
| @solana/kit | 8.3.0 |
| @coral-xyz/anchor | 0.32.1 |

## TODO 3 — what I had to pass, and why

Calling `getContributeInstructionAsync` with the minimum input, TypeScript
still demands four accounts:

- **required:** `contributor`, `mintToRaise`, `fundraiser`, `vault`
- **optional (derived for me):** `contributorAccount`, `contributorAta`,
  `tokenProgram`, `systemProgram`

`contributor` and `mintToRaise` are required for the boring reason: they are
not PDAs at all, so there is nothing to derive them from. The interesting pair
is `fundraiser` and `vault`.

The rule is visible in the IDL. A seed is either an **account the caller
passes** or a **field inside an account**, and Codama's resolver can only use
what it already holds — it derives, it does not fetch. So the moment a seed
names `someAccount.someField`, the finder would have to read that account to
compute its own address.

In `contribute`:

```
fundraiser          seeds = ["fundraiser", fundraiser.maker]
vault               seeds = [fundraiser, <token program>, fundraiser.mint_to_raise]   (ATA)
contributor_account seeds = ["contributor", fundraiser, contributor]
contributor_ata     seeds = [contributor, <token program>, mint_to_raise]             (ATA)
```

`fundraiser` is seeded on `fundraiser.maker` — a field of the very account
being derived. You would need the account to find the account, so it has to be
passed in. `contributor_account` and `contributor_ata` seed only on accounts
already in hand (`fundraiser`, `contributor`, `mint_to_raise`), so they resolve.

In `initialize` the same account is optional, because the seed is different:

```
fundraiser seeds = ["fundraiser", maker]      <- `maker` is an account input
vault      seeds = [fundraiser, <tp>, mint_to_raise]
```

`maker` is passed as its own account there, so `findFundraiserPda({ maker })`
has everything it needs — and once `fundraiser` resolves, `vault` resolves too.
`initialize`, `check_contributions` and `refund` all call `findFundraiserPda`
in the generated client; `contribute` is the only one that does not.

**This is a program-design choice, not a Codama limitation.** `refund` declares
`maker` as an explicit account and seeds off `maker.key()`, so its
`fundraiser` is optional again. `contribute` chose not to take `maker`, which
saves one account in every contribute transaction and costs every client the
ability to derive the PDA. Neither is wrong; it is a trade, and the generated
client makes the trade visible.

## Bonus

Attempted and passing. `getContributeInstructionAsync` builds the instruction
from the same four inputs, `toWeb3Instruction` converts the Kit shape
(`programAddress` / `accounts[{address, role}]`) into the web3.js shape
(`programId` / `keys[{pubkey, isSigner, isWritable}]`), and
`provider.sendAndConfirm` sends it. The vault grows by exactly `AMOUNT`, so a
Codama-built instruction and an Anchor-built one are the same transaction as
far as the program is concerned.

The cap is 10% of the target per contributor — 3 tokens of a 30-token target —
and `before()` already contributed 1, so the bonus contribution is the second
of a possible three.

## One thing that surprised me

I assumed `vault` was only required as collateral damage: that once
`fundraiser` could not be derived, anything seeded on it followed. That is not
why. `vault` is an ATA of `(fundraiser, mint_to_raise)`, and I was passing
`fundraiser` explicitly — so the resolver had it. It is still required because
the IDL seeds the vault on **`fundraiser.mint_to_raise`**, the mint recorded
*inside* the fundraiser account, not on the `mint_to_raise` account sitting
right there in the same instruction. Those are the same value at runtime, and
`contribute` even enforces it with `has_one = mint_to_raise`. But the resolver
will not assume that: one is an input it holds, the other is a field it would
have to fetch, and it refuses to fetch.

So `vault` and `fundraiser` are each required for their own reason, and both
reasons are the same rule applied twice.

## One repo note

`anchor test` ran zero tests for me until I quoted the glob in `Anchor.toml`:

```diff
-test = "yarn run ts-mocha -p ./tsconfig.json -t 1000000 tests/**/*.ts"
+test = "yarn run ts-mocha -p ./tsconfig.json -t 1000000 'tests/**/*.ts'"
```

Anchor runs that script through `sh`, which has no `globstar`, so `tests/**/*.ts`
degrades to `tests/*/*.ts` and matches exactly one file — `tests/helpers/kit-adapter.ts`,
which contains no tests. Quoting it hands expansion to mocha, which understands
`**`. The unquoted form only looks fine in the earlier fundraiser repo because
`tests/` had no subdirectory there, so nothing matched and the literal string
reached mocha untouched.
