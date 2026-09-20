# Sep 2026 job crawler

2026-09-19 · @Someone

## Purpose

A nightly crawler that updates an artifact with the top 50 job postings matched to me. This spec holds goals, profile and automation.

- **Algorithm spec:** [Sep 2026 job crawler — Algorithm](./job-crawler-algorithm.md) (sources, filters, ranking)

## Goal

Show Senior to Staff software engineer jobs I can actually take.

- **Levels:** Senior, Staff, and title equivalents (e.g. Senior II, Lead, Principal)
- **Locations:** Metro Vancouver, BC, or Remote within Canada
- **Every job must include a direct link to its posting**

## Candidate profile

- **Minimum level:** Senior
- **Primary stack:** Fullstack TypeScript, React + Node
- **Also:** Python
- **Databases:** NoSQL or relational, both fine

## Automation

A scheduled task runs nightly and updates the job board artifact with the top 50 jobs.

- **Schedule:** 6:00 PM Vancouver time daily, year-round (01:00 UTC during PDT, 02:00 UTC during PST)
- **Output:** top 50 new jobs, never shown on a previous night, each with its posting link
- **Inputs each run:** this spec and the Algorithm spec, read fresh by link
- **Sourcing, filtering and ranking:** defined only in the Algorithm spec

## Seen jobs

Jobs shown on earlier nights are saved in a separate store, one document per job. Each run skips any URL already in it, then adds the new 50.

- **Key:** job posting URL, hashed to a document id (first 32 hex characters of its SHA-256)
- **Value:** the full URL, title, company, date first shown
- **Location:** the job board's database, in seen/\<hash> documents (see Job board data)
- Each run writes only the jobs it adds; earlier documents are never rewritten.

## Job cards

- **Required:** job title, job posting link
- **Optional (shown when available):** posted date, job description, salary, remote-friendly or not

## Job board data

The board is published at [Nightly Fifty](https://claude.ai/artifact/XZCbzgNH52FHnBRPNePS25). The nightly run writes to its database; the page never needs republishing.

- **nights/\<YYYY-MM-DD>** (one doc per run): `runDate`, `runAt` (ISO time), `sourcesChecked` (board names), `stats` {`fetched`, `qualified`}, `jobs` (array, ranked)
- **Each job:** `rank`, `title`, `url` (required); `company`, `board`, `location`, `remote` (true/false), `postedDate` (YYYY-MM-DD), `salary` {`min`, `max`, `currency`}, `stackMatch` (list), `description` (max 1,200 chars)
- **seen/\<hash>** (one doc per job; hash is the first 32 hex characters of the URL's SHA-256): `url`, `title`, `company`, `firstShown` (YYYY-MM-DD)
- Only the owner or editors can write; the page shows the latest 30 nights.

## Spec rules

- Keep every spec concise.
- No spec file may exceed 1,000 lines. If one does, prompt the user to split it into separate units of work.
- Read the latest version before editing; other chats may have changed it.
- Log every change in the Changelog.

## Open questions

- None right now.

## Changelog

- 2026-09-19: Created with goal, profile, automation and spec rules.
- 2026-09-19: Resolved open questions: 6 PM year-round, title equivalents count, 50 new jobs nightly with a seen-jobs store keyed by URL, job card fields.
- 2026-09-19: Published the job board and defined its data format.
- 2026-09-20: Seen jobs moved from one document per month to one document per job, keyed by a hash of the URL. The monthly document was rewritten in full on every run, which grew to roughly 250 KB by month end and made concurrent runs collide on a single document; per-job documents mean a run writes only what it adds. Migrated the 72 existing entries and removed seen/2026-09.
- 2026-09-20: Snapshot of the board database exported to db/ in this repository.
