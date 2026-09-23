# Building with Jev

This repo holds an agent skill for writing and improving programs that call Jev, TypeSafe's System One model.

The skill covers question design (Choice, Score, Noul), state structure, answer composition in code, confidence thresholds, and diagnosis of questions that answer wrong or with low confidence. It targets `jev-1.13`.

The skill lives in [`skills/jev/SKILL.md`](skills/jev/SKILL.md).

## Install

### Claude Code plugin

Run these two commands inside Claude Code:

```
/plugin marketplace add dbreunig/building-with-jev-skill
/plugin install jev@building-with-jev
```

### Skills CLI

The [skills CLI](https://github.com/vercel-labs/skills) installs the skill into Claude Code, Codex, Cursor, and other agents:

```bash
npx skills add dbreunig/building-with-jev-skill
```

### Manual

Copy the skill folder into your personal skills directory:

```bash
git clone https://github.com/dbreunig/building-with-jev-skill
cp -r building-with-jev-skill/skills/jev ~/.claude/skills/jev
```

## Use

Claude loads the skill on its own when a task involves Jev or TypeSafe questions. You can also invoke it by name with `/jev`.

When installed as the `jev` plugin, invoke the skill explicitly with `/jev:jev`.

## Sources

The guidance draws on the [TypeSafe docs](https://docs.typesafe.ai). `SKILL.md` lists the specific pages.
