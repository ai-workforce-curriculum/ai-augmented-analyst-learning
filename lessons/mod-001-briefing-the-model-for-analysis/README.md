# Module 001: Briefing the Model for Analysis

> **Role:** AI-Augmented Data Analyst · **Org:** AI Workforce Curriculum · **Level:** L20

An AI-assisted analysis has two halves you own: the **brief** you hand the model and the **check** you run on what it returns. Skip the brief and your check has to catch everything — dialect bugs, wrong tables, silently dropped rows. Catch it in the brief and the check gets short. This module is about the first half. The next one is about the second.

## Why a brief

A production analysis is a thousand small decisions the model can't see: which of three "users" tables is the one that excludes test accounts, whether cancelled orders count, how your warehouse spells `DATE_TRUNC`, whether the number lands in a board deck or a bug ticket. Left to its defaults, a model will pick the most common pattern on the public internet. That pattern is almost never the one you need.

The symptom is familiar: the query runs, the number is round and plausible, and it's wrong in a way nobody notices for a week. Nothing in the output flags the problem — the model answered the question it inferred, not the one you had.

A brief closes the gap between the question in your head and the question the model sees.

## What a good brief contains

The checklist below is the minimum for a non-trivial analysis. Paste it in or thread the items through your prompt — the form matters less than the fact that each one is present.

| Item | Why it matters |
|------|----------------|
| **Schema and grain** | Which tables, what each row represents, the primary key, and any soft-delete / status columns. Without grain the model will often count rows that shouldn't count. |
| **The real question, bounded** | The question, plus its time window, segment, and population. "Weekly active users" is three decisions (which action, which week boundary, which users) the model will make for you if you don't make them first. |
| **Dialect and runtime** | Postgres vs BigQuery vs Snowflake vs DuckDB; pandas vs SQL; notebook vs CLI. Dialect features silently fail to port — `QUALIFY` is not in Postgres, `ILIKE` is not in BigQuery. <!-- needs-research: confirm current dialect feature matrix against each vendor's SQL reference --> |
| **Shape of the answer** | A single number, a time series, a cohort table, a chart? The shape changes the query. A model asked for "retention" without a shape will produce whichever one is most common in its training data. |
| **What's been ruled out** | Prior attempts, known-bad joins, tables that look relevant but aren't. Without this, the model will re-walk paths you've already closed. |
| **Where the number lands** | Bug triage, a slide for the leadership deck, an investor Q&A, a regulator filing. This sets your error budget and your tolerance for "roughly right". |

Three items on this list — grain, the bounded question, and the landing — are the ones most analysts leave implicit and most often regret. Make them explicit first.

## Three items most briefs skimp on

**Grain.** The grain of a table is what one row represents, exactly. Not "orders" — one row per *order line*, or per *order*, or per *order revision*? If `fct_orders` carries one row per revision and you count rows to get orders, every order with a changed shipping address counts twice. Models reach for familiar patterns: if your table looks like most public e-commerce schemas they've seen, the model will write `COUNT(*)` where you needed `COUNT(DISTINCT order_id)`. Write the grain next to the table name every time, even when it feels tedious. The one-line test: can you finish the sentence "one row per ___" without hedging? If not, the brief isn't ready — you need to read the table or ask whoever owns it.

**The bounded question.** A verbal question is usually three or four questions in a trench coat. "How did signups do last week?" is really: which signup event (started, completed, verified), which week (calendar, rolling 7-day, business week), which product surface, which segment. Pin each down in your brief. If you genuinely cannot pin one, say so explicitly — "ambiguous: counting completed signups; flag if this excludes a meaningful population." Ambiguity you've named is not the same thing as ambiguity the model inferred and buried in a `WHERE` clause.

**The landing.** Where the number lands sets the error budget. A number that lives inside a slide for a product retro can be approximate and labelled; a number cited in a quarterly earnings call cannot. The landing also determines what you need to preserve alongside the number: a slide needs a short caption, a bug ticket needs the SQL, a regulatory filing needs lineage and timestamps. State the landing before the question — it changes which answers are even acceptable. A brief that says "this is for the board deck on Thursday" is doing more work than the three paragraphs of context that follow it.

## What a brief is not

- **It is not a specification.** You are not writing the query for the model. If you can write it, write it. The brief is for work where the model is faster than you are at the mechanics but doesn't know the domain.
- **It is not a style guide.** "Make it clean" and "add good comments" push on the wrong part of the output. The query being pretty does not make it correct.
- **It is not a prompt template.** Templates help you remember the checklist, but every analysis has its own schema, grain, and landing. Fill in the items, don't just paste the headings.

## Discovering the schema when you don't already know it

The "paste the whole schema" anti-pattern is tempting because sometimes you genuinely don't know which tables you need. The correct move is a discovery step first, not a bigger prompt.

Most warehouses expose their metadata through `INFORMATION_SCHEMA` (BigQuery, Snowflake, Redshift, Postgres) or an equivalent catalog (`system.tables` in Trino, `pragma_table_info` in DuckDB). <!-- needs-research: confirm exact names and paths for current versions of each warehouse's metadata catalog --> Ask the model (or run it yourself) a narrow query like: "Which tables in `events_curated` have a column named `customer_id` or `user_id`, and when were they last written to?" The answer scopes the search without flooding the prompt.

The output of discovery belongs *in* your brief, not before it. By the time you sit down to write the brief, you should be able to name the one or two tables that matter and tell the model which it's using. If you still can't choose, your brief says so explicitly: "two candidates — `fct_orders` (grain: order, excludes cancellations) and `fct_order_items` (grain: line item, includes cancellations). Use `fct_orders` unless the question requires per-item detail, and flag if you switch."

The signal that you're skipping discovery is simple: you catch yourself typing "find the right table" into the prompt. That isn't a brief; it's hope. Do the discovery query, read the three tables that come back, and only then start writing.

## Dialect gotchas worth naming in the brief

Dialect items are the quiet killer: the query runs on the wrong engine, produces a number, and hides its mistake behind valid-looking output. A handful recur often enough to be worth naming in the brief itself:

- **Window function placement.** `QUALIFY` works in Snowflake, BigQuery, and Databricks SQL; it is not in standard Postgres, so the same query there needs a subquery around a window function. <!-- needs-research: confirm QUALIFY availability in current Postgres and ClickHouse releases -->
- **Case-insensitive string match.** `ILIKE` exists in Postgres and Snowflake but not in BigQuery (use `LOWER(col) LIKE LOWER(pattern)` or `REGEXP_CONTAINS`). If the brief does not pin the engine, the model will often write `ILIKE` on a BigQuery warehouse and the query will fail loudly — better than silent failure, but still a round trip you did not need.
- **Date truncation.** `DATE_TRUNC('week', ts)` differs across warehouses in what "week" means by default. Postgres uses ISO weeks starting Monday; BigQuery requires an explicit `WEEK(MONDAY)` argument; Snowflake's behaviour depends on the `WEEK_START` session parameter. <!-- needs-research: verify current defaults for WEEK_START in Snowflake and the week-part semantics documented by BigQuery --> If the brief does not pin "ISO weeks, Monday start", the model will pick a plausible default and the number will shift by a day or two of users.
- **Timezone defaults.** Timestamp columns can be stored with or without a timezone, and `CURRENT_TIMESTAMP` behaves differently across engines. If your brief says "October 2026, UTC" but your warehouse stores timestamps in `America/Los_Angeles`, the model's filter is off by seven or eight hours at the month boundary — enough to move a monthly count noticeably.
- **NULL handling in aggregation.** `COUNT(col)` skips NULLs; `COUNT(*)` does not. `SUM` over an empty set returns NULL, not 0, in standard SQL. If your brief needs a zero-filled time series, say so, or the model will produce one with gaps and gloss over them in the chart.

A compact dialect block in the brief — three or four lines naming the engine, the week convention, the timezone, and any one or two gotchas your team has been bitten by — makes several invisible failures visible before the query ever runs.

## A worked brief

The question your product manager asked, verbatim: "How many customers used the new export feature last month?"

A thin prompt would restate that question. A real brief looks more like:

> Warehouse: **BigQuery**, project `analytics-prod`, dataset `events_curated`.
>
> Tables: `fct_feature_events` (grain: one row per feature interaction; PK `event_id`; `is_internal` flags Anthropic-internal test traffic and must be excluded), `dim_customers` (grain: one row per customer; `status` in `{active, churned, trial}`).
>
> **Question:** Count distinct **paying** customers (`dim_customers.status = 'active'` AND `plan_tier != 'free'`) who triggered at least one event where `feature_name = 'csv_export'` in **October 2026, UTC**. One row per customer counted once. We care about distinct customers, not events.
>
> Output shape: a single integer and the SQL that produced it. No breakdown.
>
> Landing: a slide for Thursday's board meeting. Rough is not acceptable; we will cite the exact number.
>
> Ruled out: `fct_pageviews` — the export button is wired to an event, not a page.

Two things to notice:

1. Every ambiguous word in the original question has been replaced by a decision. "Customers" became "paying customers with status = active". "Last month" became "October 2026, UTC". "Used" became "triggered at least one event".
2. The landing changes what counts as correct. "A slide for Thursday's board meeting" tells the model (and you) that off-by-a-few is a real defect, not a rounding artefact.

## A second worked brief: when the shape changes everything

Same stakeholder, different question: "Are new customers sticking around?"

This is a retention question, and "retention" is at least four different charts with four different grain decisions. A real brief looks more like:

> Warehouse: **BigQuery**, project `analytics-prod`, dataset `events_curated`.
>
> Tables: `fct_feature_events` (one row per feature interaction; PK `event_id`), `dim_customers` (one row per customer; `first_paid_at` is the paid-conversion timestamp). Exclude `is_internal = TRUE`.
>
> **Question:** For customers whose `first_paid_at` was in **Q2 2026 (April–June, UTC)**, show the fraction still **active** in each of weeks 1 through 12 after their `first_paid_at`. "Active" = at least one event in `fct_feature_events` within the week window. Report weekly, not daily. Cohort by `first_paid_at` month.
>
> Output shape: a table with columns `cohort_month`, `week_offset`, `cohort_size`, `active_in_week`, `retention_pct`. 12 rows per cohort, 3 cohorts, 36 rows total.
>
> Landing: internal product review next Tuesday. Trend matters more than the exact percentage; we are looking for a slope, not reporting a number up the chain.
>
> Ruled out: any definition of retention that uses app opens — our instrumentation for app opens is unreliable before June 2026, so cohort 1 would be artificially low. Stick to feature events.

Notice how much the shape pulled in. A time-series cohort shape forces you to define *active*, the *cohort*, and the *window* — three things the thin version would have left to the model. Notice too that the landing ("trend matters more than the exact percentage") loosens the tolerance, which is the opposite of the first worked brief. That change propagates: it is now reasonable to ship the answer with a known approximation (the unreliable app-open data) ruled out in a sentence, rather than fixed.

The same checklist, two very different briefs. The checklist is stable; what you put into each row is not.

## Briefing a chat vs briefing an agent

The brief is the same artifact whether you hand it to a chat model one turn at a time or to an agent that runs on its own for minutes. What changes is where the brief does its work.

In a chat flow, the brief is the front of a conversation. You hand it over, read the first output, and correct or extend. The brief shapes turn one; your follow-ups shape turns two through N. If you forgot to mention the `is_internal` flag, the model's next turn can add it after you flag the gap.

In an agent flow, the brief has to carry the whole run. The agent picks its own sub-steps: which tables to inspect, which test queries to run, when to decide the answer is good enough. A thin brief given to an agent is especially dangerous — the agent will loop on its own best guess at your intent for five or fifty minutes and present a confident, structurally wrong answer. The brief needs to be stricter: explicit grain, explicit stopping conditions ("stop and report when you have a single integer; do not add a breakdown"), explicit things not to try ("do not re-derive active users by joining to the auth table — use `dim_customers.status`").

The rule of thumb: the longer the agent runs without you in the loop, the more of your judgment the brief has to carry. In chat you can intervene turn by turn. In an agent you intervene once, at the top, by writing a brief that forecloses the bad paths before they open.

## Briefing anti-patterns

- **The dump.** Pasting the whole warehouse schema in the hope the model finds the right table. It usually finds a wrong one that sounds right.
- **The verbal question.** Treating the model like a slack conversation. The question stays ambiguous and so does the answer.
- **The retry loop.** Running the query, noticing a wrong number, editing the prompt, running it again. The brief is where the loop terminates; otherwise it runs forever.
- **The silent context.** Relying on a chat history from earlier in the day. The model may not weight that context the way you think it does; write the brief as if the session started cold.

## The brief as a living artifact

The first version of a brief is almost never the last. After you run the analysis and check the answer, three things usually want editing:

- An item you left implicit and the model guessed wrong on. Promote it into the brief in the exact words that would have prevented the guess.
- A path you rediscovered was a dead end. Add it to "ruled out" so future-you does not re-walk it next quarter.
- A dialect quirk that bit you. Note it so a new teammate (or a new model version) does not hit the same wall.

Over months this builds a small library of briefs for the recurring questions in your work — weekly active users, funnel conversion, the quarterly board-deck number. The library is the point. It means a new analyst or a new model can reproduce your analysis without asking you anything, which is the same bar you would hold a well-documented script to.

A brief that stays in a scratch window is the lowest tier of this habit. Briefs that end up checked into a `/briefs/` folder alongside the dbt models or semantic-layer definitions they describe are the mature form — versioned with the code, reviewable in pull requests, searchable when the next ambiguous ask comes in. <!-- needs-research: survey emerging conventions for storing analysis briefs alongside dbt models or in semantic-layer repos -->

## Error budget, in practice

The landing tells you what kind of mistake is acceptable, but "acceptable" is a range, not a binary. Three questions make the range concrete, and all three belong in the brief when the answer is anything other than "doesn't matter".

**Can the number be revised?** A dashboard that updates weekly can absorb a correction the next week with a short note. A slide screenshotted into an investor update cannot — the number is in the deck now. A regulatory filing cannot be revised without a formal amendment. The further toward the "cannot revise" end, the tighter the brief needs to be on filters, grain, and dialect, and the more verification the next module will ask of you.

**Who else will touch this number?** If the answer is "only me, in a scratch notebook", you can tolerate a lot. If it is "the finance team will reconcile this against the billing system", the brief needs to name the source-of-truth table and the join keys finance uses, not whatever the model would guess from the schema. "Reconcilable with system X" is a landing in its own right.

**How does a wrong number get noticed?** If the stakeholder has intuition for the metric — a weekly active count their team watches daily — they will flag a bad number themselves and the cost of an error is a redo. If the number is for a question nobody has a prior for, the error ships undetected. Briefs for the second kind should pin down verification steps explicitly: "cross-check against the daily aggregate in the exec dashboard", "sum by segment should match the total, flag if not".

Writing these down in the brief, even informally, is the difference between "I am confident this is right" and "I am confident this is right, and here is what I checked." The next module is about the second half of that sentence.

## When the brief is overkill

Not every ask needs this. Use judgment.

- One-off exploration where you'll eyeball the result and throw it away: a one-liner is fine.
- Pure mechanics where you already know the exact SQL you want and the model is faster at typing it than you are.
- Tasks where the model is the wrong tool (e.g., a chart format question that a BI tool solves directly).

The signal that a brief is worth the time: **the number is going somewhere you can't take back.** Boards, customers, regulators, release decisions. If a mistake would require a retraction, brief it.

## Before you move on

Do [Exercise 01: Rewrite a thin prompt into a brief](exercises/exercise-01-rewrite-a-thin-prompt-into-a-brief.md). You'll take a realistic vague ask, work it into a brief that pins down grain, question, dialect, shape, and landing, and compare the model's output on both.

The next module, on verifying what the model returns, assumes you've done this one: a weak brief makes verification open-ended in a way that doesn't scale.
