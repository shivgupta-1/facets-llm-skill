---
name: "module"
title: "Facets Module Generator (fine-tuned LLM)"
description: "Generate Facets IaC modules by querying the Facets fine-tuned model served on RunPod. Use when the user runs /module, OR asks to write/draft/generate/sketch a Facets module, facets.yaml, or module skeleton (any intent/flavor/cloud). Not for generic non-Facets Terraform."
triggers: ["module", "facets.yaml", "facets module"]
category: "development"
tags: ["iac", "terraform", "facets-yaml", "module-development", "llm"]
icon: "🧱"
version: "1.3"
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

## Procedure

### 0. Pre-flight (before any endpoint call)

Determine if you are inside the Facets module repository (markers: `rules.md`,
`modules/`, `outputs/` at repo root).

- **Inside the repo:** read, BEFORE calling the endpoint (so endpoint calls
  run back-to-back and pay at most one cold start): the list of real output
  types (`ls outputs/`), the schema files of any types likely relevant to the
  request, and one exemplar module of a similar intent
  (`modules/<similar>/...`) for current file-layout and wiring conventions.
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

**Degenerate-output check:** if the response is empty, a bare file tree, or
truncated mid-YAML, do NOT retry with more context. Author the facets.yaml
yourself from the exemplar module and continue.

### 2. Correct the draft against ground truth (you, not the model)

The model invents `@facets/` type names (verified). Against the registry you
read in step 0:

- Replace every input/output `type:` with a real registered type; if none
  fits, flag it to the user rather than inventing one.
- Read the corrected input types' schemas — downstream Terraform must only
  dereference `var.inputs.<name>.attributes.<path>` paths that actually
  exist in those schemas.
- Fix `sample.kind` (= intent), `intentDetails` (RULE-021), and make
  `iac.validated_files` list exactly the files you will actually create.

### 3. Author the Terraform yourself

Write `variables.tf`, `main.tf`, `locals.tf`, `outputs.tf` (and
`variables_outputs.tf`/`providers.tf` if the exemplar convention uses them)
directly, following the exemplar module's current conventions:

- `variables.tf`: `var.instance` typed from the spec schema; `var.inputs`
  typed from the corrected input types' real schemas.
- `locals.tf` vs `outputs.tf`: put `output_attributes`/`output_interfaces`
  wherever the exemplar puts them — never define them in both files.
- Only reference input attribute paths verified in step 2.

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
