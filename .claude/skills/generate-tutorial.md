---
name: generate-tutorial
description: Generate a step-by-step tutorial from a documentation web page. Use when the user wants to learn about a topic by creating beginner-friendly tutorial markdown files from a URL.
---

# Generate Tutorial from Documentation Page

## Overview

This skill reads a documentation web page and produces a multi-file, beginner-friendly tutorial broken into logical steps.

## Input

The user provides a URL to a documentation page and optionally a topic slug for the output directory name.

## Process

1. **Fetch the page**: Use `WebFetch` (or `curl` via Bash if WebFetch fails due to redirects) to retrieve the full content of the documentation page.

2. **Analyze structure**: Identify the major sections and subsections of the page. Map out the logical flow of the content.

3. **Plan tutorial files**: Break the content into numbered tutorial files, one per logical section:
   - `01-introduction.md` — Always first. Covers: what it is, what it can do, prerequisites.
   - `02-xxx.md` through `NN-xxx.md` — One file per major section, named descriptively.

4. **Write each file** with these elements:
   - Clear title and brief description of what the section covers
   - Beginner-friendly explanations (assume the reader is new to the topic)
   - Code examples with inline comments explaining what each part does
   - "Key Takeaways" or "Learning Notes" section at the end of each file
   - Link to the next file in the sequence

5. **Output directory**: Place all files in `tutorials/<topic-slug>/` where `<topic-slug>` is derived from the page title (e.g., `tanstack-start-authentication`).

## Tutorial File Template

Each file should follow this general structure:

```markdown
# <Step Number>: <Title>

> <One-sentence summary of what this section covers>

## <Section Content>

<Explanations, code examples, notes>

## Key Takeaways

- <Bullet points summarizing what was learned>

---

Next: [<Next Title>](./<next-file>.md)
```

## Guidelines

- Write for beginners — explain jargon, don't assume prior knowledge of the framework
- Include all code examples from the source, with added comments for clarity
- Keep each tutorial file focused on one concept or section
- Use practical, real-world framing ("Here's why you'd use this...")
- Preserve the original source URL as a reference in the introduction file
