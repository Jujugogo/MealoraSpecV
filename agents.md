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

## Spec Language Rule

Project specifications must be written in Russian by default.

Technical identifiers, route names, database field names, status values, API names, framework names, and code-level terms may remain in English when that keeps the implementation contract precise.

Use another language for a spec only if the project owner explicitly requests it.

## Spec Folder and Status Synchronization Rule

The folder where a spec is located and the status written inside the spec must always match.

Required mapping:

```txt
draft folder -> Status: draft
checked-and-approved folder -> Status: checked-and-approved
```

This rule applies regardless of case or local folder spelling variants used in the repository, including `Draft`, `draft`, `checked-and-approved`, `checked and approved`, or similar approved-status folder names.

When moving any spec from `draft` to `checked-and-approved` after project owner approval, the agent must in the same change:

1. Move the spec into the approved-status folder.
2. Update the status inside the spec to `checked-and-approved`.
3. Re-read the moved file and verify that the path and internal status match.
4. Report the path and final status to the user.

When moving any spec back from `checked-and-approved` to `draft`, the agent must in the same change:

1. Move the spec into the draft-status folder.
2. Update the status inside the spec to `draft`.
3. Re-read the moved file and verify that the path and internal status match.
4. Report the path and final status to the user.

The agent must not leave a spec with a draft status inside an approved-status folder, or an approved status inside a draft-status folder.

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
* The agent may commit, create branches, push, and open pull requests while the spec remains in `draft/`, as long as all assumptions and unresolved points are documented in the draft spec and reported to the project owner.
* The agent must not move the spec to `checked-and-approved/` until the spec is checked and the project owner explicitly approves that move.

## Approval Ownership Rule

The agent may suggest moving a spec to `checked-and-approved/`, but only the project owner can approve this status change.

The agent must not silently approve a spec by itself.

Correct behavior:

```txt
I checked the spec and found no blocking issues.
Recommendation: this spec can be moved to checked-and-approved after project owner approval.
```

## Commit and Push Gate

Before every commit or push, the agent must perform a Spec Status Gate.

The agent must verify:

1. Every changed feature, page, API route, database schema, UI rule, or workflow is covered by a related spec.
2. The related spec is located in either `/specs/draft/` or `/specs/checked-and-approved/`.
3. The implementation does not contradict the related spec.
4. Any new decisions discovered during implementation were added back to the spec.
5. No temporary assumptions remain undocumented.
6. If the related spec is in `draft/`, the commit or pull request must clearly state that the work is based on a draft spec and is not yet checked-and-approved.
7. Every changed spec file has an internal status that matches its containing status folder.
8. No spec was moved from `draft/` to `checked-and-approved/` without project owner approval.

## Commit Blocking Rule

The agent must not commit or push if:

* The related spec is missing.
* The implementation contains assumptions not written in the related spec.
* The code behavior differs from the related spec.
* The related spec has unresolved contradictions, gray zones, or assumptions that are not documented in the spec.
* The project owner has not approved moving a spec from `draft/` to `checked-and-approved/`, but the change moves it anyway.
* Any changed spec file has an internal status that does not match its containing status folder.

In this case, the agent must stop and report:

```txt
Commit blocked: spec gate failed.

Reason:
- [list exact reason]

Required action:
- [what must be documented, clarified, or corrected before commit]
- [whether any spec move requires project owner approval]
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
   - yes / no / not required for this commit

4. Did implementation follow the spec exactly?
   - yes / no

5. Were new decisions added back into the spec?
   - yes / no / not needed

6. Are there unresolved gray zones?
   - yes / no

7. Do all changed spec files have status matching their folder?
   - yes / no

8. Commit allowed?
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
