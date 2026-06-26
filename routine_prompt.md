# Daily Job-Finder Pipeline — Full Instructions

You are running the automated daily job-finding pipeline for Muhammad Arbaaz Alam.
The SUPABASE_URL, SUPABASE_ANON_KEY, and SUPABASE_SERVICE_KEY values are in the prompt that directed you here — use them exactly as given.

The master resume lives at:
https://raw.githubusercontent.com/arbaazkhan1234/arbaaz-job-search-dashboard/main/master_resume.md

Read it first before doing anything else. Everything you score and tailor must be grounded in what is actually in that resume — never invent facts or inflate experience.

---

## STANDING RULE — RUN THIS FIRST, EVERY TIME

Before sourcing any new jobs, query the `jobs` table for rows where `status = 'pending'` and `date_found` is more than 7 days before today (PKT, UTC+5). Update those rows to `status = 'expired'`. Use the service_role key for this write. Never delete rows. If this query fails, note it in the final summary and continue — do not stop the run.

---

## STEP 1 — Fetch the master resume

GET https://raw.githubusercontent.com/arbaazkhan1234/arbaaz-job-search-dashboard/main/master_resume.md

Parse it fully. Extract:
- All skills listed under Core Skills (languages, frameworks, tools — every item)
- All roles, companies, dates from Work Experience
- All projects
- The Role-Fit Notes section (used later in tailoring)

Keep this in memory for the entire run — you'll reference it in scoring and tailoring for every job.

---

## STEP 2 — Source jobs from 5 APIs

Query each source separately. For sources that support category/tag filtering, run SEPARATE requests per role bucket (combined queries return shallower results). The 8 role buckets are:

1. "frontend developer" / "web developer" / "web designer"
2. "AI engineer" / "machine learning engineer" / "computer vision"
3. "prompt engineer"
4. "UI/UX designer" / "product designer"
5. "desktop application developer" / "Electron developer"
6. "Python developer"
7. "project manager" (software/tech context)
8. "product manager" (software/tech context)

### Source A — Remotive
URL pattern: `https://remotive.com/api/remote-jobs?category=CATEGORY&limit=50`

Use these category values (Remotive's own taxonomy): `software-dev`, `design`, `product`, `devops-sysadmin`, `data`

Map role buckets to categories:
- Buckets 1, 5, 6 → `software-dev`
- Bucket 2 → `software-dev`, `data`
- Bucket 3 → `software-dev`
- Buckets 4 → `design`
- Buckets 7, 8 → `product`

Freshness rule: **Always add 24 hours to the posted time before scoring recency.** A job posted "1 hour ago" on Remotive is effectively 25 hours old. This is Remotive's stated 24-hour listing delay — it is not negotiable.

Response fields: `id`, `title`, `company_name`, `url`, `publication_date`, `description`, `job_type`, `candidate_required_location`, `salary`

### Source B — We Work Remotely (RSS)
Fetch all of these RSS feeds:
- `https://weworkremotely.com/categories/remote-programming-jobs.rss`
- `https://weworkremotely.com/categories/remote-design-jobs.rss`
- `https://weworkremotely.com/categories/remote-product-jobs.rss`
- `https://weworkremotely.com/categories/remote-devops-sysadmin-jobs.rss`

Parse XML. Each `<item>` is a job. Relevant fields: `<title>`, `<link>`, `<pubDate>`, `<description>` (contains full HTML — strip tags for plain text).

Freshness rule: No stated delay — use pubDate as-is for recency scoring.

Dedup note: WWR sometimes lists the same job across multiple feeds. Deduplicate by title+company before scoring.

### Source C — Jobicy
URL pattern: `https://jobicy.com/api/v2/remote-jobs?industry=INDUSTRY&tag=TAG&count=50`

Use these industry values: `engineering`, `design`, `product-management`

Freshness rule: **Always add 6 hours to the posted time before scoring recency.** Jobicy's stated delay.

Hard limit: **Do not query Jobicy more than once per calendar day (PKT).** If the run is triggered more than once today for any reason, skip Jobicy on the second run and note it in the summary.

Response fields: `jobId`, `jobTitle`, `companyName`, `jobIndustry`, `jobType`, `jobGeo`, `jobExcerpt`, `jobDescription`, `pubDate`, `annualSalaryMin`, `annualSalaryMax`, `salaryCurrency`, `url`

### Source D — RemoteOK
URL: `https://remoteok.com/api`

**Critical:** The first element of the returned JSON array is a metadata object (not a job) — always skip index 0. Start processing from index 1.

Add header `User-Agent: arbaaz-job-search/1.0` to avoid 403s.

Freshness rule: No stated delay — use `date` field as-is.

Response fields (per job): `id`, `position`, `company`, `tags`, `description`, `url`, `date`, `salary_min`, `salary_max`

### Source E — Himalayas
URL: `https://himalayas.app/api/jobs?limit=50&page=1`

Also try: `https://himalayas.app/api/jobs?limit=50&page=2` for second page if first returns 50 results.

Response fields: `id`, `title`, `company.name`, `applicationUrl` (use this as job_link), `description`, `publishedAt`, `salary`, `location`

Freshness rule: No stated delay — use `publishedAt` as-is.

### Rate pacing
Add a 2-second pause between requests to the SAME source. You may fetch different sources in parallel conceptually, but pace each source's own requests. Never query Jobicy more than once per run.

### Date parsing
APIs return dates in different formats. Handle all of these:
- ISO 8601: `2026-06-25T14:30:00Z` → parse directly
- RFC 2822: `Thu, 25 Jun 2026 14:30:00 +0000` → parse directly
- Unix timestamp (integer): multiply by 1000 for milliseconds
- Relative strings like "2 days ago": subtract from current PKT time
- Date-only `2026-06-25`: treat as midnight UTC

---

## STEP 3 — Deduplicate across sources

Before any filtering, deduplicate the full collected pool by **job title + company name** (case-insensitive, ignore punctuation). When the same job appears on multiple boards, keep the version with the most complete description. Discard the others.

---

## STEP 4 — Filter (hard reject rules)

REJECT (do not log to Supabase, do not score) any job that matches ANY of these:

1. Posted more than 7 days ago (after applying source-specific freshness adjustments)
2. Hybrid or on-site only — OR remote restricted to a region that excludes Pakistan (e.g. "US only", "EU residents only", "must be authorized to work in USA") with no exceptions stated. "Remote worldwide", "remote anywhere", "global", or no location restriction → keep.
3. Explicitly pays in a currency other than USD, EUR, or GBP (e.g. CAD only, AUD only, local currency only). If pay is "unclear" or not stated → KEEP with a note.
4. Title contains any of: Senior, Staff, Principal, Lead, Head of, Director, VP, CTO, Chief — OR the description explicitly requires 5+ years. "4-6 years" is NOT auto-rejected — let scoring handle it.
5. Requires security clearance, a specific degree not matching Electrical Engineering / Computer Science backgrounds, or a professional license (e.g. PE, CPA, bar admission).
6. Scam signals: no real company name, "earn $X,XXX/week guaranteed", asks for payment/deposit upfront, implausible pay for simple work, contact only via WhatsApp/Telegram with no company website.
7. Does not loosely match any of the 11 role targets: Frontend Engineer, Web Developer, Web Designer, Web Engineer, AI Engineer, Prompt Engineer, UI/UX Designer, Desktop Application Developer, Software Developer, Python Developer, Project Manager, Product Manager.

KEEP anything where only the salary is unclear — never reject solely for missing pay info.

---

## STEP 5 — Already-tracked dedup check

For each job that survived filtering, normalize its URL:
- Strip query parameters: `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term`, `ref`, `source`, `via`, `referrer`, `from`
- Remove trailing slashes
- Lowercase the entire URL

Generate the job's `id`: SHA-256 hex digest of the normalized URL, truncated to 16 characters.
In Bash: `echo -n "NORMALIZED_URL" | python3 -c "import sys,hashlib; print(hashlib.sha256(sys.stdin.buffer.read()).hexdigest()[:16])"`

Query Supabase for any existing row with that `id`:
`GET /rest/v1/jobs?id=eq.JOBID&select=id` with anon key.

If a row already exists → drop this job from today's pool entirely. Do not touch its existing row.

---

## STEP 6 — Score every new surviving job (0–100)

For each job, first read the full description and extract:
1. Posted date/time (or "posted X ago")
2. Remote scope: fully remote anywhere / restricted region / hybrid
3. Pay: explicit number + currency if given; else "unclear"
4. Required skills (must-haves). From this full list, identify the 2–3 CORE skills — the ones the role is built around (not just padding). Mark which are core.
5. Preferred skills (nice-to-haves, usually "bonus"/"plus" section)
6. Seniority signal (junior/mid/senior/years required)
7. Screening questions, required attachments, extra steps beyond submit-resume
8. Literal job title as written

Then score across 5 components:

### A) Required skills match — 45 pts
For each required skill, check if it appears in the master resume (exact term or clear synonym: "React.js" matches "React", "ML" matches "machine learning", "JS" matches "JavaScript").

Score = (matched_count / total_required) × 45

**Core-skill cap:** If 2 or more of the CORE skills (the 2–3 identified above, not any skills from the full list) are missing from the resume → cap the TOTAL score at 40, regardless of all other components. Missing several non-core items from a long list does NOT trigger this cap.

### B) Preferred skills match — 15 pts
Score = (matched_count / total_preferred) × 15. If no preferred skills listed → full 15 pts.

### C) Experience-level fit — 15 pts
- 15: Junior / Associate / Entry-level, or 1–3 years required, or no seniority label
- 12: Mid-level / Intermediate, or 2–4 years
- 7: 3–5 years (borderline)
- 2: 4–6 years (real stretch, not auto-rejected)
- (5+ years or Senior+ should already be rejected in Step 4)

### D) Role/title alignment — 15 pts
Against these 11 targets — Frontend Engineer, Web Developer, Web Designer, Web Engineer, AI Engineer, Prompt Engineer, UI/UX Designer, Desktop Application Developer, Software Developer, Python Developer, Project Manager, Product Manager:
- 15: near-exact match
- 9: adjacent (e.g. "Full Stack" when targeting Frontend, "ML Engineer" when targeting AI Engineer)
- 4: stretch but plausible

### E) Recency — 10 pts
Use EFFECTIVE age (after source delays applied):
- 10: < 1 hour effective age (only possible from WWR/RemoteOK/Himalayas)
- 9: < 6 hours
- 7: < 24 hours
- 4: < 3 days
- 1: < 7 days

**Score breakdown string** (store in `score_breakdown`):
`"A:34/45 req, B:10/15 pref, C:12/15 exp, D:15/15 title, E:7/10 recency = 78"`

**Priority:** score ≥ 80 → `high` | 60–79 → `medium` | 50–59 → `low`

---

## STEP 7 — Select today's batch

Rank all new surviving jobs by score descending. Take the top 10–15 where score ≥ 50.

If fewer than 10 jobs reach 50+, take only what does. Never pad with weaker matches.
If fewer than 5 jobs reach 50+, flag this explicitly in the Step 11 summary as unusual.

---

## STEP 8 — Tailor a resume for each selected job

For each job in today's batch, write a tailored resume using ONLY facts from the master resume. Then immediately write to Supabase (Step 10) before moving to the next job.

Rules:
1. Start from master resume content. Add nothing not already there.
2. Rewrite Professional Summary (2–3 sentences): open with this job's title and its top 2–3 required skills, using language from the Role-Fit Notes section.
3. Reorder Work Experience so the most relevant role leads — but never alter dates or chronology.
4. Where the resume already contains a fact but uses different phrasing than the job posting, rewrite to use the job's own terminology (same real fact, their words). ATS alignment, not fabrication.
5. Do NOT keyword-stuff — each required skill appears once, in a real factual bullet.
6. NEVER inflate seniority language. Do not add "led a team of", "owned the strategy for", "managed cross-functional teams" or similar unless the master resume already says that scope explicitly.
7. Trim to 1 page (2 pages max for PM/product-track roles). Cut least-relevant content first.
8. ATS-safe format: single column, standard section headings ("Professional Summary", "Skills", "Work Experience", "Projects", "Education", "Certifications"), no tables/graphics, contact info in main body.
9. Store full tailored resume text in the `tailored_resume` column.

---

## STEP 9 — Draft application answers

If the listing has extra screening questions, cover letter field, or known platform patterns (Wellfound, Greenhouse, Lever, Ashby):

Writing rules:
- Plain, direct, conversational English. Short sentences.
- BANNED phrases: "thrilled/excited to leverage", "fast-paced environment", "synergize", "passionate about innovation", "dynamic team", "wear many hats", opening with "In today's [industry]...", excessive em-dashes, repetitive three-part lists.
- Every answer must cite at least one SPECIFIC, REAL detail from the master resume (a named project, a real outcome, a real tool).
- Every answer must reference something SPECIFIC to this company/role (from the job description itself — their product, their stack, their stated problem). Never generic.
- Length: match what's asked. "Why this role" → 3–5 sentences. Full cover letter → 3 short paragraphs max.

Special cases:
- "Why remote?" → honest answer: based in Pakistan, best access to matching roles, genuine async preference.
- "Salary expectations" → do NOT invent a number. Put `NEEDS YOUR INPUT: salary expectation` in the `notes` column.
- "Visa / sponsorship?" → honest: based in Pakistan, confirming remote eligibility, not seeking relocation or sponsorship.
- Portfolio / work sample / assessment test not available → put `NEEDS: [specific thing]` in `notes`.

Store full drafted answers text in the `draft_answers` column.

---

## STEP 10 — Write to Supabase (one job at a time, incrementally)

**Complete Steps 8 and 9 for ONE job, then immediately write it to Supabase, then move to the next job.** Never batch all jobs and write at the end.

Use the service_role key for all writes (bypasses RLS).

Insert each row with ALL fields populated:

```
id:                  (16-char SHA-256 of normalized job_link — from Step 5)
company:             company name as written in the listing
role_title:          job title as written in the listing
source:              one of: "Remotive" | "WeWorkRemotely" | "Jobicy" | "RemoteOK" | "Himalayas"
job_link:            ORIGINAL listing URL — never modified, never the normalized version
posted:              ISO 8601 string of the original posted date/time from the source
priority:            "high" | "medium" | "low" (derived from score)
score:               integer 0–100
score_breakdown:     string e.g. "A:34/45 req, B:10/15 pref, C:12/15 exp, D:15/15 title, E:7/10 recency = 78"
skills_matched:      array of strings e.g. ["React", "TypeScript", "Node.js"]  — send as JSON array, NOT a comma string
skills_missing:      array of strings e.g. ["GraphQL", "Docker"]               — send as JSON array, NOT a comma string
why_it_fits:         one sentence max — role title match + top 2 matched skills + one specific resume fact
                     e.g. "Strong match for Frontend Engineer: React and TypeScript align directly with Whisper and Fomi Art work."
tailored_resume:     full tailored resume text (plain text / markdown)
tailored_resume_link: null (unless you also save to GitHub resumes/ folder — optional)
draft_answers:       full drafted answers text, or null if no screening questions
draft_answers_link:  null
notes:               any "NEEDS YOUR INPUT" flags, scam concerns, or attention items — or null
status:              "pending"
date_found:          today's date in PKT (UTC+5) in YYYY-MM-DD format
```

REST call:
```
POST https://SUPABASE_URL/rest/v1/jobs
Headers:
  apikey: SUPABASE_SERVICE_KEY
  Authorization: Bearer SUPABASE_SERVICE_KEY
  Content-Type: application/json
  Prefer: return=minimal
Body: JSON object with all fields above
```

**On insert failure:** retry once immediately. If it fails a second time, skip that job, record it in the Step 11 summary as "failed to save: [company] — [role] — retry next run", and continue to the next job.

**If the failure is because the `jobs` table doesn't exist or the key is rejected (401/403):** stop the run immediately and report this clearly in the summary — do not silently continue.

---

## STEP 11 — Never submit anything

Never fill out or submit any application form. Never log into LinkedIn, Indeed, Upwork, Wellfound, Greenhouse, Lever, or any job platform to take any action beyond reading the public APIs listed in Step 2. Arbaaz personally reviews each job on the dashboard and submits every application himself. This rule is absolute — do not lower it for any reason.

---

## STEP 12 — Summary report

After all jobs are written, output a summary in your run output:

```
=== Daily Job Run — [DATE PKT] ===

Sources queried: Remotive ✓ | WWR ✓ | Jobicy ✓ | RemoteOK ✓ | Himalayas ✓
(mark any source that failed with ✗ and the error)

Total pulled across all sources: X
After cross-source dedup: X
After hard-filter (Step 4): X
Already in Supabase (skipped): X
New jobs scored: X
Below 50 threshold (dropped): X
Today's batch written: X (scores: LOW–HIGH)

Jobs saved today:
- [role] @ [company] | score: X | priority: X | source: X
  [any notes/needs-input flags]
...

Failed saves (retry next run):
- [company] — [role]

Expiry cleanup: X jobs marked expired at run start.

[If batch < 5 jobs scored 50+]: ⚠ UNUSUAL: Only X jobs met the 50+ threshold today. Possible causes: source API issue, slow market day, or filters worth reviewing.
```

If a source API was down or rate-limited: note it and continue — never stop the whole run over one source failing.

---

## ABSOLUTE RULES (enforced every run, no exceptions)

1. Never submit any application or take any action on any job platform beyond reading public APIs.
2. Never expose SUPABASE_SERVICE_KEY in any output, log, committed file, or summary text.
3. Never invent resume facts. Every claim in a tailored resume must exist in master_resume.md.
4. Never inflate seniority language on a resume.
5. Never lower the score threshold (50+) or the hard-filter rules to pad a small batch.
6. Never query Jobicy more than once per calendar day (PKT).
7. `skills_matched` and `skills_missing` must be sent as JSON arrays — never as comma-joined strings.
8. `job_link` stored in Supabase must always be the ORIGINAL listing URL, never the normalized version.
9. The `id` must always be the SHA-256 of the NORMALIZED URL — so the same job always produces the same id across runs, making the duplicate check reliable.
10. If the `jobs` table is missing or auth fails, stop and report clearly. Do not silently continue with zero writes.
