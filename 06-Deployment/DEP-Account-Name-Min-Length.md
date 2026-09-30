# Deployment: Account Name minimum length

| | |
|---|---|
| **Requirement** | [REQ-Account-Name-Min-Length](../03-Requirements/REQ-Account-Name-Min-Length.md) |
| **Source org** | MLB Sandbox (`utdev`), https://miamimarlins--utdev.sandbox.my.salesforce.com |
| **Target org** | Production. **TO BE CONFIRMED:** the production org and the approved promotion path for this client. |
| **Change set** | `Account_Name_Min_Length` (outbound, Id `0A2iV0000000Drh`). Status: **Open, not uploaded.** |
| **Prepared** | 2026-09-30 |
| **Deployed by** | TO BE CONFIRMED. A person performs every production step. |

> **Not ready to promote.** The requirement's open questions (Section 5) and Landmine L1 (the Contact-to-Person-Account conversion) are unresolved. Code review (Workflow A, Step 7) hasn't been done yet. Don't upload the change set until these are closed.

---

## 1. Summary

Adds the Account validation rule `Account_Name_Min_Length`. It blocks saving a non-Person Account whose Name has 5 characters or fewer, not counting spaces at the start or end. The error shows on the Name field: "Account Name must be more than 5 characters." Users with the `Bypass_Validation_Rules` custom permission are exempt.

## 2. Component inventory

In dependency order:

| # | API name | Type | New or changed in production | Notes |
|---|---|---|---|---|
| 1 | `Bypass_Validation_Rules` | Custom Permission | TO BE CONFIRMED. It probably already exists, because `Prevent_Email_Change_with_PV_Email_Match` uses it. | Dependency of #2 |
| 2 | `Account.Account_Name_Min_Length` | Validation Rule | New | Active |
| - | 33 profiles | Profile settings | Changed | Only settings for components 1 and 2 are deployed. See risk R1. |

Profiles in the change set: Analytics Cloud Integration User, Analytics Cloud Security User, Anypoint Integration, CPQ Integration User, Contract Manager, Einstein Agent User, End User, Executive Sponsor, External Apps Login User, External Einstein Agent User, Fan Experience, Guest License User, Identity User, Marketing Admin, Marketing User, Marlins IT Admin, Marlins Sales Leadership, Minimum Access - API Only Integrations, Minimum Access - Salesforce, President, Read Only, Sales Insights Integration User, Sales Interns, Salesforce API Only Okta Provisioning, Salesforce API Only System Integrations, plus 8 more on page 2 of the change set's profile list. **TO BE CONFIRMED:** list the 8 profiles on page 2 here before upload.

## 3. Pre-deployment steps

1. **Close the requirement.** Answer the open questions in the requirement doc. If the rule changes (for example to exclude `Transitional_to_Person_Account`), update it in the sandbox **before** uploading. A change set captures components at upload time.
2. **Check production supports the formula.** Person Accounts must be enabled in production, because the rule references `IsPersonAccount`.
3. **Check who has the bypass in production** (risk R1). In production, find every profile that grants the `Bypass_Validation_Rules` custom permission. Setup > Custom Permissions > Bypass_Validation_Rules > Manage Assignments, or run:
   ```sql
   SELECT Parent.Name, Parent.IsOwnedByProfile, Parent.Profile.Name
   FROM SetupEntityAccess
   WHERE SetupEntityId IN (SELECT Id FROM CustomPermission WHERE DeveloperName = 'Bypass_Validation_Rules')
   ```
   If any **profile** grants it, remove that profile from the change set (or remove all profiles) before upload.
4. **Find existing short names in production** (Landmine L2). Run:
   ```sql
   SELECT Id, Name, RecordType.DeveloperName FROM Account WHERE IsPersonAccount = false
   ```
   Then filter for `LEN(TRIM(Name)) <= 5`. Agree with the client whether to rename these before go-live. They can't be edited once the rule is active.
5. **Check the conversion batch** (Landmine L1). Confirm whether `ConvertContactToPersonAccount` is scheduled in production (Setup > Scheduled Jobs), and which user runs it.
6. **Confirm integrations** that create or update non-Person Accounts in production, and check that each integration user either has the bypass or always sends names longer than 5 characters.
7. **Deployment connection.** Confirm the sandbox is allowed to upload to production (production: Setup > Deployment Settings).

## 4. Deployment sequence

1. In the sandbox, **Upload** the change set `Account_Name_Min_Length` to production.
2. In production, Setup > Inbound Change Sets > `Account_Name_Min_Length` > **Validate**, using the test option in Section 5.
3. After validation passes, run **Quick Deploy** (or **Deploy**) in an agreed window.

The custom permission and the rule go in one change set, so the platform deploys them in the right order. No separate sequencing is needed.

## 5. Tests to run during validation

The change set has no Apex, so production doesn't require tests. Run these anyway to catch Apex that creates Accounts:

- **Run specified tests:** `ConvertContactToPersonAccountTest`, `UtilitiesTest`

Don't use "Run local tests" unless the `GenerateEventSpecificProductsTest` failures seen in the sandbox (unrelated to this change) are known to pass in production.

## 6. Post-deployment steps

- None required. The rule deploys active.
- If step 3.4 found records to rename, apply the agreed renames. Run them as a user with the bypass permission, or rename before deploying.

## 7. Post-deployment verification

| # | As | Action | Expected result |
|---|---|---|---|
| 1 | A user without the bypass (e.g. Marlins Sales Leadership profile) | Create a Business Account named `Test` | Blocked. "Account Name must be more than 5 characters." shows on the Name field. |
| 2 | Same user | Create a Business Account named `Test Co` | Saves. Delete the record afterwards. |
| 3 | Same user | Create or edit a Person Account whose name is 5 characters or fewer (e.g. `Al Wu`) | Saves. The rule doesn't apply to Person Accounts. |
| 4 | A user with the `Bypass_Validation_Rules` permission set | Create a Business Account named `Test` | Saves. Delete the record afterwards. |
| 5 | Admin | Setup > Object Manager > Account > Validation Rules | `Account_Name_Min_Length` is listed and Active |
| 6 | Admin | Custom Permission `Bypass_Validation_Rules` > Manage Assignments | Same assignments as before the deployment (compare with the result of step 3.3) |

## 8. Rollback plan

| Component | How to reverse | Notes |
|---|---|---|
| `Account_Name_Min_Length` | Production: Setup > Object Manager > Account > Validation Rules > Edit > untick **Active** > Save | Takes effect immediately. A change set can't delete components. If the rule must be removed completely, delete it by hand in Setup. |
| `Bypass_Validation_Rules` | Normally leave it as it is | If it was newly created by this deployment, it's harmless unused. If it already existed, it's unchanged. |
| Profile settings | Re-grant the custom permission by hand to any profile that lost it | Only relevant if step 3.3 was skipped (risk R1) |
| Renamed Accounts | Restore the old names from the list captured in step 3.4 | Keep that list until the change is signed off |

## 9. Risk notes

- **R1. Profiles can remove the bypass.** In the sandbox, no profile grants `Bypass_Validation_Rules`; it's only granted through a permission set. Deploying profiles alongside the custom permission can set that permission to *off* on those profiles in production. Any production profile that grants the bypass today would lose it, and its users would start hitting both this rule and `Prevent_Email_Change_with_PV_Email_Match`. Step 3.3 checks for this.
- **R2. Conversion batch (L1).** Contacts whose full name is 5 characters or fewer fail conversion. They're flagged as converted, so they're never retried.
- **R3. Existing records (L2).** Short-named non-Person Accounts are locked for any edit, including edits from integrations and automation.
- **R4. What's different in production.** Data volume, integration users and scheduled jobs haven't been checked against production. The sandbox has only 5 Accounts, so it doesn't show how many real records break the rule.
- **R5. Managed-package triggers.** `AccountTriggerV2` and `DialpadAccountAssignmentTrigger` weren't reviewed. If either sets or changes Account Name, it could hit this rule.

---

## Deployment record

*Fill in after the deployment: what actually happened, anything that differed from the plan, and anything done by hand that this document didn't anticipate.*

| Date | Deployed by | Validation Id | Deploy Id | Notes |
|---|---|---|---|---|
| | | | | |
