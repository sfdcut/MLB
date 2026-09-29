# Coding Standards

The rules every change on MLB must meet, whether you wrote the code or an AI tool did. The step-by-step process these rules come from is in [AI-Workflow.md](AI-Workflow.md).

In short: **one agreed change, bulk-safe, secure, tested with real assertions, verified against the org, and reviewed by a second developer.**

---

## 1. Scope of a change

- **One change, one review.** Change only what was agreed in the approach. No clean-up of nearby code "while you're there". If something else needs fixing, raise it as its own change.
- **Get the approach agreed before the code.** Before any code is written, write down which components change, what gets added, what's deliberately left alone, and how it will be tested.
- **Keep changes reviewable.** Break larger work into slices that can each be reviewed in one sitting.

## 2. Apex

- **Bulk-safe to 200 records.** Every trigger, handler and service method must handle a full batch of 200 records.
- **No SOQL or DML inside loops.** Query once, collect the changes, then write once.
- **Enforce security.** Check field-level security (FLS) and object-level access in Apex and custom UI components. Don't assume elevated access. See [Technical-Architecture.md](../02-Knowledge-Base/Technical-Architecture.md#4-authorization-model).
- **One trigger per object, with the logic in a handler.** For example, `CaseTrigger` delegates to `utsCls_CaseTriggerHandler`. Extend the existing handler rather than adding a second trigger.
- **Follow the agreed patterns.** The team agrees the trigger, handler and naming patterns before anyone generates code. After that, one reviewed reference class, with its tests, is *the* pattern to copy.

> **To be confirmed:** the `utsCls_` prefix above is taken from an example in the AI guide. Confirm the full naming convention for classes, triggers, test classes, flows and fields with the team and record it here.

## 3. Metadata and API names

- **Use only API names that exist in the org.** Never invent a field, object or class name.
- **Check what you didn't supply.** When an AI tool writes code, get it to list:
  1. every field or object API name it used that you didn't explicitly provide
  2. every assumption it made about org behaviour that it couldn't verify

  Check every item on both lists against the org before any code goes in.
- **Give the tool whole classes, not snippets.** An AI tool can't reason about a method it can't see.

## 4. Automation and order of execution

- **Know what already runs on the object.** Before changing an object, inventory its triggers, record-triggered flows, validation rules and anything that reads or writes the fields you're touching.
- **Check the order of execution.** Confirm the order your trigger and the existing flows run in, and that the order doesn't cause a conflict.
- **Look for hidden automation.** Automation that runs under an integration user or in a nightly batch won't be in the metadata you gathered unless you look for it. Record every one in the **Landmines** section of the project context block.
- **Choose flow or Apex deliberately.** When comparing a record-triggered flow, new Apex, or extending the existing handler, weigh: how it interacts with existing automation, bulk behaviour, testability, who can maintain it (admin or developer), and the long-term cost.

## 5. Tests

Every change ships with a test class that covers four scenarios:

| Scenario | What it proves |
|---|---|
| Positive path | The change does what was asked when the criteria are met |
| Negative path | Nothing happens when the criteria aren't met |
| Bulk path | It works correctly with 200 records |
| No-access user | It behaves correctly for a user without edit access on the object |

Rules:

- **Every test asserts a specific outcome:** a field value, a record count or an expected exception.
- **No coverage padding.** No tests without assertions, and no filler assertions that are always true.
- **Don't mock away the logic under test.**
- **Use only verified field API names** in test data.
- **Run tests in the sandbox,** from **Setup > Apex Test Execution** or the Developer Console **Test** menu. Then run the other tests on every object you touched, not just your own. A green test class in a red org isn't a pass.
- **Read the assertions yourself.** Green doesn't mean correct.

## 6. Security and secrets

Full details are in [Technical-Architecture.md](../02-Knowledge-Base/Technical-Architecture.md). The rules that apply to code:

- No credentials, tokens or secrets in Apex source, static resources or metadata files.
- Use named credentials and external credentials for outbound integrations.
- Custom metadata and configuration values are for non-secret settings only.
- Never write tokens, secrets or sensitive payloads to logs, debug statements or exception messages.

## 7. Before it ships

A change is ready for production only when all of these are true:

- [ ] Every API name and assumption has been checked against the org
- [ ] The test class covers all four scenarios with real assertions, and the other tests on the touched objects pass
- [ ] You've walked through the process in the sandbox as a user with the affected profile
- [ ] A second developer has reviewed it against the review checklist (AI assistance doesn't replace or shorten review)
- [ ] There's a deployment document that includes manual steps, post-deployment checks and a rollback plan
- [ ] You've asked what's different about production: data volume, sharing rules, automation that only exists there

## 8. Working with AI tools

- **The developer owns the code.** You're responsible for everything that ships, whoever or whatever wrote it.
- **Stop the correction spiral.** If three rounds of prompting haven't produced working code, write it yourself.
- **Start fresh when a session drifts.** If the tool's output ignores context it was just given, start a new session and paste everything again instead of arguing with it.
- **Deployment is always done by hand.** The tool can draft the deployment document, but a person performs every production deployment action.
