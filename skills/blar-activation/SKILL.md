---
name: blar-activation
description: Evaluate and improve customer activation with the Blar connector. Use for activation curves, reach, nudge and support analyses, the gold-standard review, and editing Blar documentation or features.
---

# Blar activation

Blar onboards a company's customer accounts. Each account moves through an activation pipeline of
stages, receives nudges on WhatsApp and in the product's widget, and talks to an AI agent that
answers from the company's documentation and hands off to the team when it cannot. The Blar
connector exposes this company's own data and lets admins change the documentation and the feature
catalog.

This skill runs one loop: measure activation, find where it breaks, change what the connector can
change, and measure again.

## When to use

- How activation is going: activation curves, activation rate, cohorts, accounts that stall.
- Whether Blar's work is landing: reach, interaction, nudge replies, hand-offs, support quality,
  what users ask.
- A periodic review of the activation process (the gold standard).
- Correcting documentation, curating features, reordering or publishing them.

## Before starting

1. Call `get_activation_filters`. If the Blar tools are unavailable, ask the user to connect the
   Blar connector and sign in with their Blar account, then stop.
2. Agree on these once and repeat them in every report:
   - **Pipeline:** the primary one unless the user names another (`is_primary` in the filters).
   - **Time zone:** an IANA name such as `America/Mexico_City`. Every date and cohort uses it.
   - **Cutoff:** the moment of the read, by default now.
   - **Exclusions:** test and internal accounts, by name or external id. Ask for them; never guess.
   - **Language** of the report.
3. Turn relative dates ("last month") into calendar dates before calling any tool.

## Workflow

1. **Frame the question.** Pick the analyses from [references/gold-standard.md](references/gold-standard.md).
   A full review runs G1 to G13 in order; a single question runs the matching analyses only.
2. **Pull the data.** Follow [references/reading-data.md](references/reading-data.md). Prefer the
   summaries (account rows, `get_nudge_funnel`, `get_activation_health_history`, `get_task_summary`,
   `get_session_topics`); read timelines and lists only to look at a few cases. Read every page with
   the filters unchanged, and keep the raw rows (in a file when code execution is available).
3. **Compute.** Use the definitions below exactly. Prefer code over mental arithmetic.
4. **Report** in the format below.
5. **Propose changes.** Give every finding one lever. The levers the connector can pull are in
   [references/configuring.md](references/configuring.md). Change nothing until the user confirms the
   exact change: show what will change, from what, to what.
6. **Report gaps.** When the connector cannot answer a question, call `send_feedback` once per
   missing capability, then say in the report what could not be measured.

## Definitions

Use these terms with exactly these meanings. Never redefine one inside a report.

| Term | Definition |
| --- | --- |
| Account | One row of `list_activation_accounts` for the chosen pipeline, after exclusions |
| Entry | The account's `entered_at` |
| Age | Cutoff minus entry, in days (decimals kept) |
| Cohort | The Monday-to-Sunday week of the entry, in the report time zone |
| Activated | `completed_at` is set. Blar clears it if the account is reopened or lost after completing, so an account counts as activated only while it stays completed |
| Days to activate | `completed_at` minus `entered_at`, in days |
| Activated within N | Days to activate ≤ N |
| Mature for N | Age ≥ N |
| Rate at N | Accounts activated within N ÷ accounts mature for N, both counted over the same group |
| Delivered, answered, interaction, hand-off, team touch | As Blar's server defines them; read them from `engagement` on the account rows and from `get_nudge_funnel`, never recount them from timelines (definitions in [references/reading-data.md](references/reading-data.md)) |
| Reached in week 1 | `engagement.first_nudge_delivered_at` is before entry + 7 days |
| Interacted in week 1 | `engagement.first_interaction_at` is before entry + 7 days |
| Lost by day d | `status` = `lost` and `status_changed_at` minus `entered_at` ≤ d |
| First week | From the entry up to, but not including, entry + 7 days |

## Reading rules

- **Current values.** Status, stage, segment, country and contacts describe the account now, not at
  entry. Say so wherever they split a result.
- **Country.** An account's `country` is its oldest contact's. To count accounts by country, query
  with the `countries` filter and use `count`; do not tally the field.
- **Maturity.** Never put accounts of different ages in one rate. Use Rate at N, and draw a cohort
  line as solid only while every account in the cohort is mature for that day.
- **Minimum size.** Do not compare groups with fewer than 10 accounts on either side; show them as
  "too few to compare".
- **Fixed denominators.** A row or a line keeps one denominator from start to end.
- **Link, not cause.** There is no control group in this data. Write "accounts that X activated more
  often", never "X raised activation". Only a random holdout shows cause; suggest one when the user
  needs it.
- **No invented figures.** If the connector did not return a number, write "not available" and why.

## Report format

1. **Title:** the main finding, as one sentence.
2. **Scope line:** pipeline, cutoff, time zone, accounts counted, accounts excluded.
3. **Summary:** four points, in this order: the activation curve (G1), then the three findings that
   most change what the team should do. Each point is one sentence, one chart or table, and one
   caption line. A caveat that changes the reading goes in the sentence.
4. **Full report:** every analysis run, each with its numbers, how it was computed, "How sure we are"
   and "What this does not say".
5. **Next steps:** for each finding, the lever, who acts (the user through the connector, the team,
   or Blar), and which analysis will show whether it worked, and when.
6. **Since last review:** if the user provides the previous report, compare against it, and name any
   figure that changed definition or was withdrawn.

## References

- [references/reading-data.md](references/reading-data.md): every read tool, what it returns, how to
  paginate it, and its traps.
- [references/gold-standard.md](references/gold-standard.md): the 13 analyses (question, data,
  computation, how to read it, lever) and the review cadence.
- [references/configuring.md](references/configuring.md): changing documentation, the review queue,
  and features; permissions; what only Blar can change.
