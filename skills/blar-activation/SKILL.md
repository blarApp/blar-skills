---
name: blar-activation
description: Understand and improve customer activation with the Blar connector. Use for activation curves, what counts as an activated account, account health, reach, nudge and support analyses, and editing Blar documentation or features.
---

# Blar activation

Blar onboards a company's customer accounts. Each account moves through an activation pipeline of
stages, receives nudges on WhatsApp and in the product's widget, and talks to an AI agent that
answers from the company's documentation and hands off to the team when it cannot. The Blar
connector exposes this company's own data and lets admins change the documentation and the feature
catalog.

This skill helps the company see how its accounts activate, find where activation breaks, and
decide what to change.

## When to use

- How activation is going: activation curves, rates, cohorts, account health, accounts that stall.
- What counts as an activated account, which features are core, which milestones matter.
- Whether Blar's work is landing: reach, interaction, nudge replies, hand-offs, support quality,
  what users ask.
- Correcting documentation, curating features.

## Before measuring

1. Call `get_activation_filters`. If the Blar tools are unavailable, ask the user to connect the
   Blar connector and sign in with their Blar account, then stop.
2. Agree on what activation means. Follow
   [references/defining-activation.md](references/defining-activation.md):
   - Look in your memory and the project's instructions for an earlier agreement, and confirm it.
   - Otherwise, read the board and the core features Blar has configured and show them. Then ask
     what the company counts as activated, whether those are its core features, and whether those
     milestones are the steps an account has to take to stay engaged.
   - In the same message, ask for the time zone, the accounts to leave out and the language.
   - Wait for the answers, then suggest saving them.
3. Turn relative dates ("last month") into calendar dates before calling any tool.

## Measuring

- **Activation curves.** Call `get_condition_completion` with the agreed condition and read every
  page. Group the rows into weekly cohorts by `entered_at`, in the agreed time zone. Each cohort's
  curve is the share of its accounts activated by each day since entry (G1 in
  [references/analyses.md](references/analyses.md)). `rate_at_days` gives the rate at 7, 14, 30 and
  60 days.
- **Health.** Call `get_activation_health_history` and chart the accounts in each band per day.
- **An open question,** such as "how is our activation going?": answer with the activation curves by
  weekly cohort on the agreed definition and the health chart, then make suggestions.
- **Any other question:** pick the analyses that answer it from
  [references/analyses.md](references/analyses.md). Read the data as
  [references/reading-data.md](references/reading-data.md) explains: summaries first, rows only to
  look at a few cases. Compute with code when you can, not by mental arithmetic.

## Answering

- Answer the question asked. There is no fixed format: choose the charts and tables that answer it.
- End with suggestions, as many as the question and the company's situation call for. Each one
  carries:
  - the evidence behind it;
  - who acts: the user through the connector, the company's team, or Blar's team;
  - how to tell whether it worked.
- When the connector cannot answer something, say what could not be measured and why. Call
  `send_feedback` once per missing capability.

## Changes

- **Documentation and features:** change them only after the user confirms the exact change: what
  changes, from what, to what. See [references/configuring.md](references/configuring.md).
- **The board** (milestones, thresholds, stages) is changed only by Blar's team. Suggest the change,
  send it with `send_feedback` (`suggestion`) and the finding behind it, and ask the user to raise it
  with their Blar contact.

## Definitions

Use these terms with exactly these meanings.

| Term | Definition |
| --- | --- |
| Account | One account the connector returns for the chosen pipeline, after the accounts left out. Accounts Blar marks as test accounts are never returned |
| Entry | The account's `entered_at` |
| Age | The cutoff minus the entry, in days (decimals kept) |
| Cohort | The Monday-to-Sunday week of the entry, in the agreed time zone |
| Activated | The account met the agreed condition: its `met_at` from `get_condition_completion` is set |
| Days to activate | `met_at` minus `entered_at`, in days. A negative value means the account did it before entering, and counts as day 0 |
| Activated within N | Days to activate ≤ N |
| Mature for N | Age ≥ N |
| Rate at N | Accounts activated within N ÷ accounts mature for N, both counted over the same group (`rate_at_days`) |
| Delivered, answered, interaction, hand-off, team touch | As Blar's server defines them. Read them from `engagement` on the account rows and from `get_nudge_funnel`; never recount them from timelines (definitions in [references/reading-data.md](references/reading-data.md)) |
| Reached in week 1 | `engagement.first_nudge_delivered_at` is before entry + 7 days |
| Interacted in week 1 | `engagement.first_interaction_at` is before entry + 7 days |
| Lost by day d | `lost_at` minus `entered_at` ≤ d |
| First week | From the entry up to, but not including, entry + 7 days |

## Accuracy rules

- **Say what each number counts:** the definition of activation, the period, the time zone, and the
  accounts left out.
- **Current values.** Status, stage, segment, country and contacts describe the account now, not at
  entry. Say so wherever they split a result.
- **Country.** An account's `country` is its oldest contact's. To count accounts by country, query
  with the `countries` filter; do not tally the field.
- **Maturity.** Never put accounts of different ages in one rate. Use Rate at N, and draw a cohort
  line as solid only while every account in the cohort is mature for that day.
- **Minimum size.** Do not compare groups with fewer than 10 accounts on either side; show them as
  "too few to compare".
- **Fixed denominators.** A row or a line keeps one denominator from start to end.
- **Link, not cause.** There is no control group in this data. Write "accounts that X activated more
  often", never "X raised activation". Only a random holdout shows cause; suggest one when the user
  needs it.
- **No invented figures.** If the connector did not return a number, write "not available" and why.

## References

- [references/defining-activation.md](references/defining-activation.md): reading the board and the
  core features, the questions to ask, saving the agreement, and turning it into a condition.
- [references/analyses.md](references/analyses.md): the analyses to draw on (question, data,
  computation, how to read it, lever).
- [references/reading-data.md](references/reading-data.md): every read tool, what it returns, how to
  paginate it, and its traps.
- [references/configuring.md](references/configuring.md): changing documentation and features,
  permissions, and what only Blar can change.
