# Skills

A collection of Claude Code skills for applying development methodologies.

## Skills

| Skill | Description |
|-------|-------------|
| [neg-space](./neg-space/README.md) | Negative Space Programming — specification, invariants, and rejection-first testing |
| [tigerskill](./tigerskill/README.md) | TigerStyle — safety, performance, and experience principles from TigerBeetle |

## How skills work

Skills are triggered by keywords in prompts. When a trigger phrase is detected, Claude loads the relevant `SKILL.md` and applies the methodology to the task at hand.

## Adding a skill

Each skill lives in its own directory with this structure:

```
skill-name/
├── SKILL.md       # Trigger keywords and methodology instructions for Claude
├── README.md      # Human-readable description of the skill
└── references/    # (optional) Source material, papers, or examples
```
