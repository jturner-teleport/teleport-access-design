# GitOps pattern: per-app edit scoping for Application Admins

This directory answers the question raised in `OPEN_QUESTIONS.md` #5:
Teleport's RBAC has no native way to let an Application Admin edit
*only* their own `app` resource — `rules: [{resources: [app],
verbs: [update]}]` grants edit rights to **every** app in the cluster,
because create/update/delete on dynamic resources isn't label-scoped
(see README.md's RBAC notes and the discussion this repo is based on).

So instead of trying to express "edit only my app" in Teleport RBAC,
this pattern moves that boundary into **git**, where per-path
ownership and review already exist as first-class features.

## How it works

```
Application Admin edits gitops/apps/<their-app>.yaml
        │
        ▼
   Opens a PR
        │
        ▼
CODEOWNERS requires review from that app's own admin team
(and only that team — they can't approve changes to someone
 else's app file)
        │
        ▼
   PR merged to main
        │
        ▼
CI pipeline runs, authenticates as a Machine ID bot via
GitHub Actions OIDC (no static token), and applies ONLY the
changed files with `tctl create -f <file>`
        │
        ▼
  Teleport cluster updated
```

The bot's Teleport role (`roles/role-app-definitions-bot.yaml`) has
broad `app: [create, update, delete]` rights — same as any
`app`-editing role would need, since that can't be narrowed in
Teleport itself. **The actual per-app boundary is enforced by
CODEOWNERS + branch protection, not by Teleport RBAC.** No human ever
holds `app:update` directly; only the CI bot does, and the bot only
acts on reviewed, merged diffs.

## Files in this example

- `apps/example-app.yaml` — one `app` resource definition per app.
  Add one file per application here.
- `CODEOWNERS.example` — maps each app's file to the team/admin who
  owns it. Rename to `CODEOWNERS` and place at the repo root (or
  `.github/CODEOWNERS`) to activate.
- `roles/role-app-definitions-bot.yaml` — the Teleport role granted to
  the CI bot identity, not to any human.
- `tokens/github-actions-app-bot.example.yaml` — a GitHub Actions
  OIDC join token for the bot (no static secret, per the earlier
  recommendation to avoid minting shared tokens for things like this).
- `ci-example/apply-app-changes.yml` — example GitHub Actions workflow.
  Copy into `.github/workflows/` in your real app-definitions repo to
  activate (left out of `.github/workflows/` here so this example
  repo doesn't run it).

## Caveats specific to this pattern

1. **Protect the CODEOWNERS file and the workflow file themselves.**
   If an Application Admin can edit `CODEOWNERS` or
   `.github/workflows/apply-app-changes.yml`, they can grant
   themselves (or remove) ownership of other apps' files, or change
   what the bot applies — a direct escalation path. Both files should
   be owned by the platform/Super Admin team in `CODEOWNERS` itself,
   and branch protection should require their review on any change to
   `CODEOWNERS` or `.github/workflows/`.

2. **Enforce branch protection with no bypass.** "Require review from
   Code Owners" must apply to everyone, including repo admins —
   otherwise an app admin with elevated git permissions could merge
   past the CODEOWNERS gate.

3. **The bot's Teleport role is intentionally broad.** This pattern
   doesn't reduce the bot's Teleport-side permissions — it moves the
   *human* boundary to git. Treat the bot's credentials/join method
   with the same care as any role with cluster-wide `app:update`,
   and keep its CI job's logic minimal and auditable (only apply
   files that changed in the merged diff, nothing else).

4. **This pattern generalizes.** The same approach works for any
   resource kind where Teleport RBAC can't scope create/update to a
   per-resource owner — e.g. `db`, `kube_cluster`, `windows_desktop`
   definitions — not just `app`.
