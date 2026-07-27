---
name: "module"
title: "Facets Module Generator (fine-tuned LLM)"
description: "Generate Facets IaC modules using the team's fine-tuned model as the author of every file. Use when the user runs /module, OR asks to write/draft/generate/sketch a Facets module, facets.yaml, or module skeleton (any intent/flavor/cloud). Not for generic non-Facets Terraform."
triggers: ["module", "facets.yaml", "facets module"]
category: "development"
tags: ["iac", "terraform", "facets-yaml", "module-development", "llm"]
icon: "🧱"
version: "2.2"
---

# Facets Module Generator

The fine-tuned model (Qwen3.6-35B `qwen36-facets-nothink`, RunPod queue-based
serverless) **writes every module file and every revision**. You (Claude) are
the orchestrator: you plan, feed it context in its trained dialects, review
its output against repo ground truth, and send it fix requests. 

**Authorship rule (non-negotiable):** you never author or hand-edit module
file content yourself unless the user explicitly asks you to, or explicitly
approves after the model has exhausted its fix rounds on a file. Your edits
are limited to review, validation, and orchestration.

## Configuration

Env vars (values from the platform team). `jq` and `curl` required
(preflight: `command -v jq curl`).

- `FACETS_LLM_ENDPOINT` — RunPod endpoint id (current: `2xd61lauejq03o`)
- `FACETS_LLM_KEY` — inference API key

If either is missing, stop and show this snippet (restart session after adding):

```bash
# add to ~/.zshrc — values from the platform team (#platform-eng)
export FACETS_LLM_ENDPOINT="2xd61lauejq03o"
export FACETS_LLM_KEY="<team-inference-key>"
```

**Cost note:** a full module is ~6 endpoint calls on the happy path and up to
~18 worst-case (every cap exhausted). Each file has a TOTAL budget of 2 fix
calls across ALL steps (generation review + raptor failures combined); the
facets.yaml reconcile in step 3 gets exactly 1. When any budget is exhausted,
ask the user — never loop the endpoint.

## The endpoint call (reusable)

For every call: write the prompt to a temp file with the Write tool (never
interpolate user or file text into shell), then:

```bash
set -eu
PROMPT_FILE="$1"   # e.g. /tmp/facets-call.txt
MAXTOK="${2:-4096}"
REQ=$(jq -n --rawfile prompt "$PROMPT_FILE" --argjson mt "$MAXTOK" '{
  input: {
    model: "qwen36-facets-nothink",
    temperature: 0.25, top_p: 0.95, max_tokens: $mt,
    chat_template_kwargs: {enable_thinking: false},
    messages: [
      {role: "system", content: "You help developers write and review Facets IaC modules. Follow the Facets module conventions precisely."},
      {role: "user", content: $prompt}
    ]
  }
}')
R=$(printf '%s' "$REQ" | curl -sS --max-time 120 -X POST \
  "https://api.runpod.ai/v2/$FACETS_LLM_ENDPOINT/runsync" \
  -H "Authorization: Bearer $FACETS_LLM_KEY" -H "Content-Type: application/json" -d @-)
JID=$(printf '%s' "$R" | jq -r '.id // empty')
STATUS=$(printf '%s' "$R" | jq -r '.status // "UNKNOWN"')
DEADLINE=$(( $(date +%s) + 900 ))
while [ "$STATUS" = "IN_QUEUE" ] || [ "$STATUS" = "IN_PROGRESS" ]; do
  [ $(date +%s) -gt $DEADLINE ] && { echo "TIMED OUT (job $JID)"; exit 1; }
  sleep 15
  R=$(curl -sS --max-time 60 -H "Authorization: Bearer $FACETS_LLM_KEY" \
    "https://api.runpod.ai/v2/$FACETS_LLM_ENDPOINT/status/$JID")
  STATUS=$(printf '%s' "$R" | jq -r '.status // "UNKNOWN"')
done
[ "$STATUS" = "COMPLETED" ] || { echo "JOB $STATUS:"; printf '%s' "$R" | jq -c '.error // .' | head -c 400; exit 1; }
printf '%s' "$R" | jq -r '.output.choices[0].message.content // ""'
```

Save as `/tmp/facets-llm-call.sh` once, then `bash /tmp/facets-llm-call.sh <prompt-file> [max_tokens]`.
First call after idle can wait minutes in IN_QUEUE (GPU cold start) — tell the user.

## Trained dialects (use these EXACTLY — the model degrades on any other shape)

- **facets.yaml (scratch):** `I need a new Facets module: intent \`<intent>\`, flavor \`<flavor>\`, version 1.0, targeting <clouds>. It should: <requirements>. Start with the facets.yaml.` (max_tokens 8192)
- **variables.tf:** `Given this facets.yaml, write the module's variables.tf:` + one ```yaml fence
- **main.tf:** `Given the facets.yaml and variables.tf below for the Facets module with intent \`<intent>\`, flavor \`<flavor>\`, version 1.0, implement main.tf.` + facets.yaml fence + variables.tf fence
- **locals.tf:** `Given this facets.yaml, write the locals.tf that defines output_attributes and output_interfaces.` + one ```yaml fence
- **outputs.tf:** `Write outputs.tf for the Facets module with intent \`<intent>\`, flavor \`<flavor>\`, version 1.0, given its locals.tf:` + one ```hcl fence (the REAL generated locals.tf — never an empty or invented one)
- **Fix (any file):** `Fix the following issue in \`<file>\` of the \`<intent>/<flavor>/1.0\` module: <one clear issue list>.` + the current file in a fence (+ the facets.yaml fence if the fix depends on the spec)

Never attach more context than the pairing shown — extra pasted files
verifiably degrade this model. One file request per call.

## Procedure

### 0. Plan (before any endpoint call)

Confirm you are inside the Facets module repository (markers: `rules.md`,
`modules/`, `outputs/` at repo root). **Outside the repo:** degraded mode —
you may run only the facets.yaml call, mark every `@facets/` type
`# UNVERIFIED`, tell the user full generation requires the repo, and stop.

Inside the repo, read BEFORE calling (so endpoint calls run back-to-back):

1. **Type-to-schema map**: `grep -H '^name:' outputs/*/outputs.yaml` — the
   declared `@facets/...` name → schema path (directory names diverge from
   type names; never guess paths). **Sanity-check the map before trusting
   it**: it must be non-empty and must contain at least one type you can see
   used by an existing module (e.g. grep a module's facets.yaml for its input
   type and confirm the map resolves it). An empty or partial map means the
   glob missed the layout (nested dirs, `.yml`, quoted keys) — adapt the
   search; never run the STOP rule against a map you haven't sanity-checked.
2. Schemas of likely-relevant types; enumerate their `attributes.*` and
   `interfaces.*` paths with types (scalar/list/map) as a scratch checklist.
3. One **exemplar module** of similar intent: its top-level `*.tf` file set
   (some carry `variables_outputs.tf`/`providers.tf`) and where it defines
   `output_attributes`/`output_interfaces` (locals.tf vs outputs.tf).

Present the user a one-paragraph plan: intent, flavor, clouds, file set,
which file carries the wiring, and the input types you expect to use.

### 1. facets.yaml (endpoint, scratch dialect)

Call with the scratch dialect. **Degenerate check** (after stripping one
optional ```yaml fence): unusable if empty, a file tree, not parseable as a
YAML mapping, missing any of `intent`/`flavor`/`version`/`spec`, or visibly
truncated. Degenerate → ONE retry via the same call; still degenerate → ask
the user whether you may author it yourself.

**On acceptance, build the SHAPE TABLE** (this makes every later check
executable): one row per spec field — spec declaration → expected HCL type →
legal access operators. Translation rules: JSON-Schema `array` → `list(...)`
(operators: index `[n]`, `for`, `length`); `patternProperties`/
`additionalProperties` object → `map(...)` (operators: key lookup, `lookup()`,
`for k,v`, `keys()`); fixed-`properties` object → `object({...})` with the
EXACT field set (operators: attribute access only). Add one row per step-0
input attribute/interface path with its schema type. Every landing file is
checked against this table.

**Review** (you): every input/output `type:` must exist in the type map;
`sample.kind` = intent; `intentDetails` per RULE-021. Anything wrong → **fix
dialect** call(s), max 2, e.g. "replace input type `@facets/foo` with
`@facets/<real>`; set sample.kind to `<intent>`". If a needed attribute
exists on no registered type, STOP and tell the user (name input, path,
types checked).

### 2. Terraform files (endpoint, one call per file, exemplar order)

For each file in the exemplar's file set, call with that file's exact
dialect. **If the exemplar wires outputs in `outputs.tf` without a
`locals.tf`:** there is NO trained dialect for outputs.tf-from-facets.yaml
(training only ever paired outputs.tf with a locals.tf). Treat this as an
experimental off-dialect call with a budget of 1: attempt it once via the
fix dialect against the facets.yaml; if the result fails any check, do not
retry — ask the user whether you may author that one file yourself.

Write each ACCEPTED file immediately to a staging directory
(`mktemp -d` at step start) so whole-set checks can run as files accumulate;
step 3 moves the set into `modules/...` at the end.

**Per-file degenerate check:** the response must contain the expected
top-level HCL block (`variable`/`resource`/`data`/`locals`/`output`) with
actual assignments — commentary-only responses, empty blocks, or unclosed
braces are degenerate. Degenerate or review-failing → **fix dialect** with a
precise issue list (e.g. "the file contains only comments; produce the
actual locals block with output_attributes and output_interfaces
assignments").

**Fix-loop rules (apply to EVERY fix call in any step):**
- Every fix output re-enters the FULL landing review: degenerate check,
  checklist, shape table, semantic-values check. A fix that introduces a new
  failure consumes the file's budget like any other round.
- Budget: 2 fix calls TOTAL per file across steps 2–4 combined. Exhausted →
  present the remaining issue and ask the user whether you may author or
  hand-edit that file yourself. This escape applies to every path.
- If ANY fix changes an already-accepted upstream file (e.g. variables.tf
  retyped during the raptor loop), re-run the shape and coherence checks on
  every downstream file in the staging dir — this costs zero endpoint calls
  — and route any new failures through their remaining budgets.

**Per-file review checklist (you):** every `var.inputs.<n>.attributes.<p>` /
`.interfaces.<p>` dereference appears on the step-0 checklist (indexing only
beneath list/map paths); every `local.*` referenced is defined; every
`var.*` declared; wiring defined in exactly ONE file
(`grep -En '^\s*(output_attributes|output_interfaces)\s*=' <staging-dir>/*.tf`
must show exactly one assignment each, in the convention file; re-run it in
step 3 after the final write to `modules/...`).

**Type-shape consistency (hard requirement — checked on EVERY file as it
lands AND on every fix output):** diff the file's declarations and accesses
against the step-1 SHAPE TABLE. Failures include: variables.tf typing a
field differently than the table; any access using an operator not legal
for the table's type — `lookup()`/`keys()`/`for k,v`/`for_each` on a list,
`[0]`/splat on a map, `object({...})` field sets that don't match the spec's
exact fields; and any `var.inputs...` access whose operator doesn't match
the input attribute's schema type. Quote both sides in the fix call (inline
in the issue text — inline quotes are not fences and don't violate the
pairing rule), e.g. "variables.tf types `databases` as `map(object)` but
the facets.yaml spec declares an array — retype it as `list(object({...}))`."
Also reject: outputs.tf re-assigning `output_attributes`/`output_interfaces`
when another file already carries them; any symbol assigned to itself
(circular reference).

**Semantic-values check (hard requirement):** any credential, secret,
password, token, or connection-string field in `output_attributes`/
`output_interfaces` must trace to a resource attribute, a verified
`var.inputs...` path, or a secret reference — NEVER a literal string or an
unrelated field (a fix round once assigned an instance NAME as a credential;
shape checks cannot catch this class).

### 3. Assemble and reconcile

Write the model's accepted outputs to `modules/<intent>/<flavor>/1.0/`
verbatim (transcription is not authorship). If `iac.validated_files`
disagrees with the actual file set, route a fix-dialect call on facets.yaml
to correct it.

### 4. Validate

Verify the validation command against the installed raptor first
(`raptor --help`; the expected form is `raptor create iac-module -f
<module-path> --dry-run`, but confirm before the first run). Failures:
summarize each raptor error into ONE plain-language issue scoped to ONE file
(never paste raw multi-file/ANSI error dumps into the fix prompt) → fix-dialect
call on the offending file with raptor's error text as the issue (max 2
rounds per file), then re-run. Still failing → present the remaining errors
and ask the user whether you may fix directly. Report security-scan findings
in a table; never skip validation. Check the repo new-module checklist
(icon, project-type, catalog) and grep `rules.md` yourself for any
RULE-number claims.

## Known model limits

- Strongest at facets.yaml. Probed on real-module context (2026-07-27):
  variables.tf, main.tf, and the fix dialect verified good; locals.tf
  degenerated into commentary on the first try (recovered by one fix round in
  field testing). Degrades on synthetic or oversized context — hence the
  strict dialect pairings and the degenerate checks.
- Invents `@facets/` type names and attribute paths — hence the step-0
  checklist; never trust, always verify.
- No raptor CLI knowledge; hallucinates raptor commands confidently.
- Rambling commentary instead of code = known failure mode; catch it with the
  degenerate check and one precise fix round.
