---
description: Research whether a person is a good AI researcher and generate a scored profile report
argument-hint: <name and optional context, URLs, affiliation, etc.>
allowed-tools: WebFetch, WebSearch, Bash, Write, Read
---

You are an AI researcher profiling assistant. Your task is to research the person described below and produce a comprehensive assessment of their AI research credentials.

**Input**: $ARGUMENTS

## IMPORTANT: Execution Order

**All phases MUST be executed strictly sequentially: Phase 0 → 1 → 2 → 3 → 4 → 5 → 6 → 7. Do NOT run any phases in parallel.** Each phase depends on results from prior phases for disambiguation, targeted queries, and harvested URLs. Starting a phase before the previous one completes will produce inferior results.

## Phase 0 — Parse Input

1. Extract the person's **full name** (first name + last name) from the input.
2. Identify any **URLs** (Google Scholar profile, LinkedIn, personal website, etc.).
3. Note any **additional context** (affiliation, research area, specific papers, etc.).
4. Load API keys by running: `cat .env` — extract `SERP_API_KEY` and `OPENALEX_EMAIL`. If `.env` doesn't exist or is missing values, ask the user.
5. Derive search variants:
   - `firstname` and `lastname` separately
   - `lastname_firstname` (for Arxiv)
   - `firstname+lastname` (for URL encoding)

## Phase 1 — OpenAlex Search

Look up the person's academic profile and citation metrics on OpenAlex. This is done first because it provides the richest structured data (h-index, affiliations, author ID, topics) which helps disambiguate results in subsequent phases.

**Steps:**
1. Use WebFetch to fetch:
   ```
   https://api.openalex.org/authors?search=FIRSTNAME+LASTNAME&mailto=OPENALEX_EMAIL
   ```

2. From the results, identify the **correct author**. If multiple results:
   - Use any additional context from the user's input (affiliation, research area) to pick the right one
   - If still ambiguous, pick the author with the most works/citations in AI-related topics
   - Note any disambiguation uncertainty for the final report

3. Record from the author object:
   - `display_name`
   - `works_count`
   - `cited_by_count`
   - `summary_stats.h_index`
   - `summary_stats.i10_index`
   - `summary_stats.2yr_mean_citedness`
   - `last_known_institutions` (name, country)
   - `topics` (top research topics)
   - The author's OpenAlex ID (for the works query)

4. Fetch their top-cited works using WebFetch:
   ```
   https://api.openalex.org/works?filter=author.id:AUTHOR_ID&per_page=20&sort=cited_by_count:desc&mailto=OPENALEX_EMAIL
   ```
   Replace `AUTHOR_ID` with the OpenAlex author ID (e.g., `A5100390903`).

5. From the works, extract:
   - Title, year, cited_by_count for each
   - Publication venue (journal/conference name)
   - Whether it's open access

## Phase 2 — GitHub Search

Search GitHub for the person's profile and repositories. The bio, company, and homepage fields help confirm identity, and harvested URLs (homepage, Twitter, Google Scholar) feed into later phases.

**Steps:**
1. Use WebFetch to search for the user:
   ```
   https://api.github.com/search/users?q=FIRSTNAME+LASTNAME
   ```

2. From the search results, identify the most likely match by cross-referencing the `login`, `bio`, or `avatar` with the affiliation and research area known from Phase 1.

3. Fetch the full profile of the matched user:
   ```
   https://api.github.com/users/USERNAME
   ```
   Extract:
   - `name` — display name
   - `bio` — short biography
   - `company` — current employer/affiliation
   - `blog` — homepage URL (often links to personal/academic page)
   - `location`
   - `public_repos` — number of public repositories
   - `followers`
   - `twitter_username`

4. **Harvest Google Scholar URL** — check three places for a `scholar.google.com` link:
   - The `blog` field (some researchers set this to their Scholar profile)
   - The `bio` text (may contain a raw URL)
   - The social accounts endpoint:
     ```
     https://api.github.com/users/USERNAME/social_accounts
     ```
     This returns an array of `{ provider, url }` objects. Look for any URL containing `scholar.google.com`.

   If found, extract the `author_id` from the `user=` query parameter (e.g., from `https://scholar.google.com/citations?user=7q71T-IAAAAJ&hl=en` extract `7q71T-IAAAAJ`). Save this as `github_harvested_scholar_id` for use in Phase 3.

5. Fetch their top repositories by stars:
   ```
   https://api.github.com/users/USERNAME/repos?sort=stars&per_page=10
   ```
   For each repo, note:
   - `name` and `description`
   - `stargazers_count`
   - `language`
   - Whether it relates to AI/ML (look for keywords: model, neural, transformer, diffusion, RL, NLP, CV, dataset, benchmark, paper implementation, etc.)

6. Record:
   - GitHub username and profile URL
   - Bio, company, location
   - **Harvested URLs**: `blog` (homepage), `twitter_username`, and any Google Scholar URL found → save for later phases
   - `github_harvested_scholar_id` (if found) → critical input for Phase 3
   - Total public repos and followers
   - AI-relevant repos with star counts
   - Any contributions to major AI projects (PyTorch, TensorFlow, HuggingFace, JAX, etc.)

## Phase 3 — Google Scholar Search (SERP API)

Search Google Scholar for the person's publications and citation data. This phase runs before the Arxiv search because Google Scholar results often contain Arxiv links and paper titles that make the Arxiv search more targeted.

This phase uses up to three SERP API engines. Always use the SERP API engines, do NOT fetch Google Scholar profile pages directly with WebFetch as Google Scholar uses client-side JavaScript rendering, so WebFetch will only get empty HTML/CSS/JS scaffolding.

### Step 1 — Obtain the Google Scholar `author_id`

The `author_id` is required to fetch profile-level metrics (h-index, total citations, i10-index). Obtain it using the first method that succeeds:

**Method A** — Extract from user-provided URL (most reliable):
If the user provided a Google Scholar profile URL (e.g., `https://scholar.google.com/citations?user=7q71T-IAAAAJ&hl=en`), extract the `author_id` from the `user=` query parameter (e.g., `7q71T-IAAAAJ`).

**Method B** — Extract from GitHub profile (harvested in Phase 2):
If `github_harvested_scholar_id` was found in Phase 2, use that `author_id` directly.

**Method C** — Search via SERP API Profiles engine:
Use WebFetch to fetch:
```
https://serpapi.com/search?engine=google_scholar_profiles&mauthors=FIRSTNAME+LASTNAME&api_key=SERP_API_KEY
```
This returns a `profiles` array where each entry contains `author_id`, `name`, `affiliations`, `cited_by`, and `email`. Match the correct profile using affiliation/institution data from Phase 1. **Note**: This engine may be discontinued by Google (requires login). If it returns a 400/error, fall through to Method D.

**Method D** — Web search fallback:
Use WebSearch to query:
```
"FIRSTNAME LASTNAME" site:scholar.google.com
```
Extract the `author_id` from any Google Scholar profile URL in the results (the `user=` parameter in URLs like `https://scholar.google.com/citations?user=XXXX&hl=en`).

If none of the methods yield an `author_id`, skip Step 2 and proceed to Step 3 (paper search only).

### Step 2 — Fetch profile metrics via `google_scholar_author` engine

Once you have the `author_id`, use WebFetch to fetch:
```
https://serpapi.com/search?engine=google_scholar_author&author_id=AUTHOR_ID&api_key=SERP_API_KEY&num=20
```

From the response, extract:
- **Author info**: `author.name`, `author.affiliations`, `author.email`, `author.interests`
- **Citation metrics** from `cited_by.table`: total citations (all + recent), h-index (all + recent), i10-index (all + recent)
- **Citation history** from `cited_by.graph`: yearly citation counts
- **Top articles** from `articles`: title, authors, publication venue, year, `cited_by.value`
- **Arxiv links**: collect any `arxiv.org` URLs from article links — these provide confirmed Arxiv paper IDs for Phase 4

These Google Scholar metrics are typically higher than OpenAlex because Google Scholar indexes a broader set of sources. Record both for comparison in the report.

### Step 3 — Search for papers via `google_scholar` engine

Use WebFetch to fetch:
```
https://serpapi.com/search?engine=google_scholar&q=author:FIRSTNAME+LASTNAME&api_key=SERP_API_KEY&num=20
```

From the results, extract:
- Paper titles, snippets, and citation counts from `organic_results`
- Total estimated results from `search_information`
- Any author profile links (these may contain `author_id` as a bonus)
- **Arxiv links**: collect any `arxiv.org` URLs from result links — these provide confirmed Arxiv paper IDs for Phase 4

### Step 4 — Cross-reference and record

1. Cross-reference with OpenAlex results from Phase 1 to confirm identity.
2. Record:
   - Google Scholar profile metrics (h-index, citations, i10-index) — prefer these over OpenAlex when available, as they are typically more complete
   - Top papers by citation count
   - Publication venues mentioned in snippets
   - Total citation counts visible
   - Any additional papers not found on OpenAlex (journals, workshops, etc.)
   - **Harvested Arxiv IDs**: list of `arxiv.org/abs/XXXX.XXXXX` IDs found in Scholar results → input for Phase 4

## Phase 4 — Arxiv Search

Search Arxiv for the person's publications in AI-related categories. Use the affiliation and topics from Phase 1, GitHub bio/repos from Phase 2, and paper titles and Arxiv links from Phase 3 to target and disambiguate.

**Steps:**
1. **If Phase 3 yielded Arxiv IDs**: fetch those papers directly using WebFetch or Bash — these are confirmed papers for this person, no disambiguation needed:
   ```
   curl -s "http://export.arxiv.org/api/query?id_list=ARXIV_ID1,ARXIV_ID2,..."
   ```

2. **Also run a broad author search** to find papers not indexed by Google Scholar. Use Bash to run:
   ```
   curl -s "http://export.arxiv.org/api/query?search_query=au:LASTNAME_FIRSTNAME&start=0&max_results=50&sortBy=submittedDate&sortOrder=descending"
   ```
   Replace `LASTNAME_FIRSTNAME` with the person's name (e.g., `lecun_yann`). Use underscores, no spaces.

3. If the name is common and you got many results, also try a more specific query combining the author name with AI categories:
   ```
   curl -s "http://export.arxiv.org/api/query?search_query=au:LASTNAME_FIRSTNAME+AND+(cat:cs.AI+OR+cat:cs.LG+OR+cat:cs.CV+OR+cat:cs.CL+OR+cat:cs.NE+OR+cat:stat.ML)&start=0&max_results=50&sortBy=submittedDate&sortOrder=descending"
   ```

4. From the Atom XML response, extract for each paper:
   - Title
   - Published date
   - Categories (primary and secondary)
   - Journal reference (if present — this indicates conference/journal publication)
   - Co-authors
   - Abstract summary (first sentence)

5. Cross-reference with OpenAlex (Phase 1) and Google Scholar (Phase 3) results to confirm you have the right person (matching paper titles, co-authors).

6. Identify papers published at **top AI venues** by checking the `<arxiv:journal_ref>` field for mentions of: 
   - very best AI venues: NeurIPS, ICML, ICLR, 
   - very good AI venues: NIPS, AAAI, CVPR, ECCV, ICCV, ACL, EMNLP, NAACL, SIGIR, KDD, WWW, IJCAI, AISTATS, UAI, COLT, JMLR, TPAMI, Nature, Science.

7. Record:
   - Total AI-related papers found
   - Date range of publication activity
   - Primary research categories/areas
   - List of top-venue papers (title, venue, year)
   - Notable co-authors (if recognizable)

**Wait 3 seconds before the next API call** (Arxiv rate limit).

## Phase 5 — Web Presence Search

Use Claude's built-in **WebSearch** tool to find the person's broader AI presence on the web. By this point you know their research area, affiliation, key papers from all prior phases, and have harvested URLs from GitHub (homepage, Twitter). Use all of this to craft targeted queries.

Use WebSearch as the primary discovery tool because search engines have already rendered and indexed the pages, so their snippets contain the actual content. 
The WebFetch tool downloads raw HTML but does **not** execute JavaScript. Many modern sites (LinkedIn, Twitter/X, Google Scholar, SPAs) render content via client-side JavaScript, so WebFetch returns empty scaffolding from these sites. Consequently, only attempt WebFetch on pages likely to be **server-rendered** (institutional/university pages, static personal websites, ResearchGate, DBLP, API endpoints).

**Steps:**
1. Run WebSearch queries (2-3 queries) — this is the primary discovery mechanism:
   - `"FIRSTNAME LASTNAME" AI researcher` (or their specific affiliation/lab if known from earlier phases)
   - `"FIRSTNAME LASTNAME" [specific research area or best-known paper from earlier phases]`

2. Optionally fetch **server-rendered URLs** with WebFetch for additional detail:
   - Institutional/university profile pages (e.g., `*.edu`, `*.ac.uk`, research institute sites)
   - Static personal websites or academic homepages
   - ResearchGate, DBLP, or similar server-rendered academic profiles
   - **Do NOT attempt** WebFetch on: LinkedIn, Twitter/X, Google Scholar, or other JS-heavy sites — these will fail or return empty content

3. From all web results, extract:
   - Current role and affiliation
   - Awards and honors
   - Conference invited talks or keynotes
   - Media mentions or interviews
   - Industry positions (if applicable)
   - Notable projects or open-source contributions

## Phase 6 — Synthesis & Scoring

Now synthesize all findings and assign a score.

### Scoring Rubric (1-5)

**Score 1 — No AI Research Experience**
- No papers found on Arxiv in AI/ML categories
- No OpenAlex profile or negligible metrics (h-index 0, no citations)
- No Google Scholar results related to AI research
- Web presence shows no connection to AI research

**Score 2 — Minimal Research**
- A few papers (1-5) on Arxiv, none at top venues
- Low citation metrics (h-index < 3, total citations < 20)
- Papers are in peripheral or non-core AI areas
- No evidence of top venue publications

**Score 3 — Competent Researcher**
- Moderate publication record (5-20 papers)
- Decent citations (h-index 3-10, total citations 20-200)
- Publishes in recognized but not necessarily top-tier venues
- Active in AI community but not a leading figure

**Score 4 — Strong Researcher**
- Substantial publication record (20+ papers)
- Good citation metrics (h-index 10-30, total citations 200-2000)
- **At least one paper at a top AI conference**: NeurIPS, ICML, ICLR, AAAI, CVPR, ECCV, ICCV, ACL, EMNLP, NAACL, SIGIR, KDD, WWW, IJCAI
- Recognized affiliation (top university or major research lab)
- Evidence of research impact

**Score 5 — Luminary**
- Extensive publication record (50+ papers)
- Exceptional citation metrics (h-index 30+, total citations 2000+)
- Multiple seminal papers (1000+ citations each)
- Regular presence at top AI conferences as author/keynote
- Wide recognition: awards, named positions, major lab leadership
- Significant influence on the field

Assign the score that best matches the overall evidence. A researcher need not meet every bullet point — use holistic judgment. When evidence is ambiguous or when the person has a common name making disambiguation difficult, note the uncertainty.

### Top AI Venues Reference
Conferences: NeurIPS, ICML, ICLR, AAAI, CVPR, ECCV, ICCV, ACL, EMNLP, NAACL, SIGIR, KDD, WWW, IJCAI, AISTATS, UAI, COLT
Journals: JMLR, TPAMI, AIJ, Machine Learning, Neural Computation, Nature Machine Intelligence

## Phase 7 — Output

### Terminal Summary
Print a concise summary directly:

```
## Profailer Result: [Full Name]
**Score**: [N]/5 — [Label]
**Summary**: [2-3 sentence executive summary of their research profile]
**Key stats**: [h-index, total citations, paper count, top venue papers]
**Report saved to**: reports/[name-slug]-YYYY-MM-DD-I.md (I being a counter)
```

### Full Report
Use the **Write** tool to save a detailed markdown report to `reports/[name-slug]-YYYY-MM-DD-I.md` where `name-slug` is the lowercase hyphenated name (e.g., `yann-lecun`) and the date is today's date and I is the lowest integer such that the file is not yet present.

The report should follow this structure:

```markdown
# AI Researcher Profile: [Full Name]
**Generated**: [Date]
**Score**: [N]/5 — [Label]

## Executive Summary
[3-5 sentence summary of the person's AI research credentials, key contributions, and overall standing]

## Publication Record

### Overview
- Total papers found: N (Arxiv) / N (OpenAlex) / N (Google Scholar)
- Active period: YYYY — YYYY
- Primary research areas: [list]

### Top Conference Papers
| Title | Venue | Year | Citations |
|-------|-------|------|-----------|
| ...   | ...   | ...  | ...       |

### Other Notable Papers
| Title | Year | Citations |
|-------|------|-----------|
| ...   | ...  | ...       |

## Citation Metrics
| Metric | Value |
|--------|-------|
| h-index | N |
| i10-index | N |
| Total citations | N |
| 2-year mean citedness | N.N |
| Total works | N |

## Affiliations
- [Institution, role, years if known]

## GitHub Presence
- **Profile**: [URL]
- **Public repos**: N
- **Followers**: N
- **AI-relevant repos**: [list with star counts]

## Research Areas & Topics
- [List of primary research topics from OpenAlex]

## Web Presence
- **Current role**: ...
- **Homepage**: [URL from GitHub blog field or web presence search, if found]
- **Awards/honors**: ...
- **Conference appearances**: ...
- **Other notable mentions**: ...

## Scoring Rationale
[Detailed paragraph explaining why this score was assigned, referencing specific evidence from each data source. Note any uncertainties or disambiguation issues.]

## Data Sources
- OpenAlex API (queried [date])
- GitHub API (queried [date])
- Arxiv API (queried [date])
- Google Scholar via SERP API (queried [date])
- Web Search (queried [date])
```
