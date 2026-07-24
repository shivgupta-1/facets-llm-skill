---
name: "module"
title: "Facets Module Generator (fine-tuned LLM)"
description: "Generate Facets IaC modules (facets.yaml + Terraform) by querying the Facets fine-tuned model served on RunPod. Use when the user runs /module followed by what they want, or asks to draft a Facets module, facets.yaml, or module skeleton using the Facets LLM."
triggers: ["module", "facets.yaml", "facets llm"]
category: "development"
tags: ["iac", "terraform", "facets-yaml", "module-development", "llm"]
icon: "🧱"
version: "1.0"
---

# Facets Module Generator

Queries the company's fine-tuned Facets model (Qwen3.6-35B `qwen36-facets-nothink`, served on RunPod serverless) to draft Facets IaC modules. The model is trained on the Facets module repository: facets.yaml conventions, module standards, output types, and validation rules.

## Configuration

Two environment variables (employees get these from the platform team):

- `FACETS_LLM_ENDPOINT` — RunPod endpoint id (current: `t77jarug59nzvk`)
- `FACETS_LLM_KEY` — team inference API key (RunPod key)

If either is missing, stop and show the user this setup snippet instead of calling the API:

```bash
# add to ~/.zshrc — values from the platform team (#platform-eng)
export FACETS_LLM_ENDPOINT="t77jarug59nzvk"
export FACETS_LLM_KEY="<team-inference-key>"
```

## Procedure

1. **Build the request.** Use the user's prompt (the text after `/module`) as the user message, verbatim plus any concrete requirements they stated earlier in the conversation (cloud, intent name, flavor, inputs). Keep it short and specific — the model was trained on direct instructions, NOT on long context dumps. Do NOT paste repository files into the prompt.

2. **Use EXACTLY this system prompt** (the model was fine-tuned with it; changing it degrades output):

   ```
   You help developers write and review Facets IaC modules. Follow the Facets module conventions precisely.
   ```

3. **Call the endpoint** (RunPod queue API, synchronous). Cold start note: if no
   worker is warm, the job waits in the queue while one boots (typically 2–8 min;
   the request below blocks until done). Tell the user it's warming up if the
   first call is slow.

   ```bash
   jq -n --arg prompt "<USER PROMPT HERE>" '{
     input: {
       model: "qwen36-facets-nothink",
       temperature: 0.7, top_p: 0.95, max_tokens: 4096,
       chat_template_kwargs: {enable_thinking: false},
       messages: [
         {role: "system", content: "You help developers write and review Facets IaC modules. Follow the Facets module conventions precisely."},
         {role: "user", content: $prompt}
       ]
     }
   }' | curl -sS --max-time 900 -X POST \
     "https://api.runpod.ai/v2/$FACETS_LLM_ENDPOINT/runsync" \
     -H "Authorization: Bearer $FACETS_LLM_KEY" \
     -H "Content-Type: application/json" -d @- \
   | jq -r '.output.choices[0].message.content // ("ERROR: " + (.|tostring))'
   ```

   If the response has `"status": "IN_QUEUE"` or `"IN_PROGRESS"` instead of output
   (runsync returns early after ~90s), poll
   `https://api.runpod.ai/v2/$FACETS_LLM_ENDPOINT/status/<id>` every 15s until
   COMPLETED, then read `.output.choices[0].message.content`.

4. **Present the result** as the model's draft, clearly labeled as coming from the Facets LLM. If the user wants the files, write them into the proper repo layout (`modules/{intent}/{flavor}/{version}/`).

5. **Always validate before calling it done.** The draft is a starting point, not a finished module:
   - If in the module repo, run `raptor create iac-module -f <module-path> --dry-run` and report results.
   - Check the new-module checklist (icon, project-type entry, catalog page) from the repo CLAUDE.md.
   - Review against `rules.md` — the model does NOT reliably recall rules by RULE-number; verify rule references by grepping `rules.md` yourself.

## Known model limits (set expectations)

- No knowledge of raptor CLI specifics — don't ask it raptor command questions; use `/raptor` skills instead.
- RULE-number ↔ rule-text bindings are unreliable; content knowledge is good, numbered citations are not.
- Single-shot drafting is its strength; it is not an agentic tool-user. Keep Claude in charge of file writes, validation, and iteration.
