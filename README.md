# Skills

Agent skills, one folder each, in the `SKILL.md` format that Claude Code and other agents load.

| Skill | What it is for |
| --- | --- |
| [react-components](react-components/SKILL.md) | Semantic, composed, markup-first React: semantic elements, plain names, thin hooks that derive, logic outside React. |

## Install

Link a skill into the folder your agent loads skills from:

```bash
ln -s ~/workspace/skills/react-components ~/.claude/skills/react-components
```
