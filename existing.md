# RuleSubscribe: Comparison with Existing Codebase Patterns

Every review finding from the RuleSubscribe WIP is a pre-existing pattern
copied from sibling subscribe implementations. This document records which
siblings exhibit the same behaviour.

## Summary Table

| # | Finding | route | addr | link | neigh | New issue? |
|---|---------|-------|------|------|-------|------------|
| 1 | Capitalised error strings, `%v` not `%w` | Yes | Yes | Yes | Yes | No |
| 2 | No message-type guard before deserialise | Yes (absent) | No (has guard) | — | — | Split convention |
| 3 | `NlFlags` mask includes inapplicable flags / undocumented | Yes | N/A (no NlFlags) | N/A | N/A | No |
| 4 | `listExisting` missing `NLM_F_REQUEST` | Yes | Yes | Yes | Yes | No |
| 5 | Test helper timeout resets inside loop | Yes (`expectRouteUpdate`) | Yes (`expectAddrUpdate`) | — | — | No |
| 6 | Data race on `lastError` in test | Yes | Yes | — | — | No |
| 7 | `NLM_F_DUMP_INTR` doesn't skip message | Yes | N/A (not checked) | — | — | No |
| 8 | No bounds-check on attribute values | Yes (all attribute parsing project-wide) | Yes | Yes | Yes | No |

## Detail

### #1 — Capitalised error strings

All four sibling `*subscribeAt` functions use:
- `fmt.Errorf("Receive failed: %v", err)`
- `fmt.Errorf("Wrong sender portid %d, expected %d", ...)`

Fixing only in RuleSubscribe would be inconsistent with the rest of the codebase.

### #2 — Message-type guard

`addrSubscribeAt` validates `RTM_NEWADDR`/`RTM_DELADDR` before parsing.
`routeSubscribeAt` does **not** validate — it passes all non-DONE, non-ERROR
messages straight to `deserializeRoute`. RuleSubscribe follows the route
pattern (its closest structural analogue).

### #3 — NlFlags mask

`RouteUpdate` uses the identical mask (`NLM_F_REPLACE | NLM_F_EXCL |
NLM_F_CREATE | NLM_F_APPEND`) and the same struct shape. `RuleUpdate` is
a direct copy. Neither has a doc comment clarifying which flags are
semantically meaningful.

### #4 — Missing NLM_F_REQUEST in listExisting

All sibling subscribe implementations omit `NLM_F_REQUEST` in their
`listExisting` dump request. Only the non-subscribe list functions
(`RuleListFiltered`, `RouteListFiltered`, etc.) include it. The kernel
treats `NLM_F_DUMP` as implying `NLM_F_REQUEST`, so this is cosmetic.

### #5 — Test helper timeout resets

`expectRouteUpdate` (route_test.go:558-574) and `expectAddrUpdate`
(addr_test.go:237-249) both create `time.After` inside the `for` loop,
exhibiting the same potential infinite-loop bug.

### #6 — Data race on lastError

`TestRouteSubscribeWithOptions` and `TestAddrSubscribeWithOptions` use
the same unsynchronised `var lastError error` pattern with a write from
the callback goroutine and a read from the test goroutine's deferred
function.

### #7 — NLM_F_DUMP_INTR fall-through

`routeSubscribeAt` checks the flag and calls `cberr(ErrDumpInterrupted)`
but does not skip the message — identical to the new RuleSubscribe code.
`addrSubscribeAt` does not check `NLM_F_DUMP_INTR` at all.

### #8 — No bounds-check on attribute values

All attribute parsing across the entire project (routes, addresses, links,
neighbours, rules) indexes into `attrs[j].Value[0:4]` etc. without length
validation. This is a project-wide convention, not a regression.

## Conclusion

The RuleSubscribe implementation is **more correct** than its siblings in
one respect: it closes the netlink socket on early error returns after
`SubscribeAt` succeeds (the `s.Close()` calls added in the setup path).
None of `routeSubscribeAt`, `addrSubscribeAt`, `linkSubscribeAt`, or
`neighSubscribeAt` do this — they all leak the socket on setup failure.

The only fix worth applying purely within RuleSubscribe scope is **#5**
(move `time.After` outside the loop) — it is a one-line change, low risk,
and prevents a real (if unlikely) infinite loop in tests.
