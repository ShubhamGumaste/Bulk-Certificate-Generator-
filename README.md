# Bulk Certificate Generator API

A backend that accepts a list of recipients in one request, generates a PDF
certificate for each valid one from a fixed template, tracks progress in a
relational database, and lets the client download the results.

**Stack:** Python 3.10+ (tested on 3.11 and 3.13), FastAPI, SQLAlchemy 2
(SQLite by default), ReportLab.

## Setup

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

## Run

```bash
uvicorn app.main:app --reload
```

Interactive docs: http://127.0.0.1:8000/docs

Tables are created on startup. Optional environment variables:

| Variable | Default | Purpose |
|---|---|---|
| `DATABASE_URL` | `sqlite:///./certificates.db` | Any SQLAlchemy URL |
| `STORAGE_DIR` | `./storage` | Where generated PDFs are written |
| `MAX_RECIPIENTS_PER_JOB` | `10000` | Largest accepted request |

## Tests

```bash
pytest
```

The tests use a temporary database and storage folder, so they never touch
real data.

## Submitting a generation request

`POST /jobs` with the certificate details and the recipient list
(`sample_request.json` is a ready-made example that includes some invalid
entries):

```bash
curl -X POST http://127.0.0.1:8000/jobs \
  -H "Content-Type: application/json" \
  -d @sample_request.json
```

```json
{
  "course_name": "Introduction to Python",
  "issuer_name": "Acme Academy",
  "issue_date": "2026-10-09",
  "signatory_name": "Dana Lee",
  "signatory_title": "Program Director",
  "recipients": [
    {"name": "Ada Lovelace", "email": "ada@example.com"},
    {"name": "Grace Hopper", "email": "not-an-email"}
  ]
}
```

`course_name`, `issuer_name` and `recipients` are required; `issue_date`
defaults to today. The response is `202 Accepted` with the job, including its
`id` and links to the endpoints below.

## Checking progress

```bash
curl http://127.0.0.1:8000/jobs/<job_id>
```

```json
{
  "id": "cb7b4d6e-…",
  "status": "completed_with_errors",
  "total": 5,
  "counts": {"pending": 0, "generated": 2, "failed": 0, "invalid": 3},
  "progress_percent": 100.0,
  "started_at": "2026-10-09T13:11:08.259457Z",
  "finished_at": "2026-10-09T13:11:08.297558Z"
}
```

Job status: `pending` → `processing` → `completed` (all generated),
`completed_with_errors` (some generated) or `failed` (none generated).

To see which recipients succeeded and which did not, and why:

```bash
curl "http://127.0.0.1:8000/jobs/<job_id>/certificates"
curl "http://127.0.0.1:8000/jobs/<job_id>/certificates?status=invalid"
```

Each entry has `position` (its index in the request), `status`, `error` and,
when generated, a `download_url`. The list is paginated with `limit` and
`offset`.

| Certificate status | Meaning |
|---|---|
| `pending` | Valid, waiting to be generated |
| `generated` | PDF is ready |
| `failed` | Data was valid but generation raised an error (can be retried) |
| `invalid` | Recipient data failed validation |

## Retrieving certificates

```bash
# One certificate
curl -OJ http://127.0.0.1:8000/certificates/<certificate_id>/download

# Every generated certificate of a job, as a ZIP (once the job has finished)
curl -OJ http://127.0.0.1:8000/jobs/<job_id>/download
```

## All endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/jobs` | Submit a request |
| GET | `/jobs/{id}` | Status, counts, progress |
| GET | `/jobs/{id}/certificates` | Per-recipient results |
| GET | `/jobs/{id}/download` | ZIP of generated certificates |
| POST | `/jobs/{id}/retry` | Re-run the `failed` certificates |
| GET | `/certificates/{id}` | One certificate's details |
| GET | `/certificates/{id}/download` | One certificate's PDF |
| GET | `/health` | Liveness check |

## Project layout

```
app/
  main.py        HTTP routes
  services.py    create job, generate certificates, counts, ZIP
  validation.py  per-recipient checks
  template.py    the certificate design (values in, PDF bytes out)
  models.py      Job and Certificate tables
  schemas.py     request/response shapes
  database.py    engine and sessions
  config.py      environment settings
  fonts/         DejaVu Sans, bundled for non-English names
tests/
```

## Design decisions

### Processing: background, in-process

`POST /jobs` validates, stores the job and its recipients, returns `202`, and
generation then runs in a FastAPI background task.

- **Why not synchronous:** generation takes about 12 ms per certificate here,
  so 5,000 recipients is roughly a minute. That is too long to hold an HTTP
  request open, and the client would see no progress.
- **Why not Celery/RQ:** a queue needs a broker and a separate worker process,
  which is a lot of setup for this scope. The trade-off is that work runs
  inside the web process, one job per worker thread.
- **What limits the trade-off:** all state is in the database, not in memory.
  The worker only picks up `pending` certificates and commits after each one,
  so on restart the app resumes unfinished jobs without regenerating what
  already exists. `services.process_job(job_id)` has no FastAPI dependency, so
  moving to a queue means calling that same function from a queue worker.
- **Assumption:** a single server process. With several processes, two could
  resume the same job at startup; that needs a row lock or a real queue.

### Two levels of validation

- **Request level (rejects with `422`, nothing stored):** malformed JSON,
  missing `course_name`/`issuer_name`, empty `recipients`, more recipients
  than the limit. If these are wrong, no certificate could be right.
- **Recipient level (job is still accepted):** missing or over-long name,
  control characters, missing or malformed email, an email repeated within the request. The entry
  is stored as `invalid` with the reason, and the others proceed. Rejecting
  5,000 recipients because of one typo would force the client to fix and
  resubmit everything.

Email is required and must be unique within a request: it is what tells two
people with the same name apart and stops a pasted-twice row from issuing two
certificates. The format check is intentionally loose (`x@y.z`).

A job whose recipients are all invalid is still created (and ends as `failed`)
so the client always gets the same kind of per-recipient report.

### Failure isolation

Each certificate is generated inside its own `try/except` and committed
separately. An error marks that one certificate `failed` with the error text
and the loop continues. `POST /jobs/{id}/retry` re-queues only the `failed`
ones; `invalid` entries are not retried because the input itself is wrong.

### Data model

Two tables: `jobs` (certificate details shared by the batch, status,
timestamps) and `certificates` (one row per submitted recipient, including
invalid ones). Job counts are computed with a `GROUP BY` over the certificate
rows instead of being stored as counters, so the summary cannot disagree with
the detail. There is an index on `(job_id, status)` for that query.

IDs are UUIDs, so download URLs cannot be guessed by counting. The certificate
ID is printed on the PDF and can be looked up at `/certificates/{id}`.

### Files

PDFs are written to `STORAGE_DIR/<job_id>/<certificate_id>.pdf`; the database
stores the relative path. Paths are built only from UUIDs, never from
user-supplied text. The readable download name (`00001_Ada_Lovelace.pdf`) is
derived at download time and stripped of unsafe characters. The ZIP is built
on request into a temporary file and deleted after it is sent.

### Template

One fixed landscape A4 design in `app/template.py`. Long names are scaled down
to fit and never cut off. DejaVu Sans is bundled so accented names render
correctly. The cost is that each PDF embeds part of the font: files are about
44 KB each (roughly 130 MB for 3,000 certificates), and embedding is most of
the generation time.

## Known limitations

- No authentication. Anyone who can reach the API can submit jobs, and anyone
  with a certificate ID can download that certificate.
- Scripts not covered by DejaVu Sans (Chinese, Japanese, Korean, Arabic
  shaping) will not render correctly; that needs a different bundled font.
- SQLite allows one writer at a time, which is fine for a single process.
  `DATABASE_URL` accepts another database such as PostgreSQL (install its
  driver), but only SQLite has been tested.
- Tables are created with `create_all`; there are no schema migrations.
- Generated files are kept indefinitely; there is no clean-up job.
- Duplicate emails are detected within one request, not across jobs.
- Retry is not guarded against two simultaneous calls for the same job.
