---
name: gaj:triage
description: |
  Batch-triage pipeline items or job alert digests using tiered AI models.
  Triggers on /gaj:triage, "triage my pipeline", "filter these jobs",
  "here's a digest", "LinkedIn alerts", "batch these jobs".
---

# gaj:triage - Batch pipeline triage

Score, filter, and surface signal from the pipeline or fresh job alert digests. Uses tiered models: Haiku for bulk classification, Sonnet for uncertain review, Opus for presentation and decisions.

## When to use

- User says "triage", "filter", "batch", "clean up my pipeline"
- User pastes a LinkedIn digest, job alert email, or list of jobs
- User has multiple pending-review items to evaluate at once

## Config

Read `~/gaj/gaj.json` at start. Required fields:

```json
{
  "models": {
    "triage": "claude-haiku-...",
    "triage_review": "claude-sonnet-..."
  },
  "blacklist": {
    "companies": ["string patterns"],
    "titles": ["string patterns"]
  },
  "profile": { ... },
  "compensation": {
    "w2_salaried": { "floor": 180000 }
  },
  "cli": {
    "working_directory": "/path/to/pipeline"
  }
}
```

If `gaj.json` is missing or fields are absent, use these defaults: triage model = haiku, triage_review model = sonnet.

## Step 0: Determine input source

**Pipeline mode (default):** Triage existing pending-review items.

```bash
cd "$working_directory" && npx tsx scripts/pipeline-cli.ts list pending-review
```

**Digest mode:** User has pasted a block of job listings. Extract each job, check against the blacklist before adding, add clean entries via CLI, then run pipeline mode on the newly added items.

If no items are found in pipeline mode, tell the user the pipeline is clean and exit.

## Step 1: Blacklist pass (no AI)

Check each item's `company_name` and `job_title` against `gaj.json blacklist.companies` and `blacklist.titles`. Matching is case-insensitive substring.

For each match, update the item status to `filtered-match` via CLI with a short reason (e.g., "blacklist: company pattern 'staffing'").

```bash
cd "$working_directory" && npx tsx scripts/pipeline-cli.ts status <id> filtered-match --reason "blacklist: <pattern>"
```

Report a summary: how many items matched per pattern. If nothing remains after this pass, skip to Step 5.

## Step 2: Haiku bulk classification

Split remaining items into batches of 25. One subagent per batch.

| Items | Subagents |
|-------|-----------|
| 1-25 | 1 |
| 26-50 | 2 |
| 51-75 | 3 |

Spawn subagents in parallel using `models.triage` (default: haiku). Each subagent receives:

- The jobs batch as a JSON array
- The user profile summary
- The scoring rubric

**Haiku subagent prompt template:**

```
You are a job triage classifier. Score each job against this candidate profile and return ONLY a JSON array. No markdown. No explanation. No wrapper text.

CANDIDATE PROFILE
Target roles: {profile.target_roles}
Preferred stack: {profile.preferred_stack}
Location preference: {profile.location_preference}
Comp floor (W2 salaried): ${compensation.w2_salaried.floor}

SCORING RUBRIC
Score each criterion 0-10, then compute weighted score:
  score = (role_fit*2 + stack*1 + company*1.5 + comp*1 + strategic*1.5) / 7

Criteria:
- role_fit (2x): Senior/staff level, AI/ML/LLM in scope, engineering not support. Junior/Mid in title = 0. On-site only = 2.
- stack (1x): TypeScript, Python, React, distributed systems, AI frameworks.
- company (1.5x): Known stage/funding/reputation. Unknown company = 5 (neutral). Known Series B+ with AI initiative = 8+.
- comp (1x): Above floor = 8+. Unstated = 5 (benefit of the doubt). Below floor = 0-3.
- strategic (1.5x): Resume impact, springboard potential, market positioning. European remote with real AI work gets a bonus.

VERDICT RULES
- score >= 7.0: verdict = "keep"
- score >= 4.5 and < 7.0: verdict = "uncertain"
- score < 4.5: verdict = "skip"

JOBS TO CLASSIFY
{jobs_json}

Return ONLY this JSON array, no other text:
[
  { "id": "<job_id>", "verdict": "keep|skip|uncertain", "score": 7.2, "reason": "one line explanation" }
]
```

After all subagents return, collect results. Update each SKIP item to status `filtered` with the subagent's reason:

```bash
cd "$working_directory" && npx tsx scripts/pipeline-cli.ts status <id> filtered --reason "<reason>"
```

## Step 3: Sonnet uncertain review

Collect all UNCERTAIN items from Step 2 into a single subagent call using `models.triage_review` (default: sonnet).

**Sonnet subagent prompt template:**

```
You are a senior job search advisor reviewing borderline job matches. The candidate has already been through a first-pass classifier. These items were flagged as uncertain. Make a final keep/skip call for each.

CANDIDATE PROFILE
Target roles: {profile.target_roles}
Preferred stack: {profile.preferred_stack}
Location preference: {profile.location_preference}
Comp floor (W2 salaried): ${compensation.w2_salaried.floor}

UNCERTAIN ITEMS (include Haiku's score and reason for context)
{uncertain_jobs_json}

Each item in uncertain_jobs_json includes:
- id, company_name, job_title, location, salary_range, source
- haiku_score: the first-pass score
- haiku_reason: the first-pass reasoning

Apply the same scoring rubric. Use your judgment on edge cases. If a role is ambiguous but has strong upside signal (e.g., unusual comp, rare company, direct inbound), lean keep.

Return ONLY this JSON array, no other text:
[
  { "id": "<job_id>", "verdict": "keep|skip", "score": 6.8, "reason": "one line final reasoning" }
]
```

Update SKIP items from this pass to `filtered` status. KEEP items remain in `pending-review` for Step 4.

## Step 4: Present results

Collect all KEEPs from Steps 2 and 3. Sort by score descending.

```
TRIAGE RESULTS
==============

| # | Score | Company | Role | Source | Signal |
|---|-------|---------|------|--------|--------|
| 1 | 8.4   | Acme AI | Staff AI Engineer | linkedin | Series B, remote, $200k |
| 2 | 7.1   | DataCorp | Senior ML Eng | email | F500, remote, unstated |

Filtered: 12 (8 blacklist, 3 Haiku skip, 1 Sonnet skip)
Kept for review: 2
```

Ask the user which items they want a full sherlock investigation on. Options: "all", specific numbers ("1, 3"), or "top N".

Wait for user input before proceeding.

## Step 5: Archive sweep

After the user has made their selections (or if there was nothing to keep), run the archive command to move stale closed items out of the active view.

```bash
cd "$working_directory" && npx tsx scripts/pipeline-cli.ts archive
```

Report final counts: active, filtered, archived.

---

## Digest extraction (digest mode only)

When the user pastes a block of content rather than triggering pipeline mode:

1. Parse the input. Extract every distinct job listing. Look for: job title, company name, location, salary (if stated), URL or source.
2. Present the raw extraction as a numbered list. Ask the user to confirm before adding anything.
3. Check each extracted job against the blacklist before adding. Skip any blacklist matches and note them.
4. Add clean jobs via CLI:

```bash
cd "$working_directory" && npx tsx scripts/pipeline-cli.ts add '<json>'
```

5. Run pipeline mode on the newly added items (they will be in `pending-review`).

Supported digest formats:
- LinkedIn job alert email (3-5 jobs with title, company, location, optional salary)
- LinkedIn recruiter digest (multiple recruiter messages, each different role)
- Forwarded email body
- Pasted plain list of companies and roles
- URLs only (ask user to paste JD text if URLs cannot be fetched)

---

## Scoring rubric

| Criterion | Weight | What it measures |
|-----------|--------|-----------------|
| Role fit | 2x | Senior/staff level, AI/ML/LLM in scope, engineering not support |
| Stack match | 1x | TypeScript, Python, React, distributed systems, AI frameworks |
| Company quality | 1.5x | Known stage, funding, reputation |
| Comp signal | 1x | Above floor, or unstated |
| Strategic value | 1.5x | Resume impact, springboard potential, market positioning |

Formula:

```
score = (role_fit*2 + stack*1 + company*1.5 + comp*1 + strategic*1.5) / 7
```

Scoring guidelines:
- Unstated salary: 5/10 (benefit of the doubt, will verify later)
- Junior or Mid in title: 0/10 on role fit
- On-site only, no relocation interest: 2/10 on role fit
- Known Series B+ or F500 with AI initiative: 8/10 on company quality
- Unknown company: 5/10 on company quality (neutral, not penalized)
- European remote with real AI work: strategic value bonus (thinner talent pool, easier placement)

---

## CLI reference

```bash
# List items by status
npx tsx scripts/pipeline-cli.ts list <status>

# Update item status
npx tsx scripts/pipeline-cli.ts status <id> <new-status> --reason "<reason>"

# Add a new item
npx tsx scripts/pipeline-cli.ts add '<json>'

# Archive stale closed items
npx tsx scripts/pipeline-cli.ts archive

# Search by keyword
npx tsx scripts/pipeline-cli.ts search '<query>'

# Pipeline stats
npx tsx scripts/pipeline-cli.ts stats
```

Valid statuses: `pending-review`, `active`, `filtered`, `filtered-match`, `applied`, `interviewing`, `offer`, `closed`, `archived`.
