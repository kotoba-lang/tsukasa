# kotoba port — measured boundary (JUN-197, 2026-10-21)

amu main `6c474c81` (origin/main worktree, shared checkout's WIP lock bypassed).
Gate run: `bin/amu check kotoba/tsukasa/ledger_core.kotoba --jvm-free`.

## Green

- `amu check --jvm-free` → `{:ok true, :effects #{}, :admitted? true}` — the
  whole core (registration verdicts, lifecycle state machine, audit counts)
  is admitted pure.

## Red (measured, not worked around)

- `amu test` → `:error :wasm-self-validation` — "emitted Wasm module does not
  validate" (kotoba/internal-error). This is the HARNESS' wasm leg failing
  module self-validation, not a test failure: no test-* ran. `--targets
  :jvm-kir,:js` is not honored (same wasm path). Cause unknown; likely a
  codegen interaction with the nested-`if` decision trees or keyword returns.
  Suspect worth one bisect: replace keyword returns with i64 codes and map
  them in the host wrapper.

## Port constraints found by measurement (drift from the .cljk original)

- `cond->` refused (unbound threading symbol); `not=` absent; `string-starts-with?`
  absent (`string-index-of` == 0 substitutes); vector literals are `:vector-i64`
  only; `vals`/`count` need canonical typed maps/bounded vectors — the map
  accumulator ledger does NOT port today; params need type annotations when a
  symbol is used at more than one type; `bit-shift-left`/`i64-or` not builtins.
- Domain codes are caller-assigned i64 in 0..7 (audit scans that range);
  the host wrapper holds domain name strings.

## Not yet replaced

- kbb gates (`kbb -M:test` 12 tests / 38 assertions green) stay as the
  executable acceptance until `amu test` passes for this file. Removing them
  before the wasm leg is green would leave the repo with no running test.
