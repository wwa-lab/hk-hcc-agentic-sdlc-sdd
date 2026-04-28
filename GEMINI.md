# GEMINI Workspace Guide

This repository contains a Claude-style skill library under `.claude/skills/`.
Gemini should not assume those skills are auto-discoverable. Instead, use this file
as the primary entry point and `.gemini/skills-index.md` as the skill catalog.

## How To Work In This Repo

When the user asks for SDLC document generation, review, planning, or implementation support:

1. Read `.gemini/skills-index.md` to find the matching workflow.
2. Open the referenced Claude skill at `.claude/skills/<skill-name>/SKILL.md`.
3. Follow that skill's instructions as the source workflow.
4. Write outputs into the repo's `docs/` structure unless the user asks otherwise.

## Preferred Docs Flow

Use these stages by default:

1. `docs/00-context`
2. `docs/01-requirements`
3. `docs/02-user-stories`
4. `docs/03-spec`
5. `docs/04-architecture`
6. `docs/05-design`
7. `docs/06-tasks`

## Skill Routing

- Requirements or raw notes to stories:
  use `req-to-user-story`
- Stories to engineering spec:
  use `user-story-to-spec`
- Spec to architecture:
  use `spec-to-architecture`
- Architecture or spec to design:
  use `architecture-to-design`
- Design to execution tasks:
  use `design-to-tasks`
- Tasks to incremental repo code:
  use `tasks-to-code`
- Tasks to greenfield, brownfield, or migration implementation:
  use `tasks-to-implementation`
- Review docs before handoff:
  use `review-doc-quality`
- Review whether code matches design:
  use `review-code-against-design`

## Output Defaults

Unless the user specifies a different filename, prefer:

- User stories:
  `docs/02-user-stories/user-stories.md`
- Spec:
  `docs/03-spec/spec.md`
- Architecture:
  `docs/04-architecture/architecture.md`
- Data model:
  `docs/04-architecture/data-model.md`
- Design:
  `docs/05-design/design.md`
- API implementation guide:
  `docs/05-design/contracts/API_IMPLEMENTATION_GUIDE.md`
- Tasks:
  `docs/06-tasks/tasks.md`

## Important Notes

- `.claude/skills/**/SKILL.md` is the authoritative workflow source.
- Do not assume hidden tool integrations from Claude-specific metadata such as
  `allowed-tools` or `argument-hint`; treat them as guidance only.
- If multiple skills could apply, choose the one that best matches the user's
  current SDLC stage.
