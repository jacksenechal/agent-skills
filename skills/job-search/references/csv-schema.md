# Tracker CSV Schema

File: `/home/jack/workspace/job-search/tracker.csv`

## Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Slug identifier. Matches resume branch suffix (`job/<id>`) and research dir (`jobs/<id>/`). E.g., `stripe-infra-eng`, `navan-ai-ex` |
| `company` | string | yes | Company name |
| `role` | string | yes | Job title |
| `url` | string | yes | LinkedIn job posting URL |
| `stage` | enum | yes | Current pipeline stage (see below) |
| `resume_branch` | string | no | Git branch in resume repo, e.g. `job/stripe-infra-eng` |
| `role_branch` | string | no | Role archetype branch used as base, e.g. `role/ai-tooling-engineer` |
| `application_url` | string | no | Direct application URL (company careers site, Greenhouse, Lever, etc.) |
| `referral_contact` | string | no | Name(s) of connection(s) identified for referral |
| `referral_status` | enum | no | `none`, `identified`, `requested`, `received` |
| `date_found` | date | yes | ISO date when job was first added (YYYY-MM-DD) |
| `date_applied` | date | no | ISO date when application was submitted |
| `date_updated` | date | yes | ISO date of last update to this row |
| `notes` | string | no | Brief freeform notes |

## Pipeline Stages

Ordered progression:

1. `discovered` — URL added, nothing else done
2. `researched` — Job description scraped and saved
3. `resume_tailored` — Resume branch created and published
4. `application_prepped` — Application form reviewed, fields documented
5. `connections_found` — LinkedIn connections searched for referrals
6. `ready_to_apply` — Everything prepared, waiting for manual submission
7. `applied` — Application submitted (manual step by user)
8. `interviewing` — In interview process
9. `offer` — Received an offer (terminal)
10. `rejected` — Application rejected (terminal)
11. `withdrawn` — User withdrew application (terminal)

## CSV Handling Rules

1. **Always use Python's `csv` module** for reading and writing. Never use raw string manipulation — job descriptions and notes can contain commas, quotes, and newlines.

2. **Preserve field order** when writing. The header row defines the canonical order.

3. **Use `csv.DictReader` and `csv.DictWriter`** for clarity and safety.

4. **Set `date_updated`** to today on every write operation.

5. **Empty optional fields** should be empty strings, not "None" or "null".
