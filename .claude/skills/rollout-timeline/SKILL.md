---
name: rollout-timeline
description: Generate and view the Miles rollout timeline — parse train.log + Harbor trial dirs into JSON and serve rollout_timeline.html. Use when regenerating timeline data, adding a parsed field, debugging missing/blank trials, or when the page shows no data.
---

# Rollout timeline

Two files, one contract: `build_timeline_data.py` parses run outputs into JSON;
`rollout_timeline.html` renders only what that JSON contains. The viewer invents
nothing — if something is missing on the page, fix the parser, not the HTML.

## Regenerate and view

```bash
# JSON must land beside the HTML (the page fetches relative paths)
python build_timeline_data.py --work-dir /path/to/runtime/work --out-dir .
# or point at inputs directly:
python build_timeline_data.py --train-log <log> --trials-dir <dir> --out-dir .

python -m http.server 8000   # fetch() is blocked on file://
```

Then <http://localhost:8000/rollout_timeline.html>. Refresh after regenerating.

Defaults: `--work-dir` falls back to `$WORK_DIR`, else `$RUNTIME/work`, else a
hardcoded `/mnt/disk_nvme1n1/...` path. Always pass `--work-dir` or
`--train-log`/`--trials-dir` explicitly unless you know the default resolves.

## The data contract

`--out-dir` receives four JSON files plus `traj/`, all gitignored:

| File | Produced by | Read by |
| --- | --- | --- |
| `timeline_metrics.json` | `parse_step_metrics` | per-step metric panel |
| `per_step_trials.json` | `build_per_step` | ROLLOUT lane trial columns |
| `trial_phase_stats.json` | `build_trial_stats` | aggregate panels |
| `timeline_source.json` | `main` | provenance header |
| `traj/<trial>.json` | symlinks, in `main` | trajectory modal on click |

`traj/` entries are **symlinks** to the trial dirs, not copies. They break if the
trials directory moves or is deleted; re-run the generator to refresh them.

## Adding a parsed field

Parser and viewer must change together:

1. Extract it in a `_parse_*` helper (`_parse_trial_log_phases`,
   `_parse_verifier_slow_tests`, `_parse_tool_exec_times` are the models).
2. Add the key to the dict appended in `parse_trials` — **both** branches: the
   cancelled-trial branch and the normal branch. Missing it from the cancelled
   branch is the usual bug.
3. If the viewer needs it per step, also copy it through `build_per_step.record`
   — that function whitelists keys, so a field added only to `parse_trials` will
   silently not reach the page.
4. For `train.log` metrics, add the key to `WANT` (via `ROLLOUT_KEYS`/
   `AGENT_KEYS`/`TRAIN_KEYS`) or `parse_step_metrics` drops it.

Every parser helper swallows exceptions and returns `None`/`{}`/`[]` on failure.
That is deliberate — a malformed trial must not kill the run — but it means a
broken regex shows up as blank UI, never as a traceback. When a field is empty,
test the regex against a real line before touching anything else.

## Debugging empty or wrong output

The generator's stdout is the first diagnostic; read the counts it prints.

- **Page shows the "Serve over HTTP" error** — opened via `file://`, or the JSON
  isn't in the same directory as the HTML.
- **0 steps / 0 windows** — `perf N:` regex missed. `step_windows` is stricter
  than `parse_step_metrics`: it requires the full `[timestamp] metrics.py:NN -`
  log prefix, so a reformatted logger silently yields zero windows and every
  trial falls out of `build_per_step`.
- **Trials parsed but none on the timeline** — window assignment failed. Trials
  land on a step when `started_at ∈ [end − rollout_time, end]`, with
  `rollout_id` preferred when present. Timestamps are parsed as **naive local
  time** on both sides; a log written in another timezone misaligns everything.
- **`gpu_gen`/`container_cpu` null** — `session_id` comes from `lock.json`'s
  `OPENAI_API_BASE` (`.../sessions/<id>/v1`) matched against `GEN session_id=`
  lines. The generator prints how many trials got an exact `gpu_gen`; a low
  count means one side of that join is missing.
- **Trajectory modal 404s** — symlink export is unconditional, so there is no
  flag to re-run with. Either the trial has no `agent/*.trajectory.json` (e.g.
  cancelled mid-flight) or the symlink went stale because the trials dir moved;
  re-running the generator refreshes it.

Trajectories are read from `mini-swe-agent.trajectory.json`, falling back to
`trajectory.json`. Turn counts handle both shapes (`steps[].source == "agent"`,
else `messages[].role == "assistant"`).

## Constraints

- Python 3.8+, **standard library only** — no third-party imports.
- The HTML is a single self-contained file: vanilla JS, no build step, no
  dependencies. Keep it that way.
- Trajectory step timestamps are written at serialization time and are **not**
  real per-turn times. Do not derive per-turn durations from them; the real
  per-turn signal is `harbor_tool_exec_msec` / `harbor_llm_wait_msec`.
