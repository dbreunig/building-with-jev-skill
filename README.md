# Building with Jev

This repo holds an agent skill for writing and improving programs that call Jev, TypeSafe's System One model.

The skill covers these topics:

- question design for Choice, Score, Noul, and Box
- state structure, including images
- answer composition in code
- confidence thresholds
- diagnosis of questions that answer wrong or with low confidence

It targets `jev-1.14` and `jev-1.13`. Images and Box questions are experimental in the TypeSafe API, and the skill marks them that way.

TypeSafe publishes its own agent skill at [typesafe-ai/skills](https://github.com/typesafe-ai/skills). That skill covers the API and helps you find places to use TypeSafe. This skill focuses on writing questions, composing answers, and fixing questions that fail.

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

## Sources

The guidance draws on the [TypeSafe docs](https://docs.typesafe.ai). `SKILL.md` lists the specific pages.
