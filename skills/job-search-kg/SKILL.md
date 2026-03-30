---
name: job-search-kg
description: >
  LinkedIn connection knowledge graph for job search. Sets up ArcadeDB, ingests
  LinkedIn exports, and queries connection warmth scores for referral targeting.
  Sub-skill of job-search. Triggers on: "set up knowledge graph", "ingest linkedin data",
  "query connections", "warmth score", "job-search-kg setup", "arcadedb".
---

# LinkedIn Connection Knowledge Graph

A local ArcadeDB graph database that models your LinkedIn connections, work history,
and message history — so you can instantly find who to ask for a referral at any company.

## What it does

When you query a company name, it returns your 1st-degree connections currently employed
there, ranked by a **warmth score** derived from message history, recency, shared employers,
and tenure. This feeds the job-search skill's Stage 5 (connections) to prioritize outreach.

## File locations

All skill infrastructure lives in this directory:
- `docker/arcadedb/docker-compose.yml` — ArcadeDB container definition
- `scripts/ingest_linkedin.py` — one-time ingestion from LinkedIn CSV exports
- `scripts/query_connections.py` — query connections by company name

LinkedIn export CSVs go in your job-search repo at `data/linkedin/` (personal, not shared).

---

## Setup

### Step 1: Start ArcadeDB

```bash
cd ~/workspace/agent-tools/skills/job-search-kg/docker/arcadedb
docker compose up -d
```

Verify it's running:
```bash
curl -s http://localhost:2480/api/v1/server -u root:playwithdata | python3 -m json.tool
```

### Step 2: Export from LinkedIn

1. Go to LinkedIn → Settings & Privacy → Data Privacy → Get a copy of your data
2. Request **all** of: Connections, Messages, Positions (work history), Education
3. Wait for the email (usually a few minutes to a few hours)
4. Download and extract CSVs to your job-search repo's `data/linkedin/` directory

Expected files:
- `data/linkedin/Connections.csv` — your 1st-degree connections
- `data/linkedin/Messages.csv` — your message history
- `data/linkedin/Positions.csv` — your own work history (for shared-employer bonus)
- `data/linkedin/Education.csv` — optional, for school nodes

### Step 3: Ingest

Run from your job-search repo root (so relative paths to `data/linkedin/` resolve):
```bash
cd ~/workspace/job-search
python3 ~/workspace/agent-tools/skills/job-search-kg/scripts/ingest_linkedin.py --me-name "Your Full Name"
```

Replace `"Your Full Name"` with your name exactly as it appears in LinkedIn messages
(this is used to identify outbound messages in your export). Takes 2–5 minutes depending
on message volume. Clears and rebuilds the graph on each run.

---

## Schema

### Vertex types

| Type | Properties |
|---|---|
| `Person` | `name`, `url` (unique LinkedIn URL), `current_title`, `connection_date`, `warmth_score` |
| `Company` | `name` (unique) |
| `School` | `name` (unique) |

### Edge types

| Type | Direction | Properties |
|---|---|---|
| `CONNECTED_TO` | Me → Person | `source`, `date` |
| `WORKED_AT` | Person → Company | `title`, `is_current`, `start_date`, `end_date` |
| `MESSAGED` | Me → Person | `timestamp`, `direction` (Inbound/Outbound) |
| `STUDIED_AT` | Me → School | `degree`, `start_date`, `end_date` |

The `Me` vertex is a `Person` with `url = "me"`.

---

## Warmth Algorithm

Each 1st-degree connection receives a warmth score:

```
warmth = log(msg_count + 1) × 20       # message depth
       + 0.9^months_since_last_msg × 20 # recency decay (max 20 if messaged this month)
       + 15 (if shared company history) # alumni bonus
       + max(0, years_connected)        # tenure bonus (1 point per year)
```

Interpretation:
- **Score > 20**: Genuinely warm — real interaction history, high response likelihood
- **Score 10–20**: Mild warmth — some tenure or occasional contact; treat as a soft tie
- **Score < 10**: Cold in practice — 1st-degree in name only; treat like 2nd-degree for outreach

---

## Querying

### Query by company

Run from the job-search repo directory:
```bash
python3 ~/workspace/agent-tools/skills/job-search-kg/scripts/query_connections.py "Company Name"
```

Returns: name, warmth score, current title, LinkedIn URL — sorted by score descending.

**Note:** Only matches connections who list the company name exactly as it appears in their
LinkedIn profile. The live LinkedIn search in job-search Stage 5 is authoritative. This
query is used to add warmth context to results found there.

### Direct SQL/Cypher queries

```bash
# Count all persons in the graph
curl -s http://localhost:2480/api/v1/command/KnowledgeGraph \
  -u root:playwithdata \
  -H "Content-Type: application/json" \
  -d '{"language":"sql","command":"SELECT count(*) FROM Person"}'

# Find a person by name
curl -s http://localhost:2480/api/v1/command/KnowledgeGraph \
  -u root:playwithdata \
  -H "Content-Type: application/json" \
  -d '{"language":"cypher","command":"MATCH (p:Person) WHERE p.name CONTAINS \"Smith\" RETURN p.name, p.warmth_score, p.current_title"}'
```

---

## Sub-Commands

### `setup`

Walk through the full setup (Steps 1–3 above). Prompt the user at each step.

1. Check Docker is running: `docker info`
2. Start ArcadeDB: `docker compose up -d` in `docker/arcadedb/`
3. Verify API: curl check above
4. Prompt: "Have you downloaded your LinkedIn data export? Where are the CSV files?"
5. Move/copy CSVs to `~/workspace/job-search/data/linkedin/` if needed
6. Ask: "What is your full name as it appears in LinkedIn messages?"
7. Run ingestion
8. Verify with a count query

### `ingest`

Re-run ingestion from existing CSVs (e.g., after a fresh LinkedIn export):
```bash
cd ~/workspace/job-search
python3 ~/workspace/agent-tools/skills/job-search-kg/scripts/ingest_linkedin.py --me-name "Your Full Name"
```

Ask the user for their name if not known. Print the ingestion output.

### `query <company>`

Query connections at a company and display the table:
```bash
python3 ~/workspace/agent-tools/skills/job-search-kg/scripts/query_connections.py "<Company Name>"
```

### `status`

Verify ArcadeDB is running and the database is populated:
```bash
# Check container
docker ps --filter name=arcadedb

# Count persons
curl -s http://localhost:2480/api/v1/command/KnowledgeGraph \
  -u root:playwithdata \
  -H "Content-Type: application/json" \
  -d '{"language":"sql","command":"SELECT count(*) as total FROM Person"}'
```

---

## ArcadeDB Credentials

Default development credentials:
- **URL**: `http://localhost:2480`
- **Username**: `root`
- **Password**: `playwithdata`
- **Database**: `KnowledgeGraph`

Change the password for production use by updating `JAVA_OPTS` in the docker-compose.yml
and the `ARCADE_PASS` constant in both scripts.
