# Shingou signal hash-commitment log

Append-only log of every signal bucket published by [shingou.io](https://shingou.io).

At publish time, the ingestion engine writes one line per (symbol, interval, bucket):

```json
{"symbol":"BTC-USD","interval":"1h","bucket":"2026-07-04T10:00:00.000Z","hash":"<sha256>"}
```

`hash` is the SHA-256 (hex) of the canonical string

```
symbol|interval|bucket|score|confidence|direction|news_volume|novelty_score|reconstructed
```

where `symbol`, `interval` and `bucket` are the line's own fields **verbatim**
(the API renders the same instant as `+00:00` instead of `.000Z`, so use the
committed string, not the API's serialization), and the remaining fields are
exactly the values `GET /v1/history/sentiment` returns for that bucket. Numbers
are rendered the way JavaScript renders them (`String(x)`, e.g. `0.65`, `0`,
`false`). Files live under `log/YYYY-MM-DD.jsonl`, keyed by the bucket's UTC date.

Day files are kept sorted, so a commit can insert lines anywhere in the file.
No line is ever changed or removed. The git history is the proof.

## Verify it yourself

One command. No dependencies. Node 18 or newer, plus a free API key from
[shingou.io](https://shingou.io):

```bash
git clone https://github.com/shingou-io/shingou-signal-log.git
cd shingou-signal-log
node verify.mjs --key YOUR_API_KEY
```

The script fetches each bucket from the live API, recomputes the hash, and
compares it to the committed line. It paces itself to fit the free plan's rate
limit.

Two plan limits shape which day is worth reading. A free key sees 7 days of
history, and non-major symbols are served with a 24h delay, so buckets outside
those windows are reported as skipped rather than failed. The script picks a day
that clears both by default. Pass `--day 2026-07-27` to choose one yourself, and
note that a day which verifies nothing exits non-zero instead of looking like a
pass.

## What this proves

Each commit is timestamped by GitHub when the bucket is published. To verify that
Shingou never rewrote history: fetch a historical bucket from the API, recompute
the hash from the fields above, and check it equals the line committed at publish
time. A changed value would produce a different hash than the one committed.
Visibly.

It proves **forward from first deploy**. The first committed bucket is
`2026-07-04T08:00:00.000Z`. The API serves 11 signals
from earlier that same day, at buckets `00:00`, `04:00` and `07:00` UTC, and none
of them have a line here. Four of the 11 were collected live and seven are
labeled `reconstructed: true`. These are not lines that went missing. They were
already written when hash publishing started. Checked across the whole log on
2026-09-13, they are the only buckets the API serves without a committed hash.

Buckets labeled `reconstructed: true` (the archival backfill) are written once
and labeled as reconstructions. They are sold and documented as exactly that,
not as live-collected history.

## Log events

**2026-07-06T05:00Z was published twice.** The hourly run committed 30 hashes at
05:01 UTC. The engine v2 rollout re-dispatched the same hour at 05:28 UTC, and
the re-scored values differed on 26 of 30 symbols. The re-run overwrote the
served bucket and committed the new hashes. Both sets of lines remain in the
log, because lines are never removed. The API's current values recompute to the
05:28 hashes; the 05:01 lines are superseded. This is the log doing its job: a
re-publish leaves permanent, timestamped evidence. `verify.mjs` reports
superseded lines separately from mismatches.

**2026-07-18T11:00Z was published twice for the same reason.** A dispatcher outage
that day ended with a catch-up run re-scoring the hour; the re-scored values
differed on 6 of the bucket's symbols, and those earlier lines are superseded.
Both re-publishes are ordinary log events, not corrections to history: the
superseded lines stay in the files forever.

**2026-09-12T19:00Z and 2026-09-12T23:00Z were published twice, and the cause is
worth naming.** Both hours ingested, scored and committed their hashes on time.
What failed was the small row the engine writes to record that the hour ran. The
database returned a gateway timeout, and that write is allowed to fail quietly so
a bookkeeping outage can never stop ingestion. The scheduler reads those rows to
decide whether an hour still needs a run. With the row missing it saw two hours
that had never run, re-dispatched them at half past, and the second pass re-ran
the hour from scratch. Five lines are superseded, three on 19:00 and two on
23:00. The API serves the values from the second pass. The bookkeeping write now
retries, and a run that loses it anyway records itself when it finishes, so a
timeout costs a minute of blindness instead of a whole second run.

As of 2026-09-13 the log holds 13,982 lines. 37 of them are superseded, across
the four buckets above.

## Known caveat: lines before 2026-07-05T12:00Z

Lines committed before bucket `2026-07-05T12:00:00.000Z` were hashed over the
engine's full-float64 values, but the database columns store 32-bit floats. That
covers the log's first 28 hours, from `2026-07-04T08:00:00.000Z` to
`2026-07-05T11:00:00.000Z`, and 80 lines. 58 of the 80 lost precision on the way
into the database, so **the committed hash does not recompute from API reads**.
The other 22 landed on values a 32-bit float stores exactly, and those still
verify. This was a precision bug in hash construction, not a rewrite: the commit
timestamps still prove *when* each bucket was published, and every one of those
rows is still marked live-collected with the creation time it had on the day.
Fixed in engine commit `403950d` (signal fields are now rounded to a float4-safe
6 decimals before storing and hashing). Every line from bucket
`2026-07-05T12:00:00.000Z` onward recomputes exactly. The pre-fix lines are left
in place unaltered. This log is append-only.
