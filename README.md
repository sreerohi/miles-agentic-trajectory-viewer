# rollout-timeline

Turn a Miles multi-turn RL training log plus its Harbor trial directories into
an interactive HTML timeline of every rollout step and every agent trial.

The timeline shows, per training step, a **ROLLOUT lane** (per-trial spawn /
install / agent / verifier phases, GPU-generation vs container-CPU split, turns,
reward) and a **TRAINER lane** (train and train-wait time), with click-through to
each trial's full agent trajectory.

## Contents

| File | Role |
| --- | --- |
| `build_timeline_data.py` | Parser/generator. Reads the training log + trial dirs, writes the JSON the page needs. |
| `rollout_timeline.html` | Static viewer. Loads the generated JSON from its own directory and renders the timeline. |

The HTML invents nothing — it only displays what the generator extracts.

## How it works

```
 train.log ──┐
             ├─▶  build_timeline_data.py  ──▶  timeline_metrics.json
 trials/   ──┘                                 per_step_trials.json
                                               trial_phase_stats.json
                                               timeline_source.json
                                               traj/<trial>.json  (symlinks)
                                                        │
                                               rollout_timeline.html  (reads them)
```

### Inputs

1. **Miles `train.log`** — the trainer/rollout/session-server stdout. The generator reads:
   - `perf N:` / `rollout N:` metric dicts → per-step metrics
   - rollout-side `perf N:` timestamps → each step's `[end − rollout_time, end]` wall-clock window
   - `GEN session_id=<hex> gen_time=<s>s` lines → GPU generation time per session
2. **Harbor trials directory** — one subdirectory per trial. Per trial it reads:
   - `result.json` → phase timings (`environment_setup`, `agent_setup`, `agent_execution`, `verifier`), start/finish, verifier reward
   - `lock.json` → `session_id` (to split agent time into GPU-gen vs container-CPU)
   - `trial.log` → fine-grained phase markers; also the start time for cancelled trials
   - `verifier/test-stdout.txt` → pytest `--durations=0` slow-test table
   - `agent/*.trajectory.json` → turn count and tool/LLM wait times

### Outputs (written to `--out-dir`)

| File | Contents |
| --- | --- |
| `timeline_metrics.json` | Per-step perf / agent / rollout / train metrics |
| `per_step_trials.json` | Each step's trials mapped into its window (phases, reward, turns, offset) |
| `trial_phase_stats.json` | Run-wide per-phase duration distribution (min/p50/p90/max/mean/n) |
| `timeline_source.json` | Provenance: log mtime, parse time, step count |
| `traj/<trial>.json` | Symlinks to each trial's trajectory, fetched on click |

## Usage

### 1. Generate the data

Point `--out-dir` at the directory that holds `rollout_timeline.html` so the JSON
and `traj/` land right beside it:

```bash
python build_timeline_data.py \
  --train-log /path/to/train.log \
  --trials-dir /path/to/trials \
  --out-dir .
```

Or, if your run keeps `train.log` and `cc_trials/` under one runtime work dir:

```bash
python build_timeline_data.py --work-dir /path/to/runtime/work --out-dir .
```

The script prints how many steps, sessions, and trials it parsed, then writes the
four JSON files and the `traj/` symlinks.

### 2. View the timeline

The page uses `fetch`, so it must be served over HTTP (opening the file directly
will be blocked by the browser). From this directory:

```bash
python -m http.server 8000
```

Then open <http://localhost:8000/rollout_timeline.html>. Refresh the page after
re-running step 1 to pick up new data.

## Requirements

- Python 3.8+ (standard library only — no third-party packages)
- A browser, to view the generated page
