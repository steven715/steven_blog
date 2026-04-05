# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a personal knowledge base / blog repository. It contains pure Markdown articles organized by topic — no build system, no framework, no dependencies.

## Content Structure

- **diary/** — Daily journal entries (English), organized by month (`diary/2026-01/diary_YYYY_MM_DD.md`)
- **AI/** — AI and coding methodology notes
- **c++/** — C++ technical notes (data synchronization, lock strategies)
- **java/** — Java learning notes
- **Software/** — Software design principles, including the Architecture Playbook
- **Linux核心/** — Linux kernel knowledge
- **leetcode/** — LeetCode problem solutions
- **Netwrok/** — Network/VM notes
- **上課心得/** — Course notes
- **個人體悟/** — Personal reflections
- **電影心得/** — Movie reviews

## Writing Conventions

- Articles are plain Markdown with no frontmatter
- Content is written in **Traditional Chinese** (繁體中文), except diary entries which are in **English**
- Diary filenames follow: `diary_YYYY_MM_DD.md`
- Technical articles use `#` for title, `##` for sections, with code blocks where appropriate

## Custom Skills

- `/architect [業務場景描述]` — Generates system architecture designs based on the Architecture Playbook (`Software/architect-playbook.md`). Outputs: system definition, object inventory (Business/Model/Data objects), communication design, threading model, and business flow.

## Architecture Playbook Key Concepts

The playbook (referenced by the architect skill) defines three object types:
1. **Business Objects** (Calculator, Handler, Manager, Controller) — business logic
2. **Model Objects** (Updater, DTO) — data read/write responsibility
3. **Data Objects** — anemic models, field definitions only

Objects communicate via Model Object interfaces with event callbacks. Multi-threading uses independent preparation threads with pointer-based atomic updates under write locks.

## Git Conventions

- Commit messages are in Chinese, concise (e.g., "更新技術文章內容", "加上AI學習")
- Diary commits use the date as message (e.g., "2026-02-24-diary")
