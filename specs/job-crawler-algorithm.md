# Sep 2026 job crawler — Algorithm

2026-09-19 · @Someone

How the nightly run sources, filters and ranks jobs. Goals, profile and schedule live in the main spec: [Sep 2026 job crawler](./job-crawler.md)

## Sources

Only jobs posted on applicant-tracking job boards qualify. The posting link must point to one of these boards.

- Greenhouse
- Ashby
- Lever
- BambooHR
- Jobvite

Aggregators (LinkedIn, Indeed, etc.) are not sources.

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
