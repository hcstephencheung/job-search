# Board database snapshot

A point-in-time export of the [Nightly Fifty](https://claude.ai/artifact/XZCbzgNH52FHnBRPNePS25)
artifact database, taken 2026-09-20. Each file is one document, holding exactly the
document body as stored — no wrapper, no metadata.

This is a snapshot for history and diffing, not the live store. The crawler and the
board page both read and write the artifact database directly; nothing here is loaded
at runtime, and editing these files changes nothing.

## Layout

```
db/
  nights/<YYYY-MM-DD>.json   one per run
  seen/<hash>.json           one per job ever shown
```

## nights/&lt;YYYY-MM-DD&gt;

One document per run, written once when the run finishes.

| Field | Notes |
| --- | --- |
| `runDate` | YYYY-MM-DD |
| `runAt` | ISO timestamp of the run |
| `sourcesChecked` | board names reached this run |
| `stats.fetched` | postings pulled from all boards before filtering |
| `stats.qualified` | postings surviving the disqualifiers, before the top-50 cut |
| `jobs` | ranked array, at most 50 |

Each job carries `rank`, `title` and `url` (always present), plus `company`, `board`,
`location`, `remote`, `postedDate`, `salary` {`min`, `max`, `currency`}, `stackMatch`
and `description` (≤1,200 chars) where the posting supplied them.

## seen/&lt;hash&gt;

One document per job posting ever shown, so a run writes only the jobs it adds.
The document id is the first 32 hex characters of the SHA-256 of the posting URL:

```python
hashlib.sha256(url.encode()).hexdigest()[:32]
```

Body is `url`, `title`, `company`, `firstShown`. The full URL lives in the body because
the id is a hash and cannot be reversed.

Before 2026-09-20 this was a single `seen/<YYYY-MM>` document holding a map keyed by
URL. That shape was rewritten in full on every run — about 250 KB by the end of a month,
and a collision point for concurrent runs — so it was migrated to one document per job.

## Current contents

- `nights/2026-09-19` — 22 jobs (seeding run)
- `nights/2026-09-20` — 50 jobs, 6,921 postings fetched, 93 qualified
- `seen/` — 72 documents, equal to the union of both nights
