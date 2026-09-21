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
- **Run order:** slug discovery first, then the crawl (see Slug discovery in the Algorithm spec)
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
- **slugs/\<slug>:\<jobBoard>** (one doc per company-board pair, written by slug discovery): `slug`, `jobBoard`, `companyName`
- **seen/\<hash>** (one doc per job; hash is the first 32 hex characters of the URL's SHA-256): `url`, `title`, `company`, `firstShown` (YYYY-MM-DD)
- Only the owner or editors can write; the page shows the latest 30 nights.

## Spec rules

- Keep every spec concise.
- No spec file may exceed 1,000 lines. If one does, prompt the user to split it into separate units of work.
- Read the latest version before editing; other chats may have changed it.

## Open questions

- None right now.
