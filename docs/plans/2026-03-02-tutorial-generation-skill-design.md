# Tutorial Generation Skill — Design

## Goal

Create a Claude Code skill that reads a web documentation page and produces a multi-file, beginner-friendly tutorial in markdown.

## Skill Definition

- **Location:** `.claude/skills/generate-tutorial.md`
- **Trigger:** User asks to generate a tutorial from a documentation URL
- **Input:** A URL to a documentation page
- **Output:** A directory of numbered markdown files under `tutorials/<topic-slug>/`

## Tutorial Structure

```
tutorials/<topic-slug>/
  01-introduction.md    — What it is, what it can do, prerequisites
  02-<section>.md       — First major section
  03-<section>.md       — Second major section
  ...
  NN-<section>.md       — Final section
```

### File format

Each tutorial file includes:
- A clear title and brief description
- Explanations written for beginners
- Code examples (copied/adapted from the source, with annotations)
- Learning notes or tips where helpful

## First Tutorial

- **Source:** https://tanstack.com/start/latest/docs/framework/react/authentication
- **Output:** `tutorials/tanstack-start-authentication/`

## Implementation Steps

1. Create `.claude/skills/generate-tutorial.md` skill file
2. Fetch the TanStack Start authentication docs page
3. Analyze the page structure and identify sections
4. Generate tutorial files in `tutorials/tanstack-start-authentication/`
5. Git commit, push, and create PR
