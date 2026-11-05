# Exercise 01: Rewrite a Thin Prompt into a Brief

**Role:** AI-Augmented Data Analyst · **Time:** 45–60 minutes · **Output:** a written brief, two model outputs, and a short diff note

## Goal

Feel the difference, in a single analysis, between a thin prompt and a real brief. By the end you'll have paired evidence that the brief changed the query, not just the model's tone.

## Setup

Pick one of the following, in order of preference:

1. A real analysis ask from your current work. Scrub identifying data if you need to.
2. A public dataset you already know (e.g., the [NYC Taxi trip data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page), [MovieLens](https://grouplens.org/datasets/movielens/), or [GitHub Archive](https://www.gharchive.org/)) and a question a non-technical stakeholder might ask.
3. A sample warehouse from your BI tool's docs. <!-- needs-research: link to specific vendor sample datasets (BigQuery public datasets, Snowflake sample data) with their schema docs -->

If you don't have model access, do steps 1–4 and write out what you'd *expect* the model to produce in each case; the diff is still real.

## Steps

1. **Capture the ask, verbatim.** Write down exactly what the stakeholder asked, in the words they used. Don't clean it up.
2. **Pass 1 — thin prompt.** Hand the model the ask as written, with no additional context. Save the output: the SQL / code and the number.
3. **Write the brief.** Using the checklist from the module (schema and grain, bounded question, dialect and runtime, shape, what's ruled out, where the number lands), produce a written brief for the same question. Aim for one screen of text, no more.
4. **Pass 2 — briefed prompt.** Hand the model the brief. Save the output.
5. **Diff the two.** Side by side, note every meaningful difference: which tables each query used, which filters, how joins were written, what the dialect looked like, what shape the result took, and — crucially — whether the numbers match.
6. **Pick the winner, honestly.** Which output would you have shipped? If the thin prompt happened to be right, say so: a brief that doesn't change the answer still has value for a reader, but you should know when it didn't move the query.

## What good looks like

- Your brief replaces every ambiguous word in the original ask with a concrete decision. "Last month", "customers", "active", "the main table" are all pinned down.
- The diff names specific lines or clauses that changed, not "the briefed version was better".
- If the two queries produced different numbers, you can point to the clause responsible.
- If they produced the same number, you can explain whether that was luck (the thin prompt happened to guess your grain) or signal (the question was unambiguous enough that briefing was overkill).

## Reflection questions

Write a sentence or two on each:

- Which of the six brief items did you forget on your first draft? Which do you now think is most load-bearing for the kind of analysis you do?
- Where did the briefed output still need human verification? (This is the handoff to the next module.)
- If a colleague asked you the same ask verbally in the hallway, how would you steer that conversation toward a briefable question before opening a notebook?

## Keep it

Save the brief. Over the curriculum you'll build a small library of briefs for the recurring questions in your work; the goal is that a new analyst (or a new model version) could pick one up and reproduce the analysis without asking you anything.
