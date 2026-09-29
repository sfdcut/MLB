# Requirement: Account Name minimum length

| | |
|---|---|
| **Workflow** | A: Small change or break-fix |
| **Status** | Built in sandbox, not approved. Waiting on the open questions below. |
| **Source** | Verbal/chat request, 2026-09-29. No ticket number yet. |
| **Org** | MLB Sandbox (`utdev`), https://miamimarlins--utdev.sandbox.my.salesforce.com |
| **Component** | Validation rule `Account.Account_Name_Min_Length` (active in sandbox, Id `03diV0000000IszQAE`) |

> **Process deviation:** the validation rule was deployed to the sandbox before this requirement was written or the approach agreed (Workflow A, Steps 1 to 4 were skipped). This document records the requirement after the fact. The rule stays active in the sandbox while the open questions are answered.

---

## 1. The requirement (Workflow A, Step 1)

| Item | Detail |
|---|---|
| **What the change is** | An Account can't be saved if its Name has 5 characters or fewer. The user sees an error on the Account Name field. |
| **Object and process** | Account, on create and update. Which business process this protects is **TO BE CONFIRMED**. |
| **Who it affects** | Everyone who creates or edits a non-Person Account, including integrations and batch jobs, unless they have the `Bypass_Validation_Rules` custom permission. |
| **How we'll know it worked** | See the acceptance criteria below. |

### Original request

> "account name must be more than 5 letters, otherwise throw a error"

### Acceptance criteria

1. Saving a non-Person Account named with 5 or fewer characters fails with the error "Account Name must be more than 5 characters." on the Account Name field.
2. Saving a non-Person Account named with 6 or more characters succeeds.
3. Spaces at the start or end don't count toward the length.
4. Person Accounts are not affected. *(Assumption A1, not confirmed.)*
5. Users with the `Bypass_Validation_Rules` custom permission are not affected. *(Assumption A2, not confirmed.)*

## 2. What was built

```
AND(
    NOT($Permission.Bypass_Validation_Rules),
    NOT(IsPersonAccount),
    LEN(TRIM(Name)) <= 5
)
```

- **Error message:** "Account Name must be more than 5 characters."
- **Error location:** Account Name field

## 3. Assumptions (not stated in the request)

| # | Assumption | Why it was made |
|---|---|---|
| A1 | Person Accounts are excluded | Their Name is the person's first and last name, so real fans ("Al Wu") would be blocked from saving. |
| A2 | `Bypass_Validation_Rules` users are exempt | It matches the existing rule `Prevent_Email_Change_with_PV_Email_Match`. |
| A3 | "Letters" means characters | The formula counts every character, including digits, punctuation and spaces in the middle. "AB 12" is 5 characters and is blocked. |
| A4 | The rule applies to all non-Person record types | The request didn't name any. |

---

## 4. Impact analysis (Workflow A, Step 3)

Carried out against the sandbox on 2026-09-29.

### 4.1 Inventory of what runs on Account

| Component | Type | What fires it | Reads or writes Account.Name? | Affected by this change? |
|---|---|---|---|---|
| `ConvertContactToPersonAccount` | Apex batch and schedulable | Batch run or schedule | **Writes it.** Creates one transitional Account per Contact with `Name = FirstName + ' ' + LastName` | **Yes. See Landmine L1.** |
| `AccountTriggerV2` | Apex trigger (active) | Account DML | Unknown. Its source wasn't in the unmanaged-code scan (probably a managed package) | **TO BE CONFIRMED** |
| `DialpadAccountAssignmentTrigger` | Apex trigger (active) | Account DML | Unknown. Its source wasn't in the unmanaged-code scan (probably the Dialpad package) | **TO BE CONFIRMED** |
| `Account_Conversation_Status_Do_Not_Email_Do_Not_Contact_True` | Record-triggered flow, before save | Create and update | Not reviewed | **TO BE CONFIRMED.** It runs before validation, so a record it saves is also checked by this rule. |
| `Prevent_duplicate_Alternate_Email` | Validation rule | Save | No | No |
| `Prevent_Email_Change_with_PV_Email_Match` | Validation rule | Save | No | No |
| `Prevent_Mobile_Creation` | Validation rule | Create | No | No |
| `Sales_Rep_Owner_Change_Last_Activity_30` | Validation rule | Owner change | No | No |

### 4.2 Record types affected

The rule applies to every non-Person record type: `Business_Account`, `Transitional_to_Person_Account`, `Vendor`, `Sponsorship`, `Internal` and `Sales_Vendor`. `PersonAccount` is excluded.

### 4.3 Users and integrations

| User | Has the bypass? | Notes |
|---|---|---|
| GCP_Integration (Salesforce API Only System Integrations) | Yes, via permission set | Exempt |
| Christopher Craven (System Administrator) | Yes, via permission set | Exempt |
| Jonathan Riseberg (Marketing Admin) | Yes, via permission set | Exempt |
| Platform Integration User, Automated Process, Integration User, Insights Integration, SalesforceIQ Integration, Data.com Clean | No | **TO BE CONFIRMED** whether any of these create or update non-Person Accounts |
| Everyone else, including other System Administrators | No | The rule applies to them |

### 4.4 Existing data (sandbox)

- 4 non-Person Accounts and 1 Person Account.
- 1 existing record breaks the rule: **"Miami"** (`001iV000000g5DGQAY`, record type `Sales_Vendor`). It can't be edited until it's renamed.
- Production volume, and how many production records have short names, is **TO BE CONFIRMED** before promotion.

### 4.5 Apex tests

- Ran all local tests (`RunLocalTests`): 17 ran, 13 passed, 4 failed.
- The 4 failures are all in `GenerateEventSpecificProductsTest`. They fail in `GenerateEventSpecificProducts.generateAssetsForEvent` with "There are no active Stadium Assets". That code doesn't touch Account, so these failures are **not caused by this change**. They should be logged as a separate issue.
- Test data that creates Accounts uses names longer than 5 characters (`'Test Account'` in `UtilitiesTest` and `ConvertContactToPersonAccountTest`), so it isn't affected. `TestUtilities.createAccount(name)` has no callers in the org's own code today. Any future caller passing a short name will fail.
- No test class for this rule exists yet (Coding Standards, Section 5).

### 4.6 Landmines

**L1. The Contact-to-Person-Account conversion will fail, without warning, for short names.**
`ConvertContactToPersonAccount` inserts a transitional Account (record type `Transitional_to_Person_Account`, `IsPersonAccount = false`) named after the Contact's full name. The `NOT(IsPersonAccount)` exclusion doesn't apply at that point, so any Contact whose full name is 5 characters or fewer ("Al Wu", "Jo Li") fails the insert.

Because the insert uses `allOrNone = false`, nothing stops. The code then sets `Has_Conversion_Error__c = true` and **also `Converted_to_Person_Account__c = true`** on the failed Contact. The batch only picks up Contacts where `Converted_to_Person_Account__c = FALSE`, so those Contacts are **never retried**. They are left unconverted and flagged as converted.

- Whether this batch is scheduled or still in use is **TO BE CONFIRMED**.
- Possible fixes (to be agreed, not built): exclude `Transitional_to_Person_Account` from the rule, or make sure the running user has the bypass permission.

**L2. Existing short-named Accounts are locked for edits.**
Any existing non-Person Account with a name of 5 characters or fewer can't be saved for any reason (owner change, address update and so on) until it's renamed. That includes updates from integrations and automation.

### 4.7 Other findings (separate changes, not in scope)

- `Prevent_duplicate_Alternate_Email` compares `RecordType.Name <> "Business_Account"`. The record type's Name (label) is "Business Account", with a space, so that check is always true and the rule also runs on Business Accounts. It probably should use `RecordType.DeveloperName`.
- 4 failing tests in `GenerateEventSpecificProductsTest` (see 4.5).

---

## 5. Open questions for the requester

1. Which business process or data-quality problem is this for? Is there a ticket number?
2. Should Person Accounts be excluded? (A1)
3. Should `Bypass_Validation_Rules` users be exempt? (A2)
4. Does "letters" mean any character, or only alphabetic letters? (A3)
5. Should the `Transitional_to_Person_Account` record type be excluded, so the conversion batch isn't broken? (L1)
6. Should any other record types be excluded, such as `Vendor` or `Internal`? (A4)
7. What should happen to existing short-named Accounts, like "Miami"? Should they be renamed before go-live? (L2)

## 6. Next steps (Workflow A)

- [ ] Get answers to the open questions (Step 1)
- [ ] Confirm the TO BE CONFIRMED items: the managed triggers, the flow, the integration users and the batch schedule (Step 3)
- [ ] Agree the final approach and update the rule to match (Step 4)
- [ ] Add the rule metadata to `force-app/` and a test class covering the positive, negative, bulk and no-access/bypass paths (Steps 5 and 6)
- [ ] Walk through it in the sandbox as an affected profile, then get a second developer's review (Step 7)
- [ ] Write the deployment document in `06-Deployment/` (Step 8)
