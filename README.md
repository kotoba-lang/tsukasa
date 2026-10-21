# tsukasa

司（つかさ）— to administer. **the itonami node ledger.** A node declares who
it is, how independent its failures are, and which lifecycle state it is in;
tsukasa keeps that record and refuses declarations that cannot be trusted.
That is all it does.

Naming: metaphor name (司), owner-approved 2026-10-21, registered in
`manifest/concept-vocabulary.edn` (com-junkawasaki/root). Candidates that were
rejected for sibling slots: `hako`（箱）and `fune`（船）— reserved for a future
container / microVM exec backing, not used here.

**This library decides nothing about where work runs.** Every function takes
small values and returns small values — records, verdicts, counts. No sockets,
no processes, no scheduling, no keys.

## The boundary, stated against each neighbour

| layer | repo | owns | tsukasa must not |
|---|---|---|---|
| node ledger (this) | `tsukasa` | registration records, declaration validation, lifecycle transitions, failure-domain audit | — |
| cluster control plane | `murakumo` | enrollment, heartbeat, job placement, `fleet ps` observation | observe or place |
| T6 placement | `sahai` | leases, budgets, checkpoints, epoch fencing | lease or schedule |
| VM contract | `kototama` | reduction, state, authority receipt | execute |
| machine / grants | `aiueos` | boot, devices, grant decisions | grant or attest |
| storage node mechanics | `kura-node` | shard movement, audits, payment | move bytes |

## What it refuses

The two declarations below have **no default**. A default is a lie about
somebody's reliability — the same rule `kura-node` and `kura-coordinator`
enforce for storage nodes, applied to itonami nodes generally.

1. **`:node/failure-domain` is required.** A node that does not declare how
   independent its failures are cannot be counted toward any durability
   claim. Format: a non-empty map (e.g. `{:provider "b2" :site "home"}`).
   How independent the domains are is the *caller's* policy; tsukasa only
   refuses the undeclared.
2. **`:node/availability` is required and closed.** Exactly `:always-on` or
   `:intermittent`. Omitted → refused, not defaulted. Unknown value → refused.

Refusals carry a **pinned reason keyword** (see `tsukasa.ledger/reasons`) and
leave the ledger unchanged. A test asserts the reason literal, not just "it
failed" — a refusal for the wrong reason is a different bug.

## Lifecycle

```text
:proposed → :admitted → :active ⇄ :draining → :retired
                ↘──────────────────────────→ :retired
```

`:retired` is terminal. An illegal transition is refused with
`:tsukasa/illegal-transition` and leaves the ledger unchanged.

## Layout

```text
src/tsukasa/ledger.cljk     pure model: registration, lifecycle, audit
test/tsukasa/ledger_test.cljk
test/tsukasa/test_runner.cljk
```

## Gates

```bash
kbb -M:test    # test-runner; throws on any failure
kbb -M:lint    # clj-kondo --fail-level error
```

kbb は移行中の互換ゲート（JUN-197）。正ゲートは `amu check --jvm-free` +
`amu test` に差し替え中 — `kotoba/tsukasa/ledger_core.kotoba` は check 緑
（`:ok true :effects #{}`、2026-10-21 実測）だが `amu test` の wasm leg が
`wasm-self-validation` internal error で赤。実測境界は
`docs/kotoba-port-boundary.md`。wasm leg が緑になるまで kbb ゲートは残す。

## Not implemented (measured boundary, not a plan)

- No persistence: the ledger is a value. Where records live (kotobase, EDN
  files, git) is the caller's decision.
- No network: enrollment and heartbeat belong to murakumo; tsukasa never
  learns node state on its own.
- No DID verification: `:node/did` is checked for shape (`did:key:` prefix)
  only. Signature verification is aiueos/kotoba-id territory.
- No placement arithmetic: survivability analysis beyond the failure-domain
  count in `audit` is kura's, not this repo's.
