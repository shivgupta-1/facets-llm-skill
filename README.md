# facets-llm — Claude Code skills for the Facets fine-tuned model

`/module <what you want>` drafts a Facets IaC module via guided multi-step
generation: the team's fine-tuned model drafts the facets.yaml (its specialty),
your Claude verifies it against the real type registry, writes the Terraform,
and validates with raptor. Best used inside the Facets module repository.

## How it works

1. The fine-tune is called **once** and drafts only the `facets.yaml` spec — it
   never writes Terraform, and it invents `@facets/` type names.
2. Your Claude corrects that spec against the real `outputs/` type registry and
   the input-type schemas, then authors every Terraform file itself.
3. Your Claude validates the result with `raptor create iac-module --dry-run`
   and reports what still needs your attention.

## Install (once)

Prereqs: `jq` and `curl` on PATH; GitHub access to this repo.

```
/plugin marketplace add shivgupta-1/facets-llm-skill
/plugin install module@facets-llm
```

Add the env vars (values from the platform team), then **restart Claude Code
from a fresh shell** so both the plugin and the env vars load:

```bash
# ~/.zshrc
export FACETS_LLM_ENDPOINT="2xd61lauejq03o"
export FACETS_LLM_KEY="<team inference key>"
```

## Use

```
/module a facets module for an s3 bucket with versioning and lifecycle rules
```

(If `/module` collides with another command, the namespaced form is
`/module:module`.) First request after idle takes a few minutes — GPU cold
start; later requests are fast. The endpoint id is not a secret; the key is —
treat it like a password, and report leaks to the platform team for rotation.
