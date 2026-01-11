# Security Policy

## Scope

This repository contains documentation, templates, and configuration files for Claude Code skills. It does not contain executable code that runs in production environments.

However, the templates and configuration examples provided may be used in production systems, so we take security seriously.

## Reporting a Vulnerability

If you discover a security issue, please report it responsibly.

### What to Report

- Security issues in configuration templates (e.g., insecure defaults)
- Sensitive data accidentally committed (API keys, credentials, etc.)
- Vulnerabilities in recommended patterns or practices
- Issues with GitHub Actions workflows

### How to Report

**For sensitive issues:**

Email [rrat@duck.com](mailto:rrat@duck.com) with:

- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if any)

**For non-sensitive issues:**

Open a [GitHub issue](https://github.com/Roly67/cc-skills/issues/new) with the `security` label.

### Response Timeline

- **Acknowledgment**: Within 48 hours
- **Initial assessment**: Within 7 days
- **Resolution**: Depends on severity and complexity

## Security Best Practices

When using skills from this repository:

1. **Review before use** — Always review templates and configurations before applying them
2. **Update placeholders** — Replace all placeholder values (e.g., `{CompanyName}`, `{email}`)
3. **Check for secrets** — Never commit actual secrets, even in config templates
4. **Keep updated** — Watch this repository for security-related updates

## Supported Versions

| Version | Supported          |
|---------|-------------------|
| 2.x.x   | :white_check_mark: |
| < 2.0   | :x:                |
