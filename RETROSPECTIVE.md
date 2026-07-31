# /module — retrospective: serving the Facets fine-tune + building the skill

Companion doc to the fine-tuning retrospective (private repo
`shivgupta-1/facets-finetune-data`, `RETROSPECTIVE.md`). That one covers how the
model was trained; this one covers **how it went to production and became
`/module`** — the deployment saga, every architecture pivot of the skill, and
the first end-to-end module it shipped.

---

## Current state (July 2026)

| Thing | State |
|---|---|
| **Serving** | RunPod **queue-based** serverless, endpoint `2xd61lauejq03o` (`qwen36-facets-nothink-qb`), scale-to-zero, ctx 32,768 |
| **Image** | `ghcr.io/shivgupta-1/facets-llm:qb-v4` (private GHCR): official llama.cpp CUDA server + baked 21GB GGUF + a static curl+jq shell shim implementing the QB job loop |
| **Skill** | `/module` v2.2 (this repo) — install via `/plugin marketplace add shivgupta-1/facets-llm-skill` |
| **Division of labor** | the **fine-tune writes every module file** (one trained-dialect call per file); **Claude orchestrates** — plans from the `outputs/` type registry + an exemplar module, reviews each file, sends fix requests, runs `raptor create iac-module --dry-run` |
| **Proven E2E** | `postgres_database/k8s/1.0` generated end-to-end (9 endpoint calls, one approved hand-fix), raptor dry-run fully green — branch `feat/postgres-database-k8s-module` of facets-modules-redesign |
| **Ops cheatsheet** | `facets-modules-redesign/finetune/runpod-serverless/MODEL_CARD.md` (private) — invoke shapes, sampling, cold-start behavior, worker env contract |

---

## Phase 1 — getting the model served (July 22–24)

### The load-balancing dead end

First attempt used RunPod **LOAD_BALANCING** serverless: llama.cpp server in a
worker, gateway routes HTTP to it. Across **~30 worker lifecycles** — every GPU
type, datacenter, and config permutation tried — the gateway never routed a
single request to provably-listening workers ("timed out waiting for worker";
health probe/registration never connected). Workers were healthy; the platform
side never wired them in. Unresolved; support ticket drafted. A full day burned.

### Queue-based: worked on the first correct attempt

Pivoted to RunPod's original **queue-based** protocol: the worker polls a job
queue instead of receiving routed HTTP. No SDK, no Python — a small shell shim
(static curl + jq) does `job-take → local llama-server on 127.0.0.1:8080 →
job-done`. Caller does `POST /runsync` (or `/run` + `/status/{id}` polling) with
an OpenAI chat body inside `{"input": ...}`.

### Deployment gotchas that cost real time

| Gotcha | Lesson |
|---|---|
| llama-server downloading the 21GB GGUF at boot lost the race against RunPod's ~15-min unhealthy-worker kill | **bake the model into the image**; never download at boot |
| macOS `tar` poisons OCI layers with AppleDouble/xattr entries | `COPYFILE_DISABLE=1 tar --no-xattrs --uid 0 --gid 0`, or build layers on Linux |
| Rebuilding a 21GB image for every change | `crane append` + `crane mutate --entrypoint` — the model layer is reused by digest, so rebakes cost **seconds** |
| The "48GB PRO" GPU pool silently includes Blackwell **MIG 2g.48gb** slices (2/7 compute) | they work under QB, just slower; GUI per-GPU ticks are the only precise exclusion |
| Thinking mode leaked (`content` empty, output in `reasoning_content`) despite the GGUF's no-think template | kill it **twice**: `LLAMA_ARG_REASONING=off` (server) AND `chat_template_kwargs.enable_thinking=false` (per request) |
| `runsync` returns `IN_QUEUE` after ~90s on cold start | poll `/status/{id}`; the queue absorbs image-pull cold starts instead of gateway-timing-out |

---

## Phase 2 — the skill, iteration by iteration

The git history is the changelog; the pivots were driven by **measurement, not
vibes**:

| Version | What changed | Why |
|---|---|---|
| v1.0 | First `/module`: call the endpoint, ask for a module | naive; assumed the model could produce whole modules in one go |
| v1.0.x | Injection-safe prompt flow, full polling state machine, explicit invocation only | post-review hardening; endpoint recreated with global scheduling |
| v1.2 | Allow auto-invocation on natural-language module requests | usability |
| **v1.3** | **Pivot #1: fine-tune drafts facets.yaml ONLY; Claude authors all Terraform** | assessment showed **zero Terraform** in 3 scratch trials; dataset measurement confirmed why — 0/142 scratch-dialect training completions contain Terraform (context tasks are single-file, never a full module). The model was never taught to emit multi-file modules unprompted |
| v1.4 / v1.4.1 | Verification hardening after adversarial pipeline review (Codex): type-to-schema map, degraded-mode branch, precise wiring assertion, objective degenerate-output test | close review findings before widening scope |
| **v2.0** | **Pivot #2 (reversal): fine-tune authors ALL files again — but one per-file trained-dialect call at a time; Claude plans/reviews/fix-loops** | a probe battery on *real-module context* validated the per-file dialects; the earlier "per-file is unreliable" verdict turned out to be an artifact of synthetic-context probes. Claude writes content only on explicit user request |
| v2.1 | Cross-file type-shape consistency check (spec schema is the source of truth) | catches the model drifting between files |
| **v2.2** | All P0–P2 patches from a max-effort adversarial review | the version that shipped the E2E module |

Key architecture facts of v2.2:
- **Trained dialects, exactly**: the model degrades on any prompt shape it
  wasn't trained on. The skill embeds the exact system prompts + user-message
  shapes from the training set (scratch dialect for facets.yaml, context
  dialect per Terraform file, fix dialect for revisions).
- **Registry over memory**: `@facets/*` output types are verified against the
  real `outputs/` registry, never trusted from the model.
- **Sampling**: temp 0.3 for drafting, max_tokens 8192 (trained facets.yaml
  drafts reach ~9k tokens).

---

## Phase 3 — first full E2E run (July 27, v2.2): SUCCESS

Target: `postgres_database/k8s/1.0` — create databases on existing Aurora/RDS
via an idempotent k8s psql job.

- **9 endpoint calls**: facets.yaml + 1 fix; variables.tf + 1 fix; main.tf +
  2 fixes; locals.tf + 1 fix; outputs.tf + 1 fix — all authored by the
  fine-tune, all fixes routed back through the fix dialect.
- **One approved hand-fix**: HCL string concatenation — the model consistently
  writes `+` for strings (Python habit) instead of HCL interpolation. Now a
  named anti-pattern candidate for a skill-side fix template and a v3 training
  item.
- `raptor create iac-module --dry-run`: **all validations passed**, output
  types validated.
- Every predicted failure mode appeared **and was caught by the check designed
  for it** — the review/fix-loop architecture did its job.
- Committed on `feat/postgres-database-k8s-module` (PR not opened; control-plane
  upload not run).

---

## What went right

- **Measure before architecting**: the v1.3 pivot came from counting Terraform
  in 142 training completions, not from guessing; the v2.0 reversal came from
  re-probing with real context after distrusting a synthetic probe.
- **Queue-based over load-balanced**: choosing the boring, older protocol ended
  a day of platform-side mystery instantly.
- **Baked image + crane**: 21GB model layer reused by digest; config iterations
  cost seconds instead of image rebuilds.
- **Adversarial review gates** (v1.4, v2.2): external max-effort reviews before
  each capability expansion; all P0–P2 findings patched before the E2E attempt.
- **Designed checks caught every predicted failure** in the live run — nothing
  escaped to the human except the one genuinely novel HCL anti-pattern.

## What to fix in v3 (training items)

1. **Multi-file module completions** in the dataset — the reason the pipeline
   exists at all is that scratch training data never contained full modules.
   Teach that, and the orchestration can thin out.
2. **HCL string concatenation** — add anti-pattern pairs (`+` → `${}`/`join`).
3. Raptor CLI knowledge is in the v2 model (trained after this deployment —
   the deployed checkpoint is v1 and hallucinates raptor); redeploy with the
   v2/v3 checkpoint to remove the skill's raptor-knowledge workarounds.
