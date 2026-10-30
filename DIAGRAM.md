# How roles and Access Lists work together

Two different mechanisms decide who ends up holding a given Teleport
role in this design. This diagram traces both paths end to end. See
`README.md` for the prose version of this table.

```mermaid
flowchart LR
    subgraph SSO["Centralized: Google Workspace via OIDC Connector"]
        direction TB
        G1["Group:<br/>teleport-super-admins@acmecorp.io"]
        G2["Group:<br/>teleport-auditor-admins@acmecorp.io"]
        G3["Group:<br/>teleport-integration-admins@acmecorp.io"]
    end

    subgraph AL["Delegated: Access Lists (self-service)"]
        direction TB
        ALO["Access List: org-example-team-access<br/>Owner = Org Delegated Admin"]
        ALA["Access List: app-example-app-access<br/>Owner = Application Admin<br/>Members = App Users"]
    end

    subgraph Roles["Teleport Roles (the actual ACL rules)"]
        direction TB
        R1["editor (built-in)"]
        R2["auditor (built-in)"]
        R3["role-integration-admin"]
        R4["role-org-delegated-admin"]
        R5["role-application-admin"]
        R6["role-app-readonly"]
    end

    G1 -->|claims_to_roles| R1
    G2 -->|claims_to_roles| R2
    G3 -->|claims_to_roles| R3
    ALO -->|grants.roles| R4
    ALA -->|grants.roles, as owner| R5
    ALA -->|grants.roles, as member| R6
```

## What each piece actually is

```mermaid
flowchart TD
    A["Connector (claims_to_roles)<br/>Automatic. No human review.<br/>Edited only by Super Admins."]
    B["Access List<br/>Owner manages Members.<br/>Membership itself IS the grant.<br/>Supports periodic audit/recertification."]
    C["Role<br/>The actual allow/deny rules.<br/>Doesn't know or care how a user got it —<br/>connector and Access List both just attach one."]

    A --> C
    B --> C
```

The role is the same kind of object either way — `editor`,
`role-application-admin`, etc. don't change behavior based on how
someone acquired them. The connector and Access Lists are just two
different **front doors** to the same back room. Centralized tiers use
the connector because membership changes rarely and should stay under
tight, central control. Delegated tiers use Access Lists because
membership changes often and an org/app owner — not a Super Admin —
should be the one managing it.

## The one thing this diagram doesn't show

Editing the underlying `app` resource definitions (not role
assignment, but the app's own URI/labels/etc.) uses neither mechanism
— that's the GitOps + CODEOWNERS pattern in `gitops/README.md`,
because neither roles nor Access Lists can scope a `create`/`update`
verb to a single resource instance. See `OPEN_QUESTIONS.md` #5.
