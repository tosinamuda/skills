# Skills

Agent skills, one folder each, in the `SKILL.md` format that Claude Code and other agents load.

| Skill | What it is for |
| --- | --- |
| [react-components](react-components/SKILL.md) | Semantic, composed, markup-first React: semantic elements, plain names, thin hooks that derive. |
| [business-logic](business-logic/SKILL.md) | Where a front-end app's rules live and how they are layered, and how to pull them out of components into tested pure functions. |

## Install

Link a skill into the folder your agent loads skills from:

```bash
ln -s ~/workspace/skills/react-components ~/.claude/skills/react-components
ln -s ~/workspace/skills/business-logic ~/.claude/skills/business-logic
```
