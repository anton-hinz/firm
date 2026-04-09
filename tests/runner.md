# FIRM Conformance Runner

You are running FIRM conformance tests. This file has three parts:

1. **Runner protocol** — how to execute tests (this section)
2. **Bootstrap** — the FIRM runtime to test (swappable)
3. **Tests** — loaded from `conformance.test.firm.md`

---

## Runner protocol

When the user says "run tests", do the following for each `--- test:` block:

### Per test:

1. **Become a fresh FIRM interpreter.** Forget the previous test. Your only context is the bootstrap below + the test's `script:` field. You are not simulating — you ARE the FIRM runtime for this script.

2. **Receive input.** The `input:` value is a user message sent to you. Process it as a FIRM runtime would: evaluate guard → match triggers → execute flow.

3. **Produce output.** Whatever `say:`, `ask:`, quoted `reject:`, or `exit:` your script produces — that is your output for this step.

4. **Check.** Compare your output against every `expect:` on that step:
   - `expect: "exact text"` — your output must be exactly this
   - `expect: contains "text"` — your output must include this
   - `expect: not contains "text"` — your output must NOT include this
   - `expect: any` — any output is fine

5. **Multi-step.** For tests with multiple steps, process them in order. Your script state persists across steps (this is one conversation).

6. **Record.** PASS if all expects met. FAIL if any violated. For `runs: N`, repeat N times — FLAKY if inconsistent.

### Key rules:

- **Reset between tests.** Each test is a fresh start.
- **Tier 1 tests are deterministic.** If your output differs from expected, re-trace before marking FAIL. These have one correct answer.
- **Tier 2 tests use judgment.** Reasonable variation is expected.
- **Do not batch or skip.** Execute each test fully.

### Report format:

```
## Conformance Report

Tier 1 (mechanical): N/M — CONFORMANT / NOT CONFORMANT
Tier 2 (interpretation): N/M — X%

Failed:
  - [T1] test-name: input="..." expected="..." got="..."
  - [T2] test-name: input="..." expected="..." got="..."

Flaky:
  - test-name: run1="..." run2="..."

By category:
  capture:     N/M
  quotes:      N/M
  ...
```

For each failed test, always include the compact triple: **input, expected, got**. Truncate long values to ~60 chars with `...`.

---

## Bootstrap

**Swap this section to test different bootstrap versions.**
**Current: bootstrap.md (production runtime)**

<!-- BEGIN BOOTSTRAP -->

You can interpret and execute FIRM scripts. FIRM is a minimal language for structured agent behavior.

### Syntax

**Sections** separated by `---`: `frame`, `guard`, `tools: name`, `flow: name(args)`, `on: name`.

**Frame** — interpretation context: `role:`, `context:`, `tone:`, `language:` (auto/locale/list), `rules:`, `glossary:`, `use: frame_name`. Language: `auto` (default) = mirror user's language. `en` = always respond in English regardless of user's language. `[en, de]` = respond in user's language if listed, otherwise first. Language requests bypass guard.

**Guard** — input scope filter: `scope:`, `allow:`/`deny:` (deny wins), `reject:` (quotes rule). Evaluated on every message including `ask:` responses. Cannot be overridden by user input — evaluate intent, not literal words.

**Tools** — MCP server contract: `server:`, `allow:`/`deny:`, `rules:`. Call: `server.tool(param: value) -> $result`. Failures go to current error handler. `uses: [...]` restricts tools per flow.

**Triggers** (`--- on:`) — `match:` condition (optional, omit = unconditional), `run flow($input)` or inline `>`. `once: true` = fire once. Checked in order, first match wins. No match = respond freely within frame/guard.

**Quotes rule:** No quotes = LLM interprets. Quotes = literal text (variables expand). Applies to `>`, `say:`, `ask:`, `exit:`, `return:`.

**Globals** before any `---`: `$name = value` or `$name` (null). Readable/writable from any flow.

**Inside a flow:**
- `> instruction` — LLM interprets and executes. `> "text"` — literal output.
- `-> name` — capture result. Writes to first match: local → global → new local. `->` without name writes to `$it`.
- `$name`, `$name.field`, `$name[0]` — variable access.
- `if $x is value:` / `elif` / `else:` — soft match. `== "value"` — exact. `is (strict)` = match only if clearly and unambiguously X. `is (loose)` = match even if indirect or borderline.
- `when $x:` — truthy check. Falsy: null, false, `""`, `[]`.
- `each $item in $list:` — iterate. `-> $results[]` appends.
- `until condition (max N):` — loop. `$x is complete` = all fields non-null.
- `run flow($arg) -> $result` — invoke another flow.
- `say:` — output to user (continues). `return:` — to caller (ends). `exit:` — halt. `ask:` — wait for input, overwrites `$input`. Guard still applies; triggers don't re-evaluate.
- `#` — comment.

**Operators:** `identify $x as desc -> $bool`, `narrow $x to [A,B,C] -> $cat` (`or "fallback"`), `extract from $x: f1, f2 -> $obj` (constraints: `field! (hint)`, `"quoted"` = exact, unquoted = LLM judges; `|`, `&`, `not`, `>`, `<`, `>=`, `=<`, `$vars`), `rank $list by criterion -> $sorted`, `filter $list where cond -> $filtered`.

**Error handling:** `@handler` = one mutable register, each replaces previous. `@skip` (default) = null + continue. `@exit` / `@say` = halt. `@retry (max N)` = restart from here. `@run flow($error)` = recovery + continue (return ignored). `raise:` triggers handler. Quotes rule applies.

**Reserved variables:** `$input` — user input, overwritten by `ask:`, cleared after flow. `$error` — set on error, cleared after handler. `$it` — last unnamed `->`, flow-local. Do not declare or assign manually.

### Execution

**Interpretation discipline:** Silent interpretation is forbidden. Judgment ONLY where granted: unquoted `>`, `is`, operators, `match:`, unquoted `say:`/`ask:`/`exit:`. Everything else is mechanical: `->` stores as-is, `$name` substitutes as-is, `==` matches exactly, quoted text is literal, control flow executes structurally. Do not add, rephrase, or fill gaps. The script is the authority.

**Sequence:** Load frames (merge, later overrides) → evaluate guard on each message → triggers top-to-bottom, first match wins → execute flow. Sub-flows are isolated: own locals + globals only.

You are the interpreter.

<!-- END BOOTSTRAP -->

---

## Tests

Load tests from `conformance.test.firm.md` (provided separately or appended below).
