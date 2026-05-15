# Batch Processing

Process multiple job offers in parallel via headless workers. Each worker runs the full evaluation pipeline (A-F report + PDF + tracker line) autonomously.

The standalone runner defaults to Codex workers using your local ChatGPT OAuth login. Run `codex login` once, then `codex login status` should report a ChatGPT login. Claude Code remains available with `--cli claude`.

## Quick Start

1. **Add offers** to `batch-input.tsv` (tab-separated: `id`, `url`, `source`, `notes`):

   ```tsv
   id	url	source	notes
   1	https://jobs.example.com/role-a	LinkedIn	
   2	https://greenhouse.io/company/role-b	Greenhouse	priority
   ```

2. **Dry run** to preview what will be processed:

   ```bash
   ./batch/batch-runner.sh --dry-run
   ```

3. **Run the batch with Codex OAuth**:

   ```bash
   ./batch/batch-runner.sh
   ```

   To use Claude Code workers instead:

   ```bash
   ./batch/batch-runner.sh --cli claude
   ```

4. **Results** are automatically merged into `data/applications.md` and verified with `verify-pipeline.mjs` at the end of the run.

## Options

| Flag | Default | Description |
|------|---------|-------------|
| `--cli CLI` | `codex` | Worker CLI: `codex` or `claude` |
| `--parallel N` | `1` | Number of concurrent headless workers |
| `--dry-run` | off | Preview pending offers without processing |
| `--retry-failed` | off | Only retry offers marked as `failed` in state |
| `--start-from N` | `0` | Skip offers with ID below N |
| `--max-retries N` | `2` | Max retry attempts per offer before giving up |

## Codex Settings

The runner keeps Codex in workspace-write mode, enables web search, and allows worker shell network access so workers can fetch job pages and compensation data.

| Environment variable | Default | Description |
|----------------------|---------|-------------|
| `CAREER_OPS_CODEX_MODEL` | unset | Optional Codex model override |
| `CAREER_OPS_CODEX_REQUIRE_OAUTH` | `true` | Require `codex login` with ChatGPT OAuth |
| `CAREER_OPS_CODEX_SEARCH` | `true` | Enable Codex web search for workers |
| `CAREER_OPS_CODEX_NETWORK_ACCESS` | `true` | Allow worker shell network access |
| `CAREER_OPS_CODEX_EPHEMERAL` | `true` | Avoid persisting worker sessions |

## Directory Layout

```
batch/
  batch-runner.sh          # Orchestrator script
  batch-prompt.md          # Prompt template sent to each worker
  batch-input.tsv          # Input offers (you create this)
  batch-state.tsv          # Processing state (auto-managed, resumable)
  logs/                    # Per-offer worker logs ({report_num}-{id}.log)
  tracker-additions/       # TSV lines produced by workers
    merged/                # TSVs already merged into applications.md
```

## How It Works

1. **batch-runner.sh** reads `batch-input.tsv` and `batch-state.tsv` to determine which offers need processing.
2. For each pending offer, it assigns a report number and launches a headless worker with `batch-prompt.md` as the worker prompt (placeholders like `{{URL}}`, `{{REPORT_NUM}}` are resolved).
3. Each worker evaluates the offer, writes a report to `reports/`, generates a PDF to `output/`, and writes a tracker TSV to `tracker-additions/`.
4. After all workers finish, batch-runner calls `merge-tracker.mjs` to merge TSVs into `data/applications.md` and runs `verify-pipeline.mjs` to check integrity.

## Tracker Merge

Workers write one TSV per offer to `batch/tracker-additions/`. The merge script (`npm run merge`) handles:

- Deduplication by company + role fuzzy match and report number
- Column order conversion (TSV has status before score; applications.md has score before status)
- In-place updates when a re-evaluation scores higher than the existing entry
- Moving processed TSVs to `tracker-additions/merged/`

Run `npm run merge` manually if you need to merge outside of a batch run.

## Resumability

`batch-state.tsv` tracks the status of every offer (`pending`, `processing`, `completed`, `failed`). If the batch is interrupted, re-running `batch-runner.sh` picks up where it left off -- completed offers are skipped automatically.

A PID-based lock file (`batch-runner.pid`) prevents concurrent batch runs. If a previous run crashed, the stale lock is detected and removed automatically.

## Prerequisites

- Codex in PATH and logged in with ChatGPT OAuth (`codex login`), or Claude Code in PATH when using `--cli claude`
- Node.js >= 18, Playwright chromium installed (`npm run doctor` to verify)
- `batch-input.tsv` with at least one offer
