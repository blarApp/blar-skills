# The gold standard: analyses that evaluate and improve activation

Thirteen analyses in four parts. Each one gives the question it answers, the data, the
computation, how to read it, and the lever. The terms (Rate at N, Delivered nudge, Interaction…)
are the ones defined in SKILL.md.

**Cadence:**
- **Weekly:** G1, G3, G6 and G13.
- **Monthly:** all thirteen.
- **After any change:** G10, once the accounts that entered after the change are mature for the
  comparison day.

## Part A: Outcome

### G1. Activation curve by cohort

- **Question:** how fast does each weekly cohort of entering accounts activate?
- **Data:** `list_activation_accounts` for the pipeline, every page, exclusions applied.
- **Computation:** for each cohort c and each day d from 0 up to the age of c's oldest account:
  - The value is the accounts in c activated within d, divided by all accounts in c.
  - Draw one line per cohort, with days since entry on the x axis and % activated on the y axis.
  - The line is solid while d ≤ the age of c's youngest account, and dashed after that, because
    beyond it not every account has lived that long.
- **Churn ceiling:** also draw, per cohort, the share of accounts not lost by day d ("Lost by day
  d" in SKILL.md). It is the most the cohort can still activate.
- **Read it:**
  - Compare cohorts at the same d. A newer cohort above an older one at the same day activates
    faster.
  - Report each cohort's Rate at 7, 14 and 30 where it is mature.
  - Allow segment, country and project filters, with the minimum size rule.
- **Lever:** none directly. This is the headline; G2 to G13 explain it.

```python
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo
from collections import defaultdict

def cohort_curves(rows, cutoff, tz):
    zone = ZoneInfo(tz)
    by_cohort = defaultdict(list)
    for r in rows:
        entered = datetime.fromisoformat(r["entered_at"]).astimezone(zone)
        monday = (entered - timedelta(days=entered.weekday())).date()
        age = (cutoff - entered).total_seconds() / 86400
        done = r["completed_at"]
        days = (datetime.fromisoformat(done) - entered).total_seconds() / 86400 if done else None
        by_cohort[monday].append((age, days))
    curves = {}
    for monday, accts in sorted(by_cohort.items()):
        youngest, oldest = min(a for a, _ in accts), max(a for a, _ in accts)
        curves[monday] = [
            (d, sum(1 for _, t in accts if t is not None and t <= d) / len(accts), d <= youngest)
            for d in range(0, int(oldest) + 1)
        ]
    return curves
```

The computation returns, for each cohort, a list of (day, share activated, solid) points. The
cutoff must be time-zone aware.

### G2. Rate at the same age, by group

- **Question:** which segments, countries and products activate more slowly?
- **Data:** the G1 rows. For countries, re-query with the `countries` filter and use `count`.
- **Computation:**
  - Rate at N for N = 7, 14, 30 and 60, split by segment, by country and by project.
  - Also split by entry month, to show the trend.
  - Only groups with at least 10 accounts mature for N are compared.
- **Read it:**
  - The gap between groups at the same N, and its direction across entry months.
  - A group lagging at 7 but level at 60 starts slowly. A group lagging at 60 stays behind.
- **Lever:** for a lagging segment, country or product, review the features restricted to it, and
  their order and documents (configuring.md).

### G3. Where active accounts stand, and for how long

- **Question:** at which stage do accounts accumulate, and which are stuck?
- **Data:** `list_activation_accounts` with `statuses` = `active` (report `on_hold` apart), and
  `get_activation_health_history` for the trend.
- **Computation:**
  - The share of active accounts in each `current_stage` and each `health` band, by segment.
  - The median `days_in_stage` per stage.
  - From the history: the daily band counts, `deterioration_rate` and `recovery_rate` for the
    window.
- **Read it:**
  - The stage with the largest share of `off_track` accounts is the bottleneck.
  - A falling recovery rate means accounts that slip stay slipped.
  - Live health and the daily history are different reads; do not mix them in one comparison.
- **Lever:** the features and documents behind that stage's steps (configuring.md). Check G6 for the
  nudges about those steps, and `list_account_milestones` for which step blocks the stuck accounts.

## Part B: Blar's work

G4 to G9 read the `engagement`, `owner` and `health` fields of the account rows and the server-side
summaries, so they cover every account. Open timelines only to look at a few cases.

### G4. Reach in the first week

- **Question:** does Blar get through to new accounts?
- **Data:** account rows mature for 7, and `get_nudge_funnel` for the channel split and failures.
- **Computation:**
  - The share reached in week 1.
  - The share never reached (`engagement.nudges_delivered` = 0).
  - From the funnel, grouped by `channel` over the same entry cohort: `accounts_delivered` and
    `failed`.
- **Read it:** high non-reach with few failures means Blar has no way to contact those accounts. A
  high `failed` count points to the channel itself.
- **Lever:**
  - Contact data and channel setup belong to Blar's team.
  - List the accounts never reached so the team can contact them.

### G5. Interaction in the first week

- **Question:** do new accounts engage, and who starts it?
- **Data:** account rows mature for 7.
- **Computation:** the share that interacted in week 1, by segment and by owner.
- **Read it:** low interaction with high reach (G4) means messages arrive but do not engage. Look at
  G6.
- **Lever:** G6 shows which nudges fail to engage.

### G6. Nudge funnel

- **Question:** which nudges get answered, by channel and by step?
- **Data:** `get_nudge_funnel` over the review window, grouped by `channel`, then by `milestone`,
  then by `follow_up_number`.
- **Computation:**
  - Per nudge: delivered ÷ nudges, then answered ÷ delivered.
  - Per account: `accounts_answered` ÷ `accounts_delivered`.
  - Widget outcomes: accepted, snoozed, dismissed.
- **Read it:**
  - A step with high delivery and few answers has a message problem.
  - A follow-up number that adds almost no answers is wasted contact.
  - `read` is a minimum.
- **Lever:**
  - The description and documents of the feature the step teaches (configuring.md).
  - Wording, timing and the number of follow-ups belong to Blar's team.

### G7. Follow-up after a yes

- **Question:** when an account says yes but does not finish, does Blar come back?
- **Data:** `get_nudge_funnel` with `statuses` = `active` (accounts not completed), grouped by
  `milestone`.
- **Computation:** `answered_without_follow_up_7_days` ÷ `answered`, per step.
- **Read it:** each counted answer is an account that showed intent and then heard nothing about that
  step for a week. To list the accounts, use `list_activity` with `origins` = `nudge` and that
  `milestone`, plus `list_account_milestones`.
- **Lever:** the follow-up cadence belongs to Blar's team. Send the counts and the list.

### G8. Interaction and activation

- **Question:** are accounts that engage with Blar more likely to activate?
- **Population:** accounts mature for 60. If fewer than 10 accounts fall in any band, use the largest
  N among 14, 30 or 60 that clears 10.
- **Computation:** Rate at N per band of `engagement.interactions_first_14_days`: 0, 1, 2 to 3, and
  4 or more.
- **Read it:**
  - This is a link, not a cause: accounts already set on activating may also talk more.
  - A steep rise across bands shows which engagement level to aim for.
- **Lever:** aim reach and nudges (G4, G6) at moving accounts out of the 0 band.

## Part C: Links and changes

### G9. Executives, tasks and activation

- **Question:** which executives' portfolios move, and do their tasks go with it?
- **Data:**
  - account rows (`owner`, `health`, `days_in_stage`, `engagement.team_touches_first_14_days`);
  - `get_task_summary` for the window;
  - `list_tasks` to drill in.
- **Computation:**
  - Per owner: accounts, Rate at N of their accounts mature for N, and the share off track.
  - Per executive, from the task summary: `tasks_completed`, the share of commitments kept on time
    (`commitments_on_time` ÷ `commitments_due`), and `median_due_to_done_hours`.
  - Rate at N for accounts with at least one team touch in their first 14 days, against those
    without.
  - Compare owners only within the same segment, country and product, with at least 10 accounts
    each.
- **Read it:**
  - A link, not a cause: owners are given different accounts, and teams touch struggling accounts
    more.
  - The share off track and the commitments kept show where to coach.
- **Lever:** the team's own playbook. Overdue work is in `list_tasks` with `moment` = `due_at` and
  `statuses` = `open`.

### G10. Before and after a change

- **Question:** did a change (a feature published, documents fixed, a new pipeline) go with faster
  activation?
- **Computation:**
  - Fix the change date T before looking at the results.
  - Compare the Rate at N of accounts that entered in the 8 weeks before T with those that entered
    after T, at the largest N where both groups have at least 10 accounts mature for N.
  - For two pipelines running at once, compare them by `pipeline_definition_id` in the same way.
- **Read it:** this compares periods, not a trial. Anything else that changed between them is part of
  the difference. Name what else changed.
- **Lever:** keep or revert the change. For a causal answer, ask Blar for a random holdout.

## Part D: Support and knowledge

### G11. Support quality

- **Question:** does the AI agent resolve, and how fast?
- **Data:** `get_support_metrics` for the review window and the equal window before it;
  `get_conversation_volume` for the trend.
- **Computation:**
  - `automation_rate`, first response times, resolution time, `unanswered_outbound`.
  - CSAT average, always shown with `csat_total_responses` and `csat_response_rate`.
- **Read it:** a CSAT average from fewer than 10 responses is "too few to compare". A falling
  automation rate with rising volume points to new questions the documentation does not cover:
  run G12.
- **Lever:** G12 and G13.

### G12. What users ask

- **Question:** what do users need help with, and is it covered?
- **Data:** `get_session_topics` for the window; `list_sessions` with `tags` to read examples.
- **Computation:**
  - Per topic: sessions, and escalated ÷ sessions.
  - For the five largest topics, run `search_documentation` with a typical question from that topic
    and record whether a passage answers it.
  - When most sessions are untagged, classify a sample of 50 untagged summaries into at most 10
    written topics before counting.
- **Read it:** a large topic with a high escalation share and no answering passage is a documentation
  gap.
- **Lever:** write or correct the document (configuring.md).

### G13. Hand-offs and why

- **Question:** why does the AI agent hand threads to the team?
- **Data:** `list_conversations` with `was_escalated` = true and activity in the window. Read every
  one if there are 30 or fewer; otherwise a random sample of 30.
- **Computation:** for each conversation, read the AI's hand-off note (`notes`, `author_type` = `ai`)
  and `list_answer_sources`(`conversation_id`). Classify the hand-off as one of:
  - (a) no sources: the agent did not search;
  - (b) the right document absent: a content gap;
  - (c) the right document present: wrong content, or not used;
  - (d) needs a person: an action, a commercial decision or an account change.
- **Read it:** the share of each class. (b) and (c) are fixable through the connector.
- **Lever:**
  - (b) and (c): correct or write the document (configuring.md).
  - (a): `send_feedback`.
  - (d): the team's process.
