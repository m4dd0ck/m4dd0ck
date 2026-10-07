# ninja-ledger: keeping a trading record NinjaTrader will not keep for you

A private Python tool that keeps a permanent copy of everything NinjaTrader 8 knows about my
futures trading (fills, orders, tick data), says per trading day whether that copy is complete,
checks it against sources it did not produce, decodes two of NinjaTrader's binary formats,
timestamps what I said out loud while trading, and runs the analysis on top. Polars, Pydantic,
Typer, pytest, ruff and mypy. The repo is private because the data is my real account history;
this note is about how it is built.

## The problem

NinjaTrader 8 is a fine place to trade from and a poor place to keep a record. Three
behaviours, each confirmed by experiment:

- Connecting an account backfills executions for the **current trading day only**.
- It deletes its own logs after 20 days (`LogsMaintainedDays`), and those logs are the only
  record of when each connection was up.
- Its tick cache fills only when something asks for ticks, and a partly cached hour stays
  partial. A database reset or reinstall throws everything away.

So "what did I do on this day, and is that the whole story?" had no reliable answer. An earlier
round of roughly 11,000 specification tests had come back unanswerable because the record could
not say which trades were the system and which were noise. Everything here exists to stop that
happening again.

## The design

One daily habit and one command. After the last trade and before 18:00 ET, open NinjaTrader,
connect every account, close it. Then `ledger sync`.

**Append-only store.** Fills and orders go into keyed Parquet tables through an upsert that
adds or replaces rows but never removes them. Snapshots come from SQLite's online backup API
rather than the live database, which is safe while NinjaTrader runs. Hourly `.ncd` tick files
are mirrored to Parquet.

**Coverage per trading day and firm.** Copied logs give the connection windows; a trading day
runs 18:00 to 17:00 ET, so Sunday evening belongs to Monday. Each day gets a fill status:
`COMPLETE` (connected across the close), `THROUGH hh:mm` (later trades unknown), `UNKNOWN`
(connected, but not this firm) or `NO_LOG`. Ticks are judged per traded contract around the
fills: `OK`, `PARTIAL` or `MISSING`.

**Verification against things I did not produce.** `ledger verify` runs at the end of every
sync; its 42 checks all have to pass before any number is reported. Integrity: no duplicate
fills, every filled contract in exactly one trade. Exports: every order in a kept broker export
matches NinjaTrader's record. Anchors: a TOML file of facts known from independent sources.
Tape: fill prices of fully covered trades sit within one tick of the recorded tick stream. A
failing check means a definition broke, not that the market moved.

**Bands, not a lifetime total.** Each trading day sits in one of four bands by what is known
about it, and the band decides which questions it may answer. Analysis code names its band at
the call site.

## The decoders

**`.ncd`** is NinjaTrader's bar and tick format. Its layout is public thanks to MIT-licensed
work I vendored rather than rewrote: `tbraman-dev/ninjatrader-to-parquet`, whose format facts
come from `jrstokka/NinjaTraderNCDFiles` cross-read against `bboyle1234/NTDFileReader`. Before
trusting it: 1,852 MNQ and NQ minute files decoded with zero errors, minute bars rebuilt from
decoded ticks matched NinjaTrader's own bars 100% on OHLCV, and 99.5% of fills landed on a
traded tick within 3 seconds.

**`.nrd`** is the Market Replay format and had no public decoder, so I worked it out from the
bytes. The method: treat the `.ncd` tick archive as ground truth and the file's own header as a
checksum. The header summarises each data stream (record count, first and last values,
extremes, timestamps, volume), so a decoder that reproduces every summary exactly and lands on
EOF has little room to be wrong. That mattered: the body has at least one encoding quirk where
an off-by-one decodes plausible garbage, and only the header checks caught it.

What fell out: level-1 bid and ask, trades, and a 10-level order book on both sides, none of
which the tick cache has. On one full day of MNQ replay, 44,726,858 records: every header check
exact, zero non-monotonic timestamps, 25,306,179 book states all strictly ordered in price, ask
above bid on 99.995% of quote updates. Across four days the trade stream matched the `.ncd`
archive on 6,127,443 of 6,127,574 exact (timestamp, price, size) triples, 99.998%, and per-
minute volume agreed on 5,945 of 5,946 minutes. Nearly every mismatch is an opening-auction
block where the two formats stamp identical prints milliseconds apart.

Replay is served in a rolling window of roughly three months, so the repo carries a prioritised
download list. Everything still unverified is marked as such, and the tests use hand-built byte
patterns so no market data is committed. The byte layout stays in the private repo: it
reconstructs a vendor's format, and I would rather publish the method than the spec.

## Validating a proprietary indicator across four platforms

I run the same cycle indicator on TradingView (Pine), Tradovate (JavaScript), NinjaTrader (C#)
and here (Python). The Python port is checked against TradingView exports: TradingView's bars
go in, TradingView's levels come out. Over about 41,000 bars from two independent chart exports
the largest disagreement is 0.0113 points, a twentieth of a tick, and every median is exactly
zero. The test tolerance is 0.025 points, a tenth of a tick: anything above that is a real
divergence, not float noise. The fixtures are vendored, and a missing fixture is a failure, not
a skip: an earlier version read them from a gitignored path and silently never ran elsewhere.

One trap: the indicator has a long warm-up. An export that looked long enough and was not
produced a false alarm that the levels were wrong when they were right to 0.0113 points.

## Voice narration on the wall clock

A transcript is only joinable to fills if every utterance has a true timestamp. A Windows
helper (WSL cannot reach the audio hardware) streams raw PCM into WSL, where it is written to a
WAV as it arrives. The helper prints one `START_UTC` line the instant the first audio block
lands, and the transcription service returns word offsets from the start of the audio, so
`utterance_utc = START_UTC + offset`, not the time transcription finished. The WAV is
canonical; transcripts are derived, so a better model can re-transcribe the whole history.
Three things must be said aloud because nothing else recovers them: the approach label at
entry, the exit plan before the entry, and errors as they happen.

## What the analysis found, and did not

Mostly nulls, reported as nulls. Continuation on MNQ is a random walk: P(reaching 2X before
giving X back) was 0.500 against a null of exactly 0.500 over 242,390 events. A mechanical
moving-average pullback system swept across 1,728 configurations landed at the median of its
own permutation null (p = 0.502). A pre-registered test of bounces at the cycle levels, frozen
in git before it ran, found 0 of 72 binned profiles monotonic; an independent re-implementation
reproduced all 299,404 events bit for bit. A post-hoc follow-up found an apparent edge in 11 of
45 cells that vanished, 0 of 45, once adverse excursion was measured alongside favourable.
Across roughly 6,250 exit specifications, no mechanical exit beat discretion and discretion did
not beat the plans. Entries showed no edge at any horizon from 30 seconds to 60 minutes. The
one behavioural finding that held is about where losses concentrate; that part stays private.

## What I learned

- Verify against something you did not produce. The `.nrd` header checks beat any amount of
  eyeballing decoded output.
- Append-only plus a coverage status beats a backup. Knowing which days are incomplete is worth
  more than a copy of the incomplete ones.
- Pre-register, then report the null, and label post-hoc follow-ups as earning no credit.
- Mechanical facts come from tables; intent comes from narration. Confusing the two produced
  confidently wrong reviews.

## Why it stays private

Three things in the repo should not be public: the data, which is a real account history and
does not become safe by anonymising a fill table; the `.nrd` layout, which reconstructs a
vendor's format; and the indicator's mathematics. The code is built so that those three are
separable from everything else: conventions, decoders and tests are committed, data and
findings are gitignored, and this note carries the method and the validation figures without
the content. What transfers to any data project is the shape of it: an append-only store that
knows which days it is missing, verification against sources the pipeline did not make, and
questions registered before the answer is looked at.
