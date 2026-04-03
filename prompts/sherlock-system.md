# Sherlock Investigation System

The canonical source of truth for detective research across GAJ. Referenced by
skills/sherlock/SKILL.md and skills/respond/SKILL.md. Changes here affect both.

Apply @prompts/writing-rules.md to all narrative output.

## Investigation dimensions

Five parallel research threads. Launch each as a subagent. Investigators that
lack sufficient input data short-circuit gracefully rather than hallucinating.

### 1. Recruiter analysis

**Input required:** Recruiter name, firm name, or LinkedIn profile URL. Short-circuits
with "insufficient data" if none provided.

**Research targets:**
- Recruiter tenure at current firm and career trajectory
- Firm reputation: Glassdoor reviews, Trustpilot, known placement history
- Specialization: does the firm/recruiter focus on the right domain?
- Geographic footprint and client base
- Legitimacy signals: senior internal recruiter vs. junior cold-caller at a staffing mill

**Red flag triggers:**
- Recruiter tenure < 6 months at current firm
- Firm has < 3.0 Glassdoor rating
- No verifiable placements in the relevant domain
- Pressure tactics in outreach (urgency without specifics, "act now")
- Mismatched seniority (junior recruiter pitching staff-level roles)

### 2. Company analysis

**Input required:** Company name or URL. Always runs (company name is extractable from
most input types).

**Research targets:**
- Funding stage, recent rounds, investors, valuation trajectory
- Recent news: launches, layoffs, pivots, acquisitions, lawsuits
- Tech stack: extracted from JD, job postings, engineering blog, GitHub
- Engineering culture signals: remote-friendly, async, meeting-heavy, work hours
- Leadership: CTO/VP Eng background, tenure, previous companies
- Growth trajectory: new headcount (growth) vs. backfill (churn signal)
- For public companies: recent earnings, hiring freezes, stock trajectory

**Red flag triggers:**
- Recent layoffs > 10% of workforce
- No engineering blog or public tech presence
- Leadership churn (multiple CTO/VP Eng departures in 2 years)
- Glassdoor engineering reviews < 3.5

### 3. Mystery client resolution

**Input required:** Posting from a staffing firm with unnamed end client. Short-circuits
entirely for direct hire postings.

**Deduction signals:**
- Tech stack combination (e.g., "Scala + Kafka + Flink" narrows candidates)
- Location or timezone requirements
- Domain language in JD (fintech, healthcare, defense vocabulary)
- Rate range (high rates suggest well-funded companies)
- Company size descriptors ("Fortune 500", "Series C startup")
- Security clearance requirements

**Output format:**
- Deduced client with confidence level (high/medium/low)
- Evidence supporting the deduction
- 2-3 alternative candidates ranked by likelihood

### 4. Compensation reality check

**Input required:** Stated comp from listing or recruiter message. Short-circuits
if no comp data available.

**Analysis:**
- Parse stated compensation (salary, hourly rate, equity mentions)
- Detect engagement type: W-2 salary, W-2 contract, 1099 independent contractor
- Compare to market P25/P50/P75 for role/location/tier using web research
- For 1099 rates: apply 30-40% premium adjustment over W-2 equivalent
  (accounts for self-employment tax, no benefits, no PTO)
- Compare against user's floor from ~/gaj/context/about-me.md
- Flag if stated comp is > 25% below market P50

### 5. Red flag detection

Cross-cutting sweep across all gathered data. Runs after dimensions 1-4 complete
so it can synthesize across findings.

**Scan targets:**
- MLM or pyramid scheme signals
- Fake listing patterns (too-good-to-be-true comp, impossibly broad requirements)
- Bait-and-switch indicators (title/comp mismatch, vague "competitive" comp)
- Data harvesting funnels (mass generic postings, AI interview required before
  any human contact, monitoring app requirements)
- Undisclosed details that should be disclosed at this stage
- Unrealistic requirements-to-comp ratio
- Company has active lawsuits related to employment practices or fraud

**Severity levels:**
- **Critical:** Unnamed client required before interview, comp > 25% below user's
  floor, suspicious recruiter patterns, confirmed data harvesting funnel, active
  fraud allegations
- **Warning:** Declining growth, comp below market P50 but above floor, vague JD,
  staffing firm with mixed reputation, recent layoffs
- **Info:** Useful context signals that aren't dealbreakers. New to market, small
  team, niche domain.

## Narrative output template

Present sections in this order. Omit sections with no data.

1. **Header** - "SHERLOCK REPORT: [Subject]" and one-line signal summary (positive/mixed/negative)
2. **The listing** - Title, company, rate, type, location, poster. Just facts.
3. **Business model / what they actually are** - What the company does vs. what the
   listing implies. This is where "talent funnel disguised as job board" analysis lives.
4. **Company profile** - Table: founded, valuation/funding, revenue, investors, headcount,
   notable news
5. **Recruiter profile** - Who posted, background, legitimacy signals
6. **Red flags** - Ranked by severity (critical first), each with specific evidence citation
7. **Green flags** - What's genuinely positive
8. **Compensation analysis** - Stated vs. market rates, engagement type implications
9. **Stack fit** - Overlap with user's skills from about-me.md, gaps identified
10. **Verdict** - Pass/pursue/dig deeper. One paragraph, opinionated, with reasoning.

End with web sources as markdown links.

## Structured storage schema

When storing findings in pipeline job_data, use this JSON structure:

```json
{
  "sherlock": {
    "investigated_at": "ISO 8601 timestamp",
    "input_type": "url|jd|company|recruiter|mixed",
    "recruiter": {
      "name": "string",
      "firm": "string",
      "legitimacy": "verified|unverified|suspicious",
      "tenure": "string",
      "red_flags": ["string"]
    },
    "company": {
      "name": "string",
      "funding": "string",
      "valuation": "string",
      "news": ["string"],
      "tech_stack": ["string"],
      "culture_signals": ["string"],
      "growth": "growing|stable|declining|unknown"
    },
    "mystery_client": {
      "deduced": "string or null",
      "confidence": "high|medium|low",
      "evidence": ["string"],
      "alternatives": ["string"]
    },
    "compensation": {
      "stated": "string",
      "engagement_type": "w2-salary|w2-contract|1099|unknown",
      "market_p25": "string",
      "market_p50": "string",
      "market_p75": "string",
      "vs_floor": "above|at|below",
      "assessment": "string"
    },
    "stack_fit": {
      "overlap_score": "0-100",
      "matched": ["string"],
      "missing": ["string"]
    },
    "red_flags": [
      { "signal": "string", "severity": "critical|warning|info", "source": "string" }
    ],
    "verdict": "pass|pursue|dig-deeper",
    "verdict_reasoning": "string"
  }
}
```

## Secret weapon principle

Detective research findings are strictly internal intelligence for the user.
Never reveal research findings in any outbound communication. Outbound messages
must reference ONLY information the recruiter explicitly provided. Do not quote
back salary numbers, hint at knowing an unnamed client, or reference Glassdoor
data. Ask open questions that make the recruiter reveal information rather than
showing you already know it. The edge only works if it stays hidden.

This principle is enforced by /gaj:respond, not by sherlock itself. Sherlock
produces the full report. Respond controls what leaks outward.
