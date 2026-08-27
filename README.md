# Milkbasket GRN Scheduler

Pulls Milkbasket (Reliance) **GRN** documents out of Gmail every 3 hours, extracts the line items, and writes them to Google Sheets.

## Pipeline

```
Gmail  --->  Google Drive  --->  LlamaParse  --->  Google Sheets
```

| Stage | What happens |
|---|---|
| **1. Gmail -> Drive** | Searches mail from `DONOTREPLY@ril.com` matching `grn`, looking back 7 days, and saves each attachment to a Drive folder. Already-seen files are skipped. |
| **2. Extract** | Each new file is parsed by **LlamaParse** using the LlamaCloud extract agent `Milkbasket Agent`. |
| **3. Sheets** | Rows are appended to tab `mbgrn` of the tracking spreadsheet. |
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

The workflow base64-decodes the two Google secrets into `credentials.json` / `token.json` at the start of the run and deletes them in an `if: always()` cleanup step.

> [!WARNING]
> **`app.py` line 59 contains a hardcoded LlamaCloud API key** as a fallback in the `CONFIG`
> dict (`'llama_api_key': 'llx-...'`). It is redundant - the workflow already supplies
> `LLAMA_CLOUD_API_KEY` from the `LLAMA_API_KEY` secret. Replace the literal with
> `os.getenv('LLAMA_CLOUD_API_KEY', '')`, and rotate the key in LlamaCloud: deleting it
> from the file alone does not help, because it remains in git history.

## Running locally

```bash
pip install -r requirements.txt
# place credentials.json + token.json next to the script
python app.py   # schedule loop; Ctrl-C to stop
```

## Files

| File | Role |
|---|---|
| `app.py` | Gmail -> Drive -> extract -> Sheets, class `MilkbasketAutomation` |
| `.github/workflows/scheduler.yml` | 3-hourly Actions schedule |
| `requirements.txt` | dependencies |

## Adding a field

Add the field to the extract agent in LlamaCloud (`Milkbasket Agent`), then widen the header row on the `mbgrn` sheet tab.
