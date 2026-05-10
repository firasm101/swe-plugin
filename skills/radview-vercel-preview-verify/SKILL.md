---
name: radview-vercel-preview-verify
description: Verify a change shipped on Vercel preview before merging to master. Use whenever the task affects user-visible /admin/* or /:tenantSlug/* surfaces. Replaces upstream's generic "test it" instruction with the screenshot ritual specific to this repo.
---

# How to verify a change on Vercel preview

Verification is not "did the build pass" — that's already enforced by CI. Verification is "does the new behavior look right when a real user lands on it." For UI-affecting changes, this means a screenshot and an explicit comparison.

## When to use this skill

- Any change to `dashboard/src/admin/*` (operator UI)
- Any change to `dashboard/src/<tenantSlug>/*` (customer UI)
- Any change to `dashboard/api/*.ts` that returns user-visible JSON (operator dashboards)
- Any new `/admin/*` or `/:tenantSlug/*` route

When NOT to use:
- Pure backend changes (crons, ingestion scripts)
- Migration-only PRs (use the `information_schema` verify in the migration skill)
- Documentation-only PRs

## Step 1 — Find the preview URL

After your PR pushes, Vercel posts a comment with the preview URL. The pattern is:

```
https://radview-marketing-intelligence-git-<branch-slug>-<vercel-team>.vercel.app
```

You can also check the latest deploy via the Vercel API:

```bash
curl -sS -H "Authorization: Bearer $VERCEL_TOKEN" \
  "https://api.vercel.com/v6/deployments?projectId=prj_MFCRqFA0uzRnSkPNZQd03krTSicr&limit=1" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['deployments'][0]['url'])"
```

The dashboard's stable preview alias is `dashboard-gray-chi-75.vercel.app`.

## Step 2 — Operator login (for /admin/* screenshots)

`/admin/*` routes require operator auth. The flow is:

1. Hit `https://<preview>.vercel.app/admin/login`
2. Use the operator JWT or magic link from your existing browser session — Claude in Chrome's `select_browser` tool reuses an authenticated session
3. After login you should be on `/admin/health` or similar landing page

For customer routes (`/:tenantSlug/*`) use the synthetic tenant: `radview` (tenant_id=1) or `synthetic-9990` if testing tenant isolation.

## Step 3 — Navigate to the surface affected

Open a new tab via Claude in Chrome (`tabs_create_mcp`), then `navigate` to the relevant URL on the preview domain. Common targets:

- `/admin/health` — invariants pass/fail status
- `/admin/services` — services catalog
- `/admin/data-lineage` — table writers/readers
- `/admin/credentials` — operator-set per-tenant credentials
- `/admin/synthetic` — synthetic tenant operator panel
- `/:tenantSlug/dashboard` — customer dashboard
- `/:tenantSlug/strategy` — customer strategy view

## Step 4 — Take the screenshot

Use Claude in Chrome's `computer` tool with `action="screenshot", save_to_disk=true`. The screenshot path is returned in the tool result. Attach it to the verification artifact.

Two-screenshot rule: when changing existing UI, take a screenshot BEFORE your change (master) and AFTER (your branch's preview). Side-by-side in the PR comment or attached to the Jira ticket.

For new UI: one screenshot showing the new state is enough.

## Step 5 — Verify the API behind the surface

The screenshot proves the page renders. The API call proves the data is right. For new endpoints, hit the API directly:

```bash
curl -sS "https://<preview>.vercel.app/api/<endpoint>" -H "Authorization: Bearer $OPERATOR_JWT"
```

Compare the response shape against your spec. For invariant-related endpoints, also query the underlying Supabase table to confirm the data flow:

```bash
curl -sS "https://deiyfumozjyhfyxdzqlc.supabase.co/rest/v1/<table>?<filter>" \
  -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY"
```

## Step 6 — Write the verification artifact

In the PR description, under *Verification on production (post-merge)*, paste:

```
## Verification

Preview: https://<preview-url>

- /admin/health: [screenshot link or attached image]
- API: GET /api/<endpoint> → 200, response shape matches spec
- Supabase: <table> contains expected row

Compared to master at /admin/health (commit <sha>): no regression on existing invariants. New invariant <id> shows passed=true.
```

For a Jira ticket, the same artifact goes in a comment on the ticket — Claude in chat creates the comment via Atlassian Rovo MCP after the PR merges.

## Trap log

- **PR previews need login each session.** Vercel preview URLs reset cookies; operator login is a per-preview action. Use `select_browser` to reuse an authenticated browser tab when possible.
- **Stale preview after force-push.** When you `git push --force-with-lease`, Vercel may briefly serve the old build. Wait for the "Vercel" comment on the PR to update before screenshotting.
- **Cron-driven UI surfaces.** `/admin/health` shows invariant results; results only refresh on cron tick (every 30 min). For these, the screenshot proves the page renders; the cron-tick verification is a separate step (see CONVENTION.md §8 Definition of Done).
- **Don't screenshot in incognito.** Operator auth requires the session — incognito drops you to login.

## Quick reference checklist

- [ ] Preview URL identified (Vercel comment or API)
- [ ] Logged in as operator (or tenant for customer surfaces)
- [ ] Navigated to affected surface; page rendered without console errors
- [ ] Screenshot taken with `save_to_disk=true`
- [ ] Before/after screenshots if existing UI changed
- [ ] API response shape verified separately if endpoint changed
- [ ] Supabase data verified separately if data flow changed
- [ ] Verification artifact written into PR description and/or Jira comment
