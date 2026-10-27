# Gemma 4 Developer Agent

Kaggle submission for [Google – The Gemma 4 Developer Agent Competition](https://www.kaggle.com/competitions/gemma-4-developer-agent).

The scorer does **not** run your training code. It compiles `submission.zip` (`agent.yaml` at the zip root) and runs that agent on hidden issues. The host already provides data, wheels, graphs, snapshots, and `sample_submission` as notebook inputs.

**Submit one notebook.** Same pattern as the official starter and the budget-fit samples.

## Notebook to run

`src/sample/gemma-4-harness-fixed-single-agent.ipynb`

1. Create a Kaggle notebook on this competition (CPU is enough).
2. Attach the competition data (already in your screenshot: `tasks.jsonl`, `sample_submission`, `wheels`, …). This notebook does not read those files; they can stay attached.
3. Replace the default cells with this notebook, or copy it in.
4. **Internet off. Run All.**
5. Submit `/kaggle/working/submission.zip` (or Save Version and submit the notebook output).

The one code cell writes four files into the zip:

```text
agent.yaml
eval_config.yaml
configs/sampling.yaml
prompts/system.md
```

It also checks: 31B model, no LoRA, 300s / 28 calls / 8 min, no `{placeholders}` in the prompt, `agent.yaml` at zip root.

## How the agent works

Kaggle starts Gemma 4 **31B** (`gemma-4-31b-it-qat-w4a16-ct`) on 4× L4 GPUs. For each hidden bug:

1. **Container A** — repo at `/workspace`. The model uses tools (`run_command`, `read_file`, `edit_file`, …).
2. Patch = `git diff`. Test-file edits are stripped. If `submit_patch` is never called, a non-empty source diff is still graded.
3. **Container B** — fresh copy + hidden tests. Score 1 only if those tests pass.

The system prompt forces: locate in calls 1–8, **edit source by call 12**, targeted pytest only, scratch files in `/tmp`, `submit_patch` last. `submit_patch` and `get_status` are free (do not count toward 28 calls).

Whole run is **12 hours, sequential**. Default 60 min/task would time out the submission. 28 calls stop tasks that are still exploring so 8 minutes/task can still fit.

| Setting | Value |
|---|---|
| Thinking | on, budget 4096, `include_thoughts: true` (no `thinking_level`) |
| `max_output_tokens` | 8192 (stay under vLLM 32k) |
| Phase 1 command timeout | 300 s (hidden pytest on the scorer is separately fixed at 300 s) |
| Graph tools | declared; prompt uses them only after a dotted symbol path |

## Testing

No extra Python tests. The notebook **is** the check: Run All must print four zip entries and not raise.

To try the same cell locally:

```bash
# from this repo, after opening the notebook in Jupyter, Run All
# or paste the code cell into python; it writes submission.zip in the current directory
```

On Kaggle, a successful run prints:

```text
submission.zip: 4 files -> /kaggle/working/submission.zip
   agent.yaml
   eval_config.yaml
   configs/sampling.yaml
   prompts/system.md
```

## Other files in `src/sample/`

| File | Use |
|---|---|
| `gemma-4-harness-fixed-single-agent.ipynb` | **This submission** |
| `getting-started-gemma-4-developer-agent.ipynb` | Official runtime: vLLM + Evaluator on 1–2 public tasks (needs GPU) |
| `gemma-4-budget-aware-coding-agent-v2.ipynb` | Earlier prompt draft |
| `gemma-4-agent-budget-fit-single-agent.ipynb` | Earlier budget draft |
| `HARNESS_README.md` | Official harness contract |

Use getting-started only if you want to **smoke-test** the agent on public tasks. Scoring still only needs the zip from the harness-fixed notebook.

## Submit

1. Accept competition rules.
2. Run All on `gemma-4-harness-fixed-single-agent.ipynb`.
3. Upload `submission.zip`, or submit the notebook version.
4. Wait up to 12 hours. **1 submission per day.**
