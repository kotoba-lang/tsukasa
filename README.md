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
small values and returns small values — codes, verdicts, counts. No sockets,
no processes, no scheduling, no keys.

## The boundary, stated against each neighbour

| layer | repo | owns | tsukasa must not |
|---|---|---|---|
| node ledger (this) | `tsukasa` | registration verdicts, declaration validation, lifecycle transitions, failure-domain audit | — |
| cluster control plane | `murakumo` | enrollment, heartbeat, job placement, `fleet ps` observation | observe or place |
| T6 placement | `sahai` | leases, budgets, checkpoints, epoch fencing | lease or schedule |
| VM contract | `kototama` | reduction, state, authority receipt | execute |
| machine / grants | `aiueos` | boot, devices, grant decisions | grant or attest |
| storage node mechanics | `kura-node` | shard movement, audits, payment | move bytes |

## What it refuses

The two declarations below have **no default**. A default is a lie about
somebody's reliability — the same rule `kura-node` and `kura-coordinator`
enforce for storage nodes, applied to itonami nodes generally.

1. **`failure-domain` is required.** A node that does not declare how
   independent its failures are cannot be counted toward any durability
   claim. The host passes the domain as a string; how independent the domains
   are is the *caller's* policy, tsukasa only refuses the undeclared.
2. **`availability` is required and closed.** Exactly `"always-on"` or
   `"intermittent"`. Omitted → refused, not defaulted. Unknown value → refused.

Refusals carry a **pinned reason literal** — `"tsukasa/missing-failure-domain"`
and friends, the full set in `first-problem`'s branches. Tests assert the
literal, not just "it failed": a refusal for the wrong reason is a different
bug.

## Lifecycle

State codes: `0 proposed  1 admitted  2 active  3 draining  4 retired`.

```text
proposed → admitted → active ⇄ draining → retired
                ↘───────────────────────→ retired
```

`retired` is terminal. An illegal transition is refused with
`"tsukasa/illegal-transition"`; an out-of-range code is
`"tsukasa/unknown-state"` — a typo is never counted as a policy hit. A
refused transition returns no new ledger.

## Layout

```text
kotoba/tsukasa/ledger_core.kotoba   the whole component: verdicts, state
                                    machine, audit (pure, zero effects)
```

The former `.cljk` host wrapper and its kbb test suite were retired
2026-10-21 (JUN-197): the guest is the component, not a core beside one.

## Gates

```bash
amu check kotoba/tsukasa/ledger_core.kotoba --jvm-free
#   → {:ok true, :effects #{}}
amu test  kotoba/tsukasa/ledger_core.kotoba --jvm-free
#   → 18/18 passed (6 test-* × :jvm-kir :js :wasm)
```

Measured green 2026-10-21 on amu main `6c474c81`. The port constraints that
shaped this file (no keyword values in the test profile, `:vector-i64`-only
vectors, `(= bool false)` breaking wasm self-validation, …) are recorded in
`docs/kotoba-port-boundary.md`.

## Not implemented (measured boundary, not a plan)

- No persistence: the ledger is a value. Where records live (kotobase, EDN
  files, git) is the caller's decision.
- No network: enrollment and heartbeat belong to murakumo; tsukasa never
  learns node state on its own.
- No DID verification: `did` is checked for shape (`did:key:` prefix) only.
  Signature verification is aiueos/kotoba-id territory.
- No placement arithmetic: survivability analysis beyond the failure-domain
  count in `audit` is kura's, not this repo's.
- Ledger slots are bounded at 8 (the code range the audit scans). A larger
  fleet is the host's sharding problem until a sized collection is admitted.
