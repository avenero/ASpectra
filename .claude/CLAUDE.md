### 1. Project Source of Truth

The official source of truth for this project is its documentation and approved designs, NOT the existing codebase. All the project docuemntation is in the `docs` directory and Figma.

The following resources define the intended architecture, requirements, scope, product behavior and someones specified features(but not all ASpectra required). To make posible and more eficient this implementation, you need implement all the Architecture(Core, BE, and FE) and them implement each `doc/specs/*/*.md` over robust, manetenible and scalable architecture you will be implemented:

### Project Scope

@docs/scope.md

Defines:

- Project boundaries and objectives.

- In-scope and out-of-scope functionality.

- Product capabilities and limitations.

### Project Requirements

@docs/requirements.md

Defines:

- Functional and non-functional requirements.

- Expected system behavior.

- Business rules and acceptance criteria.

- System constraints and expected outcomes.

### Figma Design of Record

### Figma File ID: DpfPk8xDraLcDzBhMYwDTo

### Figma is the authoritative visual and architectural reference.

## Architecture
https://www.figma.com/design/DpfPk8xDraLcDzBhMYwDTo/Architecture?node-id=0-1&p=f&t=tG08Frxxs4F8Eqcg-0

## Screens & Product Experience
https://www.figma.com/design/DpfPk8xDraLcDzBhMYwDTo/Architecture?node-id=148-2&p=f&t=tG08Frxxs4F8Eqcg-0

The Figma file must be inspected beyond the initial frames when relevant to the task, including related boards, diagrams, components, interactions, and annotations.

### 2. Authority and Documentation Rules

### The following rules are mandatory:
- The constitution.md and specs folder are not the complete scope, only represent the requirements refined at this time. So, those doesn't means that when we finish those implementation, the complete product is done

- Documentation and approved Figma designs define the intended system. Existing code is NOT the source of truth.

- Never assume that existing code, architecture, implementations, data structures, APIs, or behaviors represent the intended design.

- Existing code must be treated as a reference for understanding the current implementation, identifying dependencies, assessing technical constraints, and evaluating reusable components.

- All new implementations and architectural changes must align with the approved documentation and Figma designs.

- Do not preserve existing implementations simply because they already exist. They may require modification, replacement, or removal to achieve the documented architecture.

- Do not introduce undocumented functionality, architectural decisions, business rules, integrations, or dependencies without justification and approval when they affect project scope or architecture.

- If documentation is incomplete, ambiguous, or contradictory, identify the issue and request clarification(Asking one By one) before making assumptions that could affect architecture, business behavior, or important technical decisions.

- Never silently resolve conflicts between authoritative sources. Explain the conflict, its impact, and the clarification required.

- If implementation reveals that the documented architecture cannot be achieved due to technical constraints, explain the limitation and propose alternatives before deviating from the approved design.

### 3. Mandatory Workflow Before Implementation

### Before modifying any code:

- Read and understand `docs/scope.md`, `docs/requirements.md`, `constitution.md` and `docs/specs/*/*`.md

- Inspect the relevant Figma architecture diagrams, product designs, annotations, and interactions.

- Understand the intended architecture and expected behavior before analyzing how the existing code implements them.

- Inspect the existing codebase to identify:

- Current architecture and component responsibilities.

- Existing dependencies and integrations.

- Components that can be reused.

- Components that must be modified, replaced, or removed.

- Potential architectural conflicts and migration requirements.

- Compare the current implementation against the documented target architecture.

- Identify gaps, dependencies, risks, and implementation implications.

- Wait for explicit approval when the task requires architectural decisions, undocumented assumptions, breaking changes, or decisions outside the approved requirements.

### 4. Implementation Principles

- Follow the documented target architecture rather than adapting requirements to the existing code.

- Maintain clear separation of responsibilities and architectural boundaries.

- Avoid unnecessary complexity, duplication, and overengineering.

- Reuse existing components only when they are compatible with the target architecture.

- Do not introduce speculative abstractions or functionality.

- Preserve backward compatibility only when explicitly required by the documented requirements or approved migration strategy.

- Do not modify project governance files or authoritative documentation without explicit approval.

- Keep implementation changes aligned with the approved scope.

- Be simple. Approach tasks in a simple, incremental way.

- Work incrementally ALWAYS. Small, simple steps. Validate and check each increment before moving on.

- Use LATEST apis


### MANDATORY CODE STYLE
- Do not overengineer. Do not program defensively. Use exception manager only when needed.

- Identify root cause before fixing issues. Prove with evidence, then fix.

- Work incrementally with small steps. Validate each increment.

- Use LATEST librery APIs.

- Favor clear, concise docstring comments. Be sparing with comments outside docstrings.

- Never use emojis in code, documentations, loggings and statements.

- Keep always README.md concise and update.

- Analys every task looking for gaps and design violations, in those cases, please stop the implementation, edit the existing design making an deep analisys, then leave a summary.

- Always leave on chat as summary, what code rule, implementations standars, arquitecture violations you found in the code or in the task you done.


### 5. Validation and Completion Criteria

A task is NOT considered complete simply because code changes have been implemented.

Upon completion:

- Execute the relevant automated tests.

- Execute applicable build, compilation, linting, and static analysis checks.

- Validate the implemented behavior against the corresponding requirements and acceptance criteria.

- Verify that architectural boundaries and responsibilities remain consistent with the approved design.

- Identify any remaining gaps, limitations, or unverified behavior.

- In the final response, provide:

- Summary of implemented changes.

- Relevant files and components modified.

- Requirements and designs addressed.

- Tests and validations executed, including their actual results.

- Known limitations or outstanding issues.

- Any decisions requiring human approval.

- Never claim that everything works correctly unless the relevant behavior has actually been verified. Clearly distinguish between implemented, tested, and verified functionality.

### 6. Documentation Maintenance

- Documentation remains the authoritative reference throughout development.

- When an approved requirement or architectural decision changes:

- Update the corresponding authoritative documentation when explicitly requested or approved.

- Keep implementation and documentation consistent.

- Never treat an undocumented code change as an automatic change to the specification.

- Identify documentation that becomes outdated after an approved architectural change.

### 7. Mandatory Rule

- DOCUMENTATION DEFINES WHAT TO BUILD. CODE DEFINES WHAT CURRENTLY EXISTS.

- Never reverse these responsibilities.