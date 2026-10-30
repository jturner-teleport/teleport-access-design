# Open Questions / Caveats

These need answers before this design goes to production. None of the
example YAML in this repo should be treated as final until these are
resolved.

## 1. Org Delegated Admin is a privilege-escalation path as currently scoped

`role-org-delegated-admin.yaml` grants `role: create/update`. Taken
literally, "can manage Roles/ACLs so each org can self-manage" lets an
org admin create a brand-new role with
`allow: { rules: [{ resources: ["*"], verbs: ["*"] }] }` and grant it
to themselves via an Access List they own — full cluster admin,
bypassing Super Admin entirely.

Options:
- Don't grant `role: create/update` directly. Limit Org Delegated
  Admins to Access List **membership/grants only**, from a fixed
  catalog of pre-approved roles that Super Admins define.
- Manage roles via GitOps/Terraform with PR review; give Org Delegated
  Admins Access List ownership only, no direct `role` resource access.

**Needs a decision** — "self-manage their access structure" and "can't
escalate to full admin" are in tension as currently scoped.

## 2. `.admin` accounts — are they distinct identities, or aliases?

If the `.admin` account (e.g. `jdoe.admin@acmecorp.io`) is an email
**alias** on the same underlying Workspace account as the normal
account, rather than a fully separate account (separate login, separate
2FA, separate OU), this design breaks: Google's `sub` and `email`
claims resolve to the primary identity, and group membership doesn't
attach cleanly to an alias the way it does to a real, separate account.

**Needs confirmation** from whoever manages the existing admin-account
convention in other systems: is `.admin` a second real account, or an
alias?

## 3. Peer-approval for Super Admin changes is process-only, not enforced by Teleport

As specified (`editor` granted via standing group membership), Teleport
does nothing to require a second approver before a Super Admin change
takes effect — a change-request/peer-review process outside Teleport
could simply be skipped. If this needs to be technically enforced
rather than procedural, `editor` can be made request-only via
`allow.request` with `thresholds: [{ approve: 2 }]` and
`required_approvers`, so a second Super Admin must actively approve
before the elevated role is active.

**Needs a decision** — is procedural (CR-based) approval sufficient, or
does this need to be technically enforced?

## 4. "Integration Admin scoped for AWS" is broader in practice

Teleport's `integration` resource type isn't sub-scoped by provider at
the RBAC layer. `resources: [integration], verbs: ["*"]` grants rights
to create/delete **any** integration type (AWS, GitHub, Okta, Jamf,
etc.), not just AWS.

**Needs confirmation** — is "AWS only" a hard requirement, or just the
primary use case today? If hard, this boundary needs to be enforced by
process/training rather than RBAC, since Teleport doesn't offer a
built-in "AWS-only" scope on the `integration` resource.

## 5. Application Admin scope is broader than "their app" — RESOLVED (pattern, not RBAC)

Two related gaps, both with the same root cause: Teleport's generic
`rules` mechanism for CRUD verbs (`create`/`update`/`delete`) on a
resource kind is cluster-wide — it isn't filtered by labels the way
connection-type access (`app_labels`, `node_labels`, etc.) is.

- `token: create` isn't restricted to App-type joins — the same
  permission can mint tokens for Node, Kube, DB, or Windows Desktop
  joins too, not just the app a given admin owns.
- `app: update` (editing an app's own resource definition — URI,
  public_addr, labels) isn't restricted to apps the admin owns either
  — granting it grants edit rights to **every** app in the cluster.
  Confirmed against Teleport's docs: `where` predicate filtering on
  `rules` only supports `session`/`session_tracker` fields, and
  Teleport explicitly documents `token` (and by the same mechanism,
  `app`) CRUD as not scoped to a resource context.

**There is no RBAC-only fix for either of these.** The scoping has to
move outside Teleport. For editing app definitions specifically, see
`gitops/README.md` — a GitOps pattern using CODEOWNERS + branch
protection to enforce the per-app boundary in git, with a CI bot
(Machine ID, GitHub Actions OIDC join, no static token) holding the
necessarily-broad `app:update` permission instead of any human. The
same pattern applies to the token-minting gap: don't give humans
`token:create` at all — use identity-based join methods (AWS/GCP/Azure/
Kubernetes/CI-OIDC) so no one needs to mint a token by hand, or route
token issuance through the same bot-plus-review pipeline.

## 6. Nested Google Groups

If `teleport-super-admins@acmecorp.io` (or the other connector groups)
are themselves nested inside broader IT groups, verify with
`tctl sso test` that membership resolves correctly before relying on
it — don't assume indirect/nested membership is picked up by the
Admin SDK call Teleport makes.

## 7. OAuth consent screen type

Confirm the Google Cloud OAuth consent screen is set to **Internal**
(restricted to `acmecorp.io` Workspace accounts), not **External**. If
it's External, any Google account can reach the login prompt (they'd
still be denied for lack of group membership, but it's unnecessary
exposure of the login flow and client ID).
