---
name: generate-tutorial
description: Generate a structured tutorial from a documentation web page. Use when the user wants to create professional-level tutorial markdown files from a URL.
---

# Generate Tutorial from Documentation Page

## Overview

This skill reads a documentation web page and produces a multi-file, professional-level tutorial broken into logical steps. The target audience is working developers who know the basics — no hand-holding, no filler.

## Language Requirement

**All generated tutorial content MUST be written in Taiwan Traditional Chinese (台灣繁體中文, zh-TW).**

- All explanations, headings, descriptions, summaries, and learning notes must be in Taiwan Traditional Chinese.
- Use natural Taiwan-style phrasing (台灣用語), not Simplified Chinese (簡體中文) or Hong Kong style (港式中文).
- Common Taiwan-specific conventions:
  - Use 「」 for quotation marks instead of " " when quoting terms in Chinese text.
  - Use Taiwan terminology: 程式（not 程序）, 資料（not 数据）, 伺服器（not 服务器）, 網路（not 网络）, 變數（not 变量）, 函式（not 函数）, 物件（not 对象）, 介面（not 接口/界面）.
- **Keep in English**: code examples, variable names, function names, library names, file paths, CLI commands, and widely recognized technical terms (e.g., API, URL, HTTP, OAuth, CSRF, XSS, JWT).
- Code comments inside code blocks should also be in Traditional Chinese.

## Input

The user provides a URL to a documentation page and optionally a topic slug for the output directory name.

## Process

1. **擷取頁面內容**: Use `WebFetch` (or `curl` via Bash if WebFetch fails due to redirects) to retrieve the full content of the documentation page.

2. **分析結構**: Identify the major sections and subsections of the page. Map out the logical flow of the content.

3. **規劃教學檔案**: Break the content into numbered tutorial files, one per logical section:
   - `01-introduction.md` — Always first. Covers: what it does, architecture overview, prerequisites.
   - `02-xxx.md` through `NN-xxx.md` — One file per major section, named descriptively.

4. **撰寫每個檔案** with these elements:
   - Clear title and concise description in Traditional Chinese
   - Direct, technical explanations — skip obvious basics, focus on how things work and why
   - Production-quality code examples with inline comments in Traditional Chinese highlighting non-obvious details
   - Edge cases, gotchas, and real-world considerations where relevant
   - 「重點整理」section at the end of each file
   - Link to the next file in the sequence

5. **Output directory**: Place all files in `tutorials/<topic-slug>/` where `<topic-slug>` is derived from the page title (e.g., `tanstack-start-authentication`).

## Tutorial File Template

Each file should follow this general structure:

```markdown
# <步驟編號>: <標題（繁體中文）>

> <一句話摘要說明本節內容>

## <章節內容>

<繁體中文說明、程式碼範例、注意事項>

## 重點整理

- <用條列式摘要本節學到的內容>

---

下一篇：[<下一篇標題>](./<next-file>.md)
```

## Guidelines

- 以有經驗的開發者為目標讀者 — 不需要解釋基礎概念，直接進入核心內容
- Code examples should be production-relevant with real-world patterns, not toy examples
- 每個教學檔案聚焦於一個概念或章節
- 涵蓋 edge cases、常見陷阱、以及實務上需要注意的細節
- 使用精簡、直接的語句 — 不要鋪陳、不要「首先讓我們了解什麼是 X」之類的開場
- 在介紹檔案中保留原始文件的來源 URL 作為參考
