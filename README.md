# Survivor · fifty seasons, one app

Every season of Survivor as a single interactive document. Season summaries,
tribe composition, boot order, the full tribal-council vote matrix, challenge
results, confessional airtime, and castaway percentile scores.

**Live site:** https://astrodataus.github.io/survivor-reality/

Built by [Astrodata](https://astrodata.us). The same document also runs as an
Omni app; this repository is the static build of it.

Demo data. Not affiliated with the broadcaster.

---

## One file, no requests

`index.html` is the entire application: markup, styles, script, typefaces and
all 11,290 rows of data. Loading it makes **exactly one request**, for the
document itself. Nothing is fetched from anywhere.

That is deliberate rather than minimalist. The Omni app sandbox does not
allowlist CDNs, so anything loaded from one works on your machine and fails
silently in production. Two things were being loaded that way in the published
app, and both are now inlined:

| Was | Now |
|---|---|
| d3 7.9.0 from `cdn.jsdelivr.net`, 280 KB | a 13 KB bundle of `d3-selection` and `d3-array` |
| Jost and JetBrains Mono from Google Fonts | 51 KB of subset woff2, base64 inlined |

The app calls `d3.select` and `d3.min`. Nothing else. Shipping the whole library
to get two functions was never necessary, and inside Omni it was not even
arriving, so the published app had been falling back to system fonts and to the
no-network branch of the vote network.

Both typefaces are SIL Open Font License. d3 is ISC. Both permit redistribution.

Total: 695 KB, **196 KB gzipped**, which is what a browser actually pulls.

---

## Linking to a season

State lives in the URL fragment, so any view can be linked or bookmarked:

```
index.html#season=28&tab=voting
```

`season` is 1 to 50. `tab` is one of `seasons`, `voting`, `challenges`,
`confessionals`, `scores`. Omitting either falls back to season 50 on the
seasons tab. The back button works.

In the Omni build the same state is carried by `omni.setParams`, which has no
equivalent outside the sandbox. The application code is identical in both; only
the runtime underneath differs.

---

## Where the data comes from

MotherDuck's `sample_survivor` tables, read through an Omni semantic model,
exported as eleven query results and frozen here. Figures are a snapshot, not a
live read, and the app says so rather than implying otherwise.

| Query | Rows | Grain |
|---|---|---|
| All Seasons List | 50 | season |
| Season Overview | 50 | season |
| Voting Stats by Season | 50 | season |
| Challenge Stats by Season | 148 | season × challenge type |
| Boot Order | 930 | season × castaway |
| Castaway Scores | 917 | season × castaway |
| Top Confessionals by Castaway | 917 | season × castaway |
| Challenge Detail | 1,156 | season × episode × challenge |
| Castaway Tribe Colors | 1,189 | season × castaway |
| Tribe Breakdown | 1,189 | season × castaway |
| Vote Network | 4,694 | season × voter × target |

Columns keep their Omni field IDs, `my_db_sample_survivor__<table>.<column>`, so
any figure on screen can be traced back to a query without a translation step.

**Vote Network is the one to read carefully.** `count` is votes cast by a
castaway against another *within a season*. It has to be keyed by season: the
same two players can meet across multiple seasons, and pooling them silently
sums two different confrontations into one number.

Scores and index counts are rounded to four decimal places, confessional
averages to two, mean viewers to whole viewers. That rounding happens once, at
build time, before anything is written.

---

## Rebuilding

```
python3 build_data.py    # eleven CSVs in raw/ -> app-data.json
python3 build.py         # -> dist/index.html, standalone, and the Omni build
python3 verify_web.py    # serves dist/ over HTTP and checks it
```

`verify_web.py` also takes an origin, which is how the deployed site gets
checked against its own bytes rather than against the local build:

```
python3 verify_web.py https://astrodataus.github.io/survivor-reality/
```

It asserts the request count, that nothing leaves the origin, that every tab
draws, that the vote network has edges and nodes, that deep links restore state,
and that headline figures match the source CSVs rather than the app's own state.

### Re-exporting the data

Three queries need a **Season** field added before export, in a draft, because
the live app filters by season server side and so never selected it: Boot Order,
Tribe Breakdown and Vote Network. Without it the first two cannot be attributed
to a season at all, and Vote Network is actively wrong rather than merely
incomplete. Set Row limit to **All possible results** on every export; the
default stops at 1,000 rows without saying so.
