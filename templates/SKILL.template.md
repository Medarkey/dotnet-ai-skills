---
name: topic-name
description: What the skill does, in one or two sentences, followed by when to use it. List the concrete triggers an agent will see in a request or a repository - file types, commands, libraries, tasks (create, review, fix, migrate). Say what it does NOT cover when a neighbouring skill does.
---

<!--
Authoring rules (delete this comment in a real skill)

Format
- Folder name = `name` field: lowercase letters, digits, hyphens; max 64 characters.
  Use the plain topic name (docker, bicep, minimal-api). The whole library targets .NET,
  so don't add a dotnet- prefix or repeat ".NET" in the text.
- Frontmatter has only `name` and `description`, so every tool can read it.
- The description decides when the skill loads: be specific, include trigger words,
  stay under 1024 characters.
- Keep SKILL.md under ~400 lines. Move long material into `references/<topic>.md`
  and link to it from here with a sentence saying when to read it.

Content
- Version-agnostic: derive versions from the project (TargetFramework, global.json, etc.).
  A version in an example is only a placeholder and must be presented that way.
- No company, product, or person names; no internal packages, URLs, or feeds.
- SKILL.md is written for the AI, not for a human reader. Write rules, not explanations:
  "Do X" / "Don't do Y: <short reason>". Add a reason only when it stops the AI from
  undoing the rule or helps it handle a case the rule doesn't cover.
- No history or background ("older versions used to..."). Human-facing explanations
  belong in the README.
- Every code sample should work after replacing placeholders (MyApp, MyApp.Api).
- Prefer platform defaults and official guidance over custom setups.
- Respect existing projects: maintain their conventions, don't migrate as a side effect.
-->

# Topic title

One or two sentences: the goal and the quality bar.

## 1. Choose an approach

Decision table: situation → recommended approach. Default first.

## 2. Patterns

Working templates with placeholders and short notes.

## 3. Rules

Grouped do / don't rules with the reason when it isn't obvious.

## 4. Anti-patterns

| Avoid | Do instead |
| --- | --- |
|  |  |

## Review checklist

- [ ] Item that a reviewer can check quickly
