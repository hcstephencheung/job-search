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
| SmartRecruiters | `api.smartrecruiters.com/v1/companies/{slug}/postings` (pages via `limit`/`offset`) | `releasedDate` (ISO 8601), unconfirmed | Reachable; no slug tried returns any posting |
| BambooHR | `{subdomain}.bamboohr.com/careers/list` | Unconfirmed | Reachable; serves a generic page, no board found |
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

Not sources, and never the date the 3-month rule runs on. Use them only to discover
which companies are hiring, then crawl that company's own ATS board for the
authoritative posting and date. None publishes an official public API.

| Aggregator | Scope | Dates shown |
| --- | --- | --- |
| [Built In Vancouver](https://builtinvancouver.org/jobs) | Vancouver tech and startup roles | Not confirmed |
| [startup.jobs](https://startup.jobs/locations/vancouver) | Startups, Vancouver filter | Full UTC timestamp to the second — best granularity found |
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
| Knix — Toronto, Remote Canada only | Lever | `api.lever.co/v0/postings/knix?mode=json` | Crawlable; `createdAt` confirmed |
| Mejuri — Toronto, Remote Canada only | Greenhouse | `boards-api.greenhouse.io/v1/boards/mejuri/jobs` | Crawlable; `first_published` confirmed |
| Article | Pinpoint | `article.pinpointhq.com/postings.json` | Crawlable; 10 postings, all undated |
| Aritzia | Workday | `aritzia.wd3.myworkdayjobs.com`, board `External` | Crawlable; 547 postings, `postedOn` confirmed |
| Best Buy Canada | Workday | `bestbuycanada.wd3.myworkdayjobs.com`, board `BestBuyCA_Career` | Crawlable; 131 postings, `postedOn` confirmed |
| lululemon | Self-hosted, not on any platform above | `careers.lululemon.com` | Platform unidentified; not crawlable |
| Endy — Toronto, Remote Canada only | Unknown, careers page on own domain | `ca.endy.com/pages/careers` | Not reachable as an ATS board |

lululemon sits on no platform listed here, so the crawler as specced cannot reach it.
The login path under `en_US/careers` hints at Oracle or Avature rather than Workday,
but that is a guess from a URL shape and needs the page source to confirm.

A Workday company board is a host plus a board name, not one URL: both are needed to
build the API path, so the table records them separately.

### Slug discovery

Slug discovery runs first, before every crawl. The company list is hand-maintained and a
wrong slug is indistinguishable from an outage: on 2026-09-20, 72 of 76 board failures
were guessed slugs that returned 404, while 4 were egress blocks.

Each run, in order:

1. **Discover.** Resolve every company under Company boards to its board and slug —
   confirming pairs already in the slug table still answer, and resolving any company
   that has no slug yet.
2. **Store.** Write each confirmed pair to the slug table (see Job board data in the
   main spec), keyed by `<slug>:<jobBoard>`.
3. **Crawl.** Fetch postings only for pairs in the slug table. A slug that did not
   resolve is never guessed at.

A run reports each board's outcome separately — reached, 404, or unreachable — rather
than a single list of sources checked.

## Disqualifiers

A job is dropped if any of these apply:

- Posted date, when the posting gives one, is older than 3 months before the run date
- No posting link
- Description states the employee must reside in the US (e.g. "must reside in the United States", "open to candidates residing in the US")
- Fails the level rule in the main spec, or the location rule below

## Location

A posting qualifies on location if any of these hold:

1. It lists Metro Vancouver, BC, or is remote within Canada.
2. It is remote across North America or the Americas.
3. It is remote in the US, or remote with no country named, **and** the company has an
   engineering team in Canada. On-site and hybrid roles outside Canada never qualify.

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

## Ranking

Stack match weighs more than pay. Pay is flexible, so no job is excluded for pay.

Signals, weighted most to least:

1. **Stack match:** stronger match to TypeScript, React + Node (and Python) ranks higher
2. **Pay tier:** pay posted and meets the threshold, then pay posted below it, then no pay posted
3. **Recency:** newer posted date ranks higher; undated postings rank last

**Pay threshold** uses the midpoint of the posted range (or the single figure if only one is given):

- CAD postings: midpoint $180k CAD or more
- USD postings: midpoint $120k USD or more, with no currency conversion

## Open questions

- None right now.
