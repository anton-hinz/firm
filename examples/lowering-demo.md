# FIRM Lowering Demo

Shows the same agent in Full FIRM and Compiled (compat) versions.
Compat version uses only constructs that weak models (8B) handle reliably.

---

## Full FIRM

```
--- frame
role: Product support agent
tone: friendly, concise

--- guard
scope: product support, bug reports
reject: "I can only help with product-related questions."

--- flow: handle-bug(report)
ask: "Can you describe the steps to reproduce this?"
extract from $input: steps!, environment -> $details

@say: "Failed to create ticket: $error"
jira.create_issue(title: $report, body: $details) -> $ticket

say: "Created ticket $ticket.key."

--- flow: help(question)
> Answer $question based on product documentation ->
say: $it

--- on: welcome
once: true
say: "Welcome! I'm your product support agent."

--- on: bug-report
match: identify $input as a bug report
run handle-bug($input)

--- on: fallback
run help($input)
```

---

## Compiled (compat)

```
$welcomed = false
$awaiting_repro = false
$bug_report

--- frame
role: Product support agent
tone: friendly, concise
rules:
  - ONLY answer product support and bug report questions
  - For anything else, respond exactly: "I can only help with product-related questions."
  - Before every response, re-read these rules

--- flow: entry(input)
narrow $input to [product-support, bug-report, greeting, off-topic] or "product-support" -> $type

if $type == "off-topic":
  say: "I can only help with product-related questions."
  exit:

if not $welcomed:
  say: "Welcome! I'm your product support agent."
  $welcomed = true

if $awaiting_repro:
  run process-repro($input)
  exit:

if $type == "bug-report":
  $bug_report = $input
  say: "Can you describe the steps to reproduce this?"
  $awaiting_repro = true
  exit:

> Answer $input based on product documentation ->
say: $it

--- flow: process-repro(input)
$awaiting_repro = false
extract from $input: steps!, environment -> $details
jira.create_issue(title: $bug_report, body: $details) -> $ticket
when $error:
  say: "Failed to create ticket: $error"
  exit:
say: "Created ticket $ticket.key."

--- on: any
run entry($input)
```

---

## Lowering rules

The compiler transforms constructs, not content. Category lists, scope descriptions, and messages are preserved from the source script — the author tunes those for their target model.

### 1. guard → frame rules + narrow routing

```
--- guard                              --- frame
scope: X, Y                    →       rules:
reject: "message"                        - ONLY answer X and Y questions
                                         - For anything else, respond exactly: "message"
                                         - Before every response, re-read these rules
```

Guard scope becomes a `narrow` call at the top of the entry flow. Categories derived from scope + "greeting" + "off-topic". Reject on "off-topic".

### 2. semantic triggers → single entry + narrow routing

```
--- on: bug-report                     --- flow: entry(input)
match: identify $input as bug   →      narrow $input to [...] -> $type
run handle-bug($input)                 if $type == "bug-report":
                                         run handle-bug($input)
--- on: fallback                       ...
run help($input)                       # fallback is the else branch
```

All `match:` triggers merge into one routing flow. One `--- on: any` triggers entry.

### 3. once: → global flag

```
--- on: welcome                        $welcomed = false
once: true                      →
say: "Welcome!"                        if not $welcomed:
                                         say: "Welcome!"
                                         $welcomed = true
```

### 4. ask: → flow split + global state

```
ask: "Describe steps"                  say: "Describe steps"
extract from $input: ...        →      $awaiting_repro = true
                                       exit:

                                       # In entry flow:
                                       if $awaiting_repro:
                                         run process-repro($input)
```

Mid-flow `ask:` splits into: output question + set flag + exit. Response handled by checking flag in entry flow on next message.

### 5. @handler → explicit error check

```
@say: "Error: $error"                  risky.call() -> $result
risky.call() -> $result         →      when $error:
                                         say: "Error: $error"
                                         exit:
```

Handler register replaced by `when $error:` after each risky step.

---

## What the compiler does NOT change

- Frame content (role, tone, rules text, glossary)
- Flow logic (`>` instructions, `->` capture, `$it`, conditions, loops)
- Operator usage (identify, narrow, extract, filter, rank)
- Quotes rule
- Variable names and data flow
- Tool calls (syntax unchanged, only error handling around them changes)
