# Sep 2026 job crawler — Algorithm

2026-09-19 · @Someone

How the nightly run sources, filters and ranks jobs. Goals, profile and schedule live in the main spec: [Sep 2026 job crawler](./job-crawler.md)

## Sources

Only jobs posted on applicant-tracking (ATS) platforms qualify. A job counts as
sourced from a platform when the run read it from that platform's API, whatever
domain the posting link lands on — companies on Greenhouse often serve postings
from their own careers domain with a `gh_jid` parameter, and those still qualify.

### Working platforms

Reachable from the crawler and confirmed in a run.

| Platform | Endpoint | Posted date |
| --- | --- | --- |
| Greenhouse | `boards-api.greenhouse.io/v1/boards/{token}/jobs`, then `/jobs/{id}` for detail | `first_published` (ISO 8601) |
| Lever | `api.lever.co/v0/postings/{slug}?mode=json` | `createdAt` (epoch milliseconds) |
| Ashby | `api.ashbyhq.com/posting-api/job-board/{name}?includeCompensation=true` | `publishedAt` (ISO 8601) |

Quirks: Lever puts the job title in `text`, not `title`. Greenhouse's list endpoint
omits `first_published`, so the run fetches each surviving posting's detail endpoint.
Of the public ATS APIs, only Greenhouse, SmartRecruiters and Recruitee publish an
updated timestamp as well as a first-published one.

### Blocked platforms

Endpoints believed to exist, but every host below is refused by this environment's
network egress policy, so none can be crawled or field-verified from here. A run must
record them as unreached rather than silently skip them.

| Platform | Endpoint | Posted date | Status |
| --- | --- | --- | --- |
| SmartRecruiters | `api.smartrecruiters.com/v1/companies/{slug}/postings` (pages via `limit`/`offset`) | `releasedDate` (ISO 8601), unconfirmed | Blocked |
| Pinpoint | `{subdomain}.pinpointhq.com/postings.json` | Unconfirmed — Pinpoint's docs do not list posting fields | Blocked |
| BambooHR | Public JSON endpoints exist, URL unconfirmed | Unconfirmed | Blocked; list API exposes no posted date |
| Workday | `{org}.wd3.myworkdayjobs.com/{board}` | Not checked | Blocked; dropped 2026-09-19 |
| Jobvite | No reliable public surface — the API is per-customer and the XML feed is opt-in, usually off | — | Blocked; recommend dropping |

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
| Knix — Toronto, Remote Canada only | Lever | `api.lever.co/v0/postings/knix?mode=json` | Crawlable; `createdAt` confirmed |
| Mejuri — Toronto, Remote Canada only | Greenhouse | `boards-api.greenhouse.io/v1/boards/mejuri/jobs` | Crawlable; `first_published` confirmed |
| Article | Pinpoint | `article.pinpointhq.com/postings.json` | Blocked |
| Aritzia | Workday | `aritzia.wd3.myworkdayjobs.com/External` | Blocked |
| Best Buy Canada | Workday | `bestbuycanada.wd3.myworkdayjobs.com/BestBuyCA_Career` | Blocked |
| lululemon | Self-hosted, not on any platform above | `careers.lululemon.com` | Blocked; platform unidentified |
| Endy — Toronto, Remote Canada only | Unknown, careers page on own domain | `ca.endy.com/pages/careers` | Not reachable as an ATS board |

lululemon sits on no platform listed here, so the crawler as specced cannot reach it.
The login path under `en_US/careers` hints at Oracle or Avature rather than Workday,
but that is a guess from a URL shape and needs the page source to confirm.

### Slug discovery

The company list a run crawls is hand-maintained, and a wrong slug is indistinguishable
from an outage: on 2026-09-20, 72 of 76 board failures were guessed slugs that returned
404, while 4 were egress blocks. A run must therefore report each board's outcome
separately — reached, 404, or unreachable — rather than a single list of sources checked.

## Disqualifiers

A job is dropped if any of these apply:

- Posted date is older than 3 months before the run date
- No posted date
- No posting link
- Fails the level or location rules in the main spec

## Ranking

Stack match weighs more than pay. Pay is flexible, so no job is excluded for pay.

Signals, weighted most to least:

1. **Stack match:** stronger match to TypeScript, React + Node (and Python) ranks higher
2. **Pay tier:** pay posted and meets the threshold, then pay posted below it, then no pay posted
3. **Recency:** newer posted date ranks higher

**Pay threshold** uses the midpoint of the posted range (or the single figure if only one is given):

- CAD postings: midpoint $180k CAD or more
- USD postings: midpoint $120k USD or more, with no currency conversion

## Open questions

- None right now.

## Changelog

- 2026-09-19: Created with sources, disqualifiers and provisional ranking.
- 2026-09-19: Resolved open questions: midpoint pay threshold ($180k CAD, $120k USD, no conversion), Workday "30+ days ago" disqualified, stack match weighs over pay.
- 2026-09-19: Dropped Workday as a source (its postings rarely show a posted date).
- 2026-09-21: Rewrote Sources from the Job board timestamp fields research. Split the platform list into working (Greenhouse, Lever, Ashby — all three posted-date fields confirmed in a run) and blocked (SmartRecruiters, Pinpoint, BambooHR, Workday, Jobvite), added the aggregator table as a discovery-only tier, and added named company boards. Recorded that a source is defined by the API a posting was read from, not by the link's domain, since Greenhouse serves many postings from company careers domains.
- 2026-09-21: Added the slug-discovery rule after a run reported 76 board failures that were 72 wrong slugs and 4 egress blocks, indistinguishable in the log.
