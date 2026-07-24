# facets-llm — Claude Code skills for the Facets fine-tuned model

## Install
```
/plugin marketplace add shivgupta-1/facets-llm-skill
/plugin install module@facets-llm
```
Set two env vars (values from the platform team):
```bash
export FACETS_LLM_ENDPOINT="t77jarug59nzvk"
export FACETS_LLM_KEY="<team inference key>"
```
Then: `/module a facets module for an s3 bucket with versioning`

First request after idle takes a few minutes (GPU cold start). The model runs on
RunPod queue-based serverless; see the platform team for keys and details.
