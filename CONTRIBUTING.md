# Contributing to cc-skills

Thank you for your interest in contributing to cc-skills! This guide will help you get started.

## Ways to Contribute

- **Add a new skill** — Share your domain expertise
- **Improve existing skills** — Fix errors, add examples, update outdated content
- **Report issues** — Found a bug or have a suggestion? Open an issue
- **Documentation** — Improve README files, add examples, fix typos

## Adding a New Skill

### 1. Create the Skill Directory

```
cc-skills/
└── your-skill-name/
    ├── SKILL.md           # Required: Entry point
    ├── README.md          # Required: Human-readable docs
    ├── LICENSE            # Required: MIT recommended
    └── ...                # Additional files as needed
```

### 2. Define SKILL.md

Every skill must have a `SKILL.md` with YAML frontmatter:

```yaml
---
name: your-skill-name
version: 1.0.0
last-updated: 2026-01-11
description: >
  Brief description of when to use this skill. Include trigger phrases
  that help Claude understand when to apply this skill.
---

# Your Skill Name

Content starts here...
```

### 3. Organize Supporting Content

Structure your skill logically:

```
your-skill-name/
├── SKILL.md              # Quick reference, decision trees, common patterns
├── README.md             # Overview, installation, what's included
├── LICENSE               # MIT license
│
├── standards/            # Technical standards and guidelines
│   ├── topic-one.md
│   └── topic-two.md
│
├── templates/            # Copy-paste ready files
│   └── config-file.json
│
└── checklists/           # Step-by-step guides
    └── setup-checklist.md
```

### 4. Update the Marketplace

Add your skill to `.claude-plugin/marketplace.json`:

```json
{
  "name": "your-skill-name",
  "description": "Brief description of your skill",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  },
  "source": "./your-skill-name",
  "category": "development",
  "tags": ["relevant", "tags", "here"]
}
```

### 5. Update Root README

Add your skill to the Available Skills table in the root `README.md`.

## Skill Quality Standards

### Content Guidelines

- **Be opinionated** — Take a clear stance on best practices
- **Be practical** — Include real-world examples and templates
- **Be current** — Keep content up to date with latest versions
- **Be concise** — Avoid unnecessary verbosity

### Structure Guidelines

- Use clear headings and hierarchy
- Include decision trees for complex choices
- Provide copy-paste ready code/config snippets
- Add checklists for multi-step processes

### What to Avoid

- Generic advice that applies to any project
- Outdated patterns or deprecated APIs
- Overly academic content without practical application
- Content that duplicates official documentation

## Pull Request Process

### Before Submitting

1. **Test your skill** — Ensure all examples and templates work
2. **Check formatting** — Consistent markdown formatting
3. **Update version** — Bump version in SKILL.md if updating existing skill
4. **Update changelog** — Document changes in SKILL.md

### PR Checklist

- [ ] SKILL.md has valid YAML frontmatter
- [ ] README.md exists with installation instructions
- [ ] LICENSE file included (MIT)
- [ ] marketplace.json updated (for new skills)
- [ ] Root README.md updated (for new skills)
- [ ] No broken links
- [ ] Examples are tested and working

### Commit Messages

Use clear, descriptive commit messages:

```
Add python-best-practices skill

- Add core standards for Python development
- Include FastAPI project template
- Add testing guidelines with pytest
```

## Updating Existing Skills

When updating an existing skill:

1. **Bump the version** in SKILL.md frontmatter
2. **Update last-updated** date
3. **Add changelog entry** documenting the changes
4. **Test affected examples** to ensure they still work

### Versioning

Follow semantic versioning:

- **Major (x.0.0)** — Breaking changes, major restructuring
- **Minor (0.x.0)** — New content, new standards, new templates
- **Patch (0.0.x)** — Fixes, typos, minor clarifications

## Code of Conduct

- Be respectful and constructive
- Focus on the content, not the person
- Welcome newcomers and help them contribute

## Questions?

Open an issue with the `question` label and we'll help you out.
