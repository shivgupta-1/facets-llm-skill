---
name: "module"
title: "Facets Module Generator (fine-tuned LLM)"
description: "Generate Facets IaC modules (facets.yaml + Terraform) by querying the Facets fine-tuned model served on RunPod. Use when the user runs /module followed by what they want."
disable-model-invocation: true
triggers: ["module"]
category: "development"
tags: ["iac", "terraform", "facets-yaml", "module-development", "llm"]
icon: "🧱"
version: "1.1"
---

# Facets Module Generator

Queries the company's fine-tuned Facets model (Qwen3.6-35B `qwen36-facets-nothink`,
RunPod queue-based serverless) to draft Facets IaC modules. The model is trained on
the Facets module repository: facets.yaml conventions, module standards, output
types, and validation rules.

## Configuration

Two environment variables (values from the platform team). `jq` and `curl` must be
installed (preflight: `command -v jq curl`).

- `FACETS_LLM_ENDPOINT` — RunPod endpoint id (current: `t77jarug59nzvk`)
- `FACETS_LLM_KEY` — inference API key

If either env var is missing, stop and show the user this snippet instead of calling
the API (they must restart their session after adding it):

```bash
# add to ~/.zshrc — values from the platform team (#platform-eng)
export FACETS_LLM_ENDPOINT="t77jarug59nzvk"
export FACETS_LLM_KEY="<team-inference-key>"
```

## Procedure

1. **Build the prompt.** Use ONLY the text the user passed after `/module` as the
   request. Do not add earlier conversation content unless the user explicitly asks
   you to include specific stated requirements — and never include anything that
   looks like credentials or customer data. Keep it short and direct (the model was
   trained on direct instructions, not context dumps). Do NOT paste repository files.

2. **Write the prompt to a temp file** (never interpolate user text into shell):
   use the Write tool to create `/tmp/facets-module-prompt.txt` containing exactly
   the prompt text.

3. **Submit and poll.** Run this as ONE Bash script (it is injection-safe: the
   prompt enters via `--rawfile`):

   ```bash
   set -eu
   REQ=$(jq -n --rawfile prompt /tmp/facets-module-prompt.txt '{
     input: {
       model: "qwen36-facets-nothink",
       temperature: 0.7, top_p: 0.95, max_tokens: 4096,
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

   Cold start: if the first poll cycles show IN_QUEUE for a few minutes, tell the
   user a GPU worker is booting (~3–8 min from cold) — this is normal and the job
   will complete.

4. **Present the result** as the model's draft, clearly labeled as coming from the
   Facets LLM. Only write files if the user asks AND you are inside the Facets
   module repository (verify markers like `rules.md` and `modules/` exist at the
   repo root first); then use the proper layout `modules/{intent}/{flavor}/{version}/`.
   Otherwise present the draft as text or ask for a target path.

5. **Always validate before calling it done.** The draft is a starting point:
   - In the module repo, run `raptor create iac-module -f <module-path> --dry-run`
     and report results.
   - Check the new-module checklist (icon, project-type entry, catalog page) from
     the repo CLAUDE.md.
   - Verify any RULE-number references by grepping `rules.md` yourself — the model
     does NOT reliably recall rules by number.

## Known model limits (set expectations)

- No knowledge of the raptor CLI — don't ask it raptor questions; use `/raptor` skills.
- RULE-number ↔ rule-text bindings are unreliable; content knowledge is good.
- Single-shot drafting is its strength; Claude stays in charge of file writes,
  validation, and iteration.
