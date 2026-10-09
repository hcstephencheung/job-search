# Sep 2026 job crawler — Algorithm

2026-09-19 · @Someone

How the nightly run sources, filters and ranks jobs. Goals, profile and schedule live in the main spec: [Sep 2026 job crawler](./job-crawler.md)

## Sources

Only jobs posted on applicant-tracking (ATS) platforms qualify. A job counts as
sourced from a platform when the run read it from that platform's API, whatever
domain the posting link lands on — companies on Greenhouse often serve postings
from their own careers domain with a `gh_jid` parameter, and those still qualify.

| Platform | Endpoint | Posted date | Status |
| --- | --- | --- | --- |
| Greenhouse | `boards-api.greenhouse.io/v1/boards/{token}/jobs?content=true` | `first_published` (ISO 8601) | Working, confirmed in a run |
| Lever | `api.lever.co/v0/postings/{slug}?mode=json` | `createdAt` (epoch milliseconds) | Working, confirmed in a run |
| Ashby | `api.ashbyhq.com/posting-api/job-board/{name}?includeCompensation=true` | `publishedAt` (ISO 8601) | Working, confirmed in a run |
| Workday | `POST {org}.wd3.myworkdayjobs.com/wday/cxs/{org}/{board}/jobs` | `postedOn`, a relative phrase | Working, confirmed in a run |
| Pinpoint | `{subdomain}.pinpointhq.com/postings.json` | None — a posting carries no date field | Working; every posting is undated |
| SmartRecruiters | `api.smartrecruiters.com/v1/companies/{slug}/postings` (pages via `limit`/`offset`) | `releasedDate` (ISO 8601), unconfirmed | Working; Equinox returns 731 postings |
| BambooHR | `{subdomain}.bamboohr.com/careers/list` (JSON, postings under `result`) | None — the list carries no date field | Working; every posting is undated |
| Jobvite | No reliable public surface — the API is per-customer and the XML feed is opt-in, usually off | — | No public board; recommend dropping |

Nothing in this table is blocked by network egress. An earlier version marked five
platforms "blocked by egress"; that was a probing error, and Workday in particular was
probed on its human-facing board URL instead of its API, which cannot return postings
however well it connects. Reachable means the host answers — it does not mean a usable
board was found, which is what Status says. A run still records each board's outcome
rather than silently skipping it.

Quirks: Lever puts the job title in `text`, not `title`. Greenhouse's list endpoint
returns `first_published` and the full description when called with `content=true`, so
a run needs no per-posting detail fetch. Of the public ATS APIs, only Greenhouse,
SmartRecruiters and Recruitee publish an updated timestamp as well as a first-published
one.

Workday takes a POST with a JSON body (`{"appliedFacets":{},"limit":20,"offset":0,
"searchText":""}`) and returns `jobPostings` carrying `title`, `locationsText`,
`externalPath` and `postedOn`; the posting URL is the board URL plus `externalPath`.
`postedOn` is always present but relative: an exact age under 30 days ("Posted Today",
"Posted 7 Days Ago"), with everything older collapsed into "Posted 30+ Days Ago". Read
the exact forms as a date. "30+ Days Ago" yields no date, so the posting is undated —
it qualifies and ranks last, which is what dropping the posted-date requirement bought.

Pinpoint's `postings.json` supersedes a deprecated `jobs.json`, which returned only the
primary posting per job. The `posted_at` field seen in research came from a third-party
aggregator's normalised schema, not Pinpoint's raw output — same caveat as BambooHR.

### Aggregators

Not sources, and never the date the 3-month rule runs on. Slug discovery reads them
only to find which companies are hiring, then crawls that company's own ATS board for the
authoritative posting and date. None publishes an official public API.

| Aggregator | Scope | Dates shown |
| --- | --- | --- |
| [Built In Vancouver](https://builtinvancouver.org/jobs) | Vancouver tech and startup roles | Not confirmed |
| [startup.jobs](https://startup.jobs/locations/vancouver) | Startups, Vancouver filter | Full UTC timestamp to the second; blocks non-browser requests (403) since 2026-10 |
| [Y Combinator](https://www.ycombinator.com/jobs/location/vancouver) | YC startups, Vancouver filter | Not confirmed |
| [Top Startups](https://topstartups.io/jobs/?job_location=Vancouver) | Startups, Vancouver filter | Not confirmed |
| [Wellfound](https://wellfound.com/location/vancouver) | Startup and tech roles | Not confirmed; blocks non-browser requests (403) |
| Indeed, ZipRecruiter, LinkedIn | General, high volume | Often relative ("3 days ago"); heavy duplication |

### Company boards

Named companies whose postings we want, and the board each one actually posts to.

| Company | Platform | Board | Status |
| --- | --- | --- | --- |
| Wealthsimple — Remote Canada | Ashby | `api.ashbyhq.com/posting-api/job-board/wealthsimple` | Crawlable; 44 postings, 38 Remote Canada |
| Jane Software | Ashby | `api.ashbyhq.com/posting-api/job-board/jane` | Crawlable; 21 postings, all Remote Canada |
| Jobber — Edmonton, Toronto, Vancouver | Ashby | `api.ashbyhq.com/posting-api/job-board/jobber` | Crawlable; 40 postings, all Canadian |
| Spare — Vancouver | Ashby | `api.ashbyhq.com/posting-api/job-board/spare` | Crawlable; 2 postings, both Vancouver |
| Gumloop — San Francisco, Vancouver | Ashby | `api.ashbyhq.com/posting-api/job-board/gumloop` | Crawlable; 12 postings, Vancouver as a secondary location |
| Prenuvo | Greenhouse | `boards-api.greenhouse.io/v1/boards/prenuvo/jobs` | Crawlable; 65 postings, 12 Vancouver |
| DoorDash Canada | Greenhouse | `boards-api.greenhouse.io/v1/boards/doordashcanada/jobs` | Crawlable; 35 postings, all Canadian, 12 Vancouver |
| EviSmart | Greenhouse | `boards-api.greenhouse.io/v1/boards/evismart/jobs` | Crawlable; 16 postings, 10 Vancouver |
| Fanatics Commerce | Greenhouse | `boards-api.greenhouse.io/v1/boards/fanaticscommerce/jobs` | Crawlable; 203 postings, 3 Canadian |
| Sonder (sonder.io) | Greenhouse | `boards-api.greenhouse.io/v1/boards/sonderaustralia/jobs` | Crawlable; 42 postings, 10 Canadian |
| Thrive Digital | Greenhouse | `boards-api.greenhouse.io/v1/boards/thrivedigital/jobs` | Crawlable; 7 postings, 4 Canadian |
| Myodetox | Lever | `api.lever.co/v0/postings/myodetox?mode=json` | Crawlable; 46 postings, mostly clinic roles |
| Mednow | Lever | `api.lever.co/v0/postings/mednow?mode=json` | Crawlable; 3 postings, all Canadian |
| Equinox+ | SmartRecruiters | `api.smartrecruiters.com/v1/companies/Equinox/postings` | Crawlable; 731 postings, 5 Canadian |
| Monark — White Rock, BC | BambooHR | `monark.bamboohr.com/careers/list` | Crawlable; 5 postings, all BC |
| Molecular You — Vancouver | BambooHR | `molecularyou.bamboohr.com/careers/list` | Crawlable; 4 postings, 3 BC |
| Knix — Toronto, Remote Canada only | Lever | `api.lever.co/v0/postings/knix?mode=json` | Crawlable; `createdAt` confirmed |
| Mejuri — Toronto, Remote Canada only | Greenhouse | `boards-api.greenhouse.io/v1/boards/mejuri/jobs` | Crawlable; `first_published` confirmed |
| Article | Pinpoint | `article.pinpointhq.com/postings.json` | Crawlable; 10 postings, all undated |
| Aritzia | Workday | `aritzia.wd3.myworkdayjobs.com`, board `External` | Crawlable; 547 postings, `postedOn` confirmed |
| Best Buy Canada | Workday | `bestbuycanada.wd3.myworkdayjobs.com`, board `BestBuyCA_Career` | Crawlable; 131 postings, `postedOn` confirmed |
| Remitly | Workday | `remitly.wd5.myworkdayjobs.com`, board `Remitly_Careers` | Crawlable; 156 postings, 21 Vancouver area |
| TELUS Health | Workday | `lifeworks.wd3.myworkdayjobs.com`, board `External` | Crawlable; 172 postings, 70 Canadian |
| TRIUMF | Workday | `triumf.wd10.myworkdayjobs.com`, board `careers-at-triumf-job-postings` | Crawlable; 6 postings, all Vancouver |
| Dexcom | Workday | `dexcom.wd1.myworkdayjobs.com`, board `Dexcom` | Crawlable; 280 postings, 2 Vancouver |
| ABC Fitness Solutions | Workday | `abcfinancial.wd5.myworkdayjobs.com`, board `ABCFinancialServices` | Crawlable; 37 postings, 1 Canadian |
| lululemon | Self-hosted, not on any platform above | `careers.lululemon.com` | Platform unidentified; not crawlable |
| Endy — Toronto, Remote Canada only | Unknown, careers page on own domain | `ca.endy.com/pages/careers` | Not reachable as an ATS board |

lululemon sits on no platform listed here, so the crawler as specced cannot reach it.
The login path under `en_US/careers` hints at Oracle or Avature rather than Workday,
but that is a guess from a URL shape and needs the page source to confirm.

A Workday company board is a host plus a board name, not one URL: both are needed to
build the API path, so the table records them separately.

### Slug discovery

Slug discovery runs first, before every crawl. A wrong slug is indistinguishable from an
outage: on 2026-09-20, 72 of 76 board failures were guessed slugs that returned 404.

Each run, in order:

1. **Find.** Look for hiring companies not yet in the slug table:
   - Web-search each ATS domain for engineering postings in Vancouver or Canada —
     `site:jobs.ashbyhq.com`, `site:job-boards.greenhouse.io`, `site:boards.greenhouse.io`
     and `site:jobs.lever.co`, each with "Vancouver" and with "Canada". A result URL
     gives the slug directly (`jobs.ashbyhq.com/<slug>/…`).
   - Read the aggregators above for company names.
2. **Discover.** Resolve every company under Company boards and every company found in
   step 1 to its board and slug — confirming pairs already in the slug table still
   answer, and resolving any company that has no slug yet.
3. **Store.** Write each confirmed pair to the slug table (see Job board data in the
   main spec), keyed by `<slug>:<jobBoard>`.
4. **Crawl.** Fetch postings only for pairs in the slug table. A slug that did not
   resolve is never guessed at.

A run reports each board's outcome separately — reached, 404, or unreachable — rather
than a single list of sources checked.

## Disqualifiers

A job is dropped if any of these apply:

- Posted date, when the posting gives one, is older than 3 months before the run date
- No posting link
- Description states the employee must reside somewhere that excludes Metro Vancouver (e.g. "must reside in the United States", "open to candidates residing in the US", "must live in the Greater Toronto Area")
- Fails the level, location or stack rule below

## Level

A posting qualifies on level if any of these hold:

1. Its title names Senior, Staff or an equivalent (see Levels in the main spec).
2. Its title names no level, and the description mentions senior or staff, asks for
   5+ years, or says the level is set after interviews ("level accordingly", "all
   levels", "regardless of starting level").
3. It is mid level — its title names no level or a mid level ("Intermediate", "Mid",
   "II") and rule 2 does not apply — and its posted pay midpoint meets the pay
   threshold (see Ranking). A mid-level posting with no pay posted is dropped.

Titles marked junior, intern, new grad or similar never qualify.

## Location

A posting qualifies on location if any of these hold:

1. It is in Canada — on-site, hybrid or remote.
2. It is remote across North America or the Americas.
3. It is remote in the US, or remote with no country named, **and** the company has an
   engineering team in Canada. On-site and hybrid roles outside Canada never qualify.

A posting with no location counts as remote with no country named.

Run the level and title checks before this one, so the team check runs only for
companies with a qualifying engineering role.

### Canadian engineering check

Checked once per company and cached in `companies/<slug>`. A `true` result is kept for
good and never checked again; a `false` result is rechecked after 30 days. A company has
a Canadian engineering team when any one of these, checked in order, shows it:

1. **The posting text.** It says the role is open to candidates in Canada ("US or
   Canada", Canada among the eligible locations).
2. **The company's own boards.** It has an engineering posting located in Canada, live
   or already in `seen/`.
3. **The approvals list.** The company appears in Canada's
   [Positive LMIA Employers List](https://open.canada.ca/data/dataset/90fed587-1364-4f33-a9ee-208181dc0b97)
   in any quarter, for a software occupation (NOC 2011 codes 2173,
   2174, 2175; NOC 2021 codes 21231, 21232, 21234). The list gives legal names, so
   match on the company name within the employer name (Findem appears as "Findem
   Technologies Inc.", Vancouver).

No evidence means the job stays disqualified.

## Stack

An agent reads each posting that has passed every other rule — title and full description —
and judges its **primary stack**: the languages and frameworks the role mainly works in,
not ones listed as nice to have or mentioned in passing. It gives each posting a tier:

| Tier | Primary stack | Examples |
| --- | --- | --- |
| Strong | TypeScript or JavaScript with React and/or Node, or Python | Fullstack TypeScript, React frontend, Node or Python backend |
| Partial | TypeScript or JavaScript with another frontend framework | Vue or Angular frontend |
| None | Anything else | iOS (Swift), Android (Kotlin), .NET / C#, Java, Go, Ruby, C++, embedded, infrastructure-only |

A posting judged **None is dropped**. When a posting accepts several primary languages
("Go, Python or TypeScript"), judge it on the best one it accepts.

Run the agent in batches (about 25 postings each), returning per posting its URL, primary
stack, tier and a one-line reason. Store each judgment in `stacks/<hash>` (same hash as
`seen/`) and reuse it on later nights instead of judging the posting again.

## Ranking

Each qualifying job scores out of 100, the sum of four parts below. Highest score ranks
first; ties go to the newer posted date, with undated postings last.

| Part | Max | Points |
| --- | --- | --- |
| Location | 30 | See Location points |
| Pay | 30 | See Pay points |
| Stack | 25 | Strong 25, Partial 10 (None is dropped — see Stack) |
| Title | 15 | See Title points |

**Location points.** A posting listing several locations takes its highest.

| Location | Points |
| --- | --- |
| Metro Vancouver | 30 |
| Remote Canada | 30 |
| Remote North America or Americas | 15 |
| Remote with no country named, passing the Canadian engineering check | 15 |
| On-site or hybrid elsewhere in Canada | 15 |
| Remote US, passing the Canadian engineering check | 3 |

**Pay points** use the midpoint of the posted range, or the single figure if only one is
given. When a posting lists several ranges, use the one for the location it qualified
on. Convert hourly pay at 2,080 hours a year, daily at 260 days and monthly at 12 months.
Partial $10k steps round in the job's favour.

| Midpoint | Points |
| --- | --- |
| CA$200k or more, or US$140k or more | 30 |
| CA$180k–200k, or US$120k–140k | 28, plus 1 per $10k above CA$180k or US$120k |
| Below CA$180k, or below US$120k | 20, minus 1 per full $10k below |
| Not posted, or in another currency | 20 |

**Pay threshold** — the bar a mid-level job must meet (see Level) — is a midpoint of
CA$180k or US$120k, with no currency conversion.

**Title points.**

| Title | Points |
| --- | --- |
| Senior or Staff | 15 |
| No level in the title, qualifying through the description (Level rule 2) | 15 |
| Lead, Principal, Architect or Distinguished | 10 |
| Mid level (Level rule 3) | 10 |
| Any posting asking for 15+ years of experience | 10 |

## Open questions

- None right now.
