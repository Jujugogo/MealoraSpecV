# Specification Verification Protocol

## Core Rule

Before implementing any feature, fixing any bug, changing architecture, or modifying UI behavior, the agent must identify the related specification document and verify that it is clear, consistent, and approved.

The agent must not treat an unclear specification as permission to guess.

## Spec Status Folders

Project specifications must be separated by status:

```txt
/specs/
  draft/
  checked-and-approved/
```

Meaning:

* `draft/` - unfinished, unverified, unclear, or not yet approved specifications.
* `checked-and-approved/` - specifications that were checked, clarified, approved by the project owner, and are safe to implement.

Only specs inside `checked-and-approved/` can be used as the final source of truth for implementation.

Any spec with contradictions, gray zones, missing decisions, or unclear requirements must stay in `draft/` until the project owner approves it.

## Gray Zone and Conflict Check

Before working from a spec, the agent must check it for:

1. Contradictions
   Example: one section says checkout requires login, another section says checkout works without accounts.

2. Gray zones
   Example: pricing logic is mentioned, but the exact rule is not defined.

3. Missing decisions
   Example: the UI says "send order", but the spec does not say whether the order goes to Supabase, Telegram, email, or WhatsApp.

4. Undefined edge cases
   Example: what happens if the cart is empty, payment fails, product is out of stock, or the user refreshes the page.

5. Unclear ownership
   Example: it is unclear whether the feature belongs to frontend, backend, Supabase, admin panel, or external service.

6. Conflicts with other specs
   Example: FeatureSpec says one behavior, but GlobalSpec or TechnicalSpec says another.

7. Conflicts with existing project rules
   Example: the implementation would break the selected stack, folder structure, naming conventions, or security rules.

## Required Agent Behavior

If the spec is clear and approved:

* Continue with implementation.
* Mention which approved spec is being used.
* Follow the spec exactly unless the user explicitly changes it.

If the spec is unclear, incomplete, or not approved:

* Do not implement the feature yet.
* Keep the spec inside `draft/`.
* Report the exact unclear points.
* Suggest concrete decisions that need to be made.
* Ask for approval or clarification before coding.

If the user explicitly asks to continue despite an unclear spec:

* The agent must clearly mark all assumptions.
* The spec must remain in `draft/`.
* The agent must not commit or push the work until the spec is approved and moved to `checked-and-approved/`.

## Approval Ownership Rule

The agent may suggest moving a spec to `checked-and-approved/`, but only the project owner can approve this status change.

The agent must not silently approve a spec by itself.

Correct behavior:

```txt
I checked the spec and found no blocking issues.
Recommendation: this spec can be moved to checked-and-approved after project owner approval.
```

## Commit and Push Gate

Before every commit or push, the agent must perform a Spec Approval Gate.

The agent must verify:

1. Every changed feature, page, API route, database schema, UI rule, or workflow is covered by a related spec.
2. The related spec is located in:

```txt
/specs/checked-and-approved/
```

3. The implementation does not contradict the approved spec.
4. Any new decisions discovered during implementation were added back to the spec.
5. No temporary assumptions remain undocumented.
6. No feature was implemented from a `draft/` spec.

## Commit Blocking Rule

The agent must not commit or push if:

* The related spec is missing.
* The related spec is still in `draft/`.
* The implementation contains assumptions not written in the approved spec.
* The code behavior differs from the approved spec.
* The spec has unresolved contradictions or gray zones.
* The project owner has not approved the spec.

In this case, the agent must stop and report:

```txt
Commit blocked: related spec is not checked and approved.

Reason:
- [list exact reason]

Required action:
- [what must be clarified before commit]
- [which spec must be approved and moved to checked-and-approved]
```

## Pre-Commit Checklist

Before committing, the agent must answer:

```txt
Pre-Commit Spec Check:

1. Related spec:
   - Path:

2. Spec status:
   - draft / checked-and-approved

3. Is the spec approved by the project owner?
   - yes / no

4. Did implementation follow the spec exactly?
   - yes / no

5. Were new decisions added back into the spec?
   - yes / no / not needed

6. Are there unresolved gray zones?
   - yes / no

7. Commit allowed?
   - yes / no
```

Commit is allowed only if the final answer is:

```txt
Commit allowed: yes
```

## Source of Truth Priority

When there is a conflict between documents, use this priority order:

1. `GlobalSpec`
2. `FunctionalMap`
3. `FeatureSpecs`
4. `TechnicalSpecs`
5. `VisualRules`
6. `UserStories`
7. `WorkPlans`

However, if a lower-level spec contains more detailed behavior and does not contradict the higher-level spec, the lower-level spec may be used for implementation details.

If two specs conflict, the agent must stop and ask for clarification before implementation or commit.

The conflicting spec must remain in `draft/` until the conflict is resolved and the project owner approves it.

## Required Report Format When Problems Are Found

When the agent finds problems in a spec, it must report them like this:

```txt
Spec Verification Result: blocked

Spec:
- [path/name]

Current status:
- draft

Problems found:

1. Conflict:
   - Section:
   - Problem:
   - Suggested fix:

2. Gray zone:
   - Section:
   - Missing decision:
   - Suggested options:

3. Missing requirement:
   - Area:
   - Why it matters:
   - Suggested addition:

Next action:
- Keep spec in draft
- Clarify the listed points
- Update the spec
- Ask project owner for approval
- After approval, move spec to checked-and-approved
```

## Required Report Format When Spec Is Ready for Approval

```txt
Spec Verification Result: ready for approval

Spec:
- [path/name]

Checked:
- No contradictions found
- No unresolved gray zones found
- No missing implementation-critical decisions found
- No conflicts with higher-level specs found

Recommendation:
- Project owner may approve this spec and move it to checked-and-approved.
```

## Required Report Format When Spec Is Already Approved

```txt
Spec Verification Result: approved

Spec:
- [path/name]

Status:
- checked-and-approved

Checked:
- No contradictions found
- No unresolved gray zones found
- No missing implementation-critical decisions found
- No conflicts with higher-level specs found

Implementation may continue.
```
