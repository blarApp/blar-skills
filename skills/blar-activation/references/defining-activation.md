# Agreeing on what activation means

Blar measures activation with a board of milestones set up for this company. The company may count
an account as activated at another point. Every curve and every rate depends on that point, so agree
on it before measuring, and measure on the company's definition. The board stays Blar's path for
getting accounts there, whichever point the company picks.

## 1. Look for an earlier agreement

Check your memory and the project's instructions for a definition agreed in an earlier conversation.
If one is there, show it in one or two lines and ask whether it still holds. Ask the questions below
only for what is missing or has changed.

## 2. Read what Blar has configured

Read this before asking. The questions are built on it.

- **The board.** Boards can differ by segment and by product, so read one of each.
  - For each segment and each product in `get_activation_filters`, call `list_activation_accounts`
    with that `segments` or `project_id` filter and `limit` = 1. Then call `list_account_milestones`
    for the account it returns.
  - Group the milestones by `stage`, in board order: `hand_off`, `set_up`, `first_value`,
    `core_value`, `repeat_value`. Skip stages with no milestones.
  - For each milestone, note its `name`, its `description`, and its condition in `progress`: the
    event (`event_type_name`) and the count it needs (`required`).
  - Milestones of one stage with the same `group_key` are alternatives: any one of them is enough.
- **The core features.** Call `list_features` with `tier` = `primary` and `status` = `published`, and
  keep the order returned.
- **Blar's own definition.** An account is activated (`completed_at`) when every stage of its board
  is done.

## 3. Ask

Show what you read, briefly: one table per distinct board (stage, milestone, condition,
alternatives) and the ordered core features. Then ask, in one message:

1. **What do you measure to call an account activated?** Offer the board as options:
   - the whole board, which is Blar's definition;
   - finishing a stage (name the stages the board has);
   - one milestone;
   - something else, such as doing something in the product a number of times.
2. **Are these your core features?** These are the features an account has to use to get value.
   Ask which are missing, and which are listed but not core.
3. **Are these the milestones an account has to complete to stay engaged?** Ask about missing steps,
   wrong thresholds (the counts in the conditions), and steps in the wrong stage.

In the same message, propose a value for each of these and ask the user to confirm it:

- the **time zone** of every date and weekly cohort, as an IANA name such as `America/Mexico_City`;
- the **accounts to leave out**, such as internal or demo accounts, by name or external id. Accounts
  Blar marks as test accounts are already left out;
- the **language** of the answers.

When Blar has no board or no core features for this company, ask the three questions without the
options. Say that, until Blar's team sets up the board, only a definition built from product events
can be measured.

Wait for the answers. Never choose a definition, a time zone or the accounts to leave out on the
company's behalf.

## 4. Save the agreement

Write the agreement as one short block: the condition, the time zone, the accounts left out, the
language and the date it was agreed. Suggest the user save it, by asking you to remember it or by
adding it to the project's instructions, so the next conversation finds it in step 1.

## 5. Turn the answer into a condition

| The company's answer | `get_condition_completion` arguments |
| --- | --- |
| The whole board | None: Blar's own definition |
| Finishing a stage | `stage` |
| One milestone | `milestone_id`, from `list_account_milestones` |
| Doing something a number of times | `events`: each `event_type_id` from `list_event_types` with its `count`. Every count has to be reached |
| Something Blar does not receive | Not measurable through the connector. Join the company's own data on `external_id` |

When the company describes the action in its own words, such as "products created", find the event
in `list_event_types` and confirm the match with the user before measuring.

## 6. Gaps between the answers and the board

Each gap becomes a suggestion, with who acts on it.

| Gap | Who acts, and how |
| --- | --- |
| A core feature is not in the primary tier, or a primary feature is not core | The user, through the connector: the feature's tier (configuring.md) |
| The core features are presented in the wrong order | The user, through the connector: `reorder_tier_features` (configuring.md) |
| A missing milestone, a wrong threshold, or a step in the wrong stage | Blar's team: `send_feedback` (`suggestion`) with the finding, and the user raises it with their Blar contact |
| The company counts activation before the end of the board | Nothing to fix. Measure on the company's point; the board keeps guiding accounts past it |
