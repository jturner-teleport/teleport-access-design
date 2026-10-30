# Teleport Access Design (Example)

This repo is a **worked example** of a Teleport RBAC + Google Workspace SSO
access structure. It's intentionally generic/obfuscated — all domains,
company names, and group addresses below are fictional
(`acmecorp.io`) and must be replaced with real values before use.

Nothing in this repo contains real credentials, keys, or secrets. Every
YAML file under `roles/`, `access-lists/`, and `connector/` uses
placeholder values and is meant to be copied and adapted, not applied
as-is.

## Access tiers

| Tier | Account type | Mechanism | Teleport role |
|---|---|---|---|
| Super Admin | `.admin` account | connector group (`teleport-super-admins@acmecorp.io`) | `editor` (built-in) |
| Auditor Admin | `.admin` account | connector group (`teleport-auditor-admins@acmecorp.io`) | `auditor` (built-in) |
| Integration Admin | normal account | connector group (`teleport-integration-admins@acmecorp.io`) | `role-integration-admin` |
| Org Delegated Admin | normal account | Access List owner | `role-org-delegated-admin` |
| Application Admin | normal account | Access List owner | `role-application-admin` |
| App User | normal account | Access List member | app-scoped role, e.g. `role-app-readonly` |

Super Admin, Auditor Admin, and Integration Admin are centrally managed
through the OIDC connector's `claims_to_roles` (see
`connector/oidc-google.example.yaml`). Org Delegated Admin, Application
Admin, and App User are intentionally **not** wired through the
connector — they're granted via [Access
Lists](https://goteleport.com/docs/identity-governance/access-lists/)
instead, so each org/app owner can manage their own membership without
a Super Admin editing the central connector every time.

## Why Access Lists for the delegated tiers

If every org or app's access got its own `claims_to_roles` entry, "each
org manages their own access structure" wouldn't really be true — every
new app or team would still require a Super Admin to edit the
connector. Access Lists let an **Owner** (an Org Delegated Admin or
Application Admin) manage **membership** and role grants on their own
list, without touching the connector or needing any Teleport admin
role themselves.

## Files in this repo

- `connector/oidc-google.example.yaml` — the OIDC connector mapping
  Google Groups to `editor` / `auditor` / `role-integration-admin`.
- `roles/role-integration-admin.yaml` — scoped for platform-level
  integrations (e.g. AWS).
- `roles/role-org-delegated-admin.yaml` — lets an org manage its own
  Access Lists; **see open question #1** before granting this as
  written.
- `roles/role-application-admin.yaml` — can mint join tokens and manage
  the Access Lists for apps they own.
- `roles/role-app-readonly.yaml` — example end-user-facing role granted
  via Access List membership (App User tier).
- `access-lists/access-list-app-example.yaml` — one Access List per
  app, owned by that app's Application Admin.
- `access-lists/access-list-org-example.yaml` — one Access List per
  org, owned by that org's Delegated Admin.
- `OPEN_QUESTIONS.md` — unresolved design questions and caveats that
  should be answered before this goes to production.

## Important: how Google Group membership actually reaches Teleport

This is the part that's easy to misunderstand and causes people to keep
re-editing the connector when they don't need to, so it gets its own
section.

Google's standard OIDC ID token (the thing a user gets back after
logging in) **does not contain group membership**. It only has basic
identity claims (`sub`, `email`, `name`, `hd`, etc.). There is no
"groups" scope you can request from Google that will put group
membership into that token for a normal web OIDC login flow.

Instead, Teleport's `google` OIDC connector resolves `groups` by making
a **separate, server-side call** to the Google Admin SDK Directory API,
authenticated as a dedicated Google Cloud **service account** using
**domain-wide delegation**. This is configured in two places:

1. **The connector YAML** — `google_service_account` (the service
   account's JSON key) and `google_admin_email` (a Workspace super
   admin the service account impersonates to call the Directory API).
2. **Google Workspace Admin console** — Security → API Controls →
   Domain-wide Delegation. You authorize the service account's
   **numeric Client ID** (not its email address) for exactly this
   scope:

   ```
   https://www.googleapis.com/auth/admin.directory.group.readonly
   ```

**The key point:** that domain-wide delegation authorization in #2
lives entirely in the Google Admin console. It is **not** part of the
connector YAML, and it does not need to be touched when you:

- Add a new Google Group
- Change a `claims_to_roles` mapping
- Add/remove users from an existing group
- Add a new tier, app, or org

All of those are just connector/role/Access-List edits on the Teleport
side — Google Workspace already knows about this scope grant and keeps
serving it for every login, because it's tied to the service account's
identity, not to any specific group or role mapping.

**You only need to revisit the domain-wide delegation setting if:**

- You rotate or replace the service account key (a new/rotated service
  account still shares the same Client ID as long as you generate a
  new *key* for the *same* service account — but if you create an
  entirely new service account, it gets a new Client ID and needs to
  be re-authorized from scratch).
- You delete and recreate the service account.
- Someone revokes the delegation in the Admin console (worth an
  occasional audit, since nothing in Teleport will proactively warn you
  if this silently gets revoked — group-based logins will just start
  failing to resolve roles).

So: rotate the **key** as often as your security policy requires
(that's just a connector YAML update), but the **delegation scope
grant** itself is a one-time (or rare) Google Admin console setup, not
something to re-touch on every access-structure change.
