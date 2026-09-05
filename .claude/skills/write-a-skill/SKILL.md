---
name: write-a-skill
description: Create new agent skills with proper structure, progressive disclosure, and bundled resources. Use when user wants to create, write, or build a new skill.
---

# Writing Skills

## Process

1. **Gather requirements** - ask user about:
   - What task/domain does the skill cover?
   - What specific use cases should it handle?
   - Does it need executable scripts or just instructions?
   - Any reference materials to include?

2. **Draft the skill** - create:
   - SKILL.md with concise instructions
   - Additional reference files if content exceeds 500 lines
   - Utility scripts if deterministic operations needed

3. **Review with user** - present draft and ask:
   - Does this cover your use cases?
   - Anything missing or unclear?
   - Should any section be more/less detailed?
   - Which existing skills touch the same code, and is the boundary with each of them written down? (see Scope and Explicit Non-Scope)

## Skill Structure

```
skill-name/
├── SKILL.md           # Main instructions (required)
├── REFERENCE.md       # Detailed docs (if needed)
├── EXAMPLES.md        # Usage examples (if needed)
└── scripts/           # Utility scripts (if needed)
    └── helper.js
```

## SKILL.md Template

```md
---
name: skill-name
description: Brief description of capability. Use when [specific triggers].
---

# Skill Name

## Quick start

[Minimal working example]

## Workflows

[Step-by-step processes with checklists for complex tasks]

## Advanced features

[Link to separate files: See [REFERENCE.md](REFERENCE.md)]
```

## Description Requirements

The description is **the only thing your agent sees** when deciding which skill to load. It's surfaced in the system prompt alongside all other installed skills. Your agent reads these descriptions and picks the relevant skill based on the user's request.

**Goal**: Give your agent just enough info to know:

1. What capability this skill provides
2. When/why to trigger it (specific keywords, contexts, file types)

**Format**:

- Max 1024 chars
- Write in third person
- First sentence: what it does
- Second sentence: "Use when [specific triggers]"

**Good example**:

```
Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when user mentions PDFs, forms, or document extraction.
```

**Bad example**:

```
Helps with documents.
```

The bad example gives your agent no way to distinguish this from other document skills.

## Scope and Explicit Non-Scope

Every skill needs a stated scope **and** a stated non-scope. Without the second one, each skill drifts toward "I will check everything", and two skills in the same pipeline produce duplicate findings for the same line of code.

Put a `## Scope boundary` section in SKILL.md with:

1. **A DO / DO NOT table.** Each DO NOT row names the skill that owns the concern, not just the topic:

   | DO | DO NOT (owner) |
   |---|---|
   | dependency direction, coupling, cycles | responsibilities inside a component — `backend-architecture-review` |
   | interface / port needed or excessive | SQL, indexes — SQL review |

2. **A one-line splitting rule** a reader can apply to a borderline case. Naming the topics is not enough when two skills touch the same code. Example: "this skill asks *whose code lives in the component*; the other asks *what the component knows about others* — if the fix is moving code, it is here; if the fix is removing an import or changing a type, it is there".

3. **Two or three borderline examples** with a verdict "here" / "not here", so the agent can pattern-match instead of guessing.

4. **A handoff rule.** Anything noticed outside scope goes into an "Out of scope" line in the report, with the owner skill named, so the next skill in the pipeline can pick it up.

Stronger than a DO / DO NOT table is an **ownership criterion** at the top of SKILL.md, so the agent can classify a finding, not just a topic:

```md
## Primary question
> What kind of work is being performed here?

A finding belongs to this skill when the problem is caused by
the wrong kind of work being located inside a component.
Fixes typically move code between components.
```

Then check the **description**: it must not contain keywords from another skill's DO column. The description is the only thing the router sees; a DO NOT section inside the file cannot undo a description that promises the same coverage. Change the description first.

If the skill is one step of a sequential pipeline, say which skill runs before it and what it takes as input from that report (a component map, a list of findings). Say that the input is a starting model to verify against the code, not a fact to trust: the previous agent may have missed something. If both skills read the same code, say how to avoid a second finding on the same line (for example: evaluate the state after the previous skill's fixes, not before).

### Scope ownership check

Before finalizing a skill:

1. Define the primary question this skill answers.
2. Define what this skill MUST NOT evaluate.
3. For every non-scope concern, name the owning skill.
4. Ensure the frontmatter description does not advertise concerns owned by another skill.
5. Ensure every finding category has exactly one owner across the pipeline. This cannot be checked inside one skill: keep an ownership matrix (finding category → owner skill) next to the skills, and update it when a skill is added or its scope changes.
6. Ensure recommendations stay within the skill's ownership boundary. A perfectly split scope leaks again through recommendations: "Controller depends directly on Prisma → introduce a Repository interface and Prisma adapter" is a dependency review written inside an architecture skill.
7. Ensure the report template has a mandatory `## Scope` block with `Evaluated:` and `Not evaluated:` lists, each not-evaluated item naming its owner. The next agent must see who is obliged to check it, not just that it was skipped.

## When to Add Scripts

Add utility scripts when:

- Operation is deterministic (validation, formatting)
- Same code would be generated repeatedly
- Errors need explicit handling

Scripts save tokens and improve reliability vs generated code.

## When to Split Files

Split into separate files when:

- SKILL.md exceeds 100 lines
- Content has distinct domains (finance vs sales schemas)
- Advanced features are rarely needed

## Review Checklist

After drafting, verify:

- [ ] Description includes triggers ("Use when...")
- [ ] SKILL.md under 100 lines
- [ ] No time-sensitive info
- [ ] Consistent terminology
- [ ] Concrete examples included
- [ ] References one level deep
- [ ] Primary question and ownership criterion at the top of SKILL.md
- [ ] DO NOT list with owner skill per item, plus borderline examples for concerns shared with a neighbour skill
- [ ] Description contains no keywords from another skill's DO column
- [ ] Recommendations stay inside the ownership boundary
- [ ] Report template has a `## Scope` block (Evaluated / Not evaluated + Owner) and an "Out of scope" line naming the owner
- [ ] Ownership matrix updated if this skill shares a pipeline with others
