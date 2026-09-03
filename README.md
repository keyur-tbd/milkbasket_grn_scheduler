# Milkbasket GRN Scheduler

Pulls Milkbasket (Reliance) **GRN** documents out of Gmail every 3 hours, extracts the line items, and writes them to Google Sheets **and Supabase**.

## Pipeline

```
Gmail  --->  Google Drive  --->  LlamaParse  --->  Google Sheets
                           \--->  LlamaExtract --->  Supabase
```

| Stage | What happens |
|---|---|
| **1. Gmail -> Drive** | Searches mail from `DONOTREPLY@ril.com` matching `grn`, looking back 7 days, and saves each attachment to a Drive folder. Already-seen files are skipped. |
| **2. Extract** | Each new file is parsed by **LlamaParse** using the LlamaCloud extract agent `Milkbasket Agent`. |
| **3. Sheets** | Rows are appended to tab `mbgrn` of the tracking spreadsheet. |
| **4. Supabase** | `supabase_sink.py` re-reads the same Drive files and upserts the rows into the `milkbasket_grn` table. Idempotent: rows are keyed by a `row_hash`, so re-runs update rather than duplicate. |
| **5. Run log** | A per-run summary is appended to tab `mb_workflow_logs`. |

## Schedule and entry points

Runs on GitHub Actions via `.github/workflows/scheduler.yml`, cron `0 */3 * * *` (every 3 hours). Can also be triggered manually with **Run workflow**.

The workflow deliberately does **not** call `main()`. `app.py` uses the `schedule` library for standalone local use, and that loop would sit idle burning runner minutes. Actions instead invokes `MilkbasketAutomation.run_scheduled_workflow()` directly.

Recent runs average **~3 minutes**.

## Required secrets

Set under **Settings -> Secrets and variables -> Actions**:

| Secret | Purpose |
|---|---|
| `GOOGLE_CREDENTIALS` | base64 of `credentials.json` (Google OAuth client) |
| `GOOGLE_TOKEN` | base64 of `token.json` (authorized refresh token) |
| `LLAMA_API_KEY` | LlamaCloud key, exported to the app as `LLAMA_CLOUD_API_KEY` |
| `SUPABASE_URL` | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | service_role key - bypasses RLS so writes succeed |

The workflow base64-decodes the two Google secrets into `credentials.json` / `token.json` at the start of the run and deletes them in an `if: always()` cleanup step.

> [!WARNING]
> **`app.py` line 59 contains a hardcoded LlamaCloud API key** as a fallback in the `CONFIG`
> dict (`'llama_api_key': 'llx-...'`). It is redundant - the workflow already supplies
> `LLAMA_CLOUD_API_KEY` from the `LLAMA_API_KEY` secret. Replace the literal with
> `os.getenv('LLAMA_CLOUD_API_KEY', '')`, and rotate the key in LlamaCloud: deleting it
> from the file alone does not help, because it remains in git history.

## Running locally

```bash
py -3.12 -m venv .venv     # 3.12 or older: llama-cloud-services breaks on 3.13+
.venv/Scripts/pip install -r requirements.txt
# place credentials.json + token.json next to the script, and create .env
python app.py   # schedule loop; Ctrl-C to stop
```

The Supabase half has its own checks, in increasing order of commitment:

```bash
python supabase_sink.py --check       # config + connectivity + tables exist
python supabase_sink.py --self-test   # insert/read/delete a synthetic row
python supabase_sink.py --run --dry-run --limit 2
python supabase_sink.py --run         # for real
```

## Files

| File | Role |
|---|---|
| `app.py` | Gmail -> Drive -> extract -> Sheets, class `MilkbasketAutomation` |
| `supabase_sink.py` | Drive -> extract -> Supabase. **Identical in every scheduler repo**; only `.env` differs |
| `.github/workflows/scheduler.yml` | 3-hourly Actions schedule |
| `requirements.txt` | dependencies |
| `.env` | `GRN_SOURCE=milkbasket` plus Supabase credentials. Gitignored |

## Adding a field

Add the field to the extract agent in LlamaCloud (`Milkbasket Agent`), then widen the header row on the `mbgrn` sheet tab.

For Supabase, a new key already lands in `raw_data` and is queryable as
`raw_data->>'key'` - a new field never fails a run. To promote it to a real
typed column, add it to the `milkbasket` entry in `SOURCES` in
`supabase_sink.py`, run `python supabase_sink.py --print-schema`, and apply the
`alter table` it emits.

## Disk guard (shared across every pipeline)

This scheduler writes to a Supabase volume shared with the Business Central
sync and the marketplace/ads loaders. Before it writes, it asks the database
whether it is allowed. **If you get an email titled `[WARN]` or `[STOP]
Supabase disk`, start here.**

```sql
-- the GRN schedulers genuinely need more room, and the volume has space:
UPDATE etl_disk_policy SET budget_gb = 8 WHERE pipeline = 'grn';

-- you resized the Supabase volume (do this EVERY time you resize):
UPDATE etl_disk_policy SET budget_gb = 100 WHERE pipeline = '_disk';

-- someone else should get the emails:
UPDATE etl_alert_config SET recipients = ARRAY['birbal@thebakersdozen.in'];
```

All thirteen GRN schedulers share one `grn` budget, because they are one
workload from the volume's point of view. A `[STOP]` means this scheduler is
refusing to write until you do one of the above. Nothing is lost: it stops
before writing and the next run continues.

`etl_alerts.py` is **identical in every pipeline repo** - do not add per-repo
logic to it. Everything configurable lives in Postgres (`etl_disk_policy`,
`etl_alert_config`), so budgets, thresholds and recipients change with an
`UPDATE` and no deploy, for all pipelines at once.

Three behaviours worth knowing:

- **No new credentials were needed.** This repo has no Postgres driver and no
  DSN, so the guard reaches the policy through PostgREST RPC using the
  `SUPABASE_URL` + service-role key it already holds.
- **It fails OPEN.** If the guard cannot run - credentials missing, database
  unreachable - it logs an error and lets the scheduler continue. A guard that
  breaks a working pipeline is worse than one that cannot check. Grep for
  `Disk guard could not run`.
- **Budgets grow themselves** into genuinely unallocated volume space, so a
  pipeline that is legitimately growing is not blocked by a number somebody
  guessed months ago. It can never grow past the volume ceiling.

Full documentation:
https://github.com/keyur-tbd/bc-supabase-sync#disk-alerts-and-auto-budgeting---start-here-if-you-got-an-email
