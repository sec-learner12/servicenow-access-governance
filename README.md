# Access Governance Engine

A ServiceNow scoped application that automates employee offboarding and access revocation: from a submitted request, through approval, to the actual roles and group memberships being removed on the platform, with a full audit trail written before every deletion.

**The full technical build report is included in this repository** (`Access_Governance_Engine_Report.pdf`) covering the complete architecture, every real bug found while building it, and end-to-end proof it works.

---

## What this does

Three governance actions, each mapped to a real HR/IT scenario:

| Action | What happens |
|---|---|
| **Review** | No automated change. Displays the user's current roles and groups (marked direct vs. inherited) so a human reviewer can decide what, if anything, needs handling manually. |
| **Revoke groups** | Removes the specific group memberships a reviewer selects. Any role the user held only because of that group is cleaned up automatically by the platform's own cascade behavior. |
| **Offboard user** | Removes every group membership and every remaining directly-assigned role, then deactivates the account. |

A request is submitted through a Service Catalog Record Producer, approved by a separate ITIL/security-roled user, and once it's marked **Closed Complete**, a Flow triggers the actual revoke logic automatically, with no manual script execution anywhere in the path.

## Why it exists

This was built as a portfolio project to demonstrate real ServiceNow development: scoped application architecture, cross-scope security boundaries, GlideAjax, Flow Designer automation, and ACL design, using decisions backed by direct testing against the platform rather than assumptions about how it should behave.

## Architecture at a glance

```
Service Catalog (Record Producer)
        │
        ▼
Access Governance Request  ──approval──▶  State = Closed Complete
        │                                         │
        ▼                                         ▼
AccessGovernanceUtils                    Flow: Look Up Record
(read-only, GlideAjax,                          │
 populates security context)                    ▼
                                    Custom Action: Process Access
                                       Governance Revoke
                                                 │
                                                 ▼
                                AccessGovernanceRevokeProcessor
                                (app scope, decides what to revoke,
                                 writes audit rows BEFORE deleting)
                                                 │
                                                 ▼
                                 global.AccessGovernanceGlobalHelper
                                (Global scope, the only code that
                                 actually deletes from sys_user_has_role /
                                 sys_user_grmember)
```

**Three Script Includes, one job each:**
- `AccessGovernanceUtils`: app-scoped, Client Callable, read-only. Looks up a user's current roles and groups for the catalog form.
- `AccessGovernanceRevokeProcessor`: app-scoped, server-only. Decides what to revoke based on the governance action, writes one audit row per item *before* anything is deleted, then calls the Global helper.
- `AccessGovernanceGlobalHelper`: **Global scope**, server-only. The only code in the entire application with a privileged execution context that lets it actually delete rows from the two protected security tables.

## Key design decisions

- **Global-scope helper, not a wider app scope.** Scoped applications can't delete from `sys_user_has_role` or `sys_user_grmember` directly, confirmed by testing, not assumed. Rather than widen the app's own privileges, a narrowly-scoped Global helper acts as the only privileged delegate, keeping every other capability read-only or app-scoped.
- **Three governance actions, not four.** An earlier design included an automated "revoke individual roles" action. It was dropped after testing showed the `granted_by` field (meant to trace which group granted an inherited role) is empty on every row in the instance, making individual attribution unreliable. Group-level and full-offboard actions cover the safe, clear cases; anything more surgical is left to human review rather than guessed at automatically.
- **Audit rows are written before the delete, one per item.** Never a single bulk summary line, and never written only on confirmed success: a failed attempt still leaves a trace, which matters more for a compliance tool than a slightly optimistic log entry.
- **`revoke_name` is a String snapshot, not a Reference.** So the audit trail stays accurate even if a role or group is later renamed or deleted.
- **Choice fields are checked by internal value, not display label.** A real bug: the branching logic checked for `"Revoke groups"` instead of the field's actual stored value, `revoke_roles_groups`, and it only surfaced once real data flowed through Flow Designer, not typed test strings.

## Tech stack

ServiceNow Studio · Script Includes (class-based, cross-scope) · GlideAjax · Catalog Client Scripts & UI Policy · Flow Designer · Custom Actions (Action Designer) · ACLs · Dynamic reference qualifiers

## The full story

The technical build report included in this repository covers all of this in depth: the data model, the ACL design, the Record Producer, and a genuine, honest walkthrough of six real bugs found while wiring up the Flow Designer automation (including a reference field that displayed correctly everywhere but was secretly still an object, not a string, three layers into the call stack). It's written the way I'd explain the project in an interview: what I built, what broke, and why I made the calls I made.
