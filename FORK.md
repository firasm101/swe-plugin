# Fork: firasm101/swe-plugin

This is a fork of [eymass/swe-plugin](https://github.com/eymass/swe-plugin) maintained for the RadView Marketing Intelligence repo.

## Why we forked

The sibling-plugin pattern (radview-plugin alongside swe-plugin) couldn't add custom router intents. When `swe-router` classifies an MKT Jira ticket, it needs a `mkt-ticket` pipeline that loads the radview skills in the right order with the right gates. That pipeline lives in `skills/swe-router/SKILL.md`, which is upstream code. To extend it without ongoing rebase pain, we forked.

Forked from upstream commit at: `8947537` (Update SKILL.md, see `git log upstream/master`).
Fork tag for our consumers: `v2.0.0-radview` (this commit).

## Divergence

Files we OWN (forked, never pull from upstream):
- `.claude-plugin/plugin.json` — our identity, our version
- `skills/swe-router/SKILL.md` — our `mkt-ticket` intent + pipeline
- `skills/radview-supabase-migration/SKILL.md` — radview-only
- `skills/radview-vercel-preview-verify/SKILL.md` — radview-only
- `skills/radview-mkt-jira/SKILL.md` — radview-only
- `FORK.md` — this file

Files we INHERIT (pull from upstream when valuable):
- All other `skills/*` (code-implementation, tests-implementation, swe-documentation, etc.)
- All `agents/*`
- `hooks/*`
- `prompts/*`
- `tools/*`
- `Makefile`, `README.md`

## Pull-from-upstream procedure

```bash
cd swe-plugin
git fetch upstream
git checkout master

# Inspect what changed upstream since last pull
git log master..upstream/master --oneline

# Merge with strategy: take upstream by default, keep ours for OWN files
git merge upstream/master --no-commit --no-ff
git checkout HEAD -- \
  .claude-plugin/plugin.json \
  skills/swe-router/SKILL.md \
  skills/radview-supabase-migration \
  skills/radview-vercel-preview-verify \
  skills/radview-mkt-jira \
  FORK.md

# Manual review: did upstream change agents we use? Skills we depend on?
git diff --cached
git commit -m "Pull from upstream eymass/swe-plugin <upstream-sha>"
git tag v2.0.<n>-radview
git push origin master --tags
```

Then in the consumer repo (radview-marketing-intelligence):

```bash
cd .claude-plugins/swe-plugin
git fetch
git checkout v2.0.<n>-radview
cd ../..
git add .claude-plugins/swe-plugin
git commit -m "Bump swe-plugin submodule to v2.0.<n>-radview"
```

## Submodule pinning rule

Consumers MUST pin to a tag, never to `master`. `master` is for our development; tags are for consumption. The radview-marketing-intelligence repo's `.gitmodules` should pin to `v2.0.0-radview` (or later released tag).

## When to NOT pull

- Upstream changed `swe-router/SKILL.md` heavily — read upstream's intent table additions and decide whether to incorporate them into ours, but never replace ours wholesale.
- Upstream removed something we depend on — keep our copy; do not let upstream removal break us.
- Upstream merged a security fix — pull immediately, even if other changes need work.

## Authority

Conflicts between upstream conventions and CONVENTION.md in the consumer repo: CONVENTION.md wins. The fork enforces CONVENTION.md as Step 0.5 in the `mkt-ticket` pipeline.
