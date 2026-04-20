# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an Obsidian-based personal knowledge base ("Second Brain") for a developer focused on Java Spring Boot, React, and System Design. Its primary purpose is structured note-taking for deep learning and interview preparation.

## Language Convention

- **Explanations**: Accented Vietnamese (Tiếng Việt có dấu). Do not use unaccented Vietnamese.
- **Technical terms, code, and identifiers**: English
- **No emoji**: Never use emoji anywhere in a note. Plain text only.

## Note Structure: 16 Sections

Every note must follow this exact 16-section structure (defined in `NOTE_CONVENTION.md`):

| # | Section | Key requirement |
|---|---------|----------------|
| 1 | **What** | 2-3 sentence definition |
| 2 | **Why** | The problem it solves; why it was created |
| 3 | **Mental Model** | An analogy/metaphor — the most important section |
| 4 | **Where it fits** | Position in the system; use text diagrams like `A -> B -> C` |
| 5 | **When to use** | Specific conditions/contexts |
| 6 | **When NOT to use** | Consequences of misuse; unsuitable cases |
| 7 | **Trade-offs** | Pros/Cons table |
| 8 | **Alternatives** | Comparison table with other solutions |
| 9 | **How** | Minimal working code example |
| 10 | **Production concerns** | Scaling, failure modes, monitoring |
| 11 | **Common mistakes** | At least 2 pairs of Mistake / Fix (plain text, no emoji) |
| 12 | **Sample project** | Hands-on exercise with a hard constraint |
| 13 | **Interview** | Core Q&A + Scenarios (3–8 items each) |
| 14 | **References** | Official docs, GitHub repo, Spec/RFC, Changelog |
| 15 | **Real-world Code** | Production-grade GitHub repos to learn real patterns from |
| 16 | **Community** | Reddit, Stack Overflow, blog posts, conference talks |

## Required Frontmatter

Every note must start with:

```yaml
---
created: yyyy-MM-dd
tags:
  - "#type/concept"       # concept | library | pattern | tutorial
  - "#status/draft"       # draft | review | done
  - "#lang/java"          # java | spring | javascript | nodejs | react | devops | database | system-design
  - "#topic/async"        # async | http | i18n | state-management | performance | error-handling
related:
  - "[[Related Note]]"
---
```

`#topic` tags are optional but recommended for cross-cutting concerns to enable queries like "all async concepts in nodejs".

## File Naming Convention

| Note type | Pattern | Examples |
|-----------|---------|---------|
| General concept / topic | Title Case with spaces | `Event Loop.md`, `Non-blocking IO.md`, `Optional.md` |
| Framework-specific topic | `<Topic> in <Framework>.md` | `RestTemplate in Spring Boot.md`, `Feign Client in Spring Boot.md` |
| Library / tool / API name | lowercase kebab-case | `tanstack-react-query.md`, `criteria-api.md`, `fnv-1a.md` |
| React hooks | camelCase; combine related hooks with `-` | `useTranslation.md`, `useQuery-useMutation.md` |
| Research / tutorial | Descriptive Title Case with spaces (parens allowed) | `Researching an Existing Module (i18n Spring Boot use case).md` |

Rules:
- Use `.md` extension for all notes.
- Do not use underscores.
- Keep names concise — they become Obsidian page titles and `[[wikilinks]]`.

## Folder Structure (PARA method)

All knowledge notes go inside `Second Brain/01-Knowledge/` in the correct subfolder:

| Tech Stack | Target folder |
|-----------|--------------|
| Java Core | `01-Knowledge/java-core/` |
| Spring Boot | `01-Knowledge/java-spring/` |
| JavaScript | `01-Knowledge/javascript/` |
| Node.js | `01-Knowledge/nodejs/` |
| React | `01-Knowledge/react/` |
| DevOps / AWS | `01-Knowledge/devops/` |
| System Design | `01-Knowledge/system-design/` |
| Database | `01-Knowledge/database/` |
| Other | `01-Knowledge/others/` |

Never place notes directly in the root `01-Knowledge/` folder. Create the subfolder if it does not exist.

Other top-level folders:
- `00-Inbox/` — unclassified drafts
- `02-Projects/` — notes tracking real ongoing projects
- `03-Resources/` — reference material, cheatsheets, articles
