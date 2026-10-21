# xhs_coding_eval_dataset

Coding-agent evaluation set with its rubrics and harness. **19 cases** in three groups:

| Group | Cases | Purpose |
|---|---|---|
| **C6 — discriminative (hill climb)** | 7 | Hard tasks that separate strong from weak agents |
| **C7 — daily work** | 11 | Everyday backend bug fixes and feature work |
| **C9 — customer-provided** | 1 | A real customer brief (single-file web app), graded with hidden browser tests |

Each case runs in its source benchmark's **exact container image**, with the original hidden tests:

- **D1:** pass rate on the hidden tests, over 3 trials per model.
- **D2:** code quality from an LLM judge (Claude Opus 5.5): Readability, Maintainability and Robustness, plus 1–2 case-specific items.

D1 and D2 are reported side by side and never combined.

## Cases

| ID | Source / task | Grader | Agent budget |
|---|---|---|---|
| C6-01 | Terminal-Bench 2.0 / filter-js-from-html | tb2_pytest | 30 min |
| C6-02 | DeepSWE v1.1 / pebble-durability-wait-apis | deepswe | 180 min |
| C6-03 | Terminal-Bench 2.0 / video-processing | tb2_pytest | 60 min |
| C6-04 | Terminal-Bench 2.0 / mteb-retrieve | tb2_pytest | 30 min |
| C6-05 | Terminal-Bench 2.0 / polyglot-c-py | tb2_pytest | 15 min |
| C6-06 | Terminal-Bench 2.0 / cancel-async-tasks | tb2_pytest | 15 min |
| C6-07 | DeepSWE v1.1 / query-persist-restored-query-state | deepswe | 180 min |
| C7-01 | DeepSWE v1.1 / drizzle-orm-window-function-builders | deepswe | 180 min |
| C7-02 | Terminal-Bench 2.0 / db-wal-recovery | tb2_pytest | 15 min |
| C7-03 | SWE-bench Verified / django__django-15280 | swebench | 30 min |
| C7-04 | SWE-bench Verified / django__django-12965 | swebench | 30 min |
| C7-05 | SWE-bench Verified / django__django-15503 | swebench | 30 min |
| C7-06 | SWE-bench Verified / django__django-14376 | swebench | 30 min |
| C7-07 | SWE-bench Verified / django__django-11532 | swebench | 30 min |
| C7-08 | SWE-bench Verified / psf__requests-6028 | swebench | 30 min |
| C7-09 | DeepSWE v1.1 / kgateway-consistent-hash-policy | deepswe | 180 min |
| C7-10 | DeepSWE v1.1 / onedump-dump-encryption-pipeline | deepswe | 180 min |
| C7-11 | DeepSWE v1.1 / geo-shapeindex-serialization | deepswe | 180 min |
| C9-01 | Customer-provided / 潮玩零售店工作台 (single-file HTML: products, orders, after-sales) | tb2_pytest (Playwright) | 60 min |

## Layout

```
dataset/
  c6/ c7/ c9/            case entries (C*-NN_*.json) + envs/<ID>/ (environment, hidden tests, reference solution)
  manifest.json          case index
evaluation/
  run_benchmark.py       entry point: --category c6|c7|c9 --entry <ID> --agent agy|claude|gold|empty --model ...
  env_runner.py          sandboxes (udocker | docker | bwrap), agent isolation, graders, trial capture
  quality_grader.py      D2 LLM judge + validity audit (leak / tampering detection)
  swebench_grade.py      SWE-bench grading (runs in the swebench==4.1.0 venv)
  rubrics/               coding_quality.json (3 common items), c6/c7/c9 category rubrics, tasks/<ID>.json (case items)
tools/
  run_queue.sh, run_all.sh   local queues (udocker; 4 trials at a time on a workstation)
  cloud/                     GCP Batch runner: xhs_cloud.py (bake/run/status/collect/down/gc), worker.py, bake.sh
  campaign_report.py ...     reports (D1/D2, item scores, time/tool calls/tokens)
guide/                   entry format, evaluation guide, rubric, container integration
results/id_mapping.json  maps IDs recorded by older runs to the current IDs (reports use it)
```

**Not included:** run results and trajectories, and the C1–C5 / C8 categories of the original benchmark. The guides still mention them in places.

## Quick start

Setup (udocker, local): see [tools/README.md](tools/README.md) → "Setup on a fresh machine". Smoke test (no agent):

```bash
AGENT=gold  MODELS=x TRIALS=1 QUALITY=off bash tools/run_queue.sh 2 30 keep c9:C9-01 c6:C6-05   # expect 1
AGENT=empty MODELS=x TRIALS=1 QUALITY=off bash tools/run_queue.sh 1 30 keep c9:C9-01           # expect 0
```

One case, one agent:

```bash
python3 evaluation/run_benchmark.py --category c7 --entry C7-04 --agent claude --model claude-opus-5-5 \
    --sandbox udocker --trials 3 --quality full
```

All 19 cases on GCP (one Batch task per trial, Docker workers, automatic teardown):

```bash
python3 tools/cloud/xhs_cloud.py bake          # once: worker image with tools + all task images
python3 tools/cloud/xhs_cloud.py run --cases all --agent agy --models gemini-3.8-flash-high --trials 3 \
    --machine-type e2-standard-16
```

On GCP, a full campaign (22 cases × 4 models × 3 trials) took about 2.5 hours; the longest trials set the total time.

## Licensing / provenance

The cases are derived from public benchmarks. Their original terms apply:

- [Terminal-Bench 2.0](https://github.com/laude-institute/terminal-bench-2): its tasks carry a canary string and must not be used as training data.
- [SWE-bench Verified](https://www.swebench.com/)
- DeepSWE v1.1
- C9-01 is a customer-provided brief.

Each entry records its source and original task ID (`maps_to_journey`, `source`).
