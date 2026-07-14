<!--
author:   Splunk Search for Beginners
version:  1.0.0
language: en
narrator: US English Female

comment:  An interactive beginner's course on searching in Splunk with SPL —
          covering keyword search, fields, the pipe, stats, timechart, eval/where,
          time ranges, search modes, and troubleshooting, with quizzes throughout.

logo:     https://liascript.github.io/course/images/showcase.png
-->

# Splunk Search for Beginners

An interactive introduction to searching in Splunk with the **Search Processing
Language (SPL)**. Work through the sections in order — each ends with a quick
check so you can test yourself as you go.

--{{0}}--
Welcome to Splunk Search for Beginners. This course walks you through the
fundamentals of searching in Splunk, from your first keyword search all the way
to summarizing data with commands like stats and timechart. Each section ends
with a short quiz so you can check your understanding before moving on.

                        --{{0}}--
> **How to use this course:** switch between **Textbook**, **Presentation**, and
> **Slides** modes using the controls at the top. Presentation mode reads the
> content aloud. Answer the quizzes inline — you get instant feedback.

> **Where to practice:** Splunk recommends learning on a free **Trial** version of
> Splunk Enterprise or Splunk Cloud Platform, loaded with the "Buttercup Games"
> tutorial sample data. The Enterprise Trial converts to a perpetual **Free**
> version (500 MB/day) after 60 days.

## 1. The Big Picture

The **Search & Reporting app** (the "Search app") is the main interface for
running searches, saving reports, and building dashboards. When you add data to
Splunk it gets **indexed** — and as part of indexing, Splunk extracts information
into name/value pairs called **fields**. Searching is mostly about retrieving
events from the index and then shaping them with fields and commands.

Core terms
==========

- **Event** — a single record (e.g. one line of a web access log), with a timestamp.
- **Index** — where your indexed data lives; searches retrieve events from it.
- **Field** — a name/value pair extracted from your data (e.g. `status=404`).
- **SPL** — the Search Processing Language; the syntax you type into the search bar.

Coming from SQL or databases?
=============================

A lot of newcomers expect Splunk to behave like a relational database, and that
mental model causes friction:

- **No schema up front.** You don't define tables and columns before loading data.
  Splunk stores raw events and extracts fields *at search time*
  ("schema-on-read"). The same data can be interpreted different ways later.
- **You read left-to-right, not inside-out.** SQL nests; SPL flows through pipes:
  *get events → transform → transform*.
- **Rough translations:** `WHERE` ≈ your base search terms; `GROUP BY` + aggregate
  ≈ `stats ... by`; `SELECT` columns ≈ `table`/`fields`; `ORDER BY` ≈ `sort`.
- **Time is a first-class citizen.** Every event has a timestamp; every search has
  a time range.

### Check your understanding

In Splunk, when are fields extracted from your data?

[( )] Up front, when you define a schema before loading data
[(X)] At search time — Splunk stores raw events ("schema-on-read")
[( )] Never — Splunk cannot extract fields
[[?]] Think about how Splunk differs from a traditional SQL database.
***
Splunk uses **schema-on-read**: it stores the raw events and extracts fields when
you search, so the same data can be interpreted different ways later. This is a
key difference from relational databases, which require a schema up front.
***

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

## 3. Basic Keyword Searches

To find events, just type keywords. Some rules to internalize early:

**AND is implied.** Typing multiple keywords automatically ANDs them together —
`buttercupgames error` is the same as `buttercupgames AND error`.

**Boolean operators must be UPPERCASE** — `AND`, `OR`, `NOT`. Lowercase `and` is
treated as a search term, not an operator.

**Wildcards** use the asterisk `*`. So `fail*` matches `fail`, `failure`,
`failed`, and so on.

> **Avoid leading wildcards.** `*error` (wildcard at the *start*) forces Splunk to
> scan far more data and is slow. Trailing wildcards like `error*` are fine.

**Quotes** matter for phrases or terms with spaces. `"login failed"` matches that
exact phrase; without quotes, `login failed` is `login AND failed` anywhere in the
event.

**Case sensitivity:** keyword searches are **not** case-sensitive (`ERROR` =
`error`), but field *values* often are (see §5).

Operator precedence
===================

When Splunk evaluates Boolean logic, the order is:

1. Terms inside **parentheses** (highest)
2. **NOT**
3. **OR**
4. **AND** (lowest)

A full example:

```
buttercupgames (error OR fail* OR severe)
```

### Check your understanding

Which of these statements about keyword search are **true**?

[[X]] `web error` is treated as `web AND error`
[[X]] `AND`, `OR`, and `NOT` must be written in uppercase
[[ ]] Keyword searches are case-sensitive (`ERROR` ≠ `error`)
[[X]] `fail*` matches `failure`, `failed`, and `failing`
[[?]] Recall the implied-AND rule, the uppercase-Boolean rule, and how wildcards behave.
***
Multiple terms are ANDed by default, Booleans must be uppercase, and trailing
wildcards expand a term. Keyword search is **not** case-sensitive — that's the
one false statement.
***

## 4. Understanding Your Results

After a search runs, results appear under four tabs. **Which tab fills in depends
on the commands you use:**

| Tab               | What it shows                                                        |
|-------------------|---------------------------------------------------------------------|
| **Events**        | The raw matching events as a list. Where keyword searches land.     |
| **Patterns**      | The most common patterns among your events.                         |
| **Statistics**    | A results table — populated by **transforming commands** (`stats`…). |
| **Visualization** | Charts built from transforming-command results.                     |

A plain keyword search (no transforming commands) only fills the **Events** tab.

> **"Why is my Statistics tab empty?"** Because you haven't used a transforming
> command yet. It only populates after `stats`, `chart`, `timechart`, `top`,
> `rare`, or similar.

The Fields sidebar
==================

Fields extracted from your results are listed on the left:

- **Selected fields** — shown inline in each event (default: `host`, `source`,
  `sourcetype`).
- **Interesting fields** — other extracted fields you can promote.

Fields starting with an underscore are internal: `_time` (the timestamp),
`_raw` (original event text), `_indextime` (when it was indexed).

### Check your understanding

You run `index=web error` and expect to see a results **table**, but the
Statistics tab is empty. Why?

[( )] The `web` index doesn't exist
[(X)] You haven't used a transforming command like `stats` yet
[( )] Statistics only works in Presentation mode
[[?]] What kind of command fills the Statistics tab?
***
The Statistics and Visualization tabs only populate after a **transforming
command** (`stats`, `chart`, `timechart`, `top`, `rare`). A plain search fills
the Events tab only.
***

## 5. Searching With Fields

Searching by **field** is more precise than keywords and is the heart of SPL. The
basic comparison syntax is `field=value`:

```
status=404
categoryid=sports
status>=500
```

**`error` vs `status=error`:** typing `error` is a *keyword* search — Splunk looks
for that string **anywhere** in the raw event. `status=error` is a *field* search
— it only matches events where the extracted field `status` has exactly that
value. Field searches are more precise and usually faster.

Field names are case-sensitive, and field **values** frequently are too
(`status=OK` may not match `status=ok`).

The metadata fields
===================

A small set of fields is assigned to **every event at index time** — before
Splunk reads the raw text. Filtering on them is the cheapest, fastest way to
narrow a search, so they belong at the **front** of your base search:

| Field        | Answers                       | Example                      |
|--------------|-------------------------------|------------------------------|
| `index`      | *Where* is it stored?         | `web`                        |
| `host`       | *Which machine* produced it?  | `web01`                      |
| `source`     | *Which file/input*?           | `/var/log/apache/access.log` |
| `sourcetype` | *What format/type* is it?     | `access_combined`            |

`sourcetype` is special: it drives how Splunk finds timestamps, breaks events,
and **extracts fields**. Fields like `status` and `clientip` exist *because* of
the sourcetype's parsing rules.

A well-shaped base search:

```
index=web host=web01 sourcetype=access_combined status>=500
| stats count by uri_path
| sort -count
```

### Check your understanding

Match each metadata field to what it answers:

[[ index ] [ host ] [ sourcetype ]]
[  (X)      ( )     ( )          ] Which named bucket the data is stored in
[  ( )      (X)     ( )          ] Which machine produced the event
[  ( )      ( )     (X)          ] What format the data is, and how fields are extracted
***
`index` = where it's stored, `host` = which machine, `sourcetype` = the format
(and the field extraction rules). All are known at index time, which is why they
make the fastest filters.
***

## 6. The Pipe: Chaining Commands

The single most important concept in SPL. The pipe character `|` sends the
results of one command into the next, left to right — like a Unix pipeline. A
search reads as: *get these events → then do this → then do this.*

```
sourcetype=access_combined status=404
| stats count by uri
| sort -count
```

Read aloud: "Find 404 events, count them grouped by URL, then sort by count
descending."

Common commands
===============

| Command       | What it does                                  |
|---------------|-----------------------------------------------|
| `stats`       | Aggregate (count, sum, avg), optionally grouped |
| `top` / `rare`| Most / least common values of a field         |
| `timechart`   | Aggregate over time (trends)                   |
| `chart`       | Aggregate across two dimensions                |
| `table`       | Pick specific columns to display               |
| `fields`      | Keep or drop fields (speeds searches)          |
| `sort`        | Order results (`-` = descending)               |
| `dedup`       | Remove duplicate values of a field             |
| `eval`        | Create or modify a field                       |
| `where`       | Filter using an expression                     |
| `rename`      | Rename a field                                 |
| `head`/`tail` | Keep first / last N results                    |

### Check your understanding

What does the pipe character `|` do in an SPL search?

[(X)] Feeds the output of one command into the next, left to right
[( )] Comments out the rest of the line
[( )] Combines two separate searches into an OR
[[?]] Think about how a Unix pipeline works.
***
The pipe passes the results of the command on its left into the command on its
right, letting you chain retrieve → transform → format steps.
***

## 7. The `stats` Command

`stats` is the workhorse of SPL. It collapses many individual events into a
**summary table**. It's a transforming command, so it fills the Statistics and
Visualization tabs.

The basic shape — the `by` clause is "group by":

```
... | stats <function>(<field>) by <field>
```

```
index=web sourcetype=access_combined | stats count             (single number)
index=web sourcetype=access_combined | stats count by status   (one row per status)
```

Common functions
================

| Function        | Returns                                     |
|-----------------|---------------------------------------------|
| `count`         | Number of events (the only one usable bare) |
| `dc(field)`     | Distinct (unique) count                     |
| `sum(field)`    | Total of a numeric field                    |
| `avg(field)`    | Mean                                        |
| `min`/`max`     | Smallest / largest value                    |
| `values(field)` | List of the **unique** values              |
| `list(field)`   | List of **all** values (with duplicates)   |

Two things beginners stumble on: `count` is the only function usable without a
field, and after `stats` runs **the original events are gone** — downstream
commands must reference the aggregated fields.

### Check your understanding

Which `stats` function counts the number of **unique** values of a field (e.g.
unique visitor IPs)?

[[dc]]
[[?]] It's a two-letter function, short for "distinct count".
***
`dc(field)` returns the distinct (unique) count — e.g. `dc(clientip)` gives the
number of unique visitors. `count` counts events, not unique values.
***

## 8. Trends Over Time: `timechart`

`timechart` is `stats` with **time built in as the grouping** — it buckets events
into time intervals and aggregates each bucket. That's exactly what a line or area
chart needs.

```
index=web sourcetype=access_combined
| timechart count by status
```

Control the bucket size with `span`:

```
... | timechart span=1h count          (hourly)
... | timechart span=5m avg(bytes)     (5-minute)
```

Any transforming command populates the **Visualization** tab, where you pick a
chart type and save it as a report or dashboard panel. Rough guidance:
**line/area** for trends, **column/bar** for comparing categories, **single
value** for one headline number.

### Check your understanding

Which command would you use to plot how many events occur **over time**?

[( )] `stats`
[(X)] `timechart`
[( )] `dedup`
[( )] `rename`
***
`timechart` automatically groups by time, producing the buckets a line/area chart
needs. `stats` gives a flat table with no built-in time dimension.
***

## 9. Calculating and Filtering: `eval` and `where`

`eval` **creates or modifies a field**; `where` **filters results using an
expression**.

`eval` — make a new field
=========================

```
... | eval kb = bytes / 1024
... | eval status_type = if(status>=500, "error", "ok")
```

`where` — filter with an expression
===================================

`where` filters **after** you've computed or aggregated things, and can compare
two fields to each other:

```
index=web sourcetype=access_combined
| stats count by clientip
| where count > 100
```

A typical pattern is *aggregate → calculate → filter*:

```
index=web sourcetype=access_combined
| stats count AS total, count(eval(status>=500)) AS errors by host
| eval error_rate = round(errors / total * 100, 2)
| where error_rate > 5
| sort -error_rate
```

> **Quick rule:** making or changing a field → `eval`. Deciding which rows to
> keep → `where`.

### Check your understanding

You want to keep only the rows where a computed `error_rate` field is above 5.
Which command do you use?

[( )] `eval`
[(X)] `where`
[( )] `rename`
[[?]] Are you creating a field, or deciding which rows to keep?
***
`where` filters rows using an expression. `eval` would be what you used *earlier*
to create the `error_rate` field in the first place.
***

## 10. Time Ranges

Every search runs over a **time range**, set with the time picker — or inline with
`earliest` and `latest`:

```
... earliest=-24h                 (last 24 hours)
... earliest=-7d@d latest=@d      (last 7 whole days to midnight)
... earliest=-15m                 (last 15 minutes)
```

Units: `s m h d w mon y`. The `@` **snaps** to a boundary — `@d` = start of
today, `-1h@h` = the top of the previous hour.

> **Timezones:** Splunk stores `_time` in UTC and displays it in *your profile's*
> timezone, so two users can see different clock times for the same event. If
> timestamps look shifted, suspect a timezone mismatch set at the sourcetype level.

### Check your understanding

What does `earliest=-24h` do?

[[ (searches the last 24 hours) | searches 24 hours in the future | sorts by hour ]]
***
`earliest=-24h` sets the start of the time range to 24 hours ago, so the search
covers the last day up to now.
***

## 11. Search Modes: Fast, Smart, Verbose

A mode selector next to the search bar changes how much work Splunk does and how
many fields it returns:

| Mode        | What it does                                              | Use when                      |
|-------------|-----------------------------------------------------------|-------------------------------|
| **Fast**    | Only fields your search names. Quickest.                  | You know what you need.       |
| **Smart**   | Default. Balances the two.                                | Most of the time.             |
| **Verbose** | Discovers and returns **every** field. Slowest.           | Exploring unfamiliar data.    |

If you expected lots of fields in the sidebar but don't see them, check whether
you're in **Fast** mode.

## 12. A Mental Model for Building a Search

1. **Start broad** with keywords or a `sourcetype`.
2. **Narrow** with field comparisons (`status=404`, `host=web01`).
3. **Set the time range**.
4. **Pipe** into a transforming command (`stats`, `timechart`).
5. **Sort / limit / format** (`sort`, `head`, `table`, `rename`).
6. **Save** as a report or visualize on a dashboard.

```
sourcetype=access_combined (error OR fail* OR status>=500)
| timechart count by status
```

## 13. Troubleshooting: "Why Do I Get No Results?"

The #1 beginner frustration. When a search returns nothing, check these **in
order**:

1. **Time range** — by far the most common cause. Widen to **All time** to
   confirm the data is there, then narrow back.
2. **Wrong index** / one your role can't see by default. Name `index=` or try
   `index=*`.
3. **Value case or exact-match mismatch** — `status=OK` won't match `status=ok`.
4. **Wrong field name**, or the field isn't extracted. Check the sidebar.
5. **Typos / unbalanced** `(` or `"`.

> **Best habit:** strip back to just the base search, confirm you get events, then
> add one filter at a time until the results disappear — the last thing you added
> is the culprit.

### Check your understanding

Your search returns zero results. Which of these are common causes worth
checking?

[[X]] The selected time range doesn't include the events
[[X]] You're searching an index your role can't see by default
[[X]] A field value's case doesn't match (`OK` vs `ok`)
[[X]] A typo or unbalanced quote/parenthesis
[[?]] Almost all "no results" problems come down to a handful of usual suspects.
***
All four are classic causes. Time range is the most frequent, but index scope,
case mismatches, wrong field names, and syntax typos all produce empty results.
***

## 14. Making Searches Faster

The governing principle is **filter early, filter cheap** — cut the data down as
soon as possible so later commands work on less.

- Narrow the **time range** first.
- Lead with index-time fields: `index`, `sourcetype`, `host`, `source`.
- Put your most specific filters at the front.
- Avoid leading wildcards (`*term`).
- Trim fields early with `| fields field1, field2`.
- Prefer `stats` over pulling every raw event.

Rule of thumb: if a search is slow, the fix is usually "search less" — less time,
fewer indexes, more specific terms — not a cleverer command.

## 15. Final Knowledge Check

Test everything you've learned.

Where should the cheapest, fastest filters (`index`, `sourcetype`) go in a search?

[(X)] At the **front** of the base search, before the first pipe
[( )] At the very end, after all transforming commands
[( )] Order doesn't matter
***
Index-time fields are known before Splunk reads raw text, so filtering on them
first eliminates the most data the earliest.
***

Fill in the command that summarizes events into a grouped table:

The command `[[stats]]` collapses events into a summary table, e.g.
`... | stats count by status`.

***
`stats` is the core aggregation command. Add a `by` clause to group the results.
***

True or false — a plain keyword search with no transforming command will fill the
Statistics tab.

[( )] True
[(X)] False
***
False. Only transforming commands (`stats`, `chart`, `timechart`, `top`, `rare`)
populate the Statistics and Visualization tabs. A plain search fills Events only.
***

Which search would run **fastest** over a busy index?

[( )] `*error*`
[(X)] `index=web sourcetype=access_combined status=500 earliest=-1h`
[( )] `error`
[[?]] Think about leading wildcards, index-time fields, and time range.
***
The middle search leads with index-time fields, a specific status, and a narrow
time range — all cheap filters. Leading-wildcard and bare-keyword searches scan
far more data.
***

## 16. Where to Go Next

Congratulations — you've covered the core of Splunk search! 🎉

Good next topics:

- **Subsearches** — run a search whose results feed into another.
- **Lookups** — enrich events with data from a table.
- **Field extraction** (`rex`, the field extractor) — pull new fields from raw text.
- **`eventstats` / `streamstats`** — aggregates alongside events.
- **Reports & dashboards** — save and visualize your searches.

Official resources
==================

- **Search Tutorial:** https://docs.splunk.com/Documentation/Splunk/latest/SearchTutorial/WelcometotheSearchTutorial
- **Search Reference (all SPL commands):** https://help.splunk.com/en/?resourceId=Platform_SearchReference_WhatsInThisManual
- **Splunk Community:** https://community.splunk.com/
- **Splunk Lantern:** https://lantern.splunk.com/

--{{0}}--
That's the end of the course. You've learned how to search with keywords and
fields, chain commands with the pipe, summarize data with stats and timechart,
calculate and filter with eval and where, work with time ranges and search modes,
and troubleshoot empty results. From here, explore lookups, field extraction, and
dashboards to keep building your skills. Happy Splunking!

> *Independent study summary compiled from Splunk's official Search Tutorial and
> documentation. Splunk and SPL are trademarks of their respective owner; this is
> not an official Splunk publication.*
