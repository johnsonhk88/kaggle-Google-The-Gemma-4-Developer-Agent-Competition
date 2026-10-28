# Code development guide

How to build, measure, and change the agent for
[Google – The Gemma 4 Developer Agent](https://www.kaggle.com/competitions/gemma-4-developer-agent).

Read this before adding files. The scorer never imports your training code. It only
compiles whatever is inside `submission.zip`.

Contract details: `src/sample/HARNESS_README.md`.
Baseline zip: `src/sample/gemma-4-harness-fixed-single-agent.ipynb`.

---

## 1. What you are building

An **ADK YAML agent** that Gemma 4 31B follows while it patches a Python repo.

```text
issue text  →  Gemma 4 31B + your prompt/tools/skills  →  git diff
                                                              ↓
                                              hidden pytest in a fresh sandbox
                                                              ↓
                                                    1.0 if tests pass, else 0.0
```

Score = resolved tasks / total. Public board ≈ 58 tasks. Hidden set ≈ 120 private
repos. Agent phase is **12 hours, sequential**. **1 submission per day.**

You do **not** build:

- a Python `agent.py` entrypoint (rejected)
- a new harness, vLLM server, or pytest grader (host already runs these)
- a CSV of patches

You **do** build, in this order:

1. A packaging notebook that writes `submission.zip`
2. A GPU eval notebook that runs your zip on public tasks and dumps traces
3. Prompt / budget changes driven by those traces
4. Optional ADK **skills** (Python scripts the agent may call)
5. Optional read-only **analyzer** sub-agent
6. LoRA only if a current host thread confirms adapters load

---

## 2. Submission contract (do not violate)

Zip root must contain exactly one of: `agent.yaml`, `agent.yml`,
`root_agent.yaml`, `root_agent.yml`.

Allowed extensions only: `.yaml` `.yml` `.md` `.txt` `.py` `.json` `.safetensors`.

One base model for every agent, from:

- `gemma-4-31b-it-qat-w4a16-ct` ← use this unless you have a measured reason not to
- `gemma-4-12b-it-qat-w4a16-ct`
- `gemma-4-e4b-it-qat-w4a16-ct`
- `gemma-4-e2b-it-qat-w4a16-ct`

`eval_config.yaml` keys the scorer actually reads:

```yaml
evaluation:
  timeout_seconds: 300      # Phase 1 run_command timeout
  max_tool_calls: 28        # submit_patch and get_status are free
  max_time_minutes: 8       # agent loop only (setup excluded)
  max_turns: 80
```

Missing a key = **no limit**. No `eval_config.yaml` = 60 min / 100 calls → 12h DNF.

On the Kaggle scorer, Phase 2 hidden pytest timeout is **fixed at 300s**. Do not
treat `timeout_seconds` as the grading timeout.

Hard limits: unpacked zip &lt; 3 GiB, vLLM context 32,768 tokens, sandbox 4 GiB RAM /
2 vCPU / no network.

`!include` paths are relative to the including file. No `..`, no absolute paths,
no `{placeholder}` in instructions except names that always exist in session
state. `hints` is **absent** when empty → `{hints}` will `KeyError` and kill the
task. The harness already puts the problem and hints in the **user** message.
Keep `prompts/system.md` free of `{name}` patterns.

---

## 3. Repo layout for development

Stay notebook-first. When you add skills, keep them **inside the zip**, not as a
local Python package.

```text
src/sample/
  gemma-4-harness-fixed-single-agent.ipynb   # packaging notebook (submit this)
  getting-started-gemma-4-developer-agent.ipynb
  HARNESS_README.md
  eval-public-tasks.ipynb                    # YOU ADD: GPU measurement
```

Target zip (what Kaggle compiles):

```text
submission.zip
├── agent.yaml
├── eval_config.yaml
├── configs/sampling.yaml
├── prompts/system.md
└── skills/                          # add only after traces exist
    ├── repo-navigation/
    │   ├── SKILL.md
    │   └── scripts/navigate.py
    └── test-discovery/
        ├── SKILL.md
        └── scripts/run_related.py
```

Do not add `src/g4agent`, pytest harness clones, or custom Docker. They never
run on the leaderboard.

---

## 4. Development loop

Every change follows the same loop. Skip a step and you are guessing.

```text
1. Change ONE thing (prompt sentence, budget number, or one skill)
2. Package zip (CPU notebook, Run All)
3. Run a FIXED public slice on 4× L4 (same task ids every time)
4. Read traces → assign each fail to a bucket
5. If resolve rate or empty-patch rate improved, keep it
6. At most one Kaggle submission per day
```

Public leaderboard noise is about ±2 tasks. Do not promote a change from a
single scored run.

Fixed slice (start here): 8–12 tasks mixed across `Textualize/rich`,
`psf/requests`, `fastapi/fastapi`, `encode/httpx`. Re-run the **same ids**
after every change.

Record per task:

| Field | Why |
|---|---|
| `instance_id`, `repo` | identity |
| `resolved` | the only score |
| `duration_seconds` | 12h budget |
| `tool_calls` | 28-call cap |
| `agent_patch` empty? | empty tree = 0 |
| call index of first `edit_file` / `write_file` | must be ≤ 12 |
| last tool name | stuck on grep vs submit |
| error / timeout | cap hit vs crash |

---

## 5. Workstreams (priority)

### W0 — Ship the baseline (done)

Notebook: `gemma-4-harness-fixed-single-agent.ipynb`

- 31B, 9 tools, no LoRA
- 8 min / 28 calls / 300s / thinking 4096
- Prompt: edit by call 12, `/tmp` scratch, `submit_patch` last, no `{placeholders}`

Submit once so you have a public score and a 12h runtime log.

### W1 — Measurement notebook (build next)

Fork `getting-started-gemma-4-developer-agent.ipynb`.

Changes to make:

1. Keep wheelhouse install + vLLM start (required on Kaggle GPU).
2. Do **not** copy `sample_submission`. Write **your** agent files (same `FILES`
   dict as the packaging notebook) into `/kaggle/working/agent` and set
   `submission_dir` to that folder.
3. Set `SAMPLE_TASKS` to an explicit id list, not `tasks[:2]`.
4. After each task, append one JSON line to
   `/kaggle/working/task_results.jsonl`.
5. Print a one-row summary: resolved count, empty patches, cap hits, mean
   duration.

Do not run the full 129 tasks until the 8–12 slice is stable. 4× L4 sessions
queue for hours; pack baseline + candidate into **one** GPU session when you
compare.

### W2 — Failure buckets (no new features)

Classify every unresolved task before writing code:

| Bucket | Evidence in the trace | Code / prompt to develop |
|---|---|---|
| A. No source edit | no `edit_file` success before timeout | Stronger call-12 rule; free `get_status` |
| B. Wrong file | patch in the wrong module | **repo-navigation skill** |
| C. Right file, edit failed | `FileEditError` / truncated `tool_call` | Smaller unique `old_string`; lower `max_output_tokens` already 8192 |
| D. Right file, wrong behavior | patch applies, hidden tests fail | Prompt: copy exact exception/message; sibling pattern |
| E. Agent ran full pytest / OOM | huge duration, exit 137 | Prompt already forbids this; skill must refuse bare pytest |
| F. Cap / 12h | `max_time_minutes` hit, or whole run errors | Drop minutes, not raise calls |
| G. Repeat loop | same grep 10+ times | Anti-repeat sentence in prompt |

Fix **one bucket per daily submission**.

### W3 — Skills (first real Python worth writing)

Skills are the only extra code the compiler will run. Each skill is a directory
with `SKILL.md` (YAML frontmatter `name` + `description`) plus optional
`scripts/*.py`.

Rules from the harness:

- Declare the directory in `agent.yaml` under `skills:`.
- ADK often requires `name` == directory name. Prefer **kebab-case**
  (`repo-navigation`) unless `compile_submission` accepts the official example
  `repo_navigation`. Validate by compiling once on Kaggle.
- Scripts run via `run_skill_script` in a **temp directory**. Address the repo
  as `/workspace`, never `.`.
- Script time counts against the task budget. Output is still truncated
  (plan for ≤ 5,000 characters).
- Only `.py` scripts. No `.sh`.

**Skill 1 — `repo-navigation`**

One script, subcommands: `find`, `show`.

- `find`: take issue keywords, run `git grep -n` / `git ls-files` under
  `/workspace`, rank paths (definition-line bonus, skip `tests/` as primary
  targets), print top 10 `path:line`.
- `show`: print ≤ 40 numbered lines around a symbol or line range.

This attacks bucket B (wrong file) without spending 6 `run_command` calls.

**Skill 2 — `test-discovery`**

Subcommands: `related`, `run`.

- `related`: map a source path or symbol to one test file (name overlap under
  `tests/`).
- `run`: `timeout 150 python -m pytest <one file> -x -q -k <keyword>`. Refuse
  argv that is bare `pytest` / `pytest .`. Print a short failure summary, not
  full logs.

This attacks buckets D and E.

Wire in `agent.yaml`:

```yaml
skills:
  - skills/repo-navigation
  - skills/test-discovery
```

Teach the prompt: prefer these skills for locate/verify; still edit with
`edit_file`; still `submit_patch` last.

Keep skills out of the zip until W1 shows that locate/verify is the bottleneck.

### W4 — Budget retune (numbers only)

After a GPU or scored run, compute:

- `f` = fraction of tasks that hit the time cap
- `m` = mean duration of tasks that stopped themselves
- worst case ≈ 15 min server + N × (cap + 0.1 min)

If worst case for N ≈ 125 exceeds ~11 hours, **lower** `max_time_minutes`.
If empty patches dominate and cap hits are rare, keep 8 minutes and fix the
prompt/skill, do not raise `max_tool_calls`.

Do not change more than one of {minutes, calls, thinking on/off} in the same
submission.

### W5 — Analyzer AgentTool (optional)

Only after W3 has a measured baseline.

```yaml
tools:
  - run_command
  - read_file
  - edit_file
  - write_file
  - get_status
  - submit_patch
  - agent_tool:
      config_path: sub_agents/code_analyzer.yaml
      skip_summarization: true
```

Analyzer: same 31B model, **read-only tools** (`run_command`, `read_file`,
graph tools). No `edit_file` / `write_file` / `submit_patch`. Short instruction:
return file paths, symbols, and a 10-line contract, then stop.

This keeps giant reads out of the coder’s 32k context. It costs extra model
turns inside 8 minutes — measure or do not ship.

### W6 — Do not build yet

| Idea | Why wait |
|---|---|
| LoRA adapters | Older runs silently dropped them on patched vLLM |
| Custom graph navigator | TF-IDF / grep beats embeddings for issue text; graphs have `calls` only, no async |
| ParallelAgent / 10-agent swarm | Tasks already sequential; extra loops burn the 8 min |
| 12B / E4B as scored model | Smaller ≠ more solves unless you measure it |
| Editing tests / adding repro files under `/workspace` | Stripped or pollute the patch; use `/tmp` |

---

## 6. Packaging notebook (how to change the agent)

All scored config lives in **one** code cell as a `FILES` dict, same as the
official samples.

When you change the agent:

1. Edit strings in `gemma-4-harness-fixed-single-agent.ipynb` (and copy the
   same dict into the eval notebook so they cannot drift).
2. CPU notebook, Internet off, Run All.
3. Confirm stdout:

```text
submission.zip: 4 files -> /kaggle/working/submission.zip
   agent.yaml
   eval_config.yaml
   configs/sampling.yaml
   prompts/system.md
```

(If skills exist, they appear in the list too.)

4. Submit that zip or the notebook version.

Inline asserts to keep:

- model string is `gemma-4-31b-it-qat-w4a16-ct`
- no `adapter:`
- `timeout_seconds: 300`, `max_tool_calls: 28` (or your measured values)
- no `{identifier}` in `prompts/system.md`
- zip names do not start with `submission/`

---

## 7. Prompt rules while you iterate

The harness already sends: problem statement, hints, budgets, workspace listing,
tool docs. Your system prompt only adds **policy**.

Must remain true:

- Edit a **source** file by call 12
- Scratch in `/tmp`, never `/workspace`
- No test files, no `pytest.ini` / `conftest.py`
- No bare `pytest`
- No `pip install`
- Graph tools only with a dotted symbol (`pkg.module.Class.method`)
- `submit_patch` last; `get_status` is free
- Incremental `edit_file` (short unique `old_string`) so `<|tool_call>` is not cut off
- Zero `{name}` placeholders

Do not paste JSON examples with braces into the prompt.

---

## 8. Tools the agent already has

You do not reimplement these. You teach when to use them.

| Tool | Counts as a call | Notes |
|---|---|---|
| `run_command` | yes | 5k char stdout/stderr; timeout = min(300s, remaining time) |
| `read_file` | yes | 150 lines **and** 10k chars |
| `edit_file` | yes | 3-tier match; fails if not unique |
| `write_file` | yes | new files only; `/workspace/repro.py` will enter the patch |
| `get_code_neighbors` | yes | symbol id, optional `edge_type=calls` |
| `search_similar_code` | yes | symbol name, not English |
| `get_code_subgraph` | yes | list of symbol ids |
| `get_status` | **no** | use before submit |
| `submit_patch` | **no** | ends the loop when the turn finishes |

If `submit_patch` is never called, the harness still grades `git diff` after
stripping tests.

---

## 9. Week plan (today → final)

Entry/team merge and final dates: confirm on the Rules page (notes had merge
2026-11-25, final 2026-12-02).

| When | Deliverable |
|---|---|
| Day 1 | Submit baseline zip; start W1 notebook |
| Days 2–4 | 8–12 task slice, bucket table |
| Week 2 | `repo-navigation` skill if bucket B dominates |
| Week 3 | `test-discovery` skill; retune minutes vs calls |
| Week 4 | Analyzer only if skills plateau; freeze two final selections |

---

## 10. Definition of done for a change

A change is shippable only if all of these are true:

- [ ] Packaging notebook Run All succeeds and lists expected zip paths
- [ ] Same `FILES` used in the eval notebook
- [ ] Fixed public slice re-run completed
- [ ] Resolve count did not drop; empty-patch count did not rise
- [ ] Worst-case `N * max_time_minutes` still fits ~11 hours with setup margin
- [ ] Prompt still has no `{placeholders}`
- [ ] One-sentence note of what changed (for the 1/day log)

If the slice did not finish, do not submit. A 12h DNF wipes the whole score.
