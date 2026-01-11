# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a **Claude Code Skills repository** containing reusable skills for Claude Code. Currently includes:

- **dotnet-coding-standards**: Comprehensive .NET development standards and best practices skill

## Structure

```
cc-skills/
└── dotnet-coding-standards/     # .NET coding standards skill
    ├── SKILL.md                 # Skill entry point and quick reference
    ├── standards/               # 16 technical standards documents
    ├── project-types/           # 5 project-type guides (Web API, Console, Worker, Library, Blazor)
    ├── checklists/              # New solution and PR review checklists
    ├── templates/               # Copy-paste ready config files
    └── snippets/                # IDE snippets for VS Code, Visual Studio, Rider
```

## Adding New Skills

Each skill should be a self-contained directory with:
1. `SKILL.md` - Entry point with YAML frontmatter defining metadata and triggers
2. Supporting documentation organized in subdirectories
3. Templates and configuration files as needed

## Key Files

- `dotnet-coding-standards/SKILL.md` - Main skill definition with version info, quick reference tables, and changelog
- `dotnet-coding-standards/templates/Directory.Build.props` - Centralized MSBuild configuration template
- `dotnet-coding-standards/checklists/new-solution-checklist.md` - Step-by-step solution setup guide

## Skill Metadata Format

Skills use YAML frontmatter in SKILL.md:
```yaml
---
skill: skill-name
version: X.Y.Z
description: Short description
globs:
  - "*.cs"
  - "*.csproj"
triggers:
  - "keyword phrase"
---
```
