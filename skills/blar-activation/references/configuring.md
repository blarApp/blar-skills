# Changing Blar through the connector

The connector can change two things:

- **The documentation** the AI agent answers from.
- **The feature catalog**, which is what activation teaches.

Everything else is changed by Blar's team; see the last section.

## Ground rules

1. **Changes are live.** They reach real accounts and the AI agent's next answers. Show the user the
   exact change (what, from what, to what) and wait for an explicit yes before calling a write tool.
2. **Read before writing.** Read the current document or feature first, and its history for
   documents.
3. **Check the result.** A response can say the change did not happen (`pending_review`, a blocked
   publish). Report what the response says, never what was intended.
4. **Never delete what you did not write** without the user's explicit yes for that item.

## Documentation

### Correct a document

1. Find it with `search_documentation` or `list_documentation`, then read it with
   `get_documentation`.
2. Call `get_documentation_history`. Keep everything the revisions marked `team_edit: true` added,
   unless it is known to be wrong: teammates usually add it after a real conversation went badly.
3. Call `update_documentation`(`knowledge_piece_id`, `content`). The content is markdown and
   replaces the whole body, so send the full merged text.
4. Read the `status`:
   - `applied`: the change is live.
   - `pending_review`: the change was NOT applied. A teammate last wrote this document, so the
     edit waits for them to accept it. Report it as waiting.

Some documents are refused because a pipeline regenerates them: API references extracted from
pages, imports from other help desks, scraped sites, and indexes. Fix those at their source.
Documents generated from the product's pages can be corrected; the correction survives
regeneration. A document published by a sync can have its content changed but not its name.

### Add a document

1. Search first. Two documents answering one question make the AI agent contradict itself, so
   prefer correcting the existing one.
2. Call `create_documentation`(`name`, `content`, `tags`). A name already held by a synced document
   replaces it, so read that document first and ask before overwriting it. A name held by any other
   kind of document is refused.
3. The new document is searchable only after indexing, a short while later. A search right away
   will not find it.

### Delete a document

`delete_documentation`(`name`) is permanent. Read the document first and confirm with the user. The
`status` is one of:

- `deleted`;
- `not_found`, also returned for documents the connector may not delete;
- `pending_review`: a teammate must confirm, and the document is still there.

### The review queue

`get_documentation_history` shows `pending_change`: the proposal's `diff` and its
`proposal_version`.

- **Accept:** `accept_documentation_review`(`knowledge_piece_id`, `proposal_version`). To keep only
  part of the proposal, pass `content` with the complete final text. Never accept a proposal that
  drops what the team wrote.
- **Reject:** `discard_documentation_review`(`knowledge_piece_id`, `proposal_version`).
- If the proposal changed since you read it, the call is refused; read the history again.
- A waiting deletion can only be confirmed in Blar's dashboard.

## Features (admins of an approved company)

### What a feature is

- **Scope:** a feature is a real capability of the company's product. It belongs to one product
  (`project_id`) or to the whole company, and can be limited to countries (`countries`; an empty
  list means all).
- **Tier,** which says how activation teaches it:
  - `primary`: a milestone of activation;
  - `secondary`: presented once, and the user can skip it;
  - `tertiary`: a tip.
- **Position:** the order features are presented in, within their tier.
- **Documents:** the source documents the feature's knowledge is generated from.
- **Amplifiers:** helper features attached to a base feature, each with the reason it helps.
- **Two states:**
  - The working state, which is what you edit.
  - The `published_snapshot`, which is what activation uses now.
  - `status` is `draft`, `publishing`, `published` or `archived`.

### Read

- `list_features`. Filters: `status`, `tier`, `search`, `project_id` or `shared_only`, `country`.
  Follow `next_offset` while `has_more` is true.
- `get_feature`(`feature_id`) returns the working state, the `published_snapshot`, the
  `publish_blockers`, and `generation_tasks`.

### Create a feature

1. `create_feature`(`name`, `description`, `tier`, `project_id`, `countries`, `position`) creates a
   draft. Without a `position`, it goes last in its tier.
2. `replace_feature_documents`(`feature_id`, `knowledge_piece_ids`) sets its source documents, in
   order.
3. `request_feature_context_generation`(`feature_id`) starts building the feature's knowledge in the
   background. Poll `get_feature` until `generation_tasks.context.state` shows it finished or failed.
4. Resolve every item in `publish_blockers`.
5. `publish_feature`(`feature_id`).

### Change a feature

- **Fields:** `update_feature`(`feature_id`, and any of `name`, `description`, `tier`, `position`,
  `countries`). Fields left out are kept.
  - **Takes effect immediately:** `countries` and `position`.
  - **Needs `publish_feature` on a published feature:** text and `tier`.
  - **Cannot change:** the product.
  - A new `tier` moves the feature to the end of that tier unless `position` is also passed.
- **Documents:** `replace_feature_documents` sends the whole list; ids left out are removed. This
  makes the generated knowledge stale, so regenerate it and wait before publishing.
- **Order of a tier:** `reorder_tier_features`(`tier`, `feature_ids`) takes every non-archived
  feature of the tier, drafts included, first to last. Read them with `list_features(tier=…)`
  first. An incomplete or mixed list is rejected. The new order is live at once.
- **Amplifiers:** `replace_feature_amplifiers`(`feature_id`, `amplifiers`) sends the whole ordered
  list of helpers, each with why it helps. Entries left out are removed. Publish the helper first,
  then the base feature: publishing a helper does not republish the base.

If a publish request errors after it was accepted, read `get_feature` before retrying.

## From finding to change

| Finding (gold-standard.md) | Change through the connector | Otherwise |
| --- | --- | --- |
| A large topic with no answering document (G12) | Write or correct the document | — |
| A hand-off where the right document was absent or wrong (G13) | Write or correct the document | — |
| A hand-off where the agent did not search (G13) | — | `send_feedback` |
| A step with high delivery and few answers (G6) | Sharpen the feature's description and documents, then republish | Message wording: Blar's team |
| A lagging segment, country or product (G2) | Feature countries, order, or a product-specific feature | — |
| A bottleneck stage (G3) | The documents, order or amplifiers of the features behind that stage | Milestone definitions: Blar's team |
| Accounts never reached (G4) | — | Contact data and channels: Blar's team |
| No follow-up after a yes (G7) | — | Cadence: Blar's team |

## What only Blar's team can change

- Pipelines, stages, milestones and their conditions.
- Nudge timing, channels, message templates and the number of follow-ups.
- Segments, accounts, contacts, and who owns an account.
- Tasks and escalation rules.

To request one of these, call `send_feedback` (category `suggestion` or `missing_capability`) with
the finding that motivates it, and tell the user to also raise it with their Blar contact.
