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

## Briefing anti-patterns

- **The dump.** Pasting the whole warehouse schema in the hope the model finds the right table. It usually finds a wrong one that sounds right.
- **The verbal question.** Treating the model like a slack conversation. The question stays ambiguous and so does the answer.
- **The retry loop.** Running the query, noticing a wrong number, editing the prompt, running it again. The brief is where the loop terminates; otherwise it runs forever.
- **The silent context.** Relying on a chat history from earlier in the day. The model may not weight that context the way you think it does; write the brief as if the session started cold.

## When the brief is overkill

Not every ask needs this. Use judgment.

- One-off exploration where you'll eyeball the result and throw it away: a one-liner is fine.
- Pure mechanics where you already know the exact SQL you want and the model is faster at typing it than you are.
- Tasks where the model is the wrong tool (e.g., a chart format question that a BI tool solves directly).

The signal that a brief is worth the time: **the number is going somewhere you can't take back.** Boards, customers, regulators, release decisions. If a mistake would require a retraction, brief it.

## Before you move on

Do [Exercise 01: Rewrite a thin prompt into a brief](exercises/exercise-01-rewrite-a-thin-prompt-into-a-brief.md). You'll take a realistic vague ask, work it into a brief that pins down grain, question, dialect, shape, and landing, and compare the model's output on both.

The next module, on verifying what the model returns, assumes you've done this one: a weak brief makes verification open-ended in a way that doesn't scale.
