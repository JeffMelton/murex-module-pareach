# murex-module-pareach

Parallel foreach for Murex, powered by rust-parallel.

This module provides an ergonomic `pareach` function that executes a Murex code block concurrently over items from stdin, aligning with rust-parallel features:

- `--ordered` (keep order)
- `--collect` (gather results)
- `--concurrency/-n` (jobs)
- `--timeout` (per-task)
- `--halt-on-error` (exit on error)
- `--progress` (progress bar)

See docs/design/pareach-module.md for the full design.

## Status

Scaffold only. The `pareach` function currently reports a helpful error until the engine integration is implemented.

## Usage (planned)

```murex
ja [1..10] -> pareach --concurrency 4 --ordered i { out ($i * 2) }
```

## Development

- Dependency: rust-parallel (https://github.com/aaronriekenberg/rust-parallel). The binary is expected to be available on PATH (name typically `parallel`).
- Tests: Behavioural `.mx` tests live under `behavioural/`. You can run them by launching `murex` and sourcing the module:

```murex
source pareach.mx
source behavioural/pareach_missing_engine.mx
source behavioural/pareach_flags_parse.mx
# Then run tests
test run *
```

Alternatively, inline `test run { ... }` blocks within the test files can be used to execute specific scenarios.
