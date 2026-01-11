# Git Workflow Standards

Git branching strategy, commit conventions, and workflow standards.

## Branching Strategy

### Branch Types

| Branch | Purpose | Created From | Merges To |
|--------|---------|--------------|-----------|
| `main` | Production-ready code | - | - |
| `develop` | Integration branch (optional) | `main` | `main` |
| `feature/*` | New features | `main` or `develop` | `main` or `develop` |
| `bugfix/*` | Bug fixes | `main` or `develop` | `main` or `develop` |
| `hotfix/*` | Urgent production fixes | `main` | `main` |
| `release/*` | Release preparation | `develop` | `main` and `develop` |

### Branch Naming Convention

```
<type>/<ticket-id>-<short-description>
```

**Examples:**
```
feature/JIRA-123-add-order-api
feature/JIRA-456-implement-payment-gateway
bugfix/JIRA-789-fix-null-reference-in-validator
hotfix/JIRA-999-security-patch
release/v1.2.0
```

**Rules:**
- Use lowercase
- Use hyphens (not underscores)
- Include ticket ID when available
- Keep description short but meaningful
- No spaces or special characters

---

## Commit Messages

### Conventional Commits Format

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

### Commit Types

| Type | When to Use |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Code style (formatting, no logic change) |
| `refactor` | Code change that neither fixes nor adds |
| `perf` | Performance improvement |
| `test` | Adding or updating tests |
| `build` | Build system or dependencies |
| `ci` | CI/CD configuration |
| `chore` | Maintenance tasks |
| `revert` | Reverting a previous commit |

### Examples

```bash
# Feature
feat(orders): add order cancellation endpoint

# Bug fix
fix(auth): resolve token expiration validation

# Documentation
docs(readme): update installation instructions

# Refactoring
refactor(payments): extract payment processing to service

# Tests
test(orders): add unit tests for OrderService

# Build/Dependencies
build(deps): upgrade Serilog to v4.0.0

# CI/CD
ci(github): add code coverage reporting

# With body and footer
feat(api): implement pagination for orders endpoint

Add cursor-based pagination to improve performance for large datasets.
Includes backward-compatible query parameters.

Closes #123
BREAKING CHANGE: offset pagination removed
```

### Commit Message Rules

1. **Subject line**
   - Use imperative mood ("add" not "added" or "adds")
   - Don't capitalize first letter after type
   - No period at the end
   - Max 72 characters

2. **Body** (optional)
   - Separate from subject with blank line
   - Explain what and why, not how
   - Wrap at 72 characters

3. **Footer** (optional)
   - Reference issues: `Closes #123`, `Fixes #456`
   - Breaking changes: `BREAKING CHANGE: description`

---

## Pull Request Process

### Before Creating PR

```bash
# Ensure your branch is up to date
git fetch origin
git rebase origin/main  # or origin/develop

# Run all checks locally
dotnet build --warnaserrors
dotnet test
```

### PR Title Format

Follow commit message format:

```
feat(orders): add order cancellation endpoint
fix(auth): resolve token expiration validation
```

### PR Description Template

```markdown
## Summary
Brief description of the changes.

## Related Issues
Closes #123

## Type of Change
- [ ] Bug fix (non-breaking change fixing an issue)
- [ ] New feature (non-breaking change adding functionality)
- [ ] Breaking change (fix or feature causing existing functionality to change)
- [ ] Documentation update
- [ ] Refactoring (no functional changes)

## Changes Made
- Added X
- Updated Y
- Removed Z

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed

## Checklist
- [ ] Code follows project style guidelines
- [ ] Self-review completed
- [ ] Documentation updated (if applicable)
- [ ] No new warnings introduced
- [ ] Tests pass locally
- [ ] Coverage threshold met (80%)

## Screenshots (if applicable)
```

### PR Review Process

1. **Author**
   - Create PR with complete description
   - Assign reviewers
   - Link related issues
   - Ensure CI passes

2. **Reviewer**
   - Review within 24 hours
   - Use [PR Review Checklist](../checklists/pr-review-checklist.md)
   - Leave constructive comments
   - Approve or request changes

3. **Merge**
   - Squash and merge to keep history clean
   - Delete branch after merge
   - Verify deployment (if applicable)

---

## Git Commands Reference

### Daily Workflow

```bash
# Start new feature
git checkout main
git pull origin main
git checkout -b feature/JIRA-123-new-feature

# Make changes and commit
git add .
git commit -m "feat(module): description"

# Push to remote
git push -u origin feature/JIRA-123-new-feature

# Keep branch updated
git fetch origin
git rebase origin/main

# After PR approval, squash merge via GitHub/Azure DevOps
```

### Fixing Mistakes

```bash
# Amend last commit (before push)
git add .
git commit --amend -m "feat(module): corrected description"

# Undo last commit (keep changes)
git reset --soft HEAD~1

# Undo last commit (discard changes)
git reset --hard HEAD~1

# Revert a pushed commit
git revert <commit-hash>
git push
```

### Rebasing

```bash
# Rebase onto main
git fetch origin
git rebase origin/main

# Interactive rebase to squash commits
git rebase -i HEAD~3

# If conflicts occur
git status                    # See conflicts
# Fix conflicts in files
git add <fixed-files>
git rebase --continue

# Abort rebase if needed
git rebase --abort
```

### Stashing

```bash
# Stash current changes
git stash

# Stash with message
git stash save "WIP: working on feature X"

# List stashes
git stash list

# Apply most recent stash
git stash pop

# Apply specific stash
git stash apply stash@{2}

# Drop a stash
git stash drop stash@{0}
```

---

## Git Configuration

### Recommended Global Config

```bash
# User identity
git config --global user.name "Your Name"
git config --global user.email "your.email@{domain}"

# Default branch name
git config --global init.defaultBranch main

# Pull strategy
git config --global pull.rebase true

# Push default
git config --global push.default current

# Auto-correct typos
git config --global help.autocorrect 10

# Better diff
git config --global diff.algorithm histogram

# Useful aliases
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --decorate"
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "reset HEAD --"
```

### .gitignore Essentials

```gitignore
# Build outputs
bin/
obj/
publish/

# IDE
.vs/
.vscode/
.idea/
*.user
*.suo

# Test results
TestResults/
coverage/

# Secrets (IMPORTANT!)
appsettings.*.json
!appsettings.json
!appsettings.Development.json.template
*.pfx
*.p12
secrets.json
.env
.env.*

# OS files
.DS_Store
Thumbs.db

# Logs
logs/
*.log

# NuGet
packages/
*.nupkg
```

---

## Branch Protection Rules

### Recommended Settings for `main`

- ✅ Require pull request reviews (1+ approval)
- ✅ Dismiss stale reviews when new commits pushed
- ✅ Require review from code owners
- ✅ Require status checks to pass (build, test)
- ✅ Require branches to be up to date
- ✅ Require linear history (squash merges)
- ✅ Do not allow force pushes
- ✅ Do not allow deletions

### GitHub Branch Protection

```yaml
# .github/settings.yml (if using probot/settings)
branches:
  - name: main
    protection:
      required_pull_request_reviews:
        required_approving_review_count: 1
        dismiss_stale_reviews: true
      required_status_checks:
        strict: true
        contexts:
          - build
          - test
      enforce_admins: false
      restrictions: null
```

---

## Release Process

### Semantic Versioning

```
MAJOR.MINOR.PATCH

MAJOR: Breaking changes
MINOR: New features (backward compatible)
PATCH: Bug fixes (backward compatible)

Examples:
1.0.0 → 1.0.1 (bug fix)
1.0.1 → 1.1.0 (new feature)
1.1.0 → 2.0.0 (breaking change)
```

### Creating a Release

```bash
# Create release branch
git checkout main
git pull
git checkout -b release/v1.2.0

# Update version numbers, changelog
# ... make changes ...
git commit -m "chore(release): prepare v1.2.0"

# Create PR to main
# After merge, tag the release
git checkout main
git pull
git tag -a v1.2.0 -m "Release v1.2.0"
git push origin v1.2.0
```

### Changelog Format

```markdown
# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

## [1.2.0] - 2026-01-10

### Added
- Order cancellation endpoint (#123)
- Email notifications for order status

### Changed
- Improved pagination performance

### Fixed
- Null reference in order validator (#456)

### Security
- Updated dependencies to address CVE-2026-1234

## [1.1.0] - 2025-12-01

### Added
- Initial release
```

---

## Git Hooks (Optional)

### Pre-commit Hook

```bash
#!/bin/sh
# .git/hooks/pre-commit

# Run formatter
dotnet format --verify-no-changes
if [ $? -ne 0 ]; then
    echo "❌ Code formatting issues found. Run 'dotnet format' to fix."
    exit 1
fi

# Check for secrets
if git diff --cached --name-only | xargs grep -l "password\|secret\|api.key" 2>/dev/null; then
    echo "❌ Potential secrets detected in staged files"
    exit 1
fi

echo "✅ Pre-commit checks passed"
```

### Commit-msg Hook

```bash
#!/bin/sh
# .git/hooks/commit-msg

commit_msg=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?: .{1,72}$"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "❌ Commit message does not follow conventional commits format"
    echo "Expected: <type>(<scope>): <description>"
    echo "Example: feat(orders): add cancellation endpoint"
    exit 1
fi

echo "✅ Commit message format valid"
```

---

## Summary

| Action | Command/Format |
|--------|---------------|
| Branch name | `feature/JIRA-123-description` |
| Commit message | `feat(scope): description` |
| PR title | `feat(scope): description` |
| Merge strategy | Squash and merge |
| Version format | `v1.2.3` (semver) |
