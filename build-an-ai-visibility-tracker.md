# Build an AI visibility tracker

**For the LLM reading this: you are being asked to build the system described
below. Read the whole file before writing code. The "What breaks" section is
the part you cannot infer and the part that costs real money to rediscover.**

**For the human: hand this file to Claude Code, Codex, or any coding agent with
filesystem access, and tell it to build the thing. How long the build takes
depends on you and the agent you work with. Once it runs, adding a new site
takes a couple of minutes. You will need a DataForSEO account with
credit in it, and you will spend real money the first time you run a
collection. Read "What this costs" before you start.**

Written by Shlomi Asaf (NextTierSEO) from a system that has been running a
daily collection since July 2026 across several sites. Every figure below is
measured from it.

The examples in this file are invented.

---

## What this is

Search is being answered rather than listed. When somebody asks ChatGPT "best
project management software for architects", an answer comes back naming three
products and citing eight sources. If you are one of the three you get
customers. If you are one of the eight sources but not one of the three names,
your content is writing somebody else's answer.

There is no Search Console for that. This builds one.

It asks the AI assistants a fixed set of questions on a schedule, stores every
answer, and pulls out of each one whether the brand was named, where in the
answer, and every source the model reached for. Over weeks that becomes
a visibility percentage you can move, a competitor board, and a list of pages
that are answering your questions without mentioning you.

**Scope of this document:** the AI visibility core. Search Console and GA4
integrations are ordinary API work and are left out on purpose: they would
double the length and teach you nothing you can't find elsewhere. What's here
is the part I couldn't find written down anywhere when I built this.

---

## What this costs

Every DataForSEO response carries a `cost` field. **Record it per run from the
first day.** It's the thing that stops a loop bug from
becoming a bill. A daily average over the fifteen days to 15 November 2026,
from the running system:

| Run | Calls a day | Cost a day |
|---|---|---|
| AI collection | 425 | **$1.80** |
| SERP queue | 108 | **$0.26** |
| Local map grid (one site) | 105 | **$0.21** |
| Search Console | 50 | free |
| GA4 | 45 | free |

That is **roughly $70 a month** for four to five sites with paid tracking,
between 18 and 58 questions each, three AI surfaces daily. Per site it comes to
**$10 to $25 a month** for an ordinary setup, measured over 30 days: the
lightest site, with 20 questions, cost $9.52, and the heaviest, with 58
questions and a map grid, cost $22.66. A heavier site costs more, and how heavy
it gets is your decision. Google's own APIs cost nothing, which is why they are
worth adding even though they are not in this document.

Per-prompt, per-surface, measured:

| Surface | Per prompt | Run it |
|---|---|---|
| `chatgpt_scraper` | $0.0040 | daily |
| `google_ai_mode` | $0.0040 | daily |
| `google_aio` | ~$0.0053 | daily |
| `claude` | $0.0234 | optional, see below |
| `gemini` | $0.0382 | optional, see below |
| `chat_gpt` (API tier) | $0.0273 | not worth it |

**The three daily surfaces are flat-rate. The other two are token-priced and
swing.** That single fact is what makes the monthly bill predictable, and it is
the first thing to get right. Turn a token-priced engine on only after
deciding what it costs you. The running system asked both of them every five
days for a while and then switched them off, because ChatGPT and Google's two
surfaces already cover most of what a site needs. Turn them on, daily if you
like, for a client who wants every engine covered and has the budget. Per
question, three flat-rate surfaces daily cost about $0.40 a month; adding
Claude and Gemini daily brings it to about $2.25.

SERP position tracking is priced by depth, not by volume: depth 10/20/50/100 is
$0.0006 / $0.00105 / $0.0024 / $0.00465 per keyword. **Depth 50 is the right
default.** Positions past 50 are noise, and depth 20 hides the entire climb for
a site averaging position 21-27, which is exactly the site you are trying to
help.

**Build a cost probe before you build anything else.** A `--test` flag that
runs one prompt on one surface and prints what it cost. You will use it more
than you expect.

---

## Shape of the system

Three parts. Keep them apart.

```
collector/     talks to the outside world. fetch only.
metrics/       reads the database. raw SQL. no loops over fetched rows.
app/           renders. does no arithmetic of its own.
```

The stack that has been running in production: **Next.js (App Router) +
SQLite via better-sqlite3 + Drizzle ORM**, Docker behind nginx on a small VPS.
SQLite is the right call, and people will tell you it isn't. Four months of
daily collection across six sites is a 1 GB file. There is no concurrency problem
worth solving here: one writer, a few readers, one nightly job.

**Worth adopting even if you change the stack:**

**Keep the collector free of platform APIs.** No filesystem, no crypto library,
no Buffer: `fetch` and WebCrypto only. It means the collection logic can later
move to a serverless edge worker without a rewrite, and it forces the
separation above. Node-specific things live in the CLI shell around it.

**Every timestamp is an ISO-8601 UTC string, never a date type.** SQLite has no
date type and neither does D1. Store strings and the schema survives a driver
swap untouched. Convert at the edges.

**Every aggregation is SQL.** Not JavaScript reducing over rows you fetched.
The moment metrics live in loops, you cannot move the database, and you cannot
read what a number means without running the code.

---

## The database

Thirty-three tables in the running system. These seven are the core; the rest are
Search Console, GA4 and reporting.

```
brands           one row per monitored business: name, domain, aliases,
                 country, language
prompts          the questions. text, cadence (daily/weekly/monthly),
                 group, active flag, last_collected_at
ai_responses     one row per question per surface per run. answer text,
                 raw payload, cost, status (ok/empty/error), collected_at
mentions         brand or competitor named in an answer, with position
                 (1 = first named) and a snippet
citations        every source URL an answer pointed at, with its domain
                 and slot
search_queries   the queries the model ran before answering (fan-out)
competitors      who to measure against. status: monitored/suggested/ignored
annotations      the change log. see "The measurement rule"
```

**Store the raw API payload on every response.** It is large and you will be
tempted to skip it. It is what lets you re-run an improved extractor over old
answers when you discover your parser missed something. You will. Prune it
after 30 days rather than never storing it.

**A unique index on prompt text per brand, case- and whitespace-insensitive.**
Without it the same question gets two rows, its answers split across two ids,
and every trend line built on it silently shows half the history.

---

## The collector

### Cadence, not a cron per prompt

Each prompt carries a cadence and a `last_collected_at`. The run asks "is this
one due?" rather than the schedule deciding. **Count calendar days elapsed, not
hours**, or a run that starts twelve minutes late begins skipping days.

This also lets you mix: the questions that move weekly run daily, the stable
ones run weekly, and the expensive surfaces run on a slower cadence of their
own, or not at all. Same loop.

### Extraction

From each answer, pull out:

**The answer text.** Response shapes differ per engine and drift between
versions. **Do not hardcode paths into the payload.** Walk it for text-ish keys
and for anything carrying a URL. You have the raw JSON stored, so a parsing
miss is recoverable, but only if you walk rather than assume.

**Mentions.** Match the brand name and its aliases in the text. Include the
domain as an alias: models write "northfield-studio.com" as often as they
write the company name. Record the position: first named beats fifth named,
and it's what gets quoted onward.

**Citations.** Every URL the answer pointed at, with its domain.

**Fan-out queries.** Several engines return the searches the model ran before
answering. This is the closest anyone gets to seeing what the model thinks your
question is really about, and it is the best content brief you will find. Other
tools charge for this.

### Two-phase queue for SERP

Post the tasks, come back for them on the next run. Rows are written `pending`
on post and filled on the next pass. Live requests cost multiples of this for
data nobody reads within the hour.

---

## The measurement rule

**This is the part that makes the numbers trustworthy.**

Movement is only ever credited from the day work began, and **that day is a
question asked of the data, not a column stored in it.**

The naive design puts a `baseline_date` on each tracked thing. It is wrong, and
here is the failure it causes: moving a baseline rewrites history, the old
question can never be asked again, and you cannot show a client both "since we
started" and "since the rewrite" from the same data.

Do this instead. Keep the complete history and hold **no baseline columns at
all**. Keep a change log: one row per deliberate thing done to the site, on
the day it shipped, with a scope (site-wide, or one page, keyword, question).
A picker offers those days and writes the choice to a query parameter. Every
screen resolves it the same way: explicit choice wins, else the most recent
logged change, else nothing. **"Nothing" means absent, not "since the beginning
of time".**

Changing the selection re-asks the question. It never destroys the ability to
ask a different one.

Charts draw the chosen day as a baseline marker and other logged changes as
numbered markers. **A change whose day is not on the chart's axis must be
reported as unplaceable, not dropped.** A change silently missing from a chart
is worse than one labelled as not shown.

### Refusing to answer

Rules that cost nothing to implement and save you from confident nonsense:

**Below seven days of readings after a baseline, print "too early" instead of a
number.** Average position over two days is noise, and a confident number built
on noise is worse than no number.

**Below about ten conversions, do not print a conversion rate.** A site with
thirty key events in ninety days, four of them from organic search, will
produce a confident-looking "1.3%" from four events. That invites a decision
the number cannot support.

**Weight average position by impressions, twice.** Within a day, because a page
can appear under several URLs and a plain average lets a variant with one
impression count like one with fifty. Across days, because Search Console's
position on a handful of impressions is sampling, not movement. A page carrying one to
five impressions a day read 58, 12, 33, 5, 34, 10, 32 on seven consecutive days
while
nothing about it changed. Drawn raw, that is a chart of Google's sampling that
any reader will take for the page moving.

---

## What breaks

### DataForSEO

**`task_post` returns status_code 20100, not 20000.** "Task Created" is the
success code for every task_post endpoint. Code that treats anything but
20000 as failure throws away task ids you have already paid for. Accept both. Then add a sweep that claims orphans from the tasks_ready
endpoint, so the system heals itself when a run dies mid-flight.

**`40101 Internal SE Server Error` is transient.** Retry with backoff like a
network failure. Three attempts clears almost all of them.

**`40106 Task completed with partial results` is not a failure.** The answer
was captured; some cited pages could not be fetched, and you were not charged
for them. Retrying buys a second charge for data already on disk. Count it as
degraded citation resolution, not as a missing answer.

**`40501` is the invalid-field error, and it can be language-specific.** On
`google_ai_mode`, some prompts that are not in English return 40501 every day,
while English prompts on the same run come back clean. A site in that language
can lose close to half its AI answers this way. It is easy to miss, because a
run that ends "partial" from ordinary noise looks exactly like one that is
losing half its data.

**If you tracker a non-English market, check coverage per site rather than
trusting a run's status.** Expected answers is prompts due times surfaces;
anything under about 90% of that, repeatedly, is a bug and not weather.

**The lesson that generalises: if your runs are always partial, partial means
nothing.** Break the errors down by code and by prompt, and alert on a code
that repeats on the same prompts across days. Scattered errors are the vendor.
The same prompts failing for a fortnight is a bug you are paying for.

**`chat_gpt` skips web search unless you send `force_web_search: true`.**
Without it the answer comes back with zero citations, looks perfectly
reasonable, and is useless for visibility. This is a silent miss, not an error.

**Gemini wraps every citation URL in a `vertexaisearch.cloud.google.com`
redirect.** The real domain is in the annotation `title`. Unwrap it or your
entire source analysis reports one domain: Google's.

**The ChatGPT scraper endpoint takes `keyword`, not `user_prompt`.** The
`llm_responses` engines take `user_prompt`. Path:
`/v3/ai_optimization/chat_gpt/llm_scraper/live/advanced`.

**Check the model, not the capability flag.** The models endpoint lists
`gpt-4o-mini` as web-search capable. The flag is permission; the model declines
the tool per query and returns `web_search: false`. That reads as a genuine
absence of citations and is really a silent miss. `gpt-4.1-mini` behaves.

**Cited URLs carry the model's own tracking.** Nearly every ChatGPT citation
ends in `?utm_source=chatgpt.com`. Anything matching a stored URL against your
citations table has to canonicalise first (strip the query string and the
trailing slash), or it matches nothing and looks like a working feature
reporting zero.

### Your own code

**SQLite binds one variable per column per row.** A single insert of a few
hundred rows exceeds the limit and throws "too many SQL variables". Chunk at
100. And do not wrap a bulk insert in a try/catch that only logs: the error
becomes a log line nobody reads.

**Schema migration tools rebuild the table to add a column.** On a database
with foreign keys pointing at that table, the rebuild fails with
`FOREIGN KEY constraint failed` and your documented migration path is simply
broken. `ALTER TABLE ... ADD COLUMN` is native and non-destructive. Use it.

**If you run a public demo on a second database file, the migration does not
touch it.** A schema change leaves the demo returning 500 while every other
screen looks healthy.

**Two scheduled jobs sharing a lock will collide, and the symptom will not look
like a collision.** A monthly job scheduled while the nightly run is still
going finds the lock taken and exits. The nightly succeeds and the dashboard
fills, so nothing looks wrong, while everything only the monthly job collects
quietly stops. Check elapsed time against your
schedule, and log a refused start loudly.

**`*/3` in cron's day-of-month field means days 1, 4, 7 … 31, then 1 again.**
Every month boundary is a one-day interval. Count calendar days in the script
instead, and only advance the stamp on a clean run so a failure retries
tomorrow rather than waiting out a full interval.

**A background job that must never fail will fail silently.** A nightly
backup built to ignore a failed push, so that it can never break collection,
will also ignore the day the push stops working for good, for example when the
dump outgrows GitHub's 100 MB file limit. If you suppress an error, count it
and surface the count. Send the dump to object storage (Cloudflare R2 or S3) as
well, and check every upload by comparing its hash with the local file.

**Charts: never let a bar disagree with the number printed on it.** A share bar
labelled to one decimal but sized from full precision draws two rows that both
read 2.8% at visibly different lengths. Round both sides the same way.

### Deployment

**Cloudflare Access intercepts `/.well-known/acme-challenge/`.** Let's Encrypt
receives the login page instead of the challenge file, HTTP-01 fails with
`unauthorized`, and **certificate renewal is broken for every hostname behind
Access**. A certificate can expire with no alarm at all,
because an unauthenticated `curl` gets the Access redirect at the edge and
returns 302. The host looks healthy from outside while every real visitor gets
a 526. Use a Cloudflare Origin CA certificate for proxied hostnames. Test the
origin from the server with `openssl s_client`, never through Cloudflare.

**If your origin has a public IP, Access is decoration.** Anyone who knows the
IP reaches the app directly and skips the login. Allowlist Cloudflare's
published ranges at the web server, and refresh that list on a schedule.

**Serve a maintenance page from the web server, not the app.** Deploys recreate
the container and visitors get a raw 502 in the gap. Return **503 with
`Retry-After`**, not 502. In nginx, error_page keeps the original status unless you
force it, and 502 reads as a broken site rather than a busy one. Inline the
logo as a data URI: a maintenance page that fetches an image from the dead
upstream shows a broken image at exactly the wrong moment.

**Never build in the directory the site is served from.** A build that fails
halfway leaves half-built files where the live site reads them, and the site
goes down with it. Anything the build downloads, such as fonts from Google, can
make it fail on a day the server cannot reach that host. Two cheap fixes: ship the fonts inside the repository, and build
in a separate directory, swapping it in only after the build succeeds and the
new version answers a health check, with an automatic rollback if it does not.

**Storage that speaks the S3 protocol is not always S3.** Uploading to
Cloudflare R2 with rclone needs two settings, `no_check_bucket` and `no_head`.
Without them the upload fails with a 501: rclone's check after an upload asks
for an object version, and R2 answers that with "not implemented".

**Back up more than the database.** A database restored without its credentials
is a system that cannot collect anything. Back up the API secrets (encrypted),
the automation platform's data volume, the web server config, and the crontab.
And write the restore order down: on the worst day nobody reconstructs it.

---

## Security and backups

**A monitoring system holds your clients' data and your API credit.** Treat it
that way from the first day, because it is far easier than adding it later.

**Private by default.** Put the dashboard behind a login (Cloudflare Access in
the running system), with robots blocked and noindex on every page, on the same
day it goes online. Make the web server accept connections only from
Cloudflare's published address ranges, so nobody skips the login by calling
the server's address directly.

**Clients see only their own brand.** Each client gets a hostname locked to
their brand. Costs, settings and your own notes stay on the owner's hostname,
and those pages return 404 everywhere else.

**Backups in three places, every night.** The database dump goes to object
storage and to a private git repository, and an encrypted set with the
credentials and the server configuration goes to Google Drive. A backup in one
place is one bad day away from no backup.

**A backup nobody has restored is a hope.** Once a week a script downloads the
newest dump, restores it into a scratch database, and compares the core tables
with the live database as it was when the dump was made. Compare with the
database as it was at dump time: a collection that runs after the dump adds
rows, and a correct backup then looks broken.

**One message a day, and an alarm from outside.** Every morning one summary
arrives on Telegram: backups, collection, sources, the last restore test, the
disk. If the server itself is dead that message never comes, so an outside
service (healthchecks.io) expects a ping from every job and raises the alarm
when one goes missing.

---

## Choosing the questions, and how often to ask them

**This decides whether the dashboard is useful. Everything else is plumbing.**

A monitoring set is not a keyword list. You are not asking what people type
into Google, you are asking what somebody would type into ChatGPT when they are
trying to decide something. Those are different sentences. "gdpr
consent tool" is a search. "what are the best GDPR compliant consent tools for
a small ecommerce site in Europe" is a prompt, and it is the one that produces
an answer with three names in it.

Write them the way a real person would type them, including the clumsiness.
Long, conversational, sometimes two questions in one. A tidy keyword returns a
tidy answer nobody asked for.

### Groups that work across very different businesses

Grouped, because a group is a story and a flat list of forty is not. These
shapes have held across a B2B services firm, a recipe site and a
professional practice:

**Brand.** "What is <company> and what do they do?" Does the model know you
exist, and does it describe you the way you describe yourself. This is the
group that tells you whether you have an entity problem before you spend months
on anything else.

**Buying intent.** "Who should I hire to build X", "best X in <city>". Few in
number, highest value, and the ones a client will ask about first. Expect to be
absent here at the start. That absence is the product.

**Category.** "What are the best X for Y", where you are one legitimate answer
among several. This is where visibility actually moves, because the model is
looking for a list and you can earn a place on it.

**Comparisons.** "X vs Y", "alternatives to X". Cheap to win relative to the
category group, because the model wants a structured comparison and few
sites write one properly. If you only have budget for one group beyond brand,
this is often it.

**Vertical or segment.** The same question narrowed to one industry, city or
constraint. Far easier to be named in than the broad version, and the traffic
converts better.

**Foundational.** "How does X work", "what is X". You will rarely convert from
these, but they are what the model reads to decide who knows the subject, and
they surface the sources it trusts. Treat them as reconnaissance rather than as
a scoreboard.

### How many

**Forty to sixty per site.** Both ends of that are deliberate.

Below about twenty, one answer swinging moves your whole percentage and you
cannot tell a trend from a mood. Above eighty you are paying daily for
questions nobody asked, and the screens get too long to read.

Keep the commercial groups small and the category groups larger. Five buying
intent questions tell you more than twenty will.

### Cadence

Not everything needs asking every day, and that is the whole cost lever.

| Group | How often | Why |
|---|---|---|
| Brand | daily | it is your canary; you want to know the day it changes |
| Buying intent | daily | small, and the ones worth watching closely |
| Category | daily | where movement shows up first |
| Comparisons | weekly | answers are stable; daily buys you repetition |
| Vertical | weekly | same |
| Foundational | weekly or monthly | these barely move |

If you use the token-priced engines, give them a slower sweep of their own
(every five days worked in the running system), so you get a periodic read
without paying for them daily. For a client whose budget allows it, run them
daily like the rest.

**Mixed cadences break the trend line.** If your prompts have mixed cadences, a day
when every prompt runs happens only as often as your slowest group. A visibility
figure from a day where nineteen prompts ran is not comparable with one from a
day where all forty-five did, and averaging them together produces movement
that is purely an artefact of the schedule.

Pick one of these on purpose:

- **Exclude partial days from trend charts.** Honest, and your trend gets one
  point per full sweep. On a mixed daily/weekly set that is one point a week.
- **Chart the daily-cadence subset as its own line.** Those prompts run every
  day and are comparable to each other, so you get a genuine daily trend at no
  extra cost, with the full-sweep line beside it.

The second is usually the right answer, and it's easy to miss.

### Keeping the set honest

**Never delete a prompt. Deactivate it.** Its answers are the history behind
every number it contributed to, and deleting the row either orphans them or
takes them with it.

**Adding a prompt starts its history at zero, not at the site's.** A screen
showing "first named: three days ago" for a question added three days ago is
technically true and completely misleading. Store when a prompt started being
tracked and show it next to anything that looks like a result.

**Review the set quarterly, not weekly.** A question that has never once
returned a mention across three months is telling you something, but it takes
three months to say it.


---

## The interface

Twenty screens in the running system. Build them in the order given under
"Order to build it in". This section says what each one is, so you know what
you are aiming at.

### Navigation

A fixed left sidebar, 232px, grouped. Group labels sit on the groups, not the
items, so a group that loses all its items to a permission filter disappears
along with its heading.

```
[ logo ]
[ brand switcher ]          ← every screen follows this selection

  Overview

AI SEARCH
  Prompts                   the questions and how each performs
  Competitors               who else gets named
  Cited Sources             what the answers are built from
  Query fan-out             what the model searched before answering

SEARCH
  SERP Tracker              classic Google positions
  Search Console            clicks, impressions, CTR, position
  Watchlist                 pages being measured before and after a change

INSIGHT
  Findings         (owner)  conclusions, with evidence
  Journey                   what visitors do once they arrive
  Traffic                   where visits come from, AI assistants included

ADMIN
  Runs & Costs     (owner)  every collection run and what it cost
  Settings         (owner)  prompts, keywords, competitors
  Sites            (owner)  add and configure a brand

[ change log ]              ← the numbered list the chart markers refer to
```

**The brand switcher is the whole multi-tenancy model.** One selection in the
sidebar, written to a cookie, and every screen reads it. Do not build
per-brand routes.

### Two audiences, one application

Screens marked `(owner)` carry your costs, your judgement calls and your
administration. A client must never see them. Decide the view **by hostname**:
`dashboard.you.com` is the owner, `clientname.you.com` is that client, locked
to their brand. Filter the navigation and **404 server-side as well**, because
a hidden link is not access control.

Give write actions the same check. A read-only view that still accepts a
replayed form submission is not read-only.

### Controls that appear on every screen

**A range filter** (7 / 14 / 30 days) in the page header, written to the query
string, never to component state. It has to survive a refresh and a pasted
link.

**The measure-from picker**: the days from your change log, as described under
"The measurement rule". Also a query parameter.

**Tooltips on every single column header and metric label.** Not only the ones
that seem unclear. Every one. In my experience it's the highest-return thing in
the whole interface, and it's the one people skip.

Write them for someone who has never heard of an impression. Say what the
denominator is, which direction is good, and when the number lies:

> **Avg. position**: average place in Google results. Lower is better;
> position 3 beats position 30. Built from very few impressions it swings
> wildly, so treat a small impression count as noise.

> **Named when cited**: being used as a source and being named are two
> different things. An AI can take a figure from this site, link to it, and
> never write the brand's name in the answer: the content did the work and the
> brand got no credit. This counts only the answers that used this site, not
> every answer to the question.

That second one exists because a column called "You named" showed 4/4 while
the question behind it had seven answers. Four of them had used that site,
and all four had also named the brand. The number was right and unreadable. **If the person who built it has to stop and work out what a column
means, a client has no chance.**

### The screens

**Overview.** The only screen most people will look at. Top row: visibility
percentage with its arithmetic under it ("104 of 2,610 answers"), and a
**standings card beside it** ranking the brand against every monitored
competitor. Never show a visibility figure without something to read it
against. Then visibility over time, a per-surface breakdown, and the prompts
that name you. Below that the classic SEO block and the change log.

**Prompts.** One row per question: when it was first tracked, when it first
named you, **when it last named you**, answers collected, mentions, visibility,
average position. First-named and last-named are different facts and you need
both. A prompt reading "first named three weeks ago" at 2% looks like a live
position, and may have named you once on that day and never again.

**Prompt detail.** Every answer collected for one question, with its text, the
sources it used, and the searches the model ran first.

**Competitors.** A trend line for the brand plus the strongest few, with the
rest folded into one named "N others" line. Thirty competitors is thirty lines
and a legend four rows deep. Share of voice per AI surface. A table of everyone
monitored, and a suggestions panel of domains mined from the citations with a
one-click promote.

**Cited Sources.** Every domain the answers were built from, ranked by how much
of the conversation it shapes. Click a domain for its pages; click a page for
the answers that used it, with the text the model wrote. **This is the outreach
list.** The rows where a site is cited and the brand is never named are the
ones worth acting on.

**Query fan-out.** The searches the models ran, grouped by the question that
triggered them. Each group is a content brief.

**Watchlist.** A page, the day it changed, and before/after as **daily
averages**, because the two windows are never the same length. Clicks, impressions,
queries, and an impression-weighted position line. Its own change markers
drawn stronger than the site-wide ones.

**Journey / Traffic.** What arrives and what it does. Sessions by channel with
AI assistants as their own channel, landing pages, outbound clicks, and the
events the site reports.

**Runs & Costs.** Every run: when, what kind, how many calls, what it cost,
status, and the errors. This is how you notice a brand quietly failing.

### Charts

**Mark the change log on every time series.** A numbered dashed line on the day,
the titles listed in the sidebar rather than under each chart. One screen can
carry six plots, and the same ten titles would be drawn under every one.

**Never draw a stacked chart of unlike things.** A single day's paid campaign
can put several hundred sessions beside a day whose earned traffic is under
twenty, and every channel the dashboard exists to measure gets flattened into
a line along the bottom. Keep paid out of the earned chart and state it
separately.

**Never let a shape disagree with its own label**, and never invent a slope.
Bars, not a line, through mostly-zero data: a line between two zero days draws
a slope nothing happened on.

**Empty states explain.** "No full collection day in this period yet. The
trendline starts once there are two" tells someone the system is working.
A blank rectangle tells them it is broken.

### Adding records

**Additions happen on the page where the record lives, in a modal, never in a
central settings form.** The trigger pre-fills what the context already knows:
adding a prompt from inside a group preselects that group. Use the native
`<dialog>` element and focus trapping comes free.

A central form cannot see where you are, so every field it forgets to ask for
produces records that fall out of the grouped views, such as a prompt saved
with no group.

### Look

Light, flat, dense. No gradients, no heavy shadows, one accent colour, a card
with a hairline border. Tabular numerals for anything in a column, or the
digits jump as they update. It is a reading instrument, not a landing page.


---

## Order to build it in

1. **Schema and one brand.** No UI. Prove the shape.
2. **One surface, one prompt, cost printed.** This is your `--test` flag and
   you will keep it forever.
3. **Extraction with the payload stored.** Mentions, citations, fan-out. Check
   the output by hand against three real answers before trusting it.
4. **The loop with cadence and a lock.** Refuse to start if another run holds
   it; clear a stale lock from a dead process.
5. **Cost ledger per run** before you scale up the prompt count.
6. **The first screen: visibility over time.** One number, one line, the
   accounting under it ("104 of 2,610 answers"). People trust a figure they can
   see the arithmetic of.
7. **The change log and the measure-from picker**, before you have any history
   to misread.
8. **Competitors, sources, fan-out.** In that order, since each is a straight
   read of data the collector is already storing.
9. **Tooltips on every column, before you show it to anyone else.** It takes an
   afternoon and it is the difference between a dashboard someone can read and
   one they have to be walked through every time.

**Do not start at step 6.** The temptation is to build screens first because
they are visible. A screen built before the measurement rule will encode the
wrong assumption and everything after it inherits that.

---

## Things worth knowing before you promise anything

**AI answer history cannot be backfilled.** Those answers were generated on
demand and never archived. Your record starts the day collection starts, and
nothing recovers the month before. Search Console, GA4 and classic rank data
are all available retroactively. This is the one that isn't. Start
collecting earlier than you think you need to.

**Answers are volatile.** The same question on the same day across three
surfaces gives three different answers, and tomorrow gives three more. A single
day's reading means nothing. This is why cadence, full-sweep-only comparisons
and refusing to answer below a threshold matter: they are the difference
between a trend and a random number generator.

**Days where only part of the prompt set ran are not comparable with a full
sweep** and must be excluded from trend charts, not averaged in. If your
cadence mixes daily and weekly prompts, a full sweep happens once a week, and
your visibility trend gets one point a week. Decide that deliberately.

**A brand with no monitored competitors has an unreadable visibility figure.**
22% is a win against a leader on 24% and a rout against one on 60%. Put the
comparison on the same screen as the number.

**Citation frequency cannot tell a competitor from a tool.** Automated
competitor discovery from cited domains, run against a consultancy that
implements software for its clients, proposed monitoring the four platforms it
builds on and sells services around, plus a research archive and a vendor's
blog. Six of the first thirty it suggested were real rivals. For a recipe site
whose citations genuinely are the sites it competes with, the same code was
almost entirely right. Discovery scales; the judgement does not, and it depends
on what the business sells.

---

## Licence and provenance

Licensed under CC BY 4.0: use it, change it and build on it, commercially
too, as long as you credit Shlomi Asaf, NextTierSEO, with a link back. No
warranty and no support: it's a description of a working system, not a
product.

If it saves you a week, say so publicly.

The story behind it, what it replaced, and how to get help building it:
https://nexttierseo.com/blog/ai-visibility-tracker/

Built and documented by **Shlomi Asaf, NextTierSEO**.
