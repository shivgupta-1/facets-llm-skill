# facets-llm — Claude Code skills for the Facets fine-tuned model

`/module <what you want>` generates a complete Facets IaC module: the team's
fine-tuned model writes every file, your Claude orchestrates — planning,
reviewing against the type registry, requesting fixes — and validates with
raptor. Best used inside the Facets module repository.

## How it works

1. The fine-tune **writes every module file** — facets.yaml first, then each
   Terraform file, one trained-dialect call at a time (~6–12 calls total).
2. Your Claude orchestrates: it plans from the real `outputs/` type registry
   and an exemplar module, reviews every generated file against them, and
   sends fix requests back to the fine-tune. It only writes file content
   itself if you explicitly ask it to. (Outside the module repo it stops
   after the facets.yaml draft.)
3. Your Claude validates with
   `raptor create iac-module -f <module-path> --dry-run`, routes failures
   back to the fine-tune, and reports what still needs your attention.

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
