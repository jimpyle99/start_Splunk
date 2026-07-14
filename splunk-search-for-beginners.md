# Splunk Search for Beginners

A starter reference for searching in Splunk, compiled from Splunk's official
documentation (help.splunk.com / docs.splunk.com). Use it as a quick guide
alongside the hands-on **Search Tutorial**, which walks you through a real
dataset (the "Buttercup Games" sample data) step by step.

> **Where to practice:** Splunk recommends learning on a free **Trial** version
> of Splunk Enterprise or Splunk Cloud Platform, loaded with the tutorial's
> sample data. This keeps your results consistent with the docs and keeps
> practice data separate from any real work data. The Enterprise Trial converts
> to a perpetual Free version after 60 days.

---

## 1. The big picture

The **Search & Reporting app** (the "Search app") is the main interface for
running searches, saving reports, and building dashboards. When you add data to
Splunk it gets **indexed** — and as part of indexing, Splunk extracts
information into name/value pairs called **fields**. Searching is mostly about
retrieving events from the index and then shaping them with fields and commands.

A few core terms:

- **Event** — a single record (e.g. one line of a web access log), with a timestamp.
- **Index** — where your indexed data lives; searches retrieve events from it.
- **Field** — a name/value pair extracted from your data (e.g. `status=404`).
- **SPL** — the Search Processing Language; the syntax you type into the search bar.

### Coming from SQL or databases?

A lot of newcomers expect Splunk to behave like a relational database, and that
mental model causes friction. The key differences:

- **No schema up front.** You don't define tables and columns before loading
  data. Splunk stores raw events and extracts fields *at search time*
  ("schema-on-read"). The same data can be interpreted different ways later.
- **You read left-to-right, not inside-out.** SQL nests (`SELECT ... FROM (SELECT
  ...)`). SPL flows through pipes: *get events → transform → transform*.
- **Rough translations:** a `WHERE` clause ≈ your base search terms; `GROUP BY` +
  aggregate ≈ `stats ... by`; `SELECT` columns ≈ `table` or `fields`; `ORDER BY`
  ≈ `sort`. Joins exist (`join`, `lookup`) but are used far less than in SQL —
  `stats` usually does what you'd reach for a join to do.
- **Time is a first-class citizen.** Every event has a timestamp and every search
  has a time range. There's nothing quite like it in standard SQL.

---

## 2. The Search Assistant

As you type in the search bar, the **Search Assistant** helps you out — think of
it as autocomplete with extras:

- Type a few letters and it shows **terms that actually exist in your data**.
  Typing `category`, for example, surfaces real values like `categoryid=sports`.
- It shows **matching searches** from your recent history, so you can re-run
  yesterday's search easily. Your history is kept even after you log out.
- Once you start typing **commands**, it displays command syntax and help.

It becomes much more useful as you learn the language, because it explains
commands as you type them.

---

## 3. Basic keyword searches

To find events, just type keywords. Some rules to internalize early:

**AND is implied.** Typing multiple keywords automatically ANDs them together:

```
buttercupgames error
```

is the same as:

```
buttercupgames AND error
```

**Boolean operators must be UPPERCASE** — `AND`, `OR`, `NOT`. Lowercase `and`
is treated as a search term, not an operator.

**Wildcards** use the asterisk `*`. So `fail*` matches `fail`, `failure`,
`failed`, `failing`, and so on.

> **Avoid leading wildcards.** `*error` (wildcard at the *start*) forces Splunk
> to scan far more data and is slow. Trailing wildcards like `error*` are fine.

**Quotes** matter when your search term contains spaces or special characters,
or when you want an exact phrase. `"login failed"` matches that exact phrase;
without quotes, `login failed` is `login AND failed` anywhere in the event. Quote
values with spaces in field searches too: `status_msg="Not Found"`.

**Are searches case-sensitive?** Keyword searches are **not** — `ERROR`, `error`,
and `Error` all match the same events. But Boolean operators (`AND`/`OR`/`NOT`)
must be uppercase, and *field values* often are case-sensitive (see §5). This
split trips people up constantly.

**Parentheses group terms.** A full example:

```
buttercupgames (error OR fail* OR severe)
```

This finds events mentioning `buttercupgames` that also mention any of `error`,
`fail`-anything, or `severe`.

### Operator precedence

When Splunk evaluates Boolean logic, the order is:

1. Terms inside **parentheses** (highest precedence)
2. **NOT**
3. **OR**
4. **AND** (lowest precedence)

When in doubt, add parentheses to make your intent explicit.

> **Tip:** You can copy search strings straight from the web tutorial into the
> search bar. Avoid copying from the *PDF* version — hidden formatting
> characters can break the search.

---

## 4. Understanding your results

After a search runs, results appear under four tabs. **Which tab fills in
depends on the commands you use:**

| Tab | What it shows |
|-----|---------------|
| **Events** | The raw matching events as a list. This is where keyword searches land. |
| **Patterns** | The most common patterns among your events (groups of similarly-structured events). |
| **Statistics** | A results table — populated when you use **transforming commands** like `stats`, `top`, or `chart`. |
| **Visualization** | Charts built from transforming-command results. |

A plain keyword search (no transforming commands) only fills the **Events** tab.

> **"Why is my Statistics tab empty?"** Because you haven't used a transforming
> command yet. The Statistics and Visualization tabs only populate after `stats`,
> `chart`, `timechart`, `top`, `rare`, or similar. A plain search fills Events
> only.

### The Events viewer

By default, events show newest-first as a **List**, with your search terms
highlighted. Each event has three columns:

- ***i*** — expand/collapse the event details (click the `>` to expand).
- **Time** — the event's timestamp (extracted at index time; if the event has
  no timestamp, Splunk adds the index time).
- **Event** — the raw event data.

You can switch the display from **List** to **Table** to see one column per
selected field.

### The Timeline

Above your results, the **timeline** is a bar chart of how many events occurred
over time. Tall bars = lots of events; gaps = quiet periods. Spikes or valleys
can flag activity bursts or downtime. You can zoom in/out and change the scale.

### The Fields sidebar

Fields extracted from your results are listed on the left in two groups:

- **Selected fields** — shown inline in each event. By default these are
  `host`, `source`, and `sourcetype`. You can add more.
- **Interesting fields** — other fields Splunk extracted that you can promote
  to selected.

**"Why don't I see the field I expect?"** A few common reasons: none of the
events in your current results actually contain it; it only appears in a small
fraction of events (Splunk hides fields present in <20% of results — click **All
Fields** to reveal them); or the field simply isn't extracted for that
sourcetype yet and needs a field extraction.

**Fields that start with an underscore** are internal fields Splunk maintains:

- `_time` — the event's timestamp (what the timeline and `timechart` use).
- `_raw` — the original, unparsed event text.
- `_indextime` — when Splunk indexed the event (may differ from `_time`).

They're hidden from the sidebar by default but you can search and use them like
any other field (e.g. `| table _time, _raw`).

---

## 5. Searching with fields

Keyword search is just the start. Searching by **field** is more precise and is
the heart of SPL. The basic comparison syntax is `field=value`:

```
status=404
categoryid=sports
status>=500
```

Field comparisons can be combined with Booleans just like keywords:

```
status=404 OR status=500
sourcetype=access_combined status!=200
```

**`error` vs `status=error` — what's the difference?** Typing `error` alone is a
*keyword* search: Splunk looks for that string **anywhere** in the raw event.
Typing `status=error` is a *field* search: it only matches events where the
extracted field `status` has exactly that value. Field searches are more precise
and usually faster, because they target a specific extracted value instead of
scanning all the raw text.

Field names are case-sensitive, and field **values** frequently are too
(`status=OK` may not match `status=ok`) — this differs from plain keyword search,
which is case-insensitive. When in doubt, check the actual values in the Fields
sidebar rather than guessing.

Use the Fields sidebar to discover what fields and values exist.

### The metadata fields: `index`, `host`, `source`, `sourcetype`

A small set of fields is assigned to **every event at index time** — before
Splunk ever reads the raw text. Because Splunk knows them up front, filtering on
them is the cheapest, fastest way to narrow a search, which is why they belong at
the **front of your base search**. They form a natural hierarchy:

| Field | Answers | Example |
|-------|---------|---------|
| `index` | *Where* is it stored? | `web` |
| `host` | *Which machine* produced it? | `web01` |
| `source` | *Which file / input* did it come from? | `/var/log/apache/access.log` |
| `sourcetype` | *What format / type* is it? | `access_combined` |

- **`index`** is the named bucket of data on disk. If you don't specify one, you
  only search the indexes your **role** allows by default — so naming it avoids
  silently missing data. `index=*` searches everything you can access (broader
  and slower).
- **`host`** is the machine/source system the data came from.
- **`source`** is the specific origin — a file path, network port, or script.
- **`sourcetype`** is the *format* of the data, and it's special: it drives how
  Splunk finds timestamps, breaks events, and **extracts fields**. Fields like
  `status`, `clientip`, and `bytes` exist *because* of the sourcetype's parsing
  rules. Many different `source`/`host` values can share one `sourcetype`.

Putting them together, a well-shaped base search looks like:

```
index=web host=web01 sourcetype=access_combined status>=500
| stats count by uri_path
| sort -count
```

All of these support wildcards, negation, and OR:

```
host=web*                     ← all hosts starting with "web"
sourcetype=cisco:*            ← cisco:asa, cisco:ios, etc.
host=web01 OR host=web02
NOT host=web01
```

**Discovering what exists** on an unfamiliar deployment:

```
index=web | stats count by sourcetype       ← sourcetypes in one index
| metadata type=sourcetypes index=*         ← sourcetypes across all access
```

> **Note on the tutorial data:** the beginner Search Tutorial rarely uses
> `index=` because all its sample data loads into the default `main` index. On a
> real multi-index deployment, leading with `index=` and `sourcetype=` becomes
> essential for both correctness and speed.

---

## 6. The pipe: chaining commands

This is the single most important concept in SPL. The pipe character `|` sends
the results of one command into the next, left to right — like a Unix pipeline.
A search reads as: *get these events → then do this → then do this.*

```
sourcetype=access_combined status=404
| stats count by uri
| sort -count
```

Read aloud: "Find 404 events, count them grouped by URL, then sort by count
descending."

The first part (before any pipe) is the **base search** that retrieves events.
Everything after a pipe **transforms** what you already have.

### Commonly-used commands for beginners

| Command | What it does | Example |
|---------|--------------|---------|
| `stats` | Aggregate (count, sum, avg, etc.), optionally grouped | `... \| stats count by status` |
| `top` | Most common values of a field | `... \| top limit=10 clientip` |
| `rare` | Least common values | `... \| rare user` |
| `timechart` | Aggregate over time (great for trends) | `... \| timechart count by status` |
| `chart` | Aggregate across two dimensions | `... \| chart count over host by status` |
| `table` | Pick specific columns to display | `... \| table _time, clientip, status` |
| `fields` | Keep or drop fields (speeds searches) | `... \| fields status, uri` |
| `sort` | Order results (`-` = descending) | `... \| sort -count` |
| `dedup` | Remove duplicate values of a field | `... \| dedup clientip` |
| `eval` | Create or modify a field | `... \| eval kb=bytes/1024` |
| `where` | Filter using expressions | `... \| where count > 100` |
| `rename` | Rename a field | `... \| rename clientip AS "Client IP"` |
| `head` / `tail` | Keep first / last N results | `... \| head 20` |

`stats`, `top`, `chart`, and `timechart` are **transforming commands** — they
reshape events into a results table, which is what populates the Statistics and
Visualization tabs.

---

## 7. The `stats` command in depth

`stats` is the workhorse of SPL. It collapses many individual events into a
**summary table** — counts, sums, averages, unique counts, and more. Because it's
a transforming command, it's what fills the Statistics and Visualization tabs.

The basic shape:

```
... | stats <function>(<field>) by <field>
```

The **`by` clause is "group by."** Without it you get one number for the whole
result set; with it you get **one row per distinct value** of the grouping field.

```
index=web sourcetype=access_combined | stats count             ← single number
index=web sourcetype=access_combined | stats count by status   ← one row per status code
```

### Common aggregation functions

| Function | What it gives you |
|----------|-------------------|
| `count` | Number of events |
| `count(field)` | Number of events where that field has a value |
| `dc(field)` | Distinct (unique) count — e.g. `dc(clientip)` = unique visitors |
| `sum(field)` | Total of a numeric field |
| `avg(field)` | Mean |
| `min(field)` / `max(field)` | Smallest / largest value |
| `values(field)` | List of the **unique** values seen |
| `list(field)` | List of **every** value (with duplicates) |
| `first(field)` / `latest(field)` | First / most recent value in the group |

### Worked examples

Requests and unique visitors per page, with renamed output columns:

```
index=web sourcetype=access_combined
| stats count, dc(clientip) AS unique_visitors by uri_path
```

Multiple stats grouped by two fields (one row per *combination*):

```
index=sales
| stats sum(price) AS revenue, avg(price) AS avg_sale by product, category
```

Total bytes per host, biggest first:

```
index=web sourcetype=access_combined
| stats sum(bytes) AS total_bytes by host
| sort -total_bytes
```

### Things beginners stumble on

- `count` is the **only** function you can use bare. Everything else needs a
  field: `sum(bytes)`, not `sum`.
- After `stats` runs, **the original events are gone** — you only have the
  summary table. Anything downstream (`where`, `sort`, display) must reference
  the **aggregated** fields, not the raw event fields.
- Use `AS` to rename outputs (`dc(clientip) AS unique_visitors`) so tables and
  charts read cleanly.

### `stats` vs `chart` vs `timechart`

All three aggregate; they differ in how they lay out the result:

| Command | Output shape | Use when |
|---------|--------------|----------|
| `stats` | Flat table, one row per `by` group | General aggregation |
| `chart` | Matrix — split across two dimensions (`... over X by Y`) | Comparing one category against another |
| `timechart` | `stats` with **time as the grouping** automatically | Trends over time (line/area charts) |

The same question, three ways:

```
... | stats count by status                 ← table of counts per status
... | chart count over host by status        ← hosts as rows, statuses as columns
... | timechart count by status              ← status counts plotted over time
```

A simple rule: `stats count by X` answers *"how many of each X?"*; swap `count`
for another function to answer *"what's the total / average / unique count of Y
for each X?"*

---

## 8. Trends over time: `timechart` and visualizations

`timechart` is `stats` with **time built in as the grouping** — it buckets your
events into time intervals and aggregates each bucket. That's exactly what a line
or area chart needs, so `timechart` is the go-to command for trends.

```
index=web sourcetype=access_combined
| timechart count by status
```

This produces one column per status code, one row per time bucket — plot it and
each status becomes its own line over time.

### The `span` option

Splunk picks a bucket size automatically based on your time range, but you can
set it explicitly with `span`:

```
... | timechart span=1h count          ← hourly buckets
... | timechart span=5m avg(bytes)     ← 5-minute buckets
... | timechart span=1d count by host  ← daily, split by host
```

Smaller spans = more detail but more points; larger spans = smoother trends.

### A note on `by` and series

With `timechart`, the `by` field becomes the **series** (the separate lines).
Splunk limits how many series it draws by default (the rest roll into an
`OTHER` column); you can raise or remove that with `limit`:

```
... | timechart count by uri_path limit=10     ← top 10 paths as lines
... | timechart count by uri_path limit=0      ← every path, no OTHER bucket
```

### Turning results into a visualization

Any transforming command (`stats`, `chart`, `timechart`) populates the
**Visualization** tab. The workflow:

1. Run a search that ends in a transforming command.
2. Click the **Visualization** tab.
3. Pick a chart type (line, column, bar, pie, area, single value, etc.).
4. Use **Format** to adjust axes, legend, colors, and labels.
5. **Save As → Dashboard Panel** (or Report) to keep it.

Rough guidance on chart types: **line/area** for trends over time
(`timechart`), **column/bar** for comparing categories (`stats ... by`),
**pie** for parts of a whole (use sparingly), and **single value** for one
headline number (`stats count`).

---

## 9. Calculating and filtering: `eval` and `where`

These two commands unlock most "real" SPL. `eval` **creates or modifies a
field**; `where` **filters results using an expression**.

### `eval` — make a new field

```
... | eval kb = bytes / 1024
... | eval full_url = host . uri_path          ← "." concatenates strings
... | eval status_type = if(status>=500, "error", "ok")
```

`eval` supports arithmetic, string functions, and conditionals. Two especially
common functions:

- **`if(condition, then, else)`** — branch on a test.
- **`case(c1, v1, c2, v2, ...)`** — multiple branches:

```
... | eval severity = case(
        status>=500, "server error",
        status>=400, "client error",
        true(), "ok")
```

(`true()` acts as the catch-all "else".)

### `where` — filter with an expression

`search` (the implicit base) filters events; `where` filters **after** you've
computed or aggregated things, using comparison expressions:

```
index=web sourcetype=access_combined
| stats count by clientip
| where count > 100                 ← only IPs with more than 100 requests
```

Because `where` evaluates expressions, you can compare **two fields** to each
other — something a base search can't do:

```
... | where bytes_out > bytes_in
```

### How they work together

A typical pattern is *aggregate → calculate → filter*:

```
index=web sourcetype=access_combined
| stats count AS total, count(eval(status>=500)) AS errors by host
| eval error_rate = round(errors / total * 100, 2)
| where error_rate > 5
| sort -error_rate
```

Read aloud: "Per host, count total requests and server errors, compute an error
rate percentage, keep only hosts above 5%, worst first." That's `stats`, `eval`,
and `where` doing exactly the jobs they're built for.

> **`eval` vs `where` quick rule:** if you're *making or changing a field*, use
> `eval`. If you're *deciding which rows to keep*, use `where`.

---

## 10. Time ranges

Every search runs over a **time range**, set with the time picker next to the
search bar. For learning on the static tutorial data, you'll often set this to
**All time** so older sample events are included. In real use, narrowing the
time range is one of the easiest ways to make searches faster.

### Setting time in the search itself

Besides the picker, you can bound time inline with `earliest` and `latest` using
relative-time syntax:

```
... earliest=-24h                     ← last 24 hours
... earliest=-7d@d latest=@d          ← last 7 whole days, up to midnight today
... earliest=-15m                     ← last 15 minutes
```

The units are `s`, `m`, `h`, `d`, `w`, `mon`, `y`. The `@` **snaps** to a
boundary — `@d` = start of today, `-1h@h` = the top of the previous hour.

### "Why are my timestamps or timezones wrong?"

A frequent real-world headache. Splunk stores `_time` in UTC internally and
displays it in **your user profile's timezone**, so two users can see different
clock times for the same event. If timestamps look shifted or events land on the
wrong day, the usual culprit is a timezone mismatch between the data source, the
forwarder, and your profile — set the timezone correctly at the sourcetype level
so events are interpreted right at index time.

### Real-time vs historical

Most searches are **historical** (they run over a fixed past window). Splunk also
offers **real-time** searches that update continuously as new events arrive.
Real-time is resource-hungry and unavailable on some licenses, so beginners
should stick to historical searches with a sensible time range.

---

## 11. Search modes: Fast, Smart, and Verbose

Next to the search bar is a mode selector that changes how much work Splunk does
and how many fields it returns. It's a common source of "where did my fields go?"
confusion.

| Mode | What it does | Use when |
|------|--------------|----------|
| **Fast** | Skips field discovery — returns only fields your search explicitly references. Quickest. | You know exactly what you need; large searches. |
| **Smart** (default) | Behaves like Verbose for event searches and Fast for transforming searches. A good all-rounder. | Most of the time — leave it here. |
| **Verbose** | Discovers and returns **every** possible field. Slowest. | Exploring unfamiliar data when you want to see everything. |

If you run a bare search and expected to see lots of fields in the sidebar but
don't, check whether you're in **Fast** mode — switching to **Smart** or
**Verbose** will surface them.

---

## 12. A beginner's mental model for building a search

1. **Start broad** with keywords or a `sourcetype` to find the right events.
2. **Narrow** with field comparisons (`status=404`, `host=web01`).
3. **Set the time range** to something sensible.
4. **Pipe** into a transforming command (`stats`, `timechart`) to summarize.
5. **Sort / limit / format** the output (`sort`, `head`, `table`, `rename`).
6. **Save** it as a report or **visualize** it on a dashboard once it's useful.

Example end-to-end:

```
sourcetype=access_combined (error OR fail* OR status>=500)
| timechart count by status
```

---

## 13. Troubleshooting: "Why do I get no results?"

The single most common beginner frustration. When a search returns nothing,
it's almost always one of these — check them in order:

1. **Time range.** By far the #1 cause. Your events may be outside the selected
   window. Widen it (try **All time**) to confirm the data is there at all, then
   narrow back down.
2. **Wrong index / an index you can't see.** If you didn't specify `index=`, you
   only searched your role's default indexes. Try naming the index explicitly, or
   `index=*` to check everywhere you have access.
3. **Value case or exact-match mismatch.** `status=OK` won't match `status=ok`,
   and `sourcetype=Access_Combined` won't match `access_combined`. Confirm the
   real value in the Fields sidebar.
4. **Wrong field name, or the field isn't extracted.** `statuscode=404` returns
   nothing if the field is actually called `status`. Check the sidebar for the
   real field name.
5. **Typos / mismatched quotes or parentheses.** An unbalanced `(` or `"` can
   silently break a search. The Search Assistant highlights syntax issues.

A good habit: **strip the search back to just the base** (e.g. `index=web`),
confirm you get events, then add one filter at a time until the results
disappear — the last thing you added is the culprit.

---

## 14. Making searches faster

Splunk searches can range from instant to painfully slow. The governing
principle is **filter early, filter cheap** — cut the data down as soon as
possible so later commands work on less.

- **Narrow the time range first.** The cheapest possible filter; a smaller window
  means less data to scan.
- **Lead with index-time fields** — `index`, `sourcetype`, `host`, `source`.
  Splunk uses these before reading raw text (see §5).
- **Be specific early.** Add your most restrictive filters at the front of the
  base search rather than filtering with `where` after the fact.
- **Avoid leading wildcards** (`*term`) — they defeat fast lookups.
- **Trim fields early** with `| fields field1, field2` so downstream commands
  carry less data.
- **Prefer `stats` over raw event retrieval** when you only need the summary —
  transforming commands are efficient and you rarely need every raw event.
- **Use a narrower search mode** (Fast) when you don't need field discovery.

Rule of thumb: if a search is slow, the fix is usually "search less" — less
time, fewer indexes, more specific terms — not a cleverer command.

---

## 15. Beyond the basics (what to learn next)

The Search Tutorial continues into:

- **Subsearches** — run a search whose results feed into another search.
- **Lookups** — enrich events with extra data from a table (e.g. map a product
  ID to a product name).
- **Field extraction** (`rex`, the field extractor) — teach Splunk to pull new
  fields out of raw text.
- **`eventstats` / `streamstats`** — add aggregate values alongside events
  without collapsing them the way `stats` does.
- **Reports** — save a search to re-run or schedule.
- **Charts & dashboards** — turn searches into visual panels.

---

## 16. Official resources

- **Search Tutorial** (hands-on, start here):
  https://docs.splunk.com/Documentation/Splunk/latest/SearchTutorial/WelcometotheSearchTutorial
- **Get started with Search:**
  https://docs.splunk.com/Documentation/Splunk/latest/Search/GetstartedwithSearch
- **Basic searches and search results:**
  https://docs.splunk.com/Documentation/Splunk/latest/SearchTutorial/Startsearching
- **Search Reference** (full catalog of SPL commands):
  https://help.splunk.com/en/?resourceId=Platform_SearchReference_WhatsInThisManual
- **Splunk Community / Splunk Answers:** https://community.splunk.com/
- **Splunk Lantern** (use cases & guidance): https://lantern.splunk.com/

> Splunk's newer docs live on **help.splunk.com**; older permalinks on
> **docs.splunk.com** still resolve. Versions differ slightly between Splunk
> Enterprise and Splunk Cloud Platform, but the search basics above are the same
> for both.

---

## Printable cheat-sheet

A one-page quick-reference distilling this lesson — search anatomy, metadata
fields, syntax rules, the command and `stats`-function tables, time-range syntax,
search modes, the "no results?" checklist, and performance tips — is available as
a companion PDF: **`splunk-search-cheatsheet.pdf`**. Print at 100% on US Letter
and keep it next to you while you practice.

---

*Compiled from Splunk's official Search Tutorial and Search documentation. Splunk
and SPL are trademarks of their respective owner; this is an independent study
summary, not an official Splunk publication.*
