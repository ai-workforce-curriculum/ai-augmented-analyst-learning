# Module 001: Briefing the Model for Analysis

> **Role:** AI-Augmented Data Analyst · **Org:** AI Workforce Curriculum · **Level:** L20

An AI-assisted analysis has two halves you own: the **brief** you hand the model and the **check** you run on what it returns. Skip the brief and your check has to catch everything — dialect bugs, wrong tables, silently dropped rows. Catch it in the brief and the check gets short. This module is about the first half. The next one is about the second.

## Why a brief

A production analysis is a thousand small decisions the model can't see: which of three "users" tables is the one that excludes test accounts, whether cancelled orders count, how your warehouse spells `DATE_TRUNC`, whether the number lands in a board deck or a bug ticket. Left to its defaults, a model will pick the most common pattern on the public internet. That pattern is almost never the one you need.

The symptom is familiar: the query runs, the number is round and plausible, and it's wrong in a way nobody notices for a week. Nothing in the output flags the problem — the model answered the question it inferred, not the one you had.

A brief closes the gap between the question in your head and the question the model sees.

## The failure modes a brief prevents

It helps to name the specific ways an un-briefed analysis goes wrong. Each of these is cheap to produce and expensive to catch downstream — and each has a direct counter in the checklist in the next section.

- **Wrong-table correctness.** The model picks a table whose name sounds right (`users`, `orders`, `events`) but whose grain or filter semantics are not what you'd pick. The query runs. The number is plausible. Nothing in the output surfaces that `users` includes soft-deleted accounts, or that `orders` includes cancelled orders at the same grain as paid ones. The brief's *schema and grain* line is the counter.
- **Boundary drift.** "Last month", "this quarter", "year to date", "returning customers" are interpreted the way the model saw them most often in training. "Last month" might become calendar month, trailing-30-day, or fiscal month depending on nearby context; "returning customer" might mean *has any prior order*, *has an order in the prior window*, or *has an order in both this window and the previous one*. The brief's *bounded question* line is the counter.
- **Silent dialect mismatch.** The model writes `QUALIFY ROW_NUMBER() OVER (...) = 1` against a Postgres warehouse that does not have `QUALIFY`, and your ORM swallows the error or falls back to a slower rewrite you did not notice. Or it writes `ILIKE` against BigQuery and the client library forgives it with a warning you miss. The brief's *dialect and runtime* line is the counter.
- **Shape substitution.** You asked for "retention"; the model gave you a weekly cohort chart because that is the most common way retention is drawn on the internet, when what you actually needed was a single number for a specific experiment arm. The brief's *shape of the answer* line is the counter.
- **Overconfident rounding.** The model, having inferred the question from the ask alone, produces a confident number without showing the uncertainty that would be obvious if the brief named the landing. Stakeholders read confidence as accuracy. The brief's *where the number lands* line is the counter.

None of these show up as errors. They show up as numbers that are wrong by an amount nobody can see without re-doing the analysis. The purpose of the brief is to stop them before the query runs, not after.

## What a good brief contains

The checklist below is the minimum for a non-trivial analysis. Paste it in or thread the items through your prompt — the form matters less than the fact that each one is present.

| Item | Why it matters |
|------|----------------|
| **Schema and grain** | Which tables, what each row represents, the primary key, and any soft-delete / status columns. Without grain the model will often count rows that shouldn't count. |
| **The real question, bounded** | The question, plus its time window, segment, and population. "Weekly active users" is three decisions (which action, which week boundary, which users) the model will make for you if you don't make them first. |
| **Dialect and runtime** | Postgres vs BigQuery vs Snowflake vs DuckDB; pandas vs SQL; notebook vs CLI. Dialect features silently fail to port — `QUALIFY` is [not in Postgres](https://www.postgresql.org/docs/current/sql-select.html), `ILIKE` is [not in BigQuery](https://cloud.google.com/bigquery/docs/reference/standard-sql/operators). |
| **Shape of the answer** | A single number, a time series, a cohort table, a chart? The shape changes the query. A model asked for "retention" without a shape will produce whichever one is most common in its training data. |
| **What's been ruled out** | Prior attempts, known-bad joins, tables that look relevant but aren't. Without this, the model will re-walk paths you've already closed. |
| **Where the number lands** | Bug triage, a slide for the leadership deck, an investor Q&A, a regulator filing. This sets your error budget and your tolerance for "roughly right". |

Three items on this list — grain, the bounded question, and the landing — are the ones most analysts leave implicit and most often regret. Make them explicit first.

### Grain, in a little more detail

Grain is the one word on the list that is most often skipped and most often fatal. "Grain" is the answer to the question: *what does a single row of this table represent?* An `orders` table might have one row per order, one row per order line item, or one row per status transition, and the three answers imply three completely different queries for the same English question. Write the grain down as a short phrase next to each table you name — *"one row per paid invoice"*, *"one row per user per day"*, *"one row per shipment event"*. The phrase should be specific enough that it tells you, without having to query, what a `SUM`, `COUNT`, or `COUNT(DISTINCT ...)` will mean.

A related trap is **mixed grain inside one table**. Event logs often blend heartbeats with interactions; subscription tables often blend trial signups with paid conversions. If a column distinguishes them, name it in the brief and name the filter. If no column distinguishes them, say so — then the model knows it is working with an ambiguous row-set and will typically ask or flag it, instead of guessing.

### Bounding the question

The practical test for "bounded" is: a second analyst, given only the brief and no access to you, should be able to tell when the answer is wrong. If "last month" is in the question, two analysts will disagree about whether today's date matters, whether "last month" is calendar or trailing-30, and whether events at 23:59 in a non-UTC timezone count. Write the time window as an interval with explicit start and end timestamps and a timezone. Do the same for the population: *"customers whose first paid invoice landed before the window"*, not *"existing customers"*. Do the same for the action: *"any row in `fct_feature_events` where `feature_name = 'csv_export'` and `is_internal = false`"*, not *"used the feature"*.

### Dialect, runtime, and the environment around them

Dialect is not just the SQL flavour. It is the whole environment the code will run in: the warehouse, the client library, the Python version, whether the query is a one-off in a notebook or a scheduled view in a dbt project. The model's defaults will drift toward whatever is most common on the open internet, which is usually recent Postgres syntax and recent pandas idioms. Both are fine; both are wrong in at least one way for most production warehouses. Name the warehouse, the project / database / schema, and (if non-obvious) the client. If the output will be reused — a view, a dashboard SQL, a scheduled notebook — name that too, because it changes whether throwaway hacks are acceptable.

### What the answer looks like, before you see it

The shape of the answer is the hardest item to pin down on a first pass, because the stakeholder often has a shape in their head they have not said out loud. A useful move: describe, in one sentence, the chart or table you would put in front of them if you had the right data already. *"A single number with a rounded comparison to the prior month."* *"A line chart, one series per plan tier, weekly."* *"A funnel with four steps."* Writing the shape forces the stakeholder's unstated expectation to the surface. Half the time the brief changes after you do this, because the shape turns out to imply a different question.

## What a brief is not

- **It is not a specification.** You are not writing the query for the model. If you can write it, write it. The brief is for work where the model is faster than you are at the mechanics but doesn't know the domain.
- **It is not a style guide.** "Make it clean" and "add good comments" push on the wrong part of the output. The query being pretty does not make it correct.
- **It is not a prompt template.** Templates help you remember the checklist, but every analysis has its own schema, grain, and landing. Fill in the items, don't just paste the headings.

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

## A second brief: when the output is not SQL

Not every analysis lands in a SQL cell. Here is the same shape of brief for a pandas task, so the pattern is visible outside the warehouse.

The ask from a colleague: "Can you pull the top spenders from our checkout export and flag the ones who haven't ordered in a while?"

A real brief:

> Runtime: **Python 3.12** in a Jupyter notebook. pandas ≥ 2.2 is available; no database connection — the input is a CSV.
>
> Input file: `checkout_export_2026_11.csv`, one row per completed order, columns: `order_id`, `customer_id`, `order_total_usd`, `ordered_at` (ISO-8601, UTC), `status` (`paid`, `refunded`, `cancelled`). Grain: one row per order. Expect ~1.2M rows; fits in memory.
>
> **Question:** For each customer, compute lifetime net spend (sum of `order_total_usd` for `status = 'paid'`, minus the same for `status = 'refunded'`). Return the top 100 customers by net spend, with a boolean column `at_risk` set to `True` when the customer's most recent `paid` order is older than 180 days as of **2026-11-01 00:00:00 UTC** (treat this date as a fixed snapshot, not "today").
>
> Output shape: a DataFrame with columns `customer_id`, `net_spend_usd`, `last_paid_at`, `at_risk`. Sorted descending by `net_spend_usd`. Written to `top_spenders_2026_11.csv` next to the input.
>
> Landing: a list my CS lead will use to assign retention outreach this week. False positives waste a call; false negatives miss revenue. We prefer precision on `at_risk`.
>
> Ruled out: do not join with `dim_customers` — the export is the source of truth for this exercise, and the warehouse copy lags by a day.

The dialect is Python instead of SQL, but the ingredients are identical: schema and grain (the CSV columns and what a row is), a bounded question (fixed snapshot date, explicit filters), a shape (the four-column DataFrame, written to a specific path), a landing (CS outreach this week, with a stated preference for precision), and a ruled-out branch (don't join the warehouse). A thin version — "pull the top spenders and flag at-risk ones" — would have left every one of those decisions to the model.

## Briefing anti-patterns

- **The dump.** Pasting the whole warehouse schema in the hope the model finds the right table. It usually finds a wrong one that sounds right.
- **The verbal question.** Treating the model like a slack conversation. The question stays ambiguous and so does the answer.
- **The retry loop.** Running the query, noticing a wrong number, editing the prompt, running it again. The brief is where the loop terminates; otherwise it runs forever.
- **The silent context.** Relying on a chat history from earlier in the day. The model may not weight that context the way you think it does; write the brief as if the session started cold.
- **The polite brief.** Softening the constraints into suggestions — *"maybe exclude internal users if you can"* instead of *"exclude rows where `is_internal = true`"*. The model reads hedges as permission to decide; write constraints as instructions, not requests.
- **The one-column fix.** Discovering a problem mid-session and patching the prompt with one extra line (*"also exclude refunded orders"*) instead of rewriting the brief. Each patch is cheap; the stack of patches silently contradicts itself and the model follows whichever version it weighted most.

## Revising the brief, not the output

A good brief still misses things on the first pass. The move when it does: edit the brief, not the output. If the model produced a query that joined the wrong table, the fix is not "please use the other table" tacked on the end of the chat. The fix is to add *"use `fct_feature_events`, not `fct_pageviews`"* to the ruled-out section and re-ask. Two reasons:

1. **The brief is the artefact that outlives the session.** The chat history disappears; the brief goes into a document, a PR description, or a dbt model comment. Patches tacked onto the end are invisible to the next person who runs the analysis (including future-you, three weeks later, with no memory of the patch).
2. **The model weights the most recent instructions most heavily, but not exclusively.** A brief that contradicts itself — "exclude internal users" near the top, "actually include everyone" at the bottom — produces output that silently splits the difference. Rewriting the relevant line is cleaner than appending a correction.

A useful habit: at the end of each analysis, if the brief needed more than one amendment, update it before closing the tab. The next version of this question — different month, different cohort, same shape — should start from the fixed brief, not the broken one.

## When the brief is overkill

Not every ask needs this. Use judgment.

- One-off exploration where you'll eyeball the result and throw it away: a one-liner is fine.
- Pure mechanics where you already know the exact SQL you want and the model is faster at typing it than you are.
- Tasks where the model is the wrong tool (e.g., a chart format question that a BI tool solves directly).

The signal that a brief is worth the time: **the number is going somewhere you can't take back.** Boards, customers, regulators, release decisions. If a mistake would require a retraction, brief it.

A rough escalator, from shortest ask to longest brief:

- *Eyeballing a trend before a standup.* A sentence is fine. If the number surprises you, promote the question and brief it before you share.
- *Answering a slack thread with a chart.* A paragraph: one line for grain, one for the time window, one for the dialect. Error budget is loose; a retraction is embarrassing but survivable.
- *A number in a document, email, or ticket that other people will quote.* The full checklist. The number will be re-run, probably by a model, probably by someone who didn't write the original question.
- *A number in an external artefact — deck, filing, customer-facing dashboard.* The full checklist, plus the brief saved alongside the artefact. The next time this number has to be defended, the brief is the evidence that the number meant what you said it meant.

## Briefs as a team artefact

Once you have written three or four briefs, they stop being one-off prompts and start being a small library. The recurring questions in any analytics team — MAUs, ARR, churn, funnel conversion, feature adoption — each want the same brief most weeks, with a date swapped in. Keeping those briefs somewhere a colleague can find them (a docs repo, a dbt model comment, a notebook template) does three useful things:

- It compresses the next analysis. Starting from a known-good brief is faster than writing a new one from the top.
- It makes definitional drift visible. If "weekly active users" means one thing in last quarter's brief and another thing in this quarter's, you can see the drift in the diff instead of discovering it in a mismatched chart.
- It is the asset you hand a new analyst or a new model version. Models change; the brief, if it was written for a reader rather than a specific model, mostly does not.

The exercise at the end of this module asks you to save the brief you write. That is the first entry in that library.

## Before you move on

Do [Exercise 01: Rewrite a thin prompt into a brief](exercises/exercise-01-rewrite-a-thin-prompt-into-a-brief.md). You'll take a realistic vague ask, work it into a brief that pins down grain, question, dialect, shape, and landing, and compare the model's output on both.

The next module, on verifying what the model returns, assumes you've done this one: a weak brief makes verification open-ended in a way that doesn't scale.
