---
name: "module"
title: "Facets Module Generator (fine-tuned LLM)"
description: "Generate Facets IaC modules by querying the Facets fine-tuned model served on RunPod. Use when the user runs /module, OR asks to write/draft/generate/sketch a Facets module, facets.yaml, or module skeleton (any intent/flavor/cloud). Not for generic non-Facets Terraform."
triggers: ["module", "facets.yaml", "facets module"]
category: "development"
tags: ["iac", "terraform", "facets-yaml", "module-development", "llm"]
icon: "🧱"
version: "1.4"
---

# Facets Module Generator

Drafts Facets IaC modules using a division of labor validated against the
fine-tune's training data: the **fine-tuned model** (Qwen3.6-35B
`qwen36-facets-nothink`, RunPod queue-based serverless) drafts the
`facets.yaml` — that is its comparative advantage (house spec idiom,
`intentDetails`, `x-ui-*` annotations). **You (Claude) author all Terraform
files yourself** from the corrected yaml plus repo ground truth. Do NOT ask
the fine-tune for Terraform: it was never trained to emit multi-file modules
and its per-file generations are unreliable on the deployed checkpoint.

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

**Cost & courtesy:** every call spends GPU time on an endpoint the whole team
shares. One call per module draft is the intended budget — do not loop retries
against the endpoint. If the call comes back unusable, use the fallback at the
end of step 1 (author the yaml yourself where possible) instead of calling again.

## Procedure

### 0. Pre-flight (before any endpoint call)

Determine if you are inside the Facets module repository (markers: `rules.md`,
`modules/`, `outputs/` at repo root).

- **Inside the repo:** read, BEFORE calling the endpoint (so the single
  endpoint call pays at most one cold start): (a) the **type-to-schema map** —
  the directory name under `outputs/` is NOT always the type name, so build the
  map from ground truth: `grep -H '^name:' outputs/*/outputs.yaml` and record
  each declared `@facets/...` name → its `outputs.yaml` path (a type must match
  exactly one entry); (b) the schema files of types likely relevant to the
  request; (c) one exemplar module of a similar intent (`modules/<similar>/...`)
  for current file-layout and wiring conventions.
- **Outside the repo (degraded mode):** you may still run step 1, but you MUST
  mark every `@facets/` type in the result with `# UNVERIFIED — check against
  outputs/ registry`, tell the user validation was impossible, and stop after
  presenting the yaml. Do not fabricate Terraform without the registry.

### 1. Fine-tune call — facets.yaml (the model's one job)

Build the prompt in the model's trained "scratch dialect", with intent and
flavor backticked, ending with the literal sentence "Start with the
facets.yaml.":

```
I need a new Facets module: intent `<intent>`, flavor `<flavor>`, version 1.0, targeting <clouds>. It should: <the user's requirements, direct and concise>. Start with the facets.yaml.
```

Use ONLY the user's module request — never earlier conversation content,
credentials, customer data, or pasted repo files (pasted context degrades
this model, verified empirically).

Write that prompt to `/tmp/facets-module-prompt.txt` with the Write tool
(never interpolate user text into shell), then run as ONE Bash script:

```bash
set -eu
REQ=$(jq -n --rawfile prompt /tmp/facets-module-prompt.txt '{
  input: {
    model: "qwen36-facets-nothink",
    temperature: 0.3, top_p: 0.95, max_tokens: 8192,
    chat_template_kwargs: {enable_thinking: false},
    messages: [
      {role: "system", content: "You help developers write and review Facets IaC modules. Follow the Facets module conventions precisely."},
      {role: "user", content: $prompt}
    ]
  }
}')
R=$(printf '%s' "$REQ" | curl -sS --max-time 120 -X POST \
  "https://api.runpod.ai/v2/$FACETS_LLM_ENDPOINT/runsync" \
  -H "Authorization: Bearer $FACETS_LLM_KEY" \
  -H "Content-Type: application/json" -d @-)
JID=$(printf '%s' "$R" | jq -r '.id // empty')
STATUS=$(printf '%s' "$R" | jq -r '.status // "UNKNOWN"')
DEADLINE=$(( $(date +%s) + 900 ))
while [ "$STATUS" = "IN_QUEUE" ] || [ "$STATUS" = "IN_PROGRESS" ]; do
  [ $(date +%s) -gt $DEADLINE ] && { echo "TIMED OUT after 15 min (job $JID)"; exit 1; }
  sleep 15
  R=$(curl -sS --max-time 60 -H "Authorization: Bearer $FACETS_LLM_KEY" \
    "https://api.runpod.ai/v2/$FACETS_LLM_ENDPOINT/status/$JID")
  STATUS=$(printf '%s' "$R" | jq -r '.status // "UNKNOWN"')
done
if [ "$STATUS" = "COMPLETED" ]; then
  printf '%s' "$R" | jq -r '.output.choices[0].message.content // ("EMPTY OUTPUT: " + (.output|tostring))'
else
  echo "JOB $STATUS:"; printf '%s' "$R" | jq -c '.error // .' | head -c 500
  exit 1
fi
```

Cold start: first call after idle can wait minutes in IN_QUEUE — tell the
user a GPU worker is booting; it will complete.

**Degenerate-output check:** treat the draft as unusable if it is

- empty,
- a bare file tree,
- truncated mid-YAML,
- YAML that fails to parse,
- mostly prose (commentary/explanation) rather than a yaml document, or
- under ~20 lines — a real facets.yaml is never that short.

In every one of those cases do NOT retry the endpoint and do NOT re-call it
with more context. Then branch: **inside the module repo**, author the
facets.yaml yourself from the exemplar module, say so to the user, and
continue with step 2. **Outside the repo (degraded mode)**, report the
unusable response and stop — do not fabricate a facets.yaml or Terraform
without the registry.

### 2. Correct the draft against ground truth (you, not the model)

The model invents `@facets/` type names (verified). Against the registry you
read in step 0:

- Replace every input/output `type:` with a real registered type; if none
  fits, flag it to the user rather than inventing one.
- **Attribute checklist (hard requirement).** For each corrected input type,
  open its schema file via the step-0 type-to-schema map (NOT by guessing
  `outputs/<type-name>/` — directory names and type names diverge) and
  enumerate BOTH the `attributes.*` and `interfaces.*` paths it exposes,
  recording each path's schema type (scalar / list / map / object). Keep that
  as a scratch list per input and carry it into step 3: every
  `var.inputs.<name>.attributes.<path>` or `var.inputs.<name>.interfaces.<path>`
  dereference you write must correspond to a listed path — indexing or map
  keys (`[0]`, `["key"]`) are allowed only beneath a listed list/map property.
  Do not rely on memory of what a type "usually" has — read the schema.
- If an attribute the module genuinely needs does not exist on any registered
  output type, **STOP** and tell the user (name the input, the missing path,
  and the types you checked). Never invent a path to make the module compile.
- Fix `sample.kind` (= intent) and `intentDetails` (RULE-021). Leave
  `iac.validated_files` for the end of step 3, after the file set is decided.

### 3. Author the Terraform yourself

Write the Terraform directly, following the exemplar module's current
conventions:

- **Decide the Terraform file set FIRST, from the exemplar as a convention
  baseline.** The usual set is `variables.tf`, `main.tf`, `locals.tf`,
  `outputs.tf`; some intents also carry `variables_outputs.tf` and/or
  `providers.tf`. Consider only top-level `*.tf` files in the exemplar —
  ignore README/docs/YAML/assets. Once decided, set the facets.yaml
  `iac.validated_files` to exactly the `.tf` files you will create — every
  created top-level `.tf` listed, no extras, no omissions.
- **Decide the wiring location from the exemplar too.** Check whether the
  exemplar defines `output_attributes`/`output_interfaces` in `locals.tf` or in
  `outputs.tf`, define them in exactly ONE file, and after writing, check
  assignments only (references and comments don't count):
  `grep -En '^\s*(output_attributes|output_interfaces)\s*=' <module-path>/*.tf`
  — require exactly one assignment line for EACH of the two symbols.
- `variables.tf`: `var.instance` typed from the spec schema; `var.inputs`
  typed from the corrected input types' real schemas.
- Only reference input attribute paths that appear on the step 2 scratch list.
- **Coherence pass (after all files are written).** Re-read the file set as a
  whole and confirm: every `local.*` referenced is actually defined; every
  `var.*` referenced is declared in `variables.tf`/`variables_outputs.tf`; and
  no file still references content you removed or relocated while editing
  another file. Fix before running validation.

### 4. Validate

- Run `raptor create iac-module -f <module-path> --dry-run`; fix and re-run
  until clean. Report security-scan findings to the user in a table; never
  skip validation silently.
- Check the repo's new-module checklist (icon, project-type entry, catalog)
  from CLAUDE.md and tell the user which items remain.
- Verify any RULE-number claims by grepping `rules.md` — the model does not
  reliably recall rules by number.

## Known model limits (why the pipeline is shaped this way)

- Trained to emit `facets.yaml` from scratch asks — never full multi-file
  modules; per-file Terraform generation is unreliable on the deployed
  checkpoint. Hence: model drafts yaml, Claude writes Terraform.
- Invents `@facets/` type names; inconsistent across runs. Hence step 2.
- No raptor CLI knowledge (it will hallucinate raptor commands confidently).
- Extra pasted context (repo files, prior conversation) degrades output.
