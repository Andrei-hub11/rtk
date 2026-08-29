# Bug: `npx next` subcommands can be reported as a successful build

## Summary

`rtk npx next <subcommand>` routes every Next.js subcommand through the
`next build` output filter. For subcommands that are not builds, this can
replace their real output with a synthetic, successful-looking build summary.

## Reproduction

1. Remove Next.js generated route types:

   ```bash
   rm -rf .next/types
   ```

2. Run Next.js route type generation through rtk:

   ```bash
   npx next typegen
   ```

3. Observe rtk's output:

   ```text
   Next.js Build
   ═══════════════════════════════════════
   Errors: 0 | Warnings: 0
   ```

4. Check whether `.next/types/routes.d.ts` was generated. In the affected
   case, it was not, and a subsequent `tsc` run could not resolve a new route.

Running the same command through the raw proxy shows the expected Next.js
output and generates the file:

```bash
rtk proxy npx next typegen
```

```text
Generating route types...
✓ Types generated successfully
```

## Expected behavior

rtk should only apply the compact Next.js build filter to `next build`.
Unsupported Next.js subcommands, including `next typegen`, should preserve the
underlying command's arguments, output, and exit code.

## Actual behavior

The `rtk npx` dispatcher routes `next` directly to the build executor:

```text
rtk npx next <args> -> next_cmd::run(<args>) -> next build <args>
```

`next_cmd::run` is designed for the `rtk next` build shortcut, so it always
appends `build` and always applies `filter_next_build`. As a result, the filter
can show `Errors: 0 | Warnings: 0` even though the requested subcommand did not
run as intended.

## Impact

This is a false-success failure mode. It can hide missing generated artifacts
and delay discovery until another tool fails. The generated summary should not
be used as evidence that an unrecognized Next.js subcommand succeeded.

## Proposed fix

Classify `rtk npx next` invocations before executing them:

| Invocation | Behavior |
| --- | --- |
| `rtk npx next build [args]` | Run `next build [args]` with the compact build filter. |
| `rtk npx next <any other subcommand> [args]` | Run without filtering and forward the real output, arguments, and exit code. |

The passthrough path should use the same command-resolution behavior as the
filtered path: prefer a local `next` binary and fall back to `npx next` when
needed.

## Regression coverage

A regression test should assert that `typegen --flag` is classified as a
passthrough invocation with all arguments intact, while `build --turbo` is the
only invocation classified for filtering. A mutation that forces `typegen`
down the build path must make that test fail.

## Relevant project pattern

This follows the existing convention of filtering only recognized invocations
and using passthrough otherwise. For example, `golangci_cmd` distinguishes
`Invocation::{FilteredRun, Passthrough}`, and `mvn_cmd` uses
`MvnPhase::Passthrough` for unsupported phases.
