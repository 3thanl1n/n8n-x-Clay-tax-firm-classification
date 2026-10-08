# Supabase Reference

## Overview
Supabase provides hosted Postgres databases with connection pooling via Supavisor (PgBouncer replacement). Used in GTM pipelines as the data source for ICP filtering queries.

## Initial Setup

### Creating a Project
1. Go to supabase.com → New Project
2. Set a database password (SAVE IT — not viewable after creation)
3. Choose a region close to your n8n instance
4. Wait for provisioning (~2 minutes)

### Uploading Data
- Use the Table Editor UI to import CSV files
- For files > 50MB on free tier: use the Supabase API to upload via Python scripts in batches
- Column types are auto-detected from CSV but verify after import (especially numeric vs text)

## Connection Pooling (Required for n8n)

### Why Pooling Matters
The free tier allows 15 direct connections. n8n batch workflows can exhaust this in seconds. The pooler multiplexes connections so n8n only uses 1-2 actual database connections regardless of batch size.

### Enabling Pooling
Navigate to: **Settings → Database → Connection Pooling section**
- Pool size: defaults to 15 (Nano tier)
- Max client connections: 200 (fixed on Nano)
- These are already enabled by default — you just need to use the right host

### Finding the Pooler Connection String
1. Click the green **"Connect"** button (top right of dashboard)
2. Select the **"Direct"** tab (NOT "Framework")
3. The connection string shows the pooler host, port, user, and database

### Connection Parameters for n8n

| Parameter | Value |
|---|---|
| Host | `aws-0-{region}.pooler.supabase.com` |
| Port | `6543` |
| Database | `postgres` |
| User | `postgres.{project-ref}` |
| Password | Your database password |
| SSL | ON |

**Common mistakes:**
- Using `db.{ref}.supabase.co` (direct host) → works but exhausts connections during batch runs
- Using `postgres` as user instead of `postgres.{project-ref}` → authentication fails on pooler
- Using port `5432` → connects to direct, not pooler
- Forgetting SSL → connection refused

### Finding Your Project Reference
The project ref is the long string in your Supabase URL:
`https://supabase.com/dashboard/project/{PROJECT_REF}`
It's also in the pooler host: `aws-0-us-east-1.pooler.supabase.com` and the user: `postgres.{PROJECT_REF}`

### Resetting Password
Settings → Database → "Reset database password" button. Warning: breaks existing connections.

## SQL Patterns

### ICP Filtering with Multi-Year Averaging
```sql
SELECT 
  ein, legal_name, dba_name, city, state,
  ROUND(AVG(total_returns)::numeric, 0) AS avg_returns_2yr
FROM irs_filings
WHERE filing_year >= (EXTRACT(YEAR FROM CURRENT_DATE) - 2)
GROUP BY ein, legal_name, dba_name, city, state
HAVING AVG(total_returns) >= {{ $json.min_returns_threshold || 400 }}
ORDER BY avg_returns_2yr DESC
LIMIT 1000;
```

**Notes:**
- `{{ $json.min_returns_threshold || 400 }}` is n8n expression syntax — sets a default of 400 if no input
- `::numeric` cast is required for ROUND in Postgres
- `EXTRACT(YEAR FROM CURRENT_DATE)` dynamically uses current year
- Scale thresholds to dataset: 400 for 287 firms, 1000-10000 for 500k firms

### Data Schema Design
For IRS e-file provider data, explode multi-year columns into rows:
```
ein | legal_name | dba_name | city | state | zip_code | filing_year | total_returns | accepted_returns | rejected_returns
```
One row per firm per year. This enables AVG aggregation across years.

## Free Tier Limits
- Database size: 500MB
- Bandwidth: 5GB
- Projects: 2 active
- Connection pool: 15
- For larger datasets: upload via API in chunks to bypass the 50MB CSV import limit
