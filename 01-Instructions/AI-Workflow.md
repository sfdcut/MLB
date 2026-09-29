# AI Workflow

How we use AI tools on MLB work. There are three workflows, one for each kind of work:

| Workflow | Use it for | Target |
|---|---|---|
| [A: Small change or break-fix](#workflow-a-small-change-or-break-fix) | Managed-services tickets, our highest-volume work | Under two hours |
| [B: New requirement in an existing org](#workflow-b-new-requirement-in-an-existing-org) | New functionality in an org with undocumented history | Discovery first, then build in slices |
| [C: SOW project](#workflow-c-sow-project) | Full builds with discovery, solution design and technical design documents | Per project plan |

The coding rules that apply to all three are in [Coding-Standards.md](Coding-Standards.md).

> **Referenced but not yet in this repo:** the *context pack* and *project context block* (Section 2.1 of the full guide), the *review checklist* (Section 9), and the definition of *Lane 1*. Until they're added, get them from the full AI guide.

---

## Workflow A: Small change or break-fix

### Step 1: Pin down what is actually being asked

Managed-services requests come in as an email, a call or a two-line ticket. Before you touch the org, write these four things in the requirement document:

1. What the change is
2. Which object and process it affects
3. Who it affects
4. How you'll know it worked

If you can't write all four, you don't have a requirement yet. Go back to the client.

> **Watch for:** requests phrased as a solution instead of a need, like "add a checkbox to Case". Ask what the checkbox is for. Half the time there's a better way to meet the real need.

### Step 2: Gather the context pack

Gather the full context pack (Section 2.1) for every object and class you're about to touch. Don't skip this because the change is small. Small changes are where invented field names do the most damage, because nobody reviews them carefully.

> **Watch for:** pasting a snippet instead of the whole class. The tool can't reason about a method it can't see, and it will confidently guess what the missing part does.

### Step 3: Ask for impact analysis before any code

This step pays for the whole exercise. You're asking what else in the org depends on the thing you're about to change, and the answer is only as good as what you paste.

```text
<paste the project context block>

Below is the current state of the Case object in this org: the field list
with API names and types, the full source of CaseTrigger and
utsCls_CaseTriggerHandler, the list of record-triggered flows on Case with
their entry criteria, and the Case validation rules.

<paste all of the above>

I need to change how the Case owner is assigned when Status moves to
Escalated. Before writing any code, tell me:
1. Everything in what I pasted that reads or writes Case.OwnerId or
   Case.Status, named specifically.
2. For each one, whether my change would affect it, and how.
3. What order the trigger and the flows will execute in, and whether that
   ordering creates a conflict.
4. What you would need to see that I have not given you, in order to be
   confident in this analysis.

Do not write any code yet.
```

Question 4 is the one developers leave out, and they shouldn't. It shows you what's missing from your context pack before the gap turns into a defect.

> **Watch for:** automation that runs under an integration user or in a nightly batch. It won't be in what you pasted unless you went looking for it. Record every one you find in the **Landmines** section of the context block.

### Step 4: Get the approach in writing before the code

Have the tool state its intent first. Reviewing an approach takes a minute. Reviewing four hundred lines of wrong code takes an hour.

```text
Give me your approach before writing anything: which classes or components
you would change, what you would add, what you would deliberately leave
alone, and how you would test it. If there is more than one reasonable
approach, give me both with the trade-offs. Wait for me to choose.
```

> **Watch for:** "I also cleaned up the older method while I was there." Say no. One change, one review.

### Step 5: Write the change in pieces you can read

```text
Implement the approach we agreed. Follow the conventions in the project
context block exactly. Bulk-safe to 200 records, no SOQL or DML in loops,
FLS enforced. Change only what we agreed.

When you are done, give me two lists:
- every field or object API name you used that I did not explicitly provide
- every assumption you made about org behaviour that you could not verify
  from what I pasted
```

Those two lists are the most valuable output in this guide. Check every item on both before you paste anything into the org. That's how you catch hallucinated metadata yourself, in two minutes, instead of a client catching it in UAT.

> **Watch for:** the correction spiral. If three rounds haven't produced working code, stop prompting and write it yourself. You'll be faster, and you'll understand what you deployed.

### Step 6: Test it in the sandbox, properly

Ask for the test scenarios explicitly. Left alone, these tools write tests that hit coverage and assert nothing.

```text
Write the test class for this change. Cover four scenarios: the positive
path; a negative path where the criteria are not met; a bulk path with 200
records; and a user who lacks edit access on Case. Every test must assert a
specific outcome, a field value, a record count, an expected exception.
No assertion-free coverage padding. Use only the field API names I provided.
```

Save the class in the sandbox and run it there, from **Setup > Apex Test Execution** or the **Test** menu in the Developer Console. Then run the other tests on the objects you touched, not just yours. A green test class in a red org isn't a pass.

> **Watch for:** tests that pass because they mock away the logic under test, and filler assertions that are always true. Read the assertions yourself. Green doesn't mean correct.

### Step 7: Check the behaviour by hand, then get a second pair of eyes

Log into the sandbox as a user with the affected profile and walk through the actual process. Automated tests confirm the code does what you told it to. Only clicking through confirms it does what the client asked for.

Then a second developer reviews the change against the review checklist (Section 9). AI assistance doesn't replace review and doesn't shorten it.

### Step 8: Write the deployment document

Every promotion to production comes with a deployment document. This is one of the best uses of AI in our process. The document is structured and repetitive, and it's exactly the thing that gets written badly at six in the evening when the deployment window opens at seven.

Draft it in Lane 1 from your own change details. For the component list, Agentforce Vibes can list what changed in the org more reliably than your memory can. The draft is only a starting point: check every component name against the org before anyone uses the document.

```text
<paste the project context block>

We are promoting the following change from the <name> sandbox to production:
<describe the change, and paste the list of components created or modified,
the classes and their test classes, any new fields with API names, flows,
permission sets, and anything configured by hand in the sandbox>

Draft the deployment document with these sections:
1. Summary of the change and the requirement reference.
2. Component inventory: every component to be deployed, with API name and
   type, grouped by dependency order.
3. Pre-deployment steps: anything that must exist in production first, and
   any configuration that cannot be deployed and must be done by hand.
4. Deployment sequence, with the reason for the ordering.
5. Tests to run during validation, named specifically.
6. Post-deployment steps: manual configuration, activations, data updates.
7. Post-deployment verification: what to check, as which profile, with the
   expected result for each check.
8. Rollback plan: exactly how to reverse this, component by component, and
   what cannot be reversed.
9. Risk notes: what is different about production, data volume, sharing,
   automation that exists only there and what could fail because of it.

For anything you cannot determine from what I gave you, write TO BE
CONFIRMED and say what you would need. Do not guess at component names.
```

Sections 3, 7 and 8 matter most:

- **Manual configuration that nobody wrote down** is the most common reason a deployment succeeds and still leaves production broken.
- **A rollback plan written after something goes wrong** is written under pressure, by someone who's no longer thinking clearly.

> **Watch for:** a component list drafted from your description instead of from the org. Check every API name in the document against the sandbox before the deployment window, not during it.

### Step 9: Promote to production, by hand

Deployment is your job, not the tool's. Work through the deployment document you just wrote:

1. Complete the pre-deployment steps.
2. Promote through the approved path for that client.
3. Validate before you deploy, and run the named tests.
4. Work through the post-deployment and verification sections as a user with the affected profile, exactly as you did in the sandbox.

Then mark up the deployment document with what actually happened: anything that differed from the plan, and anything you had to do by hand that the document didn't anticipate. File it with the client change log. The marked-up copy is what makes the next deployment on that account faster.

> **Watch for:** changes that pass in a sandbox and fail in production because of data volume, sharing rules, or automation that only exists in production. Ask what's different about production before you deploy, not after.

---

## Workflow B: New requirement in an existing org

This is the hardest mode we work in. The org carries years of history that nobody documented, so discovery matters more than the build.

### Step 1: Turn the request into a written requirement

A client email isn't a requirement. Turn it into a requirement document entry precise enough that two developers would build the same thing. Use the tool to structure and question it, then edit the result yourself.

```text
Here are my notes from the client call: <paste>.

Turn these into a requirement write-up with these sections: business need,
scope (what is in and what is explicitly out), functional behaviour,
affected objects and processes, acceptance criteria, assumptions, and open
questions.

Be strict about the difference between what the client actually said and
what you inferred. List every inference separately under assumptions, and
give me the questions I should put back to the client before we commit to
an estimate. Write it in plain business language and this goes to the client.
```

The open-questions list is the valuable output here. Send it to the client before anyone commits hours. Assumptions nobody challenges quietly become scope you're expected to deliver for free.

### Step 2: Discover the org as it actually is

Before designing anything, find out what's already running on the objects in scope. Gather the context pack for each object, then have the tool build an inventory you can keep.

```text
<paste the project context block, then the full metadata for the objects in
scope: field lists, trigger and class source, flow names with entry criteria,
validation rules, and the relevant profiles and permission sets>

Build me an inventory table of everything currently automating these
objects: component name, type, what fires it, what it does in one line, and
whether it would conflict with the requirement described below.

Requirement: <paste from the requirement document>

Then list what you cannot assess from what I gave you, and what I would
need to provide for you to assess it.
```

Keep the inventory in the requirement document, and copy the parts relevant to conflicts into the **Landmines** section of the context block. Every developer who works on this account afterwards benefits.

### Step 3: Make the design decision yourself

Use the tool to widen your options and pressure-test your reasoning. The decision stays with you and your architect.

```text
Given the inventory above and this requirement, give me two or three
implementation options for example a record-triggered flow, new Apex, or
extending the existing handler.

For each option cover: how it interacts with the automation already on the
object, bulk behaviour, testability, who can maintain it (admin or
developer), and what it costs us later. Recommend one, and tell me what
would change your recommendation.
```

> **Watch for:** a recommendation that ignores the inventory the tool just produced. That means the session has drifted. Start a fresh session and paste everything again instead of arguing with it.

### Step 4: Write the technical approach into the design document

Once you've chosen, record the decision and the reasoning. The next developer needs to know what you built, and also what you rejected and why. That's what stops the same debate from happening again in six months.

```text
Write the technical design section for the option we chose: components to be
created or modified, field-level detail with API names, automation logic and
its order of execution, error handling, security and sharing considerations,
test approach, and the deployment considerations for moving this to
production. Note the options we rejected and why in one short paragraph.
```

### Step 5: Estimate, then correct upward

These tools estimate too low on orgs with legacy debt, because nothing in your paste shows how long it takes to untangle a ten-year-old handler. Treat the output as a checklist of work items, not a number, and add your own contingency. **No estimate goes to a client without your lead's sign-off.**

### Step 6: Build it in slices, then close the documentation loop

Break the requirement into pieces small enough to review in one sitting, and run each piece through [Workflow A](#workflow-a-small-change-or-break-fix).

When the build is done:

- Write one deployment document for the whole requirement, not one per slice.
- In the same session, update the requirement document, the configuration workbook and the client change log, while the details are still in your head and in the context window.

---

## Workflow C: SOW project

On a full build, the deliverables are the discovery document, the solution design document, the technical design document, and then the org itself.

These tools help most when writing those documents and when writing tests. They help least in integration debugging. That's where people expect the most help, and it's where they get frustrated.

### Your first day on a new build

- [ ] Read the discovery and design documents before you write anything. If they don't exist yet, writing them is your job, not the code.
- [ ] Confirm which sandbox you're working in and how changes reach production on this project.
- [ ] Find or create the project context block for this org. Keep it in shared documentation, not on your machine.
- [ ] Agree the trigger, handler and naming patterns with the team before anyone generates code.
- [ ] Build one reference class with its tests, review it as a team, and paste it into every session afterwards as the pattern to follow.

The last item is worth the hour it takes. "Follow the pattern in this class" plus the actual class gives far more consistent output across ten developers than any written description of the pattern.

### Where the tools help, by phase

| Phase | Deliverable | What the tools do well | What stays yours |
|---|---|---|---|
| Discovery | Discovery document | Structure interview notes, suggest questions you haven't thought to ask, flag contradictions between stakeholders | The client conversations and the priority calls |
| Solution design | Solution design document | Draft object models, lay out options with trade-offs, flag anti-patterns and licensing implications to check | The decision, and defending it to the client |
| Technical design | Technical design document | Turn the chosen design into component-level detail, field lists and interface specifications | Security model, sharing model, integration contracts |
| Build: code | Apex, LWC | Classes and components that follow a pasted reference class, plus their tests | Design review, code review, what actually ships |
| Build: config | Flows, metadata | Flow scaffolding and metadata generation inside the org | Record types, sharing, licensing decisions |
| Integration | Interface specs | Mapping tables, mock payloads and error-handling patterns from pasted API documentation | Auth design, endpoint contracts, credential handling |
| Test | Test classes, scripts | Test classes, regression and UAT scripts from the requirement document, synthetic data | Test strategy, sign-off, exploratory testing |
| Deployment | Deployment document | Component inventory, sequencing, pre- and post-deployment steps, verification checks, rollback plan | Every deployment action in production |
| Handover | Runbooks, admin guide | Runbooks, admin guides, configuration workbooks, training material | Client training and accountability |
