<p align="center">
  <img src="https://img.shields.io/badge/Claude_Code-Skills-7c3aed?style=for-the-badge&labelColor=1e1e2e" alt="Claude Code Skills" />
  <img src="https://img.shields.io/badge/License-MIT-22c55e?style=for-the-badge" alt="MIT License" />
  <a href="https://github.com/Roly67/cc-skills/actions/workflows/ci.yml"><img src="https://img.shields.io/github/actions/workflow/status/Roly67/cc-skills/ci.yml?style=for-the-badge&label=CI" alt="CI Status" /></a>
</p>

<h1 align="center">
  <code>cc-skills</code>
</h1>

<p align="center">
  <strong>A curated collection of production-grade skills for Claude Code</strong>
</p>

<p align="center">
  <a href="#installation">Install</a> •
  <a href="#available-skills">Skills</a> •
  <a href="#usage">Usage</a> •
  <a href="#contributing">Contributing</a> •
  <a href="#license">License</a>
</p>

---

## What are Claude Code Skills?

Skills extend Claude Code's capabilities with domain-specific knowledge, templates, and best practices. When activated, skills provide Claude with contextual guidance tailored to specific technologies, frameworks, or workflows.

```
┌─────────────────────────────────────────────────────────┐
│  Claude Code  +  Skill  =  Expert-Level Assistance      │
└─────────────────────────────────────────────────────────┘
```

<br>

## Installation

### Quick Install

Add this marketplace to Claude Code with a single command:

```bash
claude /install-plugin github:roly67/cc-skills
```

### Manual Installation

<details>
<summary>Click to expand manual installation steps</summary>

<br>

**Option 1: Add as marketplace**

```bash
# Clone the repository
git clone https://github.com/Roly67/cc-skills.git

# Add to Claude Code
claude /install-plugin ./cc-skills
```

**Option 2: Configure in settings**

Add to your Claude Code settings file (`~/.claude/settings.json`):

```json
{
  "plugins": [
    "github:roly67/cc-skills"
  ]
}
```

**Option 3: Project-level configuration**

Add to your project's `.claude/settings.json`:

```json
{
  "plugins": [
    "github:roly67/cc-skills"
  ]
}
```

</details>

### Verify Installation

```bash
# List installed plugins
claude /plugins

# Check skill availability
claude /skills
```

<br>

## Available Skills

| Skill | Description | Version |
|:------|:------------|:-------:|
| **[dotnet-coding-standards](./dotnet-coding-standards)** | Enterprise .NET development standards, patterns & templates | `2.3.0` |

<br>

## Usage

### Installing a Skill

Skills are loaded automatically by Claude Code when working in repositories that match skill triggers. To manually reference a skill:

```bash
# Point Claude Code to a skill directory
claude --skill ./path/to/skill
```

### Skill Activation

Each skill defines triggers in its `SKILL.md` frontmatter:

```yaml
triggers:
  - "creating .NET solution"
  - "writing C# code"
  - "setting up Serilog"
```

When Claude detects these contexts, it applies the relevant skill's guidance.

<br>

## Repository Structure

```
cc-skills/
├── README.md                    # You are here
├── CLAUDE.md                    # Claude Code guidance for this repo
│
└── dotnet-coding-standards/     # ┐
    ├── SKILL.md                 # │ Skill entry point
    ├── standards/               # │ Technical standards
    ├── project-types/           # │ Project templates
    ├── checklists/              # │ Actionable guides
    ├── templates/               # │ Config files
    └── snippets/                # ┘ IDE snippets
```

<br>

## Contributing

### Adding a New Skill

<table>
<tr>
<td width="60">

**1**

</td>
<td>

Create a new directory with your skill name

</td>
</tr>
<tr>
<td>

**2**

</td>
<td>

Add a `SKILL.md` with YAML frontmatter defining metadata and triggers

</td>
</tr>
<tr>
<td>

**3**

</td>
<td>

Organize supporting docs in logical subdirectories

</td>
</tr>
<tr>
<td>

**4**

</td>
<td>

Include templates, examples, and checklists as needed

</td>
</tr>
</table>

### Skill Frontmatter Schema

```yaml
---
skill: my-skill-name
version: 1.0.0
description: Brief description of what this skill provides
globs:
  - "*.ext"           # File patterns this skill applies to
triggers:
  - "keyword phrase"  # Natural language triggers
---
```

### Quality Standards

> **Skills should be opinionated, practical, and production-tested.**

- ✓ Provide clear, actionable guidance
- ✓ Include real-world examples and templates
- ✓ Document the "why" behind decisions
- ✓ Keep content current and maintained

For detailed contribution guidelines, see [CONTRIBUTING.md](CONTRIBUTING.md).

<br>

## License

This project is licensed under the [MIT License](LICENSE).

---

<p align="center">
  <sub>Built for <a href="https://claude.ai/code">Claude Code</a></sub>
</p>
