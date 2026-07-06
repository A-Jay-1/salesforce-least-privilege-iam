# Least-Privilege IAM Design: Just-In-Time Salesforce Access

**A Microsoft Entra ID project demonstrating least-privilege access control, risk-based policy design, and just-in-time (JIT) privilege elevation using Privileged Identity Management (PIM) for Groups.**

---

## Problem Statement

gazephotos, a creative/tech company, already maintained a break-glass account pattern for emergency access, but had no formal process for routine elevated access to business applications. Standing privileged access — users holding admin rights indefinitely, whether they're using them or not — is a common but avoidable risk.

This project redesigns access to Salesforce (Sales Cloud) for the Sales-Direct team around a core principle: **no one holds privileged access they aren't actively using.** Access is requested, justified, time-boxed, and — for the highest-risk role — approved by a second person before it's granted.

---

## Architecture

**Identity provider:** Microsoft Entra ID
**Target application:** Salesforce (Sales Cloud) — Entra gallery app, SAML SSO
**Elevation mechanism:** PIM for Groups (eligible → active, time-bound, auto-expiring)

```
User → requests activation → [Admin only: approval required] → time-bound
group membership → SAML role claim → Salesforce access → auto-expiry → audit log
```

*(See `/screenshots` for a full visual walkthrough of every stage below.)*

---

## Role Model

Three tiers were selected from Salesforce's own built-in app role catalog (11 available) rather than inventing custom roles — a deliberate choice to reflect how IAM design works against a real vendor's role taxonomy, not a hypothetical one.

| Role | Salesforce App Role | Access | Group | Assignment Type |
|---|---|---|---|---|
| Sales Rep | Standard User | Own accounts/opportunities only | `SG-SalesDirect-App-User` | Standing (Assigned, active) |
| Salesforce Admin | System Administrator | Full config, user management, data export | `SG-SalesDirect-App-Admin` | PIM-eligible only |
| Compliance Auditor | Read only | Read-only access to activity logs/audit trail | `SG-SalesDirect-App-Auditor` | PIM-eligible, scoped |

**Why only three of eleven available roles:** Marketing User, Chatter Moderator, Contract Manager, Solution Manager, and others were deliberately excluded as out of scope for the Sales-Direct CRM use case. Provisioning roles nobody on the team needs — even when they're free and available — is itself a least-privilege violation.

---

## Key Design Decisions

### 1. Group-based, not direct, role assignment
Users are never assigned directly to Salesforce app roles. Membership flows through dedicated security groups (`SG-SalesDirect-App-*`), which scales better and is easier to audit than per-user assignment.

### 2. Assigned membership, not Dynamic
gazephotos' existing groups (`SG-Sales-Direct`, `SG-Eng-All`, etc.) use Dynamic membership rules, which are incompatible with PIM — PIM requires direct control over who is Eligible vs. Active in a group. New, purpose-built Assigned-type groups were created specifically for this project.

### 3. Risk-differentiated PIM policies
The Admin and Auditor groups are both PIM-governed, but with deliberately different policies — not copy-pasted defaults:

| Policy Setting | Admin | Auditor | Why |
|---|---|---|---|
| Max activation duration | 1 hour | 8 hours | Admin tasks are short; shrinks blast radius. Auditor review sessions run longer. |
| Approval required | **Yes** | No | Separation of duties for the highest-risk role; read-only carries less risk. |
| MFA on activation | Yes | Yes | Both roles touch sensitive data; a stolen session alone shouldn't be enough to elevate. |
| Justification required | Yes | Yes | Creates an audit trail and discourages casual requests. |
| Eligible assignment expiry | 3 months | — | Forces periodic re-justification of *eligibility itself*, not just activation. |
| Active assignment expiry | 6 months | — | Prevents indefinite standing privilege even outside activation windows. |

### 4. Standard User stays standing
Baseline sales access is low-risk and high-frequency — forcing it through PIM would add friction without a significant security benefit. Not every role needs the same level of control; least privilege means matching control to risk, not applying maximum friction everywhere.

### 5. SAML role claim value mapping
Salesforce's gallery app roles shipped with **no `value` set** — meaning that even with correct role assignment, the token sent to Salesforce at sign-in would carry no identifiable role claim. This was fixed via a direct app registration manifest edit, setting explicit values (`SystemAdministrator`, `StandardUser`, `ReadOnly`) for each role. This is a real, non-obvious gap that a from-scratch custom app wouldn't expose — discovering and fixing it demonstrates manifest-level familiarity beyond the admin UI.

---

## End-to-End Verification

Both the Admin and Auditor flows were run in full, using dedicated test identities, to confirm the design behaves as intended — not just that it's configured correctly.

**Admin flow (Requester: Araba Forson, Approver: designated approver):**
1. Baseline confirmed: no roles assigned, no standing access
2. Eligible membership granted via PIM
3. Activation requested (1 hour, justification provided)
4. Request entered a **pending approval** state — did not activate immediately
5. Approver received both an in-portal request and an automated email notification
6. Approver approved the request
7. Membership became **Active** with a visible expiry time

**Auditor flow (Requester: Kwame Baidoo, same Approver):**
1. Eligible membership granted via PIM
2. Activation requested (8 hours, justification provided)
3. Request activated **immediately** — no approval step, matching the lighter-friction policy

**The side-by-side contrast is the real evidence here:** the same mechanism (PIM for Groups) produces different runtime behavior depending on the policy attached to each group — proof the risk-based design isn't just documented, it's enforced.

**SSO verification:** Both users attempted to launch Salesforce from My Apps post-activation. Both failed with *"App is not configured for single sign-on,"* since no live Salesforce org was connected for this lab. The failure message itself confirms the request correctly reached the Salesforce Enterprise App registration with the correct app ID — validating the app-side configuration up to the point a live SSO target would be required.

**Audit trail:** PIM's audit history (Groups) shows the full governance timeline automatically — every eligible assignment, activation, and approval event, timestamped and attributed. One non-fatal failed action appears in the log unedited; a real audit trail isn't a curated all-green report, and leaving it visible is more representative of production reality than removing it.

---

## Known Limitations & Scope Boundaries

- **No live Salesforce org / SCIM provisioning.** A full implementation would pair the SAML role claim (built here) with SCIM-based provisioning to map roles to actual Salesforce Profiles or Permission Sets. Not configured here since it requires a live Salesforce Developer org with API access.
- **Approver is the tenant Global Admin**, not a dedicated non-admin approver identity, due to a limited pool of Entra ID P2 test licenses. In production, a distinct approver role would be preferable to avoid concentrating too much authority in one account.
- **Directory-wide read visibility** (any authenticated user can browse basic user/group/app info by default) was identified but not remediated. The relevant mitigation — restricting the Entra admin center via User Settings — was evaluated and intentionally not applied, since Microsoft's own guidance notes it only blocks portal UI access, not underlying Graph API data; genuine enforcement would require a Conditional Access policy targeting Microsoft Admin Portals, which is a tenant-wide decision beyond this project's scope.
- **Licensing as a design constraint.** With only two spare P2 licenses available, this project's build process is itself a small illustration of a real-world tradeoff: **P2 costs real money per user**, and organizations have to make deliberate calls about which roles/teams get JIT-governed access versus which stay on standing access — a budget constraint as much as a risk one.

---

## Repository Structure

```
salesforce-least-privilege-iam/
├── README.md
├── docs/
│   └── project-log.md          ← full step-by-step build log with reasoning
├── screenshots/
│   ├── 01-scenario/
│   ├── 02-enterprise-app/
│   ├── 03-app-roles-manifest/
│   ├── 04-groups/
│   ├── 05-role-assignment/
│   ├── 06-pim-onboarding/
│   ├── 07-pim-policies/
│   ├── 08-activation-flows/
│   └── 09-audit-trail/
```

---

## Tech / Concepts Demonstrated

Microsoft Entra ID · Enterprise Applications & SAML SSO · App Registration Manifests · Security Groups (Assigned vs. Dynamic) · Privileged Identity Management (PIM) for Groups · Risk-based access policy design · Separation of duties · Just-in-time access · Audit logging · Microsoft Entra ID P2 licensing model
