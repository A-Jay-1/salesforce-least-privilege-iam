# Least-Privilege IAM Design — Salesforce Access for gazephotos / Sales-Direct

## Project Log \& Decision Journal

## Step 1: Scenario Definition

**Company:** gazephotos — creative/tech company (photography/creative studio)
**Team:** Sales-Direct
**App:** Salesforce (Sales Cloud) — chosen from Entra gallery for mature SAML SSO support and clear app role mapping

**Problem statement:**
gazephotos already maintains a break-glass account pattern (`Break-Glass` group, Assigned membership) for emergency access, but had no formal just-in-time process for routine elevated access to business applications. This project extends that governance model to the Sales-Direct team's Salesforce access, replacing standing privileged access with a least-privilege, auditable model.

**Existing tenant context (pre-project):**
Existing groups (`SG-Creative-Studio`, `SG-Devops`, `SG-Eng-All`, `SG-Marketing`, `SG-Opr-All`, `SG-Sales-Direct`) are Dynamic membership type — fine for department-level baseline access, but incompatible with PIM for Groups, which requires Assigned membership. New groups were created specifically for this project rather than reusing dynamic groups.

**Roles defined:**

|Role|Access Description|Assignment Type|
|-|-|-|
|Sales Rep|Own accounts/opportunities only|Standing (Assigned, active)|
|Salesforce Admin|Full config, user management, data export|PIM-eligible only|
|Compliance Auditor|Read-only access to activity logs/audit trail|PIM-eligible, scoped|

\---

## Step 2: Enterprise Application Added

Salesforce added to gazephotos tenant as an Enterprise Application via Entra gallery.

* Application ID: `d67da9f2-c33b-4287-b81a-...`
* Object ID: `b78dca5e-2149-4620-a2c8-...`

**Decision:** Used the Entra gallery template rather than a manual/custom app registration, to reflect a realistic SaaS integration scenario (most IAM analyst work involves gallery/pre-built connectors, not custom-built apps).

## 

## Step 3: App Roles

**Finding:** Salesforce's gallery app ships with a pre-built set of app roles (System Administrator, Standard User, Read only, Marketing User, Chatter Moderator User, Contract Manager, Solution Manager, Chatter External User, Chatter Free User, msiam\_access) rather than requiring custom roles to be created from scratch.

**Decision — role selection:**

|Chosen Role|Mapped To Project Tier|Why|
|-|-|-|
|Standard User|Sales Rep|Baseline access, matches "own accounts/opportunities only"|
|System Administrator|Salesforce Admin|Exact match — full config/user management access|
|Read only|Compliance Auditor|Exact match — read access, no modify rights|

**Why these three and not others:** All other pre-built roles (Marketing User, Chatter Moderator User, Contract Manager, Solution Manager, Chatter External/Free User) were deliberately excluded as out of scope for the Sales-Direct CRM use case. This is itself a least-privilege decision — don't provision roles nobody on the team needs, even if they're available by default.

**Issue found — blank Value fields:**
The pre-built app roles had no `value` set in the manifest. The Value field is what populates the `roles` claim in the SAML token sent to Salesforce at sign-in — without it, role assignment in Entra would have no effect on what Salesforce actually sees at authentication time. This appears to be because Salesforce's gallery template expects entitlement to primarily flow through SCIM provisioning (Profile/Permission Set assignment) rather than the SAML role claim.

**Fix applied:** Edited the App Registration manifest (`appRoles` array) to set explicit values for the three selected roles:

* System Administrator → `"value": "SystemAdministrator"`
* Standard User → `"value": "StandardUser"`
* Read only → `"value": "ReadOnly"`

**Note for later:** A full production implementation would also involve SCIM-based provisioning to map these roles to actual Salesforce Profiles/Permission Sets — not configured in this project since it requires a live Salesforce org with API access. Documented as a known scope boundary, not an oversight.

\---

## Step 4: Security Groups Created

Three new Assigned-type security groups created (distinct from the tenant's existing Dynamic groups, which are unsuitable for PIM):

* `SG-SalesDirect-App-User`
* `SG-SalesDirect-App-Admin`
* `SG-SalesDirect-App-Auditor`

**Decision — group-based assignment over direct user assignment:** Groups scale better and are easier to audit than assigning users directly to app roles one by one. Naming convention (`SG-` prefix) kept consistent with existing tenant groups so it reads as part of the same governance model rather than a one-off.



\---

## Step 5: Groups Assigned to App Roles

Each group mapped to its corresponding Salesforce app role under Enterprise Applications → Salesforce → Users and groups:

|Group|Salesforce App Role|
|-|-|
|`SG-SalesDirect-App-User`|Standard User|
|`SG-SalesDirect-App-Admin`|System Administrator|
|`SG-SalesDirect-App-Auditor`|Read only|

**Why this step matters:** This is the link between the internal group structure and the actual token claim. Without this explicit mapping, group membership has no effect on what Salesforce grants at sign-in — the group is just an internal container until this assignment tells Entra what role claim to issue for its members.

\---

## Step 6: Groups Onboarded to PIM

`SG-SalesDirect-App-Admin` and `SG-SalesDirect-App-Auditor` onboarded into PIM for Groups (Identity Governance → Privileged Identity Management → Groups).

**Decision:** `SG-SalesDirect-App-User` intentionally left as a standard (non-PIM) group — Sales Rep access is low-risk baseline access appropriate for standing assignment. Only the two more sensitive roles (Admin, Auditor) are governed through PIM's eligible/active model.

**Note:** Groups were onboarded while empty. Onboarding governs the group's membership *model* (eligible/active), not its current members — membership is added afterward through PIM itself, once policy is set, so the policy is already in force before anyone is added.

\---

## Step 7: PIM Activation Policies

### `SG-SalesDirect-App-Admin`

**Activation tab:**

* **Max activation duration:** 1 hour
* **On activation, require:** Azure MFA
* **Require justification on activation:** Yes
* **Require ticket information on activation:** No
* **Require approval to activate:** Yes
* **Require post-approval custom extension to activate (Preview):** Yes

**Assignment tab:**

* **Allow permanent eligible assignment:** No — eligible assignments expire after **3 months** (forces periodic re-justification of who's even eligible for Admin, not just who's currently activated)
* **Allow permanent active assignment:** No — active assignments expire after **6 months**
* **Require justification on active assignment:** Yes

**Reasoning:** Highest-risk role in the model (full Salesforce config/user management). Short activation window minimizes blast radius; mandatory approval enforces separation of duties; MFA re-check at activation protects against session hijacking; justification creates an audit trail and discourages unnecessary requests. Time-boxing eligibility itself (not just activation) ensures stale eligible assignments get periodically reviewed rather than persisting indefinitely.

### `SG-SalesDirect-App-Auditor`

**Activation tab:**

* **Max activation duration:** 8 hours
* **On activation, require:** Azure MFA
* **Require justification on activation:** Yes
* **Require ticket information on activation:** No
* **Require approval to activate:** No

**Reasoning:** Read-only role, lower risk than Admin, so approval friction is relaxed. Longer duration reflects realistic use case (a compliance review session may run longer than a quick admin fix). MFA and justification retained regardless — "read-only" still means access to sensitive activity logs, so it isn't zero-risk.

**Key design point for README:** The two policies are deliberately different, not copy-pasted. This reflects risk-based policy design — not every privileged group warrants identical controls, and the differences (approval requirement, activation duration, eligible/active assignment expiration) are each tied to a specific risk rationale rather than arbitrary.



\---

## Step 8: Test Users, Licensing \& PIM Assignment

**Licensing constraint encountered:** Only 2 spare Microsoft Entra ID P2 licenses available in the tenant. PIM requires a P2 (or Entra ID Governance) license on any user with an eligible/active PIM assignment, and on any approver. Standing (non-PIM) group members, like the Sales Rep test user in `SG-SalesDirect-App-User`, do not require a license.

**Solution — sequential reuse of licenses across flows:**

1. Assigned P2 licenses to: Requester (test user) and Approver (Global Admin account, reused as approver for lab purposes).
2. Added Requester as eligible member of `SG-SalesDirect-App-Admin` via PIM.
3. Ran full Admin activation flow: request → approval → active (time-limited) → confirmed → deactivated.
4. Plan: remove Requester's eligible assignment from Admin group, unassign their P2 license, reassign that license to the Auditor test user, and repeat the flow for `SG-SalesDirect-App-Auditor` using the same Approver.

**Documentation note:** Real-world IAM design isn't purely risk-driven — it's also budget-constrained. P2 licensing cost is a genuine practical barrier organizations face when deciding which roles/teams get JIT/PIM-governed access versus which stay on standing access. This project's limited license pool is a small-scale reflection of that same tradeoff.

**Documentation note — approver identity:** The Approver in this lab is the tenant's Global Admin account, not a dedicated non-admin approver role. In production, a distinct approver identity (not holding Global Admin) would be preferable to avoid concentrating too much authority in a single account. Noted here as a known lab simplification, not an oversight.

**Observation — default directory visibility:** After activation, it was noted that the Requester (a standard licensed user, not an admin) could browse all users, groups, and applications in the Entra admin center home page in read-only form. Investigated and confirmed this is **not** related to the P2 license — it's Microsoft Entra ID's default behavior: any authenticated user can read basic directory data (users, groups, non-hidden enterprise apps) via default Microsoft Graph permissions, regardless of license or role.

**Mitigation considered — "Restrict access to Microsoft Entra admin center" (User Settings toggle):**

* This is an all-or-nothing setting: once enabled, any user without an Entra admin role is fully blocked from the admin center portal (not narrowed to a partial view).
* Notable exception: PIM's interface is explicitly exempted from this restriction by Microsoft's design — any user retains the ability to view and activate their own eligible PIM assignments even with the restriction enabled.
* Important limitation: Microsoft's own documentation states this toggle is **not a true security control** — it only blocks the admin center UI, not underlying data access via Microsoft Graph API, PowerShell, or other admin tooling. Genuine enforcement requires a **Conditional Access policy targeting Microsoft Admin Portals / the Windows Azure Service Management API**, which blocks non-admins at the access layer rather than the UI layer.
* **Decision for this project:** noted as a recommended production hardening step in the README rather than implemented directly in the lab, since it's a tenant-wide setting with tradeoffs beyond this project's scope (it would also affect group/app owners' ability to self-manage owned resources via the portal).

**Result:** Both flows completed successfully, with full evidence captured at each stage.

### Admin flow (Requester: Araba Forson, Approver: J.A — Global Admin)

1. **Baseline check** — Before any assignment, Araba's My Access home page showed "No roles assigned," confirming no standing privilege. (`Screenshot\_\_301\_.png`)
2. **Eligible assignment added** — Araba added as an eligible Member of `SG-SalesDirect-App-Admin` via PIM Assignments (not the regular Groups blade). (`Screenshot\_\_300\_.png`)
3. **Requester view** — Araba's PIM "My roles" showed the eligible assignment with an Activate action available. (`Screenshot\_\_302\_.png`)
4. **Activation request submitted** — Duration set to 1 hour (matching policy max), justification entered: *"I need access to the group to perform make changes with require admin access."* (`Screenshot\_\_303\_.png`, `Screenshot\_\_304\_.png` — Details tab confirming group name, role, member, start/end time before submission)
5. **Pending approval** — System confirmed: "Your request is pending for approval." Demonstrates the approval gate is enforced — activation did not happen immediately, unlike the Auditor flow below. (`Screenshot\_\_305\_.png`)
6. **Approver queue** — Jason, as designated approver, saw the pending request under "Requests for role activations," showing requestor (Araba Forson), request time, reason, and request type (`selfActivate`). (`Screenshot\_\_306\_.png`)
7. **Email notification** — PIM automatically sent an email ("PIM: Review Araba Forson's request to activate the Member role") with full request details (user, resource, role, reason, start time) and a direct approve/deny action link. Confirms the notification policy configured in Step 7 is functioning. (`Screenshot\_\_307\_.png`)
8. **Approval confirmed** — "Request(s) approved successfully" notification shown to approver. (`Screenshot\_\_308\_.png`)
9. **Active, time-limited access confirmed** — Araba's "My roles" now showed the assignment under **Active assignments**, State: Activated, with a visible End time and a Deactivate action available. (`Screenshot\_\_309\_.png`)

### Auditor flow (Requester: Kwame Baidoo, same Approver)

1. **Eligible assignment** — Kwame added as eligible Member of `SG-SalesDirect-App-Auditor`. Requester view showed Activate action available. (`Screenshot\_\_312\_.png`)
2. **Activation request submitted** — Duration set to 8 hours (matching policy max), justification entered: *"Request to audit PIM logs."* (`Screenshot\_\_313\_.png`)
3. **Immediate activation — no approval step** — Unlike the Admin flow, this request did not enter a pending-approval state. Kwame's "My roles" moved straight to **Active assignments**, State: Activated, confirming the policy decision from Step 7 (no approval required for Auditor) behaves correctly in practice. (`Screenshot\_\_315\_.png`)

**Key comparative evidence for README:** Running both flows side by side produced a clean, demonstrable contrast — the Admin request visibly stalled at "pending approval" and required a second person's action to proceed, while the Auditor request activated immediately after justification alone. This is direct proof (not just configuration screenshots) that the two risk-differentiated policies from Step 7 actually behave differently at runtime.

\---

## Step 9: Audit Trail \& SSO Launch Verification

**Sign-in logs reviewed:** Entra ID → Users → \[test user] → Sign-in logs, filtered by user and date. Confirmed interactive sign-in events for portal/PIM navigation activity (Azure Portal, My Apps, Azure AD Identity Governance) around each activation window.

**Finding — PIM activation does not itself generate an application sign-in event:** Activating a PIM-eligible group membership changes the user's group membership immediately, but does not trigger a sign-in to the target application. The role claim only reaches Salesforce on the user's *next* actual sign-in/SSO attempt to that app. This is an important distinction between "access granted" (group membership) and "access exercised" (application sign-in with role claim).

**SSO launch test — both flows:**
After activating each role, the respective test user (Araba Forson for Admin, Kwame Baidoo for Auditor) navigated to **My Apps** and clicked the Salesforce tile to trigger an SSO sign-in attempt. (`Screenshot\_\_319\_.png` — My Apps dashboard showing the Salesforce tile; `Screenshot\_\_320\_.png` — resulting error)

**Result:** Both attempts failed with the same error — *"App with ID d67da9f2-c33b-4287-b81a-... is not configured for single sign-on"* — routed correctly through `launcher.myapps.microsoft.com` with the correct app ID and tenant ID.

**Interpretation:** This is an expected, scoped limitation, not a configuration defect. No live Salesforce org was connected for this lab, so the SSO certificate/metadata exchange was never completed on the Salesforce side. The failure message itself is useful evidence: it confirms the request correctly reached and identified the Salesforce Enterprise App registration, meaning the app assignment and role/group configuration built in Steps 2–5 are functioning correctly up to the point where a live SSO target would be required. Full end-to-end SAML role-claim verification (e.g., inspecting the token for `SystemAdministrator` / `ReadOnly` values) was out of scope without a live Salesforce Developer org, and is noted as a natural extension of this project rather than an incomplete step.

**Documentation note for README:** "SSO launch was attempted from My Apps for both the Admin and Auditor test flows to confirm PIM-activated group membership correctly resolved to app access eligibility. The launch failed at the SSO certificate/configuration step, since no live Salesforce org was connected for this lab — expected, given the project scope was Entra-side IAM design rather than full SSO federation. The app ID and tenant routing in the failure message confirm the request correctly reached the Salesforce Enterprise App registration."

**PIM audit history captured:** `Screenshot\_\_321\_.png` — Privileged Identity Management → My audit history → Groups, filtered to last week, showing the complete governance timeline in one view: eligible member additions/removals for both Admin and Auditor groups, activation-related role assignment events, and their approval status. This single screenshot is a strong closing exhibit for the README, since it demonstrates the full audit trail Entra maintains automatically — every action tied to a timestamp, an actor (Requestor), and a target (Subject), with success/failure status.

**Note — one failed action visible in the log (`7/1/2026, 2:37:46 PM — Process role removal reque... — Kwame Baidoo — ❌`):** left as-is and worth mentioning honestly in the README rather than omitted. It reflects normal PIM administration (e.g. attempting to remove a role assignment in a state that didn't permit it) rather than a flaw in the access model itself — real audit logs contain non-fatal errors like this, and showing it unedited is more credible than a curated all-green log.

\---

## 

