---
name: generate-tutorial
description: Generate a step-by-step tutorial from a documentation web page. Use when the user wants to learn about a topic by creating beginner-friendly tutorial markdown files from a URL.
---

# Generate Tutorial from Documentation Page

## Overview

This skill reads a documentation web page and produces a multi-file, beginner-friendly tutorial broken into logical steps.

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
   - `01-introduction.md` — Always first. Covers: what it is, what it can do, prerequisites.
   - `02-xxx.md` through `NN-xxx.md` — One file per major section, named descriptively.

4. **撰寫每個檔案** with these elements:
   - Clear title and brief description in Traditional Chinese
   - Beginner-friendly explanations in Traditional Chinese (assume the reader is new to the topic)
   - Code examples with inline comments in Traditional Chinese explaining what each part does
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

- 以初學者為目標讀者撰寫 — 解釋專業術語，不要假設讀者已經了解該框架
- Include all code examples from the source, with added comments in Traditional Chinese for clarity
- 每個教學檔案聚焦於一個概念或章節
- 使用實際、貼近真實情境的說明方式（例如：「這個功能的實際用途是……」）
- 在介紹檔案中保留原始文件的來源 URL 作為參考
