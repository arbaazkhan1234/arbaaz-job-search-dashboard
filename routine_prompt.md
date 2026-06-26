# Daily Job-Finder Pipeline — Full Instructions

You are running the automated daily job-finding pipeline for Muhammad Arbaaz Alam.
The SUPABASE_URL, SUPABASE_ANON_KEY, and SUPABASE_SERVICE_KEY values are in the prompt that directed you here — use them exactly as given.

The master resume lives at:
https://raw.githubusercontent.com/arbaazkhan1234/arbaaz-job-search-dashboard/main/master_resume.md

Read it first before doing anything else. Everything you score and tailor must be grounded in what is actually in that resume — never invent facts or inflate experience.

---

## STANDING RULE — RUN THIS FIRST, EVERY TIME

Before sourcing any new jobs, query the `jobs` table for rows where `status = 'pending'` and `date_found` is more than 10 days before today (PKT, UTC+5). Update those rows to `status = 'expired'`. Use the service_role key for this write. Never delete rows. If this query fails, note it in the final summary and continue.

---

## STEP 1 — Fetch the master resume

GET https://raw.githubusercontent.com/arbaazkhan1234/arbaaz-job-search-dashboard/main/master_resume.md

Parse it fully. Extract:
- All skills listed under Core Skills (every item)
- All roles, companies, dates from Work Experience
- All projects
- The Role-Fit Notes section (used in tailoring)

---

## STEP 2 — Source jobs from 8 APIs

Run SEPARATE requests per role bucket. The 8 role buckets are:

1. "frontend developer" / "web developer" / "web designer"
2. "AI engineer" / "machine learning engineer" / "computer vision"
3. "prompt engineer"
4. "UI/UX designer" / "product designer"
5. "desktop application developer" / "Electron developer"
6. "Python developer"
7. "project manager" (software/tech context)
8. "product manager" (software/tech context)

---

### Source A — Remotive
URL: `https://remotive.com/api/remote-jobs?category=CATEGORY&limit=100`

Category values: `software-dev`, `design`, `product`, `data`

Response fields: `id`, `title`, `company_name`, `url`, `publication_date`, `candidate_required_location`, `salary`, `description`

**Freshness rule:** Always add 24 hours to posted time before scoring recency. Remotive delays listings 24h by design.

**Known issue:** Remotive sometimes returns identical results across all category queries on a given day (API-side caching bug). If all categories return the same job IDs, note it in the summary and continue — don't query repeatedly.

Use PowerShell to fetch:
```powershell
$r = Invoke-RestMethod "https://remotive.com/api/remote-jobs?category=software-dev&limit=100"
$jobs = $r.jobs
```

---

### Source B — We Work Remotely (RSS)
Fetch ALL of these feeds:
- `https://weworkremotely.com/remote-jobs.rss` (main feed — catches cross-category listings)
- `https://weworkremotely.com/categories/remote-programming-jobs.rss`
- `https://weworkremotely.com/categories/remote-design-jobs.rss`
- `https://weworkremotely.com/categories/remote-product-jobs.rss`

Parse XML. Each `<item>` is a job. Fields: `<title>`, `<link>`, `<pubDate>`, `<description>` (strip HTML tags).

**Paywalled listings:** WWR sometimes requires payment to view the full listing. When this happens:
1. Extract the company name and role title from the preview/title that IS visible
2. Use web search to find the direct company careers page: `"[Company]" "[Role]" careers apply`
3. Fetch the full description from the company's own careers page
4. Store the DIRECT company URL (not the WWR URL) as `job_link`
5. If the company careers URL cannot be found, skip this specific listing rather than storing a broken link

**Freshness rule:** No stated delay — use pubDate as-is.

---

### Source C — Jobicy
URL pattern: `https://jobicy.com/api/v2/remote-jobs?industry=INDUSTRY&count=50`

Try these industry slugs in order:
- `engineering` (for developer/AI roles)
- `design` (for UI/UX roles)
- `product-management` (for PM roles)

**If any industry slug returns 400, immediately fall back to the `tag` parameter instead:**
- `https://jobicy.com/api/v2/remote-jobs?tag=web+developer&count=50`
- `https://jobicy.com/api/v2/remote-jobs?tag=ui+designer&count=50`
- `https://jobicy.com/api/v2/remote-jobs?tag=product+manager&count=50`
- `https://jobicy.com/api/v2/remote-jobs?tag=python+developer&count=50`
- `https://jobicy.com/api/v2/remote-jobs?tag=frontend+developer&count=50`
- `https://jobicy.com/api/v2/remote-jobs?tag=AI+engineer&count=50`

The `industry` and `tag` params are mutually exclusive — never combine them in one request.

**Freshness rule:** Always add 6 hours to posted time before scoring recency.

**Hard limit:** Do not query Jobicy more than once per calendar day (PKT). If run is triggered twice in one day, skip Jobicy on the second run.

Response fields: `jobId`, `jobTitle`, `companyName`, `url`, `pubDate`, `jobGeo`, `salaryCurrency`, `annualSalaryMin`, `annualSalaryMax`, `jobDescription`

---

### Source D — RemoteOK
URL: `https://remoteok.com/api`

**Must use PowerShell with User-Agent header:**
```powershell
$r = Invoke-RestMethod "https://remoteok.com/api" -Headers @{'User-Agent'='arbaaz-job-search/1.0'}
$jobs = $r[1..($r.Count-1)]  # skip index 0 (metadata object, not a job)
```

**Pre-filter before scoring:** RemoteOK mixes many non-tech postings. Before scoring, keep only jobs where `position` or `tags` contain at least one of: `developer`, `engineer`, `designer`, `frontend`, `backend`, `python`, `react`, `typescript`, `javascript`, `ai`, `machine learning`, `product manager`, `software`, `web`, `electron`, `ui`, `ux`, `prompt`.

**Paywalled or 403 job detail pages:** If fetching an individual job's description page returns 403/paywall, use web search to find the company's direct careers page for that role — same approach as WWR paywalled listings above.

**Freshness rule:** No stated delay — use `date` field as-is.

Response fields: `id`, `position`, `company`, `tags`, `url`, `date`, `salary_min`, `salary_max`, `description`

---

### Source E — Himalayas
Primary: `https://himalayas.app/api/jobs?limit=50&page=1`
Fallback: `https://himalayas.app/jobs?format=json&limit=50`

If both return errors, skip Himalayas for that run and note it in the summary. Do not stop the whole run.

Response fields (when available): `id`, `title`, `company.name`, `applicationUrl`, `description`, `publishedAt`, `salary`, `location`

**Freshness rule:** No stated delay — use `publishedAt` as-is.

---

### Source F — Nodesk
URL: `https://nodesk.co/remote-jobs/feed/`

Parse RSS. Fields: `<title>`, `<link>`, `<pubDate>`, `<description>` (strip HTML, first 800 chars).

**Freshness rule:** No stated delay — use pubDate as-is.

---

### Source G — Working Nomads
URLs (fetch all):
- `https://www.workingnomads.com/jobs?category=development&remote=true&feed=rss`
- `https://www.workingnomads.com/jobs?category=design&remote=true&feed=rss`
- `https://www.workingnomads.com/jobs?category=product-management&remote=true&feed=rss`

Parse RSS. Fields: `<title>`, `<link>`, `<pubDate>`, `<description>`.

**Freshness rule:** No stated delay — use pubDate as-is.

---

### Source H — Remote.co
URL: `https://remote.co/remote-jobs/feed/`

Parse RSS. Fields: `<title>`, `<link>`, `<pubDate>`, `<description>` (strip HTML).

**Freshness rule:** No stated delay — use pubDate as-is.

---

### Rate pacing
Add a 2-second pause between requests to the SAME source. Respect Jobicy's once-per-day limit. Sources may be fetched in parallel across different providers.

### Date parsing
Handle all formats: ISO 8601, RFC 2822, Unix timestamp (×1000 for ms), relative strings ("2 days ago"), date-only strings (treat as midnight UTC). When in doubt, parse conservatively (assume older rather than newer).

---

## STEP 3 — Deduplicate across sources

Deduplicate the full collected pool by **job title + company name** (case-insensitive, ignore punctuation). When the same job appears on multiple boards, keep the version with the most complete description. Discard the others.

---

## STEP 4 — Filter (hard reject rules)

REJECT anything matching ANY of these. Apply them strictly in order:

**1. Age:** Posted more than 10 days ago (after applying source-specific freshness adjustments). 10 days is the window — not 7.

**2. Location — how to decide:**
KEEP the job if ANY of these are true:
- Location says "worldwide", "anywhere", "global", "remote worldwide", "fully remote", or similar open phrasing
- Location lists regions/continents that include Asia, South Asia, APAC, or a list that doesn't explicitly exclude Pakistan
- Location says "Americas, Europe, Asia, Oceania" — Asia includes Pakistan → KEEP
- Location says "EMEA" → keep (Pakistan is sometimes counted in Middle East / Asia categories)
- Location is blank or not specified

REJECT only if the listing EXPLICITLY restricts to a region that excludes Pakistan AND states no exceptions:
- "US only", "must be authorized to work in the US", "EU residents only", "UK only", "LATAM only", "Brazil only", etc.
- "Must be in [specific timezone] ±2 hours" where Pakistan (UTC+5) falls outside

When in doubt, KEEP with a note rather than reject. Missing a good job due to over-aggressive location filtering is a worse outcome than scoring and reviewing a borderline one.

**3. Pay currency:** Explicitly pays in a currency other than USD, EUR, or GBP only (e.g. CAD only, AUD only, local INR/PKR). If pay is unclear or not stated → KEEP with note.

**4. Seniority — two-part check:**
- REJECT if title contains: Staff, Principal, Lead, Head of, Director, VP, CTO, Chief
- For "Senior" in title: Do NOT auto-reject. Instead, check the description for required years of experience:
  - If description requires 5+ years → REJECT
  - If description requires ≤4 years (or no years stated) → KEEP and score normally. Some companies use "Senior" for 3-4 year roles.
- REJECT if description explicitly requires 5+ years regardless of title

**5. Disqualifying requirements:** Security clearance, professional license (PE, CPA, bar), or degree requirement that clearly doesn't match (e.g. must have Law degree, Medical degree).

**6. Scam signals:** No real company name, "earn $X,XXX/week guaranteed", asks for payment/deposit upfront, implausible pay for simple tasks, contact only via WhatsApp/Telegram with no company website, no application link anywhere.

**7. Role mismatch:** Does not loosely match any of the 11 role targets: Frontend Engineer, Web Developer, Web Designer, Web Engineer, AI Engineer, Prompt Engineer, UI/UX Designer, Desktop Application Developer, Software Developer, Python Developer, Project Manager, Product Manager.

KEEP anything where only salary is unclear — never reject solely for missing pay info.

---

## STEP 5 — Already-tracked dedup check

Normalize each job's URL:
- Strip: `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term`, `ref`, `source`, `via`, `referrer`, `from`
- Remove trailing slashes
- Lowercase

Generate `id`: SHA-256 hex of normalized URL, truncated to 16 chars.
PowerShell: `[System.BitConverter]::ToString([System.Security.Cryptography.SHA256]::Create().ComputeHash([System.Text.Encoding]::UTF8.GetBytes($normalizedUrl))).Replace('-','').Substring(0,16).ToLower()`

Query Supabase for existing row with that id:
`GET /rest/v1/jobs?id=eq.JOBID&select=id` with anon key.

If row exists → drop from today's pool entirely.

---

## STEP 6 — Score every new surviving job (0–100)

Read the full description and extract:
1. Posted date/time
2. Remote scope
3. Pay: explicit number + currency, or "unclear"
4. Required skills — from this full list, identify the 2–3 CORE skills (role is built around these). Mark which are core vs. the rest.
5. Preferred skills
6. Seniority signal (years required or level label)
7. Screening questions, required attachments, extra steps
8. Literal job title as written

### A) Required skills match — 45 pts
Score = (matched / total_required) × 45

Match logic: exact term OR clear synonym (React.js = React, ML = machine learning, JS = JavaScript, Node = Node.js, TS = TypeScript, NextJS = Next.js, LLM = large language model).

**Core-skill cap:** If 2+ of the CORE skills (2–3 identified above, not any skills from the full list) are missing → cap TOTAL score at 40 regardless of other components. Missing several non-core items from a long wishlist does NOT trigger this cap.

### B) Preferred skills match — 15 pts
Score = (matched / total_preferred) × 15. If no preferred skills listed → 15 pts.

### C) Experience-level fit — 15 pts
- 15: Junior / Associate / Entry-level, or ≤3 years required, or no label
- 12: Mid-level / Intermediate, or 2–4 years
- 7: 3–5 years (borderline)
- 2: 4–6 years (stretch)
- 0: "Senior" kept under Step 4's exception but description still says 5+ years (already caught by reject, but just in case)

### D) Role/title alignment — 15 pts
Against the 11 targets:
- 15: near-exact match ("Frontend Engineer", "UI/UX Designer", "Product Manager")
- 9: adjacent ("Full Stack Engineer" → Frontend, "ML Engineer" → AI Engineer, "Software Engineer" → Software Developer)
- 4: stretch but plausible

### E) Recency — 10 pts
Effective age after source delays:
- 10: <1 hour (only WWR/RemoteOK/Himalayas/Nodesk/WorkingNomads/Remote.co)
- 9: <6 hours
- 7: <24 hours
- 4: <3 days
- 2: <7 days
- 1: <10 days

**Score breakdown string** (store in `score_breakdown`):
`"A:34/45 req, B:10/15 pref, C:12/15 exp, D:15/15 title, E:7/10 recency = 78"`

**Priority:** ≥80 → `high` | 60–79 → `medium` | 50–59 → `low`

---

## STEP 7 — Select today's batch

Rank all new surviving jobs by score descending. Take the top 10–15 where score ≥ 50.

If fewer than 10 jobs reach 50+, take only what does. Never pad with weak matches.
If fewer than 5 jobs reach 50+, flag this explicitly in the Step 12 summary as unusual.

---

## STEP 8 — Tailor a resume for each selected job

Complete Steps 8 and 9 for ONE job, then immediately write to Supabase (Step 10), then move to the next job.

Rules:
1. Start from master resume. Add nothing not already there.
2. Rewrite Professional Summary (2–3 sentences): open with this job's title and its top 2–3 required skills, using language from the Role-Fit Notes section.
3. Reorder Work Experience so most relevant role leads — never alter dates or chronology.
4. Where resume has the same fact but different phrasing than the job posting, rewrite using the job's own terminology. Same real fact, their words. ATS alignment only.
5. Do NOT keyword-stuff — each required skill appears once, in a real factual bullet.
6. NEVER inflate seniority language. No "led a team of", "owned the strategy for", "managed cross-functional teams" unless the master resume already says that scope explicitly.
7. Trim to 1 page (2 pages max for PM/product roles). Cut least-relevant content first.
8. ATS-safe: single column, standard headings ("Professional Summary", "Skills", "Work Experience", "Projects", "Education", "Certifications"), no tables/graphics, contact info in main body.
9. Store full tailored resume text in `tailored_resume` column.

---

## STEP 9 — Draft application answers

If the listing has extra questions, cover letter field, or known platform patterns (Wellfound, Greenhouse, Lever, Ashby):

Rules:
- Plain, direct, conversational English. Short sentences.
- BANNED: "thrilled/excited to leverage", "fast-paced environment", "synergize", "passionate about innovation", "dynamic team", "wear many hats", opening with "In today's [industry]...", excessive em-dashes, repetitive three-part lists.
- Every answer must cite at least one SPECIFIC, REAL detail from the master resume (named project, real outcome, real tool).
- Every answer must reference something SPECIFIC to this company/role (their product, stack, stated problem). Never generic.
- Length: match what's asked. "Why this role" → 3–5 sentences. Full cover letter → 3 short paragraphs max.

Special cases:
- "Why remote?" → honest: based in Pakistan, best access to matching roles, genuine async preference.
- "Salary expectations" → do NOT invent a number. Put `NEEDS YOUR INPUT: salary expectation` in `notes`.
- "Visa/sponsorship?" → honest: based in Pakistan, confirming remote eligibility, not seeking relocation/sponsorship.
- Portfolio/work sample/assessment not available → put `NEEDS: [specific thing]` in `notes`.

Store full drafted answers in `draft_answers` column.

---

## STEP 10 — Write to Supabase (one job at a time, incrementally)

Use the service_role key for all writes.

Insert each row with ALL fields:

```
id:                  16-char SHA-256 of normalized job_link
company:             company name as written
role_title:          job title as written
source:              "Remotive" | "WeWorkRemotely" | "Jobicy" | "RemoteOK" | "Himalayas" | "Nodesk" | "WorkingNomads" | "RemoteCo"
job_link:            ORIGINAL listing URL (or direct company URL if source was paywalled)
posted:              ISO 8601 string of original posted date/time
priority:            "high" | "medium" | "low"
score:               integer 0–100
score_breakdown:     e.g. "A:34/45 req, B:10/15 pref, C:12/15 exp, D:15/15 title, E:7/10 recency = 78"
skills_matched:      JSON array e.g. ["React", "TypeScript"] — NEVER a comma string
skills_missing:      JSON array e.g. ["GraphQL"] — NEVER a comma string
why_it_fits:         one sentence: role match + top 2 skills + one resume fact
tailored_resume:     full tailored resume text
tailored_resume_link: null
draft_answers:       full drafted answers or null
draft_answers_link:  null
notes:               NEEDS YOUR INPUT flags, scam warnings, location uncertainty notes, or null
status:              "pending"
date_found:          today in PKT (UTC+5) as YYYY-MM-DD
```

REST call:
```
POST https://SUPABASE_URL/rest/v1/jobs
Headers:
  apikey: SUPABASE_SERVICE_KEY
  Authorization: Bearer SUPABASE_SERVICE_KEY
  Content-Type: application/json
  Prefer: return=minimal
Body: JSON with all fields above
```

On failure: retry once immediately. Second failure → skip, log in Step 12 summary, continue.
If table missing or auth fails (401/403): stop and report clearly — do not silently continue.

---

## STEP 11 — Never submit anything

Never fill out or submit any application form. Never log into LinkedIn, Indeed, Upwork, Wellfound, Greenhouse, Lever, or any job platform to take any action beyond reading public APIs. Arbaaz reviews every job on the dashboard and submits himself. Absolute rule, no exceptions.

---

## STEP 12 — Summary report

Output in run results:

```
=== Daily Job Run — [DATE PKT] ===

Sources: Remotive [✓/✗] | WWR [✓/✗] | Jobicy [✓/✗] | RemoteOK [✓/✗] | Himalayas [✓/✗] | Nodesk [✓/✗] | WorkingNomads [✓/✗] | RemoteCo [✓/✗]
(note any API issues or fallbacks used)

Total pulled: X
After dedup: X
After filters: X
Already in Supabase: X
New scored: X
Below 50 (dropped): X
Saved today: X (score range: LOW–HIGH)

Jobs saved:
- [role] @ [company] | score: X | priority: X | source: X
  [notes/flags if any]

Failed saves (retry next run): ...

Expiry cleanup: X jobs marked expired.

[If <5 jobs scored 50+]: ⚠ UNUSUAL — only X jobs met threshold. Possible cause: [API issue / slow market / note what failed]
```

---

## ABSOLUTE RULES

1. Never submit any application or take any action on job platforms beyond reading public APIs.
2. Never expose SUPABASE_SERVICE_KEY in any output, log, or file.
3. Never invent resume facts. Every claim must exist in master_resume.md.
4. Never inflate seniority language on a resume.
5. Never lower the score threshold (50+) or hard-filter rules to pad a thin batch.
6. Never query Jobicy more than once per calendar day (PKT).
7. `skills_matched` and `skills_missing` must be JSON arrays — never comma strings.
8. `job_link` must always be the original (or direct company) URL — never the normalized version.
9. `id` must always be SHA-256 of the NORMALIZED URL — same job always produces same id.
10. If `jobs` table missing or auth fails → stop and report. Do not silently continue with zero writes.
