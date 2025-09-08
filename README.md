# murex-module-pareach

Parallel foreach for Murex, powered by rust-parallel.

This module is meant to provide an ergonomic `pareach` function that executes a Murex code block concurrently over items from stdin, aligning with rust-parallel features (wip):

- `--ordered` (keep order)
- `--collect` (gather results)
- `--concurrency/-n` (jobs)
- `--timeout` (per-task)
- `--halt-on-error` (exit on error)
- `--progress` (progress bar)

See docs/design/pareach-module.md for the full design.

## Status

Minimal, currently works like the following example:

```murex
ja %[1..3] -> set array
pareach $array '{ 1 + {1} }'
```

## Usage (planned)

```murex
ja [1..10] -> pareach --concurrency 4 --ordered i { out ($i * 2) }
```

## Development

- Dependency: rust-parallel (https://github.com/aaronriekenberg/rust-parallel). The binary is expected to be available on PATH (name typically `parallel`).
- Tests: TBD
