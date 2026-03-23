# RuleSubscribe: Proposed Fixes

## #1 — Capitalised error strings; `%v` should be `%w`

**File:** `rule_linux.go:471,478`

```go
// Current:
cberr(fmt.Errorf("Receive failed: %v", err))
cberr(fmt.Errorf("Wrong sender portid %d, expected %d", from.Pid, nl.PidKernel))

// Proposed:
cberr(fmt.Errorf("receive failed: %w", err))
cberr(fmt.Errorf("wrong sender port ID %d, expected %d", from.Pid, nl.PidKernel))
```

## #2 — Missing message-type guard before `deserializeRule`

**File:** `rule_linux.go:499`

Add after the `NLMSG_ERROR` block:

```go
if m.Header.Type != unix.RTM_NEWRULE && m.Header.Type != unix.RTM_DELRULE {
	if cberr != nil {
		cberr(fmt.Errorf("unexpected message type: %d", m.Header.Type))
	}
	continue
}
```

## #3 — `RuleUpdate.NlFlags` undocumented

**File:** `rule.go:34`

```go
// Current:
// RuleUpdate is sent when a rule changes.

// Proposed:
// RuleUpdate is sent when a rule changes.
// Type is unix.RTM_NEWRULE or unix.RTM_DELRULE.
// NlFlags is only meaningful for RTM_NEWRULE; unix.NLM_F_EXCL and
// unix.NLM_F_CREATE may be set.
```

## #4 — `listExisting` request missing `NLM_F_REQUEST`

**File:** `rule_linux.go:456`

```go
// Current:
req := pkgHandle.newNetlinkRequest(unix.RTM_GETRULE, unix.NLM_F_DUMP)

// Proposed:
req := pkgHandle.newNetlinkRequest(unix.RTM_GETRULE, unix.NLM_F_DUMP|unix.NLM_F_REQUEST)
```

## #5 — `expectRuleUpdate` timeout resets inside loop

**File:** `rule_test.go:700-712`

```go
// Current:
func expectRuleUpdate(ch <-chan RuleUpdate, msgType uint16, priority int) bool {
	for {
		timeout := time.After(time.Minute)
		select {
		case update := <-ch:
			if update.Type == msgType && update.Rule.Priority == priority {
				return true
			}
		case <-timeout:
			return false
		}
	}
}

// Proposed:
func expectRuleUpdate(ch <-chan RuleUpdate, msgType uint16, priority int) bool {
	timeout := time.After(time.Minute)
	for {
		select {
		case update := <-ch:
			if update.Type == msgType && update.Rule.Priority == priority {
				return true
			}
		case <-timeout:
			return false
		}
	}
}
```

## #6 — Data race on `lastError` in `TestRuleSubscribeWithOptions`

**File:** `rule_test.go:804,812-813`

```go
// Current:
var lastError error
defer func() {
	if lastError != nil {
		t.Fatalf("Fatal error received during subscription: %v", lastError)
	}
}()
// ...
ErrorCallback: func(err error) {
	lastError = err
},

// Proposed:
var lastError atomic.Value
defer func() {
	if v := lastError.Load(); v != nil {
		t.Fatalf("Fatal error received during subscription: %v", v)
	}
}()
// ...
ErrorCallback: func(err error) {
	lastError.Store(err)
},
```
