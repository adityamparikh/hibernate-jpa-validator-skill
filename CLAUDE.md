# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code skill for Hibernate/JPA code review, inspired by Vlad Mihalcea's Hypersistence Optimizer. There is no build system — this is a pure Markdown skill, not a compiled project.

## Repository Structure

- **SKILL.md** — The main skill entry point. Contains the routing logic (Sections A–K) and inline code examples. This is what Claude loads when the skill activates.
- **references/** — Deep-dive reference documents with before/after Java code patterns. SKILL.md cross-references these by path (e.g., `references/identifier-strategies.md`).

SKILL.md and the reference files form a two-tier system: SKILL.md has the checklist and quick rules; reference files have the full explanations and edge cases.

## How the Skill Works

Section A in SKILL.md routes the user's request to specific sections (B–K). Section B (Entity Mapping Validation) is special — it runs its full checklist on every entity review, regardless of what the user asked about. This mirrors Hypersistence Optimizer's philosophy of catching issues the user didn't think to ask about.

## Editing Guidelines

- All code examples should show **before (wrong) and after (correct)** patterns with the generated SQL where relevant.
- SKILL.md sections should be self-contained enough for quick answers, with `→ See references/...` pointers for deep dives.
- Reference files are standalone — each covers one topic completely with its own code examples.
- Target stack: Hibernate 6.x, Spring Boot 3.x, Spring Data JPA 3.x, Java 17+, PostgreSQL. Note Hibernate 5 / Spring Boot 2.x differences where behavior diverges.
