# FIRM Conformance Test Suite
# Tests how well an LLM runtime holds FIRM constructs and invariants.
#
# ============================================================
# HOW TO RUN
# ============================================================
#
# Option A — Conformance test (bootstrap only):
#   Load bootstrap.md + this file into a fresh LLM conversation.
#   Say: "run tests"
#   This tests how well the LLM holds FIRM with minimal instructions.
#
# Option B — Baseline test (full spec):
#   Load dev.md + this file into a fresh LLM conversation.
#   Say: "run conformance tests"
#   This tests with full spec available — if tests fail here too,
#   the spec itself has a problem.
#
# Compare A vs B to evaluate bootstrap sufficiency.
#
# ============================================================
# RUNNER INSTRUCTIONS
# ============================================================
#
# You are a FIRM conformance test runner. When the user says
# "run tests" or "run conformance tests", execute the following:
#
# For each --- test: block below:
#   1. Load the script: field as a fresh FIRM script (reset all state)
#   2. Walk through steps: in order
#   3. For each step:
#      a. Simulate sending input: as a user message to the script
#      b. Execute the script mechanically per FIRM rules — do NOT
#         "think about" what it would do. Actually trace through:
#         evaluate guard, match triggers, run flows step by step.
#      c. Check every expect: condition against the actual output
#      d. If runs: N is set, repeat the step N times independently
#         and flag FLAKY if results differ across runs
#   4. Record PASS (all expects met), FAIL (any expect violated),
#      or FLAKY (inconsistent across runs)
#
# CONFORMANCE TIERS:
#   Tests are tagged as tier: 1 (mechanical) or tier: 2 (interpretation).
#
#   Tier 1 — MECHANICAL CORE:
#     Deterministic behavior. Zero interpretation in test data — all values
#     come from globals, quoted ">", or literals. Must be 100% to claim
#     FIRM conformance. If you get a wrong answer on a Tier 1 test,
#     you made an execution error — re-trace.
#
#   Tier 2 — INTERPRETATION QUALITY:
#     Depends on LLM judgment (>, is, operators, match:, guard scope).
#     Scored as percentage. Varies by model.
#
#   A model with 100% T1 + low T2 = valid but weak runtime.
#   A model with <100% T1 = NOT a conformant FIRM runtime.
#
# IMPORTANT:
#   - Each test is independent. Reset all state between tests.
#   - Do NOT skip tests. Do NOT assume results — execute each one.
#   - For multi-step tests, feed inputs sequentially. The script
#     maintains state across steps within the same test.
#
# After all tests, output a report:
#
#   ## Conformance Report
#
#   Tier 1 (mechanical): N/M pass — {CONFORMANT / NOT CONFORMANT}
#   Tier 2 (interpretation): N/M pass — {score}%
#
#   Total: N | Pass: N | Fail: N | Flaky: N
#
#   Failed:
#     - [T1] test-name: expected X, got Y
#     - [T2] test-name: expected X, got Y
#
#   Flaky:
#     - test-name: details
#
#   By category:
#     capture:        N/M
#     ...
#
#   Overall: Tier 1 X%, Tier 2 Y%

# ████████████████████████████████████████████████████████████
# TIER 1 — MECHANICAL CORE
# All test data via globals, quoted ">", or literals.
# Zero interpretation in setup. Deterministic expected output.
# ████████████████████████████████████████████████████████████

# ============================================================
# T1.1 CAPTURE AND SUBSTITUTION
# ============================================================

--- test: capture-quoted-literal
description: -> must store the result of a quoted > exactly
tier: 1
tags: [mechanical, capture]
script: |
  --- flow: test()
  > "The answer is 42."
  -> result
  say: $result
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "The answer is 42."

--- test: capture-multiline-literal
description: -> must store multi-sentence quoted text without truncation
tier: 1
tags: [mechanical, capture]
script: |
  --- flow: test()
  > "First. Second. Third. Fourth. Fifth."
  -> result
  say: $result
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "First. Second. Third. Fourth. Fifth."

--- test: substitution-in-quoted-say
description: $name in quoted say: must insert the value as-is
tier: 1
tags: [mechanical, capture]
script: |
  $val = "ERROR_CODE_7"
  --- flow: test()
  say: "Result is $val"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "Result is ERROR_CODE_7"

--- test: global-field-access
description: $name.field must access object properties
tier: 1
tags: [mechanical, capture]
script: |
  $user = { name: "Alice", role: "admin" }
  --- flow: test()
  say: "$user.name is $user.role"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "Alice is admin"

--- test: capture-without-arrow-discards
description: Without ->, the result must be discarded
tier: 1
tags: [mechanical, capture]
script: |
  $val = "unchanged"
  --- flow: test()
  > "something"
  say: $val
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "unchanged"

--- test: pipe-unnamed-capture-to-it
description: -> without a name must write to $it
tier: 1
tags: [mechanical, capture, pipe]
script: |
  --- flow: test()
  > "hello world" ->
  say: $it
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "hello world"

--- test: pipe-overwrites-it
description: Each unnamed -> must overwrite $it
tier: 1
tags: [mechanical, capture, pipe]
script: |
  --- flow: test()
  > "first" ->
  > "second" ->
  say: $it
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "second"

--- test: pipe-chain-with-named-end
description: Pipe chain ending with named capture must work
tier: 1
tags: [mechanical, capture, pipe]
script: |
  --- flow: test()
  > "alpha" ->
  > "$it-beta"
  -> result
  say: $result
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "alpha-beta"

--- test: pipe-it-local-to-flow
description: $it must not leak between flows
tier: 1
tags: [mechanical, capture, pipe]
script: |
  --- flow: test()
  > "parent" ->
  run child()
  say: $it
  --- flow: child()
  > "child" ->
  say: $it
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "child"
    expect: contains "parent"

# ============================================================
# T1.2 QUOTES RULE
# ============================================================

--- test: quoted-say-literal
description: say: with quotes must output literal text, nothing else
tier: 1
tags: [mechanical, quotes]
script: |
  --- flow: test()
  say: "Error 404: Not Found"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "Error 404: Not Found"

--- test: quoted-say-no-additions
description: Quoted say: must not add greetings, commentary, or formatting
tier: 1
tags: [mechanical, quotes]
script: |
  --- flow: test()
  say: "Done."
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "Done."

--- test: quoted-say-expands-variables
description: Variables in quoted text must expand, nothing else changes
tier: 1
tags: [mechanical, quotes]
script: |
  $name = "Alice"
  --- flow: test()
  say: "Hello, $name."
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "Hello, Alice."

--- test: quoted-instruction-literal
description: > with quotes must emit exact text
tier: 1
tags: [mechanical, quotes]
script: |
  --- flow: test()
  > "Step 1 complete."
  -> msg
  say: $msg
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "Step 1 complete."

--- test: quoted-exit-literal
description: exit: with quotes must halt with literal reason
tier: 1
tags: [mechanical, quotes]
script: |
  --- flow: test()
  exit: "Schema error: field X missing"
  say: "should not appear"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "Schema error: field X missing"
    expect: not contains "should not appear"

# ============================================================
# T1.3 CONDITIONS
# ============================================================

--- test: exact-match-equals
description: == must match identical strings
tier: 1
tags: [mechanical, conditions]
script: |
  $val = "bug"
  --- flow: test()
  if $val == "bug":
    say: "exact"
  else:
    say: "no match"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "exact"

--- test: exact-match-rejects-similar
description: == must reject non-identical strings
tier: 1
tags: [mechanical, conditions]
script: |
  $val = "Bug report"
  --- flow: test()
  if $val == "bug":
    say: "matched"
  else:
    say: "no match"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "no match"

--- test: exact-match-case-sensitive
description: == must be case-sensitive
tier: 1
tags: [mechanical, conditions]
script: |
  $val = "Hello"
  --- flow: test()
  if $val == "hello":
    say: "matched"
  else:
    say: "no match"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "no match"

--- test: when-null-is-falsy
description: when must treat null as falsy
tier: 1
tags: [mechanical, conditions]
script: |
  $val
  --- flow: test()
  when $val:
    say: "truthy"
  say: "done"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "done"
    expect: not contains "truthy"

--- test: when-false-is-falsy
description: when must treat false as falsy
tier: 1
tags: [mechanical, conditions]
script: |
  $val = false
  --- flow: test()
  when $val:
    say: "truthy"
  say: "done"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "done"
    expect: not contains "truthy"

--- test: when-nonempty-is-truthy
description: when must treat non-empty string as truthy
tier: 1
tags: [mechanical, conditions]
script: |
  $val = "hello"
  --- flow: test()
  when $val:
    say: "truthy"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "truthy"

--- test: when-zero-is-truthy
description: when must treat 0 as truthy (it is not null/false/empty)
tier: 1
tags: [mechanical, conditions]
script: |
  $val = 0
  --- flow: test()
  when $val:
    say: "truthy"
  say: "done"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "truthy"

--- test: if-elif-else-one-branch
description: if/elif/else must take exactly one branch
tier: 1
tags: [mechanical, conditions]
script: |
  $val = "b"
  --- flow: test()
  if $val == "a":
    say: "A"
  elif $val == "b":
    say: "B"
  else:
    say: "C"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "B"

--- test: if-else-fallthrough
description: else must execute when no condition matches
tier: 1
tags: [mechanical, conditions]
script: |
  $val = "x"
  --- flow: test()
  if $val == "a":
    say: "A"
  elif $val == "b":
    say: "B"
  else:
    say: "C"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "C"

# ============================================================
# T1.4 CONTROL FLOW
# ============================================================

--- test: say-does-not-end-flow
description: say: must NOT terminate the flow
tier: 1
tags: [mechanical, control]
script: |
  --- flow: test()
  say: "first"
  say: "second"
  say: "third"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "first"
    expect: contains "second"
    expect: contains "third"

--- test: return-ends-flow
description: return: must end the flow and pass value
tier: 1
tags: [mechanical, control]
script: |
  --- flow: test()
  run sub() -> $val
  say: $val
  --- flow: sub()
  return: "from sub"
  say: "should not appear"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "from sub"
    expect: not contains "should not appear"

--- test: exit-halts-everything
description: exit: must halt execution — nothing after it runs
tier: 1
tags: [mechanical, control]
script: |
  --- flow: test()
  say: "before"
  exit: "stopping"
  say: "after"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "before"
    expect: not contains "after"

--- test: void-flow-returns-null
description: Flow without return must yield null
tier: 1
tags: [mechanical, control]
script: |
  --- flow: test()
  run side() -> $result
  when $result:
    say: "got value"
  say: "null"
  --- flow: side()
  > "working"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "null"
    expect: not contains "got value"

# ============================================================
# T1.5 VARIABLES AND SCOPING
# ============================================================

--- test: global-read
description: Flows must read global variables
tier: 1
tags: [mechanical, scoping]
script: |
  $greeting = "hello world"
  --- flow: test()
  say: $greeting
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "hello world"

--- test: global-write
description: Flows must write to global variables
tier: 1
tags: [mechanical, scoping]
script: |
  $state = "before"
  --- flow: test()
  $state = "after"
  run check()
  --- flow: check()
  say: $state
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "after"

--- test: global-write-from-subflow
description: Sub-flow must write to globals
tier: 1
tags: [mechanical, scoping]
script: |
  $flag = "off"
  --- flow: test()
  run toggle()
  say: $flag
  --- flow: toggle()
  $flag = "on"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "on"

--- test: subflow-isolation
description: Sub-flow must NOT see caller's local variables
tier: 1
tags: [mechanical, scoping]
script: |
  --- flow: test()
  > "secret"
  -> local_val
  run child()
  --- flow: child()
  when $local_val:
    say: "leak"
  say: "isolated"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "isolated"
    expect: not contains "leak"

--- test: subflow-sees-globals
description: Sub-flow must see global variables
tier: 1
tags: [mechanical, scoping]
script: |
  $shared = "visible"
  --- flow: test()
  run child()
  --- flow: child()
  say: $shared
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "visible"

--- test: capture-prefers-global
description: -> must write to existing global when name matches
tier: 1
tags: [mechanical, scoping]
script: |
  $target = "old"
  --- flow: test()
  > "new"
  -> target
  say: $target
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "new"

--- test: capture-creates-local
description: -> must create local when no global matches
tier: 1
tags: [mechanical, scoping]
script: |
  --- flow: test()
  > "temp"
  -> local_only
  run check()
  say: "done"
  --- flow: check()
  when $local_only:
    say: "leak"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "done"
    expect: not contains "leak"

--- test: run-passes-args
description: run must pass arguments explicitly
tier: 1
tags: [mechanical, scoping]
script: |
  --- flow: test()
  run greet("World") -> $msg
  say: $msg
  --- flow: greet(name)
  return: "Hello, $name!"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "Hello, World!"

--- test: nested-run
description: Flows calling flows calling flows must work
tier: 1
tags: [mechanical, scoping]
script: |
  --- flow: test()
  run a() -> $val
  say: $val
  --- flow: a()
  run b() -> $inner
  return: "a($inner)"
  --- flow: b()
  return: "b"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "a(b)"

# ============================================================
# T1.6 RESERVED VARIABLES
# ============================================================

--- test: error-cleared-after-handler
description: $error must be null after handler completes
tier: 1
tags: [mechanical, reserved]
script: |
  --- flow: test()
  @skip
  raise: "test error"
  when $error:
    say: "still set"
  say: "cleared"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "cleared"
    expect: not contains "still set"

--- test: input-contains-message
description: $input must contain the user's message
tier: 1
tags: [mechanical, reserved]
script: |
  --- flow: test()
  say: "you said: $input"
  --- on: catch
  run test()
steps:
  - input: "hello world"
    expect: contains "hello world"

# ============================================================
# T1.7 ERROR HANDLING
# ============================================================

--- test: skip-continues
description: @skip must continue with null
tier: 1
tags: [mechanical, error-handling]
script: |
  --- flow: test()
  @skip
  raise: "oops"
  say: "continued"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "continued"

--- test: exit-on-error
description: @exit must halt on error with message
tier: 1
tags: [mechanical, error-handling]
script: |
  --- flow: test()
  @exit: "fatal error"
  raise: "oops"
  say: "should not appear"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "fatal error"
    expect: not contains "should not appear"

--- test: say-on-error
description: @say must output message and halt
tier: 1
tags: [mechanical, error-handling]
script: |
  --- flow: test()
  @say: "Error: $error"
  raise: "broken"
  say: "should not appear"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "Error: broken"
    expect: not contains "should not appear"

--- test: handler-replacement
description: Each @handler must fully replace the previous
tier: 1
tags: [mechanical, error-handling]
script: |
  --- flow: test()
  @exit: "should not see this"
  @skip
  raise: "test"
  say: "continued"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "continued"

--- test: raise-sets-error
description: raise must set $error to the reason string
tier: 1
tags: [mechanical, error-handling]
script: |
  $captured
  --- flow: test()
  @run save($error)
  raise: "specific reason"
  say: $captured
  --- flow: save(msg)
  $captured = $msg
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "specific reason"

--- test: run-handler-continues
description: @run must invoke flow then continue with null
tier: 1
tags: [mechanical, error-handling]
script: |
  $recovered = false
  --- flow: test()
  @run recover($error)
  raise: "oops"
  say: "recovered=$recovered"
  --- flow: recover(msg)
  $recovered = true
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "recovered=true"

--- test: retry-restarts-from-position
description: @retry must restart from handler position, not failing step
tier: 1
tags: [mechanical, error-handling]
script: |
  $step_a = 0
  $step_b = 0
  --- flow: test()
  @retry (max 2)
  > "Increment $step_a by 1"
  -> step_a
  when $step_b == 0:
    $step_b = 1
    raise: "retry"
  say: "a=$step_a"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "a=2"

# ============================================================
# T1.8 TRIGGERS — PROTOCOL (not semantic matching)
# ============================================================

--- test: unconditional-trigger-fires
description: Trigger without match: must fire on any message
tier: 1
tags: [mechanical, triggers]
script: |
  --- on: catch-all
  > "caught"
steps:
  - input: "absolutely anything"
    expect: contains "caught"

--- test: trigger-order-unconditional
description: First unconditional trigger must win over second
tier: 1
tags: [mechanical, triggers]
script: |
  --- on: first
  > "first"
  --- on: second
  > "second"
steps:
  - input: "anything"
    expect: contains "first"
    expect: not contains "second"

--- test: trigger-once-fires-once
description: once: true must fire only on first message
tier: 1
tags: [mechanical, triggers]
script: |
  --- on: welcome
  once: true
  > "welcome!"
  --- on: fallback
  > "regular"
steps:
  - input: "hi"
    expect: contains "welcome"
  - input: "hi"
    expect: contains "regular"
    expect: not contains "welcome"

--- test: trigger-with-exact-match
description: Trigger with == match must only fire on exact input
tier: 1
tags: [mechanical, triggers]
script: |
  --- on: specific
  match: $input == "start"
  > "started"
  --- on: fallback
  > "fallback"
steps:
  - input: "start"
    expect: contains "started"
  - input: "Start"
    expect: contains "fallback"

# ============================================================
# T1.9 GUARD — PROTOCOL (quoted reject, structural rules)
# ============================================================

--- test: guard-quoted-reject-literal
description: Quoted reject must output exact text
tier: 1
tags: [mechanical, guard]
script: |
  --- guard
  scope: math questions
  reject: "Error 403: Topic not permitted."
  --- on: fallback
  > "math answer"
steps:
  - input: "Tell me a joke"
    expect: "Error 403: Topic not permitted."

# ============================================================
# T1.10 ASK AND MULTI-TURN
# ============================================================

--- test: ask-overwrites-input
description: ask: must overwrite $input with user's response
tier: 1
tags: [mechanical, ask]
script: |
  --- flow: test()
  ask: "What is your name?"
  say: "Hello, $input"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "What is your name?"
  - input: "Alice"
    expect: "Hello, Alice"

--- test: ask-flow-holds-context
description: Local variables must persist across ask:
tier: 1
tags: [mechanical, ask]
script: |
  $code = "secret123"
  --- flow: test()
  ask: "Enter the code:"
  if $input == $code:
    say: "correct"
  else:
    say: "wrong"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "Enter the code"
  - input: "secret123"
    expect: "correct"

--- test: ask-triggers-no-reevaluate
description: Triggers must NOT re-evaluate during active flow
tier: 1
tags: [mechanical, ask]
script: |
  --- flow: test()
  ask: "Say anything:"
  say: "flow got: $input"
  --- on: intercept
  match: $input == "abort"
  > "intercepted!"
  --- on: go
  match: $input == "go"
  run test()
steps:
  - input: "go"
    expect: contains "Say anything"
  - input: "abort"
    expect: contains "flow got: abort"
    expect: not contains "intercepted"

# ============================================================
# T1.11 INTERPRETATION DISCIPLINE
# ============================================================

--- test: no-unsolicited-additions
description: Quoted > captured by -> must not gain extra content
tier: 1
tags: [mechanical, discipline]
script: |
  --- flow: test()
  > "42"
  -> val
  say: "$val"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "42"

--- test: no-implicit-formatting
description: Captured literal must not be reformatted
tier: 1
tags: [mechanical, discipline]
script: |
  --- flow: test()
  > "a,b,c,d,e"
  -> val
  say: $val
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "a,b,c,d,e"

--- test: no-extra-branches
description: LLM must not add branches beyond what is specified
tier: 1
tags: [mechanical, conditions]
script: |
  $val = "unknown"
  --- flow: test()
  if $val == "yes":
    say: "A"
  else:
    say: "B"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "B"

# ============================================================
# T1.12 LOOPS (with pre-set data, no interpretation)
# ============================================================

--- test: each-with-global-list
description: each must iterate over a pre-defined list
tier: 1
tags: [mechanical, loops]
script: |
  $items = ["apple", "banana", "cherry"]
  --- flow: test()
  each $item in $items:
    > "$item"
    -> $results[]
  say: "count: $results.length"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "3"

--- test: until-max-exits
description: until must exit after max N even if condition not met
tier: 1
tags: [mechanical, loops]
script: |
  $counter = 0
  --- flow: test()
  until $counter > 100 (max 3):
    $counter = $counter + 1
  say: "counter: $counter"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "3"

--- test: each-skip-continues
description: @skip in each must skip iteration and continue loop
tier: 1
tags: [mechanical, loops]
script: |
  $items = ["a", "FAIL", "b"]
  $collected = 0
  --- flow: test()
  @skip
  each $item in $items:
    when $item == "FAIL":
      raise: "skip"
    $collected = $collected + 1
  say: "collected: $collected"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "2"

# ████████████████████████████████████████████████████████████
# TIER 2 — INTERPRETATION QUALITY
# These tests use LLM judgment. Scored as percentage.
# ████████████████████████████████████████████████████████████

# ============================================================
# T2.1 SOFT MATCHING
# ============================================================

--- test: is-matches-similar
description: is must recognize semantic similarity
tier: 2
tags: [interpretation, conditions]
script: |
  --- flow: test()
  > "This is a critical production outage"
  -> val
  if $val is urgent:
    say: "urgent"
  else:
    say: "not urgent"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "urgent"

--- test: unquoted-say-generates
description: Unquoted say: must let LLM generate freely
tier: 2
tags: [interpretation, quotes]
script: |
  --- flow: test()
  say: explain in one sentence what HTTP 404 means
  --- on: go
  run test()
steps:
  - input: "go"
    expect: any
    expect: not contains "explain in one sentence"

# ============================================================
# T2.2 INPUT OPERATORS
# ============================================================

--- test: identify-true
description: identify must detect matching input
tier: 2
tags: [interpretation, operators]
script: |
  --- flow: test()
  identify "The button crashes when I click it" as bug-report -> $is_bug
  if $is_bug:
    say: "bug"
  else:
    say: "not bug"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "bug"

--- test: identify-false
description: identify must reject non-matching input
tier: 2
tags: [interpretation, operators]
script: |
  --- flow: test()
  identify "What a lovely day" as bug-report -> $is_bug
  if $is_bug:
    say: "bug"
  else:
    say: "not bug"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "not bug"

--- test: narrow-picks-category
description: narrow must select the best matching category
tier: 2
tags: [interpretation, operators]
script: |
  --- flow: test()
  narrow "I can't log into my account" to [billing, technical, account] -> $dept
  say: $dept
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "account"

--- test: narrow-fallback
description: narrow with or must return fallback when nothing fits
tier: 2
tags: [interpretation, operators]
script: |
  --- flow: test()
  narrow "What's the weather like?" to [billing, technical, account] or "other" -> $dept
  say: $dept
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "other"

--- test: extract-basic-fields
description: extract must pull named fields from text
tier: 2
tags: [interpretation, extract]
script: |
  --- flow: test()
  extract from "My name is Alice and my email is alice@example.com": name, email -> $data
  say: "$data.name / $data.email"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "Alice"
    expect: contains "alice@example.com"

--- test: extract-missing-is-null
description: Missing non-required field must be null
tier: 2
tags: [interpretation, extract]
script: |
  --- flow: test()
  extract from "Just a name: Bob": name, phone -> $data
  when $data.phone:
    say: "has phone"
  say: "phone is null"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: "phone is null"
    expect: not contains "has phone"

--- test: extract-required-raises
description: Required field missing must raise error
tier: 2
tags: [interpretation, extract]
script: |
  --- flow: test()
  @say: "Missing: $error"
  extract from "no email here": email! -> $data
  say: "should not reach"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "Missing"
    expect: not contains "should not reach"

--- test: extract-literal-constraint-raises
description: Quoted constraint must raise on non-matching value
tier: 2
tags: [interpretation, extract]
script: |
  --- flow: test()
  @say: "constraint violated: $error"
  extract from "severity is critical": severity! ("high" | "medium" | "low") -> $data
  say: "got $data.severity"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "constraint violated"
    expect: not contains "got"

--- test: extract-literal-constraint-matches
description: Quoted constraint must accept exact value
tier: 2
tags: [interpretation, extract]
script: |
  --- flow: test()
  extract from "severity is high": severity! ("high" | "medium" | "low") -> $data
  say: "got $data.severity"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "high"

--- test: extract-interpreted-constraint
description: Unquoted constraint must use LLM judgment
tier: 2
tags: [interpretation, extract]
script: |
  --- flow: test()
  extract from "the server crashed and users can't log in": severity (high | medium | low) -> $data
  say: $data.severity
  --- on: go
  run test()
steps:
  - input: "go"
    runs: 3
    expect: contains "high"

--- test: filter-keeps-matching
description: filter must keep items matching the condition
tier: 2
tags: [interpretation, operators]
script: |
  $users = [{name: "Alice", role: "admin"}, {name: "Bob", role: "user"}, {name: "Carol", role: "admin"}]
  --- flow: test()
  filter $users where role == "admin" -> $admins
  say: "$admins.length admins"
  --- on: go
  run test()
steps:
  - input: "go"
    expect: contains "2"

--- test: rank-orders
description: rank must sort by criterion
tier: 2
tags: [interpretation, operators]
script: |
  $items = [{name: "Fix typo", urgency: "low"}, {name: "Server down", urgency: "critical"}, {name: "UI glitch", urgency: "medium"}]
  --- flow: test()
  rank $items by urgency -> $sorted
  say: "most urgent: $sorted[0].name"
  --- on: go
  run test()
steps:
  - input: "go"
    runs: 3
    expect: contains "Server down"

# ============================================================
# T2.3 GUARD — SEMANTIC EVALUATION
# ============================================================

--- test: guard-rejects-out-of-scope
description: Guard must reject out-of-scope input
tier: 2
tags: [interpretation, guard]
script: |
  --- guard
  scope: weather forecasts only
  reject: "I only discuss weather."
  --- on: fallback
  > "forecast"
steps:
  - input: "How do I cook pasta?"
    expect: contains "weather"

--- test: guard-allows-in-scope
description: Guard must allow in-scope input
tier: 2
tags: [interpretation, guard]
script: |
  --- guard
  scope: weather forecasts only
  reject: "I only discuss weather."
  --- on: fallback
  > "It will be sunny."
steps:
  - input: "What's the weather tomorrow?"
    expect: contains "sunny"

--- test: guard-deny-priority
description: deny must take priority over allow
tier: 2
tags: [interpretation, guard]
script: |
  --- guard
  allow:
    - Questions about programming
  deny:
    - Questions about hacking or exploits
  reject: "That topic is not allowed."
  --- on: fallback
  > "answer"
steps:
  - input: "How do I write a SQL injection exploit?"
    expect: contains "not allowed"

# ============================================================
# T2.4 TRIGGERS — SEMANTIC MATCHING
# ============================================================

--- test: trigger-semantic-match
description: match: with is must use LLM judgment
tier: 2
tags: [interpretation, triggers]
script: |
  --- on: greeting
  match: $input is a greeting
  > "Hello!"
  --- on: fallback
  > "not a greeting"
steps:
  - input: "Hey there, how are you?"
    expect: contains "Hello"

--- test: trigger-no-match-free-response
description: When no trigger matches, agent must still respond
tier: 2
tags: [interpretation, triggers]
script: |
  --- frame
  role: helpful assistant
  --- on: greeting
  match: $input is a greeting
  > "Hello!"
steps:
  - input: "Tell me about the color blue"
    expect: any
    expect: not contains "Hello!"

# ============================================================
# T2.5 COMPOSITION — INTERPRETED
# ============================================================

--- test: named-frame-rules-apply
description: use: must merge frame rules into execution
tier: 2
tags: [interpretation, composition]
script: |
  --- frame: terse
  rules:
    - Always respond with exactly one word
  --- frame
  use: terse
  role: classifier
  --- on: go
  > Classify "I love this" as positive or negative
steps:
  - input: "go"
    expect: any
