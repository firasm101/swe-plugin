---
name: radview-mkt-jira
description: Handle a Jira MKT ticket end-to-end — pickup, in-progress comment, transition, done comment. Use whenever picking up, transitioning, or commenting on an MKT ticket. Encodes the workflow trap (transitions are mislabeled in this project) and the sole-creator rule.
---

# How to handle a Jira MKT ticket

This is the radview-specific procedure. The Atlassian Rovo MCP gives you the API surface; this skill encodes the rules.

## Authority — who creates tickets

**Claude in chat is the sole creator of MKT tickets.** Firas does not file Jira tickets; Claude Code does not file Jira tickets. When Claude Code (running locally on Firas's machine) discovers a new issue, it surfaces the issue in its report — Claude in chat then files the ticket.

Claude Code CAN:
- Comment on tickets it is actively working on
- Transition tickets it is actively working on (In Progress on pickup, Done on completion)

Claude Code CANNOT:
- Create new tickets
- Decide ticket scope (only execute against an existing ticket)

## The transition trap (CRITICAL)

The MKT project's workflow has mislabeled transitions. **The transition names in the API do NOT match the destination states.**

Empirically verified mapping (from `GET /rest/api/3/issue/<KEY>/transitions`):

| Transition ID | Transition name | Actual destination |
|---|---|---|
| 21 | "In Progress" | **Approved** ⚠️ |
| 31 | "Done" | **In Progress** ⚠️ |
| 5 | "Done" | **Done** ✅ |

Rules:
- **Done = id=5 direct.** NEVER via the "Approved" path. Going through Approved triggers an agent-automation loop on the project board and stalls the ticket.
- Always verify by calling `GET /rest/api/3/issue/<KEY>/transitions` for the current ticket before any state change. The available IDs may differ from this list depending on the current status.
- Don't trust transition NAMES — trust the ID + verify the destination by reading the `to.name` field of each transition object.

## Pickup — first comment

When Claude Code (or Claude in chat) starts work on an MKT ticket:

1. Transition to In Progress (use the transition that lands in `to.name == "In Progress"`)
2. Add a comment describing:
   - What you understood the ticket to be asking
   - The plan in 2-4 bullet points
   - Any Step 0 schema-verify findings if relevant (CONVENTION.md §7.1)

Comment via `addCommentToJiraIssue` with `contentFormat: "adf"`. ADF requires every text node be wrapped in a paragraph node — orphan text at the document root produces `INVALID_INPUT` 400 with no detail.

ADF skeleton:

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    { "type": "paragraph", "content": [{ "type": "text", "text": "Plan: ..." }] }
  ]
}
```

## During work — periodic comments

For long-running tickets (multi-PR, multi-day), comment when:
- A PR is opened against the ticket — link it
- A blocker is hit — describe and tag if it needs Firas
- A schema-verify finding changes the ticket scope — call this out explicitly (this is the MKT-415 pattern)

## Closing — Done comment + transition

Before transitioning to Done:

1. **Verify per CONVENTION.md §8 Definition of Done.** Different change types have different verifications — check the table.
2. **Add a comment** that includes:
   - What shipped (1-3 lines)
   - Verification artifact (screenshot URL, API result, cron-tick result, etc.)
   - Any followups (operator actions Firas owes — list explicitly)
3. **Transition to Done via id=5 direct.** Never via "Approved." Always verify the destination is `Done` by reading the transition response.

```bash
# Pattern (curl form for clarity; Atlassian Rovo MCP wraps this):
curl -X POST \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  https://firasm101.atlassian.net/rest/api/3/issue/MKT-XXX/transitions \
  -d '{"transition":{"id":"5"}}'
```

## Audience designation rule (MKT-392 / PR template)

Every PR for an MKT ticket declares its audience: customer / operator / engineer / infra. The audience determines:

- Which surface the change appears on (`/:tenantSlug/*` vs `/admin/*` vs `/docs/*`)
- Which CI guards apply (engineer-facing changes don't need data-lineage updates; customer-facing always do)
- Which architectural diagram tier needs updating
- The verification ritual (Vercel preview screenshot for customer/operator; doc render for engineer; runtime check for infra)

If a ticket spans multiple audiences, split into multiple PRs.

## ADR check (MKT-396)

An ADR is required when the ticket involves:
- Choosing between two architectural options (e.g., "should this be a view or a materialized view?")
- Deprecating an existing pattern
- Changing a public contract or schema in a way that breaks consumers
- Introducing a new infrastructure dependency
- Establishing a non-obvious convention

If the ticket has any of those, file an ADR at `docs/adr/NNNN-title.md` in the same PR. The PR template has a checkbox.

## Sprint membership

Default sprint is Sprint 4 (id=137). Set via `customfield_10020: 137` when creating tickets. If the project has rolled to Sprint 5+, lookup the new ID via `/rest/agile/1.0/board/<board-id>/sprint?state=active`.

## Issue type IDs

| Type | ID |
|---|---|
| Epic | 10080 |
| Task | 10077 |
| Bug | 10078 |
| Story | 10079 |

## Cloud ID

`5649b75f-4f9a-4b69-ad38-33f1a09d91f2` — required as the first argument to every Atlassian Rovo MCP call.

## Trap log

- **Transition mislabel:** id=21 "In Progress" lands in Approved, id=31 "Done" lands in In Progress. Verified empirically; do not trust names.
- **Approved automation loop:** transitioning through Approved triggers the project board's agent automation, which can stall and not auto-transition Done. Always use id=5 direct.
- **ADF orphan text:** ADF requires text wrapped in paragraph nodes. Orphan text at document root → 400 INVALID_INPUT with no body detail.
- **Container egress for Atlassian:** the bash environment proxy returns `503 DNS cache overflow` for `*.atlassian.net`. Use the Atlassian Rovo MCP tools instead of curl from `bash_tool`. If MCP is also unavailable, hand off the curl command to Firas to run from his local machine.
- **Sole creator rule:** Claude Code occasionally tries to file a ticket when discovering a new issue. The right behavior is to surface the issue in its report; Claude in chat then files. Mixing these blurs accountability for ticket scope.

## Quick reference checklist

When picking up a ticket:
- [ ] Read the ticket. Confirm scope.
- [ ] Verify CONVENTION.md §7.1 Step 0 schema if SQL is in scope.
- [ ] Comment plan (ADF, paragraph-wrapped).
- [ ] Transition to In Progress (verify destination via transitions endpoint).
- [ ] Determine audience per MKT-392.
- [ ] ADR required? If yes, draft alongside the code change.

When closing:
- [ ] PR merged + migrations applied.
- [ ] Verification artifact captured per CONVENTION.md §8 Definition of Done.
- [ ] Comment closing with shipped + verified + followups.
- [ ] Transition to Done via id=5 direct.
- [ ] CONVENTION.md updated if a canonical value changed.
- [ ] ROUTER_AUDIT.md entry appended.
