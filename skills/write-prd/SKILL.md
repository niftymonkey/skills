---
name: write-prd
description: Create a PRD through user interview, codebase exploration, and module design, saved as a local markdown file with optional GitHub issue submission. Works for new projects, features, or refactors. Use when user wants to write a PRD, create a product requirements document, plan a new feature, or spec out a new project.
disable-model-invocation: true
---

This skill will be invoked when the user wants to create a PRD. You may skip steps if you don't consider them necessary.

1. Check if the user has an explore-idea output file to use as a starting point. If so, read it and use it to pre-fill your understanding; skip or shorten the interview for areas already resolved. If not, ask the user for a long, detailed description of the problem they want to solve and any potential ideas for solutions.

2. If there's an existing codebase, explore the repo to verify assertions and understand the current state. Skip this step for greenfield projects with no code yet.

3. Interview the user relentlessly about every aspect of this plan until you reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one.

4. Sketch out the major modules you will need to build or modify to complete the implementation. If the `architect-deep` skill is installed, use it for this step; otherwise sketch the modules directly using the deep-module method (pre-deletion test, dependency classification, interface design) before any code is written. Either way, the result populates the **Implementation Decisions** section of the PRD.

Check with the user that the resulting modules match their expectations. Check with the user which modules they want tests written for.

5. Once you have a complete understanding of the problem and solution, use the template below to write the PRD. Choose the appropriate save location:

   - **`./docs/`** (in-repo): for project-specific documents, active work, design decisions, things that are already true about the repository, and technical reference.
   - **An external directory** (outside the repository): for higher-level planning, historical reference, or content that shouldn't live in a public repository. Ask the user where it should go; don't assume a path.

   If the chosen directory doesn't exist, confirm before creating it. If not in a project directory, ask where to save. After saving, ask the user if they'd also like to submit it as a GitHub issue.

<prd-template>

## Problem Statement

The problem that the user is facing, from the user's perspective.

## Solution

The solution to the problem, from the user's perspective.

## User Stories

A LONG, numbered list of user stories. Each user story should be in the format of:

1. As an <actor>, I want a <feature>, so that <benefit>

<user-story-example>
1. As a mobile bank customer, I want to see balance on my accounts, so that I can make better informed decisions about my spending
</user-story-example>

This list of user stories should be extremely extensive and cover all aspects of the feature.

## Implementation Decisions

A list of implementation decisions that were made. This can include:

- The modules that will be built/modified
- The interfaces of those modules that will be modified
- Technical clarifications from the developer
- Architectural decisions
- Schema changes
- API contracts
- Specific interactions

Do NOT include specific file paths or code snippets. They may end up being outdated very quickly.

## Testing Decisions

A list of testing decisions that were made. Include:

- A description of what makes a good test (only test external behavior, not implementation details)
- Which modules will be tested
- Prior art for the tests (i.e. similar types of tests in the codebase)

## Out of Scope

A description of the things that are out of scope for this PRD.

## Further Notes

Any further notes about the feature.

</prd-template>
