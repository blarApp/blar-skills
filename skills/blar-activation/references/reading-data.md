# Reading Blar data through the connector

Everything the connector returns belongs to the company whose member authorized the connection.
Blar's own ids (`account_id`, `contact_id`) and the company's own ids (`external_id`) are separate
fields; a missing external id stays empty and is never filled with a Blar id.

## Permissions

| Permission | Unlocks | Who |
| --- | --- | --- |
| Read | Every tool on this page | Any member; granted by default. An admin's tasks tools read every task; a non-admin reads only their own |
| Write documentation | The documentation changes in configuring.md | Granted on Blar's consent screen |
| Read features / Write features | The feature catalog | Admins of an approved company only |

If a tool answers that a permission is missing, ask the user to disconnect and reconnect the Blar
connector and approve that permission on Blar's consent screen.

## Summaries first, rows to drill down

Every tool result lands in the conversation, and a busy company produces thousands of activity rows
a month. Answer with the summaries; open rows only to look at a few cases.

| Question | Summary | Drill-down |
| --- | --- | --- |
| Accounts, their state, health, owner and engagement | `list_activation_accounts` | `list_account_activity`, `list_account_milestones` |
| How nudges landed, by channel, step and follow-up | `get_nudge_funnel` | `list_activity` (`origins: ["nudge"]`) |
| How health moved over time | `get_activation_health_history` | `list_activation_accounts` with `health` |
| What executives did with their tasks | `get_task_summary` | `list_tasks` |
| What users ask the AI agent | `get_session_topics` | `list_sessions` with `tags`, then `get_session` |
| Hand-offs | `get_support_metrics` | `list_conversations` (`was_escalated`), `get_conversation`, `list_answer_sources` |
| Support quality and volume | `get_support_metrics`, `get_conversation_volume` | — |
| What the documentation says | — | `search_documentation`, `list_documentation`, `get_documentation`, `get_documentation_history` |
| Pipelines, segments, products, countries, statuses, stages | `get_activation_filters` | — |

## Accounts

**`list_activation_accounts`** returns one row per account in one pipeline (the primary when
`pipeline_definition_id` is omitted).

- **Filters:**
  - `segments` (names, ids as strings, or `none`) and `countries` (names, ISO codes, or `none`);
  - `project_id`;
  - `statuses` (`active`, `on_hold`, `completed`, `lost`, `queued`) and `current_stages`
    (`hand_off`, `set_up`, `first_value`, `core_value`, `repeat_value`, `full_value`);
  - `health` (`progressing`, `idle`, `at_risk`, `off_track`);
  - `owners` (member ids, or `none`);
  - `entered_from` and `entered_to`: inclusive `YYYY-MM-DD`, read in `date_timezone`.

  Several values in one filter combine with OR.
- **Identity:** `pipeline_id`, `account_id`, `external_id`, `account_name`, segment, project,
  `country`, `contacts`.
- **Lifecycle:**
  - `status`, `current_stage`, `entered_at`, `completed_at`;
  - `status_changed_at`, so a lost account carries when it was lost;
  - `entered_stage_at` and `days_in_stage` (whole days in the current stage);
  - `health`: the live band, read at request time; null for any account that is not active.
- **Owner:**
  - `owner` (id, name, email; null when nobody owns the account or the owner was deactivated);
  - `owner_email` (the address the source named);
  - `owner_source` (`blar` or the CRM that names the owner);
  - `owner_changed_at` and `unassigned_reason`.
- **`engagement`**, summed on the server from the account's activity since it entered:
  - `nudges_delivered` and `first_nudge_delivered_at`. Delivered means the widget showed the nudge,
    or the channel confirmed the nudge's own message as delivered, read or replied to. On email and
    Slack, which report no delivery, a send that did not fail counts.
  - `nudges_answered`: nudges the contact replied to, or accepted in the widget.
  - `interactions`, `interactions_first_14_days`, `first_interaction_at`, `last_interaction_at`: an
    interaction is an answered nudge or a thread the contact started, timed when the contact acted.
  - `handoffs`: threads escalated to the team.
  - `team_touches_first_14_days`: calls, hand-logged work, and threads a member started.
- **Pages:** `limit` up to 500. Follow `next_offset` until it is null with identical filters. `count`
  is the total.
- **Traps:**
  - Status, stage, segment, owner and contacts are current values.
  - `country` is the oldest contact's; to count accounts by country, filter with `countries` and use
    `count`.
  - `completed_at` is cleared if the account is reopened or lost after completing.

## Totals and history

- **`get_nudge_funnel`**
  - **Window:** `sent_from` to `sent_to`, inclusive local days, at most 92 apart.
  - **Filters:** the same account filters as `list_activation_accounts`.
  - **Output:** `totals`, plus `rows` by `group_by` (any of `channel`, `milestone`,
    `follow_up_number`).
  - **Per-nudge counts:**
    - `nudges`, `delivered`, `answered`;
    - `read`: outside the widget only, and a minimum;
    - `accepted`, `snoozed`, `dismissed`;
    - `failed`;
    - `answered_without_follow_up_7_days`: answers at least a week old with no later nudge on the
      same step within the week.
  - **Per-account counts:** `accounts`, `accounts_delivered`, `accounts_answered`. These are counted
    once within each row, so the rows do not add up to the totals.
- **`get_activation_health_history`**
  - **Window:** `start_date` to `end_date`, inclusive, at most 366 days.
  - **Filters:** pipeline, product and segments.
  - **Output:**
    - `series`: one entry per day, counting accounts by band plus `no_band` and `total`, from one
      reading per day.
    - `transitions`: `observed_accounts`, `deteriorated_accounts`, `slipping_accounts`,
      `recovered_accounts`, `deterioration_rate` and `recovery_rate`. Each account counts once.
- **`get_task_summary`**
  - **Window:** `date_from` to `date_to`, at most 92 days.
  - **Output:** one row per executive (`user`) plus an unassigned row, and a `team` total. Each row
    has `commitments_due`, `commitments_on_time`, `tasks_completed`, `tasks_completed_in_blar` and
    `median_due_to_done_hours`.
  - **Filters:** `assignees`, `project_id`, `pipeline_definition_id`, `segments` (ids as strings, or
    `none`) and `countries`.
- **`get_session_topics`**
  - **Window:** `active_after` and `active_before` (ISO timestamps on the last activity).
  - **Output:** per topic tag, `sessions` and `escalated`, plus an untagged row. A session with
    several tags counts under each.
- **`get_support_metrics` and `get_conversation_volume`**
  - **Output:** the dashboard's figures: `automation_rate`, response and resolution times, CSAT
    average, response rate and responses, `unanswered_outbound`. The volume tool returns them as a
    daily series under 30 days and weekly beyond.
  - **Options:** `source` (`all`, `sessions` or `tickets`) and `project_id`.
  - **Window:** `days`, or `start_date` and `end_date` (ISO timestamps, at most 365 days apart).

## Drill-down

- **`list_account_activity`** reads one account's timeline, newest first.
  - **Account:** pass exactly one of `account_id` or `external_id`.
  - **Pages:** `limit` up to 50; follow `next_offset`.
  - **Row fields:**
    - `kind`: `ai_session`, `conversation`, `widget_call`, `phone_call`, `logged`.
    - `channel`, and `origin` (`nudge` marks a nudge).
    - `started_by`: `contact`, `member`, `ai`, `automation`.
    - `occurred_at`, `status`, `escalation`.
    - `milestone`, `feature`, `follow_up_number`.
    - `outbound`: `message`, `status`, `sent_at`, `delivered_at`, `read_at`, `replied_at`,
      `failure_reason`, `unreported_steps`.
    - `widget_nudge`: `shown_at`, `replied_at`, `outcome`, `outcome_at`, the counts.
    - For calls and logged work: `actor`, `logged_by`, `disposition`, `note`.
- **`list_account_milestones`** returns one account's board, judged now.
  - **Account fields:** `stage`, `completed`, `entered_stage_at`.
  - **Per milestone:** `stage`, `position`, `is_required`, `group_key` (alternatives), `status`
    (`pending`, `achieved`, `skipped`, `error`), `achieved_at`, `progress` (each condition's count
    against its bar), and the skip reason.
  - **Use:** next to the timeline, to see whether a step followed a nudge.
- **`list_activity`** lists activity rows across accounts, newest first.
  - **Window:** `occurred_from` to `occurred_to`, at most 31 days.
  - **Filters:** at least one of `origins`, `channels`, `kinds`, `milestone` or `account_ids`.
  - **Rows:** the timeline shape, plus `account_id`, `external_id`, `account_name`.
  - **Pages:** continue with `cursor` = `next_cursor` until it is null; `limit` up to 100.
- **`list_tasks`** lists tasks.
  - **Window:** `moment` (`created_at`, `due_at` or `closed_at`) picks which date the window reads,
    at most 92 days.
  - **Filters:** `statuses` (`open`, `done`, `cancelled`, `resolved_by_progress`), `origins`
    (`commitment`, `personal`, `rule`, `crm`), `types`, `assignees`, `account_id`.
  - **Rows:** every field: title, notes, priority, status, due, `overdue`, the agreed due, creator,
    assignee and closer, when it was created and closed, the resolution, account and contact.
  - **Pages:** follow `next_offset`.
- **`list_sessions`**
  - **Rows:** `summary`, `status`, `tags` (topic tags) and contact fields (current values).
  - **Filters:** `status`, `active_after`/`active_before`, `tags` (tag ids as strings, `none`).
  - **Pages:** `limit` up to 50; deduplicate by `id`.
  - **`get_session`** reads the messages.
- **`list_conversations` and `get_conversation`:** threads handed to the team.
  - Filter `was_escalated` = true for hand-offs, and continue with `next_active_before` while
    `has_more` is true.
  - The `notes` entry with `author_type` = `ai` is the agent's hand-off note.
- **`list_answer_sources`** takes exactly one of `message_id`, `session_id` or `conversation_id`.
  - **Empty result:** the agent did not search, or the knowledge base had nothing.
  - **The right document is absent:** a content gap.
  - **The right document is present:** not a gap. Read the document; it was ignored or it is wrong.

## Traps that change results

- **Read every page.** A capped list looks complete.
- **Activity windows are not event windows.** `active_after` and `active_before` bound the last
  message. To select by when something happened, use a tool with an event window
  (`get_nudge_funnel`, `list_activity`, `list_tasks`).
- **Live versus daily.** `health` on account rows is live; `get_activation_health_history` reads
  one stored reading per day. Do not mix them in one comparison.
- **Product events stay with the client.** The connector reports Blar's activity, the board and its
  milestones, not every event of the client's product. Join the client's own data on the
  `external_id` fields when a question needs it.

## Reporting what is missing

Call `send_feedback` once per distinct problem.

- **`category`:** `missing_capability`, `tool_error`, `confusing_output` or `suggestion`.
- **Fields:** `message` (what was missing and what you expected), `tool_name`, and `intent`.
- Leave personal data about the company's customers out of the message.
