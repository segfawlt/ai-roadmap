# Git Collaboration Workflow

## Introduction

Effective Git collaboration workflows are essential for modern software development teams. A well-defined workflow ensures that team members can work in parallel without conflicts, maintain code quality through peer review, and deploy changes safely and efficiently.

This guide covers fundamental Git concepts, popular branching strategies, and best practices for collaboration. By the end, you’ll understand how to choose and implement the right workflow for your team’s needs, whether you’re working on a small project or a large-scale cloud-native application.

**Why Git Workflows Matter:**

- **Parallel Development**: Multiple developers can work simultaneously without interfering with each other

- **Code Quality**: Structured review processes catch bugs before they reach production

- **Release Management**: Clear processes for deploying tested code to production

- **Traceability**: Every change is tracked, documented, and reversible

------------------------------------------------------------------------

## Git Fundamentals

### Repository & Commits

**Repository (Repo)**: A repository is a storage location for your project that contains all files, history, and branches. It can exist locally on your machine or remotely on platforms like GitHub, GitLab, or Bitbucket.

**Commits**: A commit is a snapshot of your project at a specific point in time. Each commit contains:

- Changes made to files

- Author information

- Timestamp

- Commit message describing the changes

- Unique SHA-1 hash identifier

**Staging Area**: Git uses a staging area (also called “index”) as an intermediate step between your working directory and the repository. This allows you to selectively choose which changes to include in your next commit.

**Basic Workflow:**

```
Working Directory → Staging Area → Repository
     (edit)       →  (git add)   → (git commit)
```

**Key Commands:**

``` bash
# Initialize a new repository
git init

# Check status of files
git status

# Add files to staging area
git add filename.txt
git add .  # Add all changes

# Commit staged changes
git commit -m "Descriptive commit message"

# View commit history
git log
git log --oneline --graph --all
```

### Branches

**What is a Branch?**: A branch is an independent line of development. It’s a pointer to a specific commit that moves forward as you create new commits. Branches allow you to work on features, fixes, or experiments without affecting the main codebase.

**Why Use Branches?**

- Isolate feature development

- Test experimental ideas safely

- Enable parallel workflows

- Facilitate code review before merging

**Default Branch**: Traditionally named `master`, now commonly `main`. This is typically your production-ready code.

**Key Commands:**

``` bash
# List all branches
git branch

# Create a new branch
git branch feature-name

# Switch to a branch
git checkout feature-name
# Or using newer syntax:
git switch feature-name

# Create and switch in one command
git checkout -b feature-name
git switch -c feature-name

# Merge a branch into current branch
git merge feature-name
# Use --no-ff to preserve branch history (recommended for feature branches)
git merge --no-ff feature-name

# Rebase current branch onto another (clean up history before PR)
git rebase main

# Delete a branch
git branch -d feature-name
```

------------------------------------------------------------------------

## Security Best Practices

### .gitignore Essentials

Prevent sensitive files from being committed:

```
# Environment and secrets
.env
.env.local
.env.*.local
*.pem
*.key
secrets/
config/local.yaml

# Credentials
credentials.json
service-account.json
.aws/
.gcloud/

# IDE and OS
.vscode/
.idea/
.DS_Store
Thumbs.db

# Dependencies
node_modules/
venv/
__pycache__/
```

If you accidentally commit a secret, assume it’s compromised. Rotate the credential immediately, then use `git`` ``filter-branch` or [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/) to remove it from history.

### Pre-commit Secrets Detection

Use tools to catch secrets before they’re committed:

``` bash
# Install detect-secrets
pip install detect-secrets

# Create baseline (ignore existing findings)
detect-secrets scan > .secrets.baseline

# Add to pre-commit hooks
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

Alternatives: `gitleaks`, `trufflehog`, `git-secrets`

------------------------------------------------------------------------

## Branch Protection & Governance

### Protected Branches

Configure branch protection rules (GitHub/GitLab):

| Rule                       | Purpose                             |
|----------------------------|-------------------------------------|
| **Require PR reviews**     | At least 1-2 approvals before merge |
| **Require status checks**  | CI must pass before merge           |
| **Require signed commits** | Verify commit authenticity          |
| **Restrict force push**    | Prevent history rewriting on main   |
| **Require linear history** | No merge commits (rebase only)      |

### CODEOWNERS

Automatic reviewer assignment based on file paths:

```
# .github/CODEOWNERS (or .gitlab/CODEOWNERS)

# Default owners for everything
*       @team-leads

# Frontend owners
/src/frontend/    @frontend-team
*.tsx             @frontend-team

# Backend owners
/src/api/         @backend-team
*.py              @backend-team

# DevOps/Infra
.github/          @devops-team
Dockerfile        @devops-team
*.yml             @devops-team
```

### Draft Pull Requests

Use draft PRs to:

- Get early feedback on WIP code

- Run CI without enabling merge

- Signal “not ready for review yet”

``` bash
# Create a draft PR via GitHub CLI
gh pr create --draft --title "WIP: Add user authentication"
```

------------------------------------------------------------------------

## Branching Strategies

### Gitflow Workflow

Gitflow is a robust branching model designed around project releases. It defines a strict branching structure with specific roles for different branch types.

**Branch Types in Gitflow:**

1.  **Main Branches:**

    - `main` (or `master`): Contains production-ready code. Every commit here represents a production release.

    - `develop`: Integration branch for features. Contains the latest development changes for the next release.

2.  **Supporting Branches:**

    - **Feature branches** (`feature/*`): Branch from `develop`, merge back to `develop`

      - Used for developing new features

      - Naming: `feature/user-authentication`, `feature/payment-integration`

    - **Release branches** (`release/*`): Branch from `develop`, merge to `main` and `develop`

      - Prepare for a new production release

      - Allow for minor bug fixes and metadata preparation

      - Naming: `release/1.2.0`

    - **Hotfix branches** (`hotfix/*`): Branch from `main`, merge to `main` and `develop`

      - Quick fixes for production issues

      - Naming: `hotfix/critical-security-patch`

**Gitflow Workflow Steps:**

```
Feature Development:
develop → feature/new-feature → develop

Release Preparation:
develop → release/1.0.0 → main + develop (tagged v1.0.0)

Hotfix:
main → hotfix/critical-bug → main + develop (tagged v1.0.1)
```

**Pros:**

- Clear structure for releases

- Parallel development of features and releases

- Good for scheduled release cycles

**Cons:**

- Complex for small teams or continuous deployment

- Multiple long-lived branches can lead to merge conflicts

- May slow down deployment velocity

### Feature Branch Workflow

A simpler alternative to Gitflow, the Feature Branch Workflow is based on a single main branch with feature branches for all new work.

**Core Principles:**

- `main` branch always contains production-ready code

- All new work happens in dedicated feature branches

- Feature branches are merged back to `main` via pull requests

- Continuous integration tests run on every branch

**Typical Flow:**

``` bash
# 1. Create feature branch from main
git checkout main
git pull origin main
git checkout -b feature/add-login

# 2. Work on feature, commit regularly
git add .
git commit -m "Add login form UI"
git commit -m "Implement authentication logic"

# 3. Push feature branch
git push origin feature/add-login

# 4. Create Pull Request (on GitHub/GitLab)
# 5. Code review and CI checks
# 6. Merge to main after approval
# 7. Delete feature branch
```

**Best Practices:**

- Keep feature branches short-lived (ideally \< 1 week)

- Sync regularly with `main` to avoid conflicts

- One feature = one branch

- Use descriptive branch names

**Trunk-Based Development** (Advanced Variation):

For teams practicing Continuous Delivery, trunk-based development takes this further:

- Developers merge to `main` (trunk) daily

- Feature branches are very short-lived (\< 1 day)

- Every merge to `main` triggers automated deployment to dev/QA

- Production releases are explicit actions with semantic versioning

**Key Consideration**: Trunk-based development requires:

- Robust automated testing (unit, integration, full pipeline tests)

- Feature flags for incomplete features

- Strong CI/CD infrastructure

- Team discipline and maturity

------------------------------------------------------------------------

## Pull Requests

### Creating Effective PRs

A Pull Request (PR) is a mechanism for proposing changes to a codebase. It facilitates code review, discussion, and quality control before merging.

**Anatomy of a Good Pull Request:**

1.  **Descriptive Title**: Summarize the change in one line

    - Good: “Add user authentication with JWT tokens”

    - Bad: “Update code”

2.  **Clear Description**: Explain what, why, and how

    ``` markdown
    ## What

    Implements JWT-based authentication for API endpoints

    ## Why

    Needed to secure user data and implement role-based access

    ## How

    - Added JWT middleware for Express
    - Created login/logout endpoints
    - Implemented token refresh mechanism

    ## Testing

    - Added unit tests for auth functions
    - Tested manually with Postman

    ## Screenshots

    [If applicable]
    ```

3.  **Small, Focused Changes**:

    - Aim for \< 400 lines changed

    - One logical change per PR

    - If larger, consider breaking into multiple PRs

4.  **Reference Issues**: Link to related issues or tickets

    - “Closes \#123”

    - “Related to \#456”

5.  **Self-Review**: Review your own PR first

    - Check for debug code, console.logs

    - Ensure tests pass

    - Verify formatting and style

**PR Checklist Before Submitting:**

``` markdown
- [ ] Code follows project style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Tests added/updated and passing
- [ ] Documentation updated
- [ ] No merge conflicts with target branch
- [ ] CI/CD pipeline passing
```

### Code Review Process

Code review is a critical quality gate that improves code quality, shares knowledge, and catches bugs early.

**Reviewer Responsibilities:**

1.  **Understand the Context**: Read the PR description and related issues

2.  **Check Functionality**: Does the code do what it claims?

3.  **Review Code Quality**:

    - Readability and maintainability

    - Adherence to coding standards

    - Proper error handling

    - Security considerations

4.  **Test Coverage**: Are critical paths tested?

5.  **Performance**: Any obvious performance issues?

**Providing Constructive Feedback:**

- **Be Specific**:

  - Good: “This function could be simplified using array.map() instead of a for loop”

  - Bad: “This code is messy”

- **Be Kind**:

  - Use “we” language: “We could consider…”

  - Ask questions: “Have you considered…?”

  - Praise good work: “Nice solution to this edge case!”

- **Categorize Comments**:

  - **Blocking**: Must be fixed before merge

  - **Non-blocking**: Suggestions for improvement

  - **Question**: Need clarification

**Author Responsibilities:**

- **Respond to All Comments**: Even if just “Fixed” or “Good point”

- **Explain Decisions**: If you disagree, explain why politely

- **Keep Discussion on Topic**: Move lengthy debates to separate discussions

- **Update the PR**: Push fixes and re-request review

**Example Review Comment Types:**

```
🔴 Blocking: This will cause a null pointer exception if user is undefined

🟡 Suggestion: Consider extracting this into a separate function for readability

💭 Question: Why did we choose this approach over using the existing utility?

✅ Approval: LGTM! Great work on the error handling
```

**Merge Strategies:**

1.  **Merge Commit**: Preserves full history with a merge commit

2.  **Squash and Merge**: Combines all commits into one (cleaner history)

3.  **Rebase and Merge**: Replays commits on top of base branch (linear history)

Choose based on your team’s preference and project needs.

------------------------------------------------------------------------

## Conflict Resolution

### Understanding Merge Conflicts

Conflicts occur when Git can’t automatically merge changes (same lines edited by different branches).

**Conflict Markers:**

``` diff
<<<<<<< HEAD
Your changes (current branch)
=======
Their changes (incoming branch)
>>>>>>> feature-branch
```

### Resolution Strategies

``` bash
# 1. See which files have conflicts
git status

# 2. Open conflicted files and resolve manually
#    - Remove conflict markers
#    - Keep the correct code

# 3. Mark as resolved
git add resolved-file.py

# 4. Complete the merge
git commit
```

**Using ours/theirs:**

``` bash
# Keep your version for all conflicts
git checkout --ours path/to/file.py

# Keep their version for all conflicts
git checkout --theirs path/to/file.py

# Keep one side for entire merge
git merge -X ours feature-branch   # Prefer current branch
git merge -X theirs feature-branch # Prefer incoming branch
```

**GUI Tools for Complex Conflicts:**

``` bash
# Launch configured merge tool
git mergetool

# Popular tools: VS Code, IntelliJ, Meld, Beyond Compare
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
```

- Sync with main frequently (`git`` ``pull`` ``--rebase`` ``origin`` ``main`)

- Keep branches short-lived

- Communicate with team about overlapping work

------------------------------------------------------------------------

## Conventional Commits

Conventional Commits is a specification for writing standardized commit messages. It provides a lightweight convention that creates an explicit commit history, making it easier to write automated tools on top of.

### The Specification

A conventional commit message follows this structure:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

**Components:**

1.  **Type** (required): Describes the category of change

2.  **Scope** (optional): Describes what part of the codebase is affected

3.  **Description** (required): Short summary of the change

4.  **Body** (optional): Detailed explanation of the change

5.  **Footer** (optional): Metadata like breaking changes or issue references

### Common Commit Types

| Type | Description | Example |
|----|----|----|
| `feat` | A new feature | `feat:`` ``add`` ``user`` ``login`` ``functionality` |
| `fix` | A bug fix | `fix:`` ``resolve`` ``null`` ``pointer`` ``in`` ``auth`` ``module` |
| `docs` | Documentation changes | `docs:`` ``update`` ``API`` ``documentation` |
| `style` | Code style changes (formatting, semicolons) | `style:`` ``fix`` ``indentation`` ``in`` ``utils.js` |
| `refactor` | Code refactoring without changing functionality | `refactor:`` ``extract`` ``validation`` ``logic` |
| `perf` | Performance improvements | `perf:`` ``optimize`` ``database`` ``queries` |
| `test` | Adding or updating tests | `test:`` ``add`` ``unit`` ``tests`` ``for`` ``auth`` ``service` |
| `build` | Build system or external dependencies | `build:`` ``update`` ``webpack`` ``configuration` |
| `ci` | CI/CD configuration changes | `ci:`` ``add`` ``GitHub`` ``Actions`` ``workflow` |
| `chore` | Other changes (maintenance tasks) | `chore:`` ``update`` ``dependencies` |
| `revert` | Reverts a previous commit | `revert:`` ``revert`` ``feat`` ``adding`` ``login` |

### Examples

**Simple commit:**

```
feat: add email validation to registration form
```

**Commit with scope:**

```
fix(auth): resolve token expiration issue
```

**Commit with body:**

```
feat(api): add pagination to user list endpoint

Implemented cursor-based pagination to improve performance
when loading large user datasets. The page size defaults to
20 items but can be configured via query parameter.
```

**Breaking change:**

```
feat(api)!: change authentication to JWT tokens

BREAKING CHANGE: The API now requires JWT tokens instead of
session cookies. All clients must update their authentication
flow to request and include Bearer tokens.

Closes #456
```

**Multiple footers:**

```
fix(payment): correct currency conversion calculation

The previous implementation used outdated exchange rates.
Now fetches real-time rates from the currency API.

Fixes #789
Reviewed-by: John Doe
Co-authored-by: Jane Smith <[email protected]>
```

### Benefits

1.  **Automatic Changelog Generation**: Tools can parse conventional commits to generate release notes

2.  **Semantic Versioning**: Commit types determine version bumps:

    - `fix` → PATCH (1.0.0 → 1.0.1)

    - `feat` → MINOR (1.0.0 → 1.1.0)

    - `BREAKING`` ``CHANGE` → MAJOR (1.0.0 → 2.0.0)

3.  **Clear Communication**: Team members understand changes at a glance

4.  **Better Git History**: Structured commits make history easier to navigate

5.  **Automation Ready**: CI/CD pipelines can trigger actions based on commit types

### Tools and Enforcement

**Commitlint**: Lint commit messages to ensure they follow conventions

``` bash
# Install
npm install --save-dev @commitlint/cli @commitlint/config-conventional

# Create config file (commitlint.config.cjs)
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'body-empty': [2, 'always'],
    'footer-empty': [2, 'always'],
  },
};
```

This repository enforces conventional commits with empty body and footer sections.

**Husky**: Git hooks to run commitlint before commits

``` bash
# Install husky
npm install --save-dev husky

# Enable Git hooks
npx husky install

# Add commit-msg hook
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit "$1"'
```

**Commitizen**: Interactive CLI for writing conventional commits

``` bash
# Install globally
npm install -g commitizen cz-conventional-changelog

# Use instead of git commit
git cz
```

### Automated Versioning & Changelog

[semantic-release](https://semantic-release.gitbook.io/) automates version bumps and changelog generation based on Conventional Commits:

``` yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Semantic Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: npx semantic-release
```

**Configuration (`.releaserc.json`):**

``` json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    "@semantic-release/npm",
    "@semantic-release/github",
    [
      "@semantic-release/git",
      {
        "assets": ["CHANGELOG.md", "package.json"],
        "message": "chore(release): ${nextRelease.version}"
      }
    ]
  ]
}
```

This automatically:

- Determines next version from commit types

- Generates CHANGELOG.md

- Creates GitHub releases with notes

- Publishes to npm (if configured)

------------------------------------------------------------------------

## Git in 2026: What Has Changed

### Modern Commands

Git has introduced more intuitive alternatives to some classic commands:

| Classic Command | Modern Alternative | Why |
|----|----|----|
| `git`` ``checkout`` ``-b`` ``feature` | `git`` ``switch`` ``-c`` ``feature` | Clearer intent — `switch` is only for branches |
| `git`` ``checkout`` ``--`` ``file.txt` | `git`` ``restore`` ``file.txt` | `restore` is only for files, avoids ambiguity |
| `git`` ``checkout`` ``main` | `git`` ``switch`` ``main` | Separates branch switching from file operations |

``` bash
# Create and switch to a new branch
git switch -c feature/add-auth

# Discard changes in a file
git restore src/config.py

# Unstage a file
git restore --staged src/config.py
```

`git`` ``switch` and `git`` ``restore` were introduced in Git 2.23. Use them in new projects. The classic `git`` ``checkout` still works and you will see it in older documentation.

### Branching Strategy Decision Tree (2026)

| Team Size      | Deployment Frequency | Recommended Strategy                   |
|----------------|----------------------|----------------------------------------|
| 1-5 engineers  | Continuous (daily)   | **GitHub Flow** or **Trunk-Based**     |
| 5-15 engineers | Weekly/bi-weekly     | **GitHub Flow** with branch protection |
| 15+ engineers  | Scheduled releases   | **GitFlow** or **GitLab Flow**         |
| Monorepo       | Continuous           | **Trunk-Based** with affected-path CI  |

### AI-Assisted Development and Git

With AI tools generating more code faster (2025 DORA Report: 98% increase in PR volume), Git discipline is more important than ever:

- **Keep feature branches short-lived** (\< 1 week) — AI-generated PRs are larger on average, making long-lived branches even riskier

- **Smaller PRs get better reviews** — PRs under 400 lines have 75% defect detection rate vs 31% for PRs over 1,000 lines

- **Conventional commits enable automation** — tools like `semantic-release` auto-generate changelogs and version bumps from commit messages

- **Branch protection is non-negotiable** — require CI checks + at least 1 review before merge to main

### Conventional Commits Deep Dive

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

| Type | When to Use | Example |
|----|----|----|
| `feat` | New feature | `feat(auth):`` ``add`` ``JWT`` ``refresh`` ``token`` ``endpoint` |
| `fix` | Bug fix | `fix(api):`` ``handle`` ``null`` ``user_id`` ``in`` ``ticket`` ``query` |
| `docs` | Documentation only | `docs:`` ``update`` ``API`` ``security`` ``cheatsheet` |
| `refactor` | Code change that neither fixes a bug nor adds a feature | `refactor:`` ``extract`` ``validation`` ``into`` ``shared`` ``module` |
| `test` | Adding or fixing tests | `test:`` ``add`` ``integration`` ``tests`` ``for`` ``booking`` ``API` |
| `chore` | Maintenance (deps, CI, tooling) | `chore:`` ``upgrade`` ``FastAPI`` ``to`` ``0.115` |
| `perf` | Performance improvement | `perf(db):`` ``add`` ``composite`` ``index`` ``on`` ``orders(user_id,`` ``status)` |
| `ci` | CI/CD changes | `ci:`` ``add`` ``preview`` ``deploy`` ``for`` ``MR`` ``branches` |

**Breaking changes:** Add `!` after type or include `BREAKING`` ``CHANGE:` in footer:

```
feat(api)!: change pagination from offset to cursor-based
```

------------------------------------------------------------------------

## Summary

**Key Takeaways:**

1.  **Git Fundamentals**

    - Repository stores complete project history

    - Commits are snapshots; stage changes before committing

    - Branches enable parallel development

    - Use modern commands (`git`` ``switch`, `git`` ``restore`) in new projects

2.  **Branching Strategies**

    - **Gitflow**: Structured approach with main, develop, feature, release, and hotfix branches. Best for scheduled releases.

    - **Feature Branch Workflow**: Simpler model with main branch and feature branches. Better for continuous delivery.

    - **Trunk-Based Development**: Advanced approach requiring mature CI/CD and testing practices.

    - Choose based on team size and deployment frequency

3.  **Pull Requests**

    - Essential for code review and quality control

    - Should be small (\< 400 lines), focused, and well-documented

    - Code review improves quality and shares knowledge

4.  **Choosing a Workflow**

    - **Use Gitflow if**: You have scheduled releases, need to support multiple production versions, or have a complex release process

    - **Use Feature Branch Workflow if**: You deploy continuously, want simplicity, or have a small to medium team

    - **Use Trunk-Based if**: You have strong CI/CD, mature testing practices, and can deploy multiple times per day

**Best Practices Across All Workflows:**

- Commit early and often with meaningful messages (conventional commits format)

- Keep branches short-lived (\< 1 week) — especially important with AI-generated code

- Use pull requests for all code changes

- Automate testing and deployment

- Document your chosen workflow for the team

- Be consistent across the team

------------------------------------------------------------------------

## References

1.  [Atlassian Git Tutorial - Comparing Workflows](https://www.atlassian.com/git/tutorials/comparing-workflows)

2.  [DORA - Trunk-Based Development](https://dora.dev/capabilities/trunk-based-development/)

3.  [Minimum CD](https://minimumcd.org/)

4.  [Git-flow Cheatsheet](https://danielkummer.github.io/git-flow-cheatsheet/)

5.  [GitHub Flow](https://docs.github.com/en/get-started/using-github/github-flow)

6.  [Conventional Commits Specification](https://www.conventionalcommits.org/)

7.  [Pro Git - Official Documentation](https://git-scm.com/book/en/v2)

8.  [GitHub Skills - Interactive Courses](https://skills.github.com/)

9.  [Semantic Release](https://semantic-release.gitbook.io/)

10. [detect-secrets - Secrets Detection](https://github.com/Yelp/detect-secrets)

## Practice

This practice guide contains hands-on exercises to reinforce your understanding of Git workflows. Complete each exercise in order to build your skills progressively.

- Git installed on your system

- Basic familiarity with command line

- A text editor of your choice

------------------------------------------------------------------------

### Exercise 1: Git Fundamentals

**Objective**: Initialize a Git repository, make commits, and understand the staging area.

**Skills Practiced**:

- Repository initialization

- Staging and committing changes

- Viewing history and differences

#### Steps

``` bash
# 1. Create a new project directory
mkdir my-git-project
cd my-git-project

# 2. Initialize Git repository
git init

# 3. Configure user information (if not done globally)
git config user.name "Your Name"
git config user.email "[email protected]"

# 4. Create a README file
echo "# My Git Project" > README.md

# 5. Check status - observe untracked files
git status

# 6. Add file to staging area
git add README.md

# 7. Check status again - observe staged files
git status

# 8. Commit the changes
git commit -m "Initial commit: Add README"

# 9. Create additional files
echo "print('Hello, World!')" > app.py
echo "Flask==2.0.1" > requirements.txt

# 10. Add all files at once
git add .

# 11. Commit with a descriptive message
git commit -m "feat: Add Python application and dependencies"

# 12. View commit history
git log --oneline

# 13. Make a change to README
echo "\nThis is a sample project for learning Git." >> README.md

# 14. View differences before staging
git diff

# 15. Stage and commit
git add README.md
git commit -m "docs: Update README with project description"

# 16. View complete history with graph
git log --oneline --graph
```

#### Expected Output

```
* abc1234 docs: Update README with project description
* def5678 feat: Add Python application and dependencies
* ghi9012 Initial commit: Add README
```

#### Verification Checklist

Repository initialized successfully

Three commits visible in history

Commit messages follow descriptive format

`git`` ``status` shows clean working tree

------------------------------------------------------------------------

### Exercise 2: Branching and Merging

**Objective**: Create branches, make changes, and merge them back.

**Skills Practiced**:

- Branch creation and switching

- Making isolated changes

- Merging branches

#### Steps

``` bash
# 1. Create and switch to a new branch
git checkout -b feature/add-config

# 2. Create a configuration file
cat > config.py << 'EOF'
DATABASE_URL = "postgresql://localhost:5432/mydb"
DEBUG = True
SECRET_KEY = "your-secret-key"
EOF

git add config.py
git commit -m "feat: Add application configuration"

# 3. Add more configuration
echo 'LOG_LEVEL = "INFO"' >> config.py
git add config.py
git commit -m "feat: Add logging configuration"

# 4. View branch status
git log --oneline --graph --all

# 5. Switch back to main branch
git checkout main

# 6. Observe that config.py doesn't exist on main
ls -la

# 7. Merge the feature branch
git merge feature/add-config -m "Merge feature/add-config into main"

# 8. Verify config.py now exists
cat config.py

# 9. Delete the feature branch (optional)
git branch -d feature/add-config

# 10. View final history
git log --oneline --graph
```

#### Expected Output

```
*   abc1234 Merge feature/add-config into main
|\
| * def5678 feat: Add logging configuration
| * ghi9012 feat: Add application configuration
|/
* jkl3456 docs: Update README with project description
* mno7890 feat: Add Python application and dependencies
* pqr1234 Initial commit: Add README
```

------------------------------------------------------------------------

### Exercise 3: Gitflow Workflow

**Objective**: Practice the complete Gitflow workflow including feature development, release preparation, and hotfixes.

**Skills Practiced**:

- Managing multiple long-lived branches

- Feature branch workflow

- Release and hotfix processes

- Tagging releases

#### Steps

``` bash
# === SETUP ===
# 1. Initialize a new repository
git init gitflow-demo
cd gitflow-demo

# 2. Create initial commit on main
echo "# My Application" > README.md
git add README.md
git commit -m "Initial commit"

# 3. Create develop branch
git checkout -b develop
echo "v0.1.0-dev" > VERSION
git add VERSION
git commit -m "chore: Initialize develop branch"

# === FEATURE DEVELOPMENT ===
# 4. Create feature branch from develop
git checkout -b feature/user-authentication develop

# 5. Work on feature - create auth module
mkdir -p src
cat > src/auth.py << 'EOF'
def login(username, password):
    """Authenticate user with credentials."""
    pass

def logout():
    """End user session."""
    pass
EOF

git add src/auth.py
git commit -m "feat(auth): Add login and logout functions"

# 6. Add password validation
cat >> src/auth.py << 'EOF'

def validate_password(password):
    """Validate password meets requirements."""
    return len(password) >= 8
EOF

git add src/auth.py
git commit -m "feat(auth): Add password validation"

# 7. Merge feature back to develop
git checkout develop
git merge --no-ff feature/user-authentication -m "Merge feature/user-authentication"
git branch -d feature/user-authentication

# 8. Create another feature
git checkout -b feature/dashboard develop

cat > src/dashboard.py << 'EOF'
def render_dashboard(user):
    """Render user dashboard."""
    return f"Welcome, {user}!"
EOF

git add src/dashboard.py
git commit -m "feat(dashboard): Add dashboard rendering"

git checkout develop
git merge --no-ff feature/dashboard -m "Merge feature/dashboard"
git branch -d feature/dashboard

# === RELEASE PREPARATION ===
# 9. Create release branch
git checkout -b release/1.0.0 develop

# 10. Bump version and prepare metadata
echo "1.0.0" > VERSION

cat > CHANGELOG.md << 'EOF'
# Changelog

## [1.0.0] - $(date +%Y-%m-%d)

### Added
- User authentication (login/logout)
- Password validation
- User dashboard
EOF

git add VERSION CHANGELOG.md
git commit -m "chore(release): Bump version to 1.0.0"

# 11. Merge release to main and tag
git checkout main
git merge --no-ff release/1.0.0 -m "Release version 1.0.0"
git tag -a v1.0.0 -m "Version 1.0.0"

# 12. Merge release back to develop
git checkout develop
git merge --no-ff release/1.0.0 -m "Merge release/1.0.0 back to develop"
git branch -d release/1.0.0

# === HOTFIX ===
# 13. Create hotfix from main (critical bug discovered!)
git checkout -b hotfix/1.0.1 main

# 14. Fix critical bug - add input validation
cat > src/auth.py << 'EOF'
def login(username, password):
    """Authenticate user with credentials."""
    if not username:
        raise ValueError("Username is required")
    if not password:
        raise ValueError("Password is required")
    pass

def logout():
    """End user session."""
    pass

def validate_password(password):
    """Validate password meets requirements."""
    return len(password) >= 8
EOF

git add src/auth.py
git commit -m "fix(auth): Add input validation to prevent empty credentials"

# 15. Update version
echo "1.0.1" > VERSION
git add VERSION
git commit -m "chore(release): Bump version to 1.0.1"

# 16. Merge to main and tag
git checkout main
git merge --no-ff hotfix/1.0.1 -m "Hotfix: Critical authentication bug"
git tag -a v1.0.1 -m "Version 1.0.1 - Security Hotfix"

# 17. Merge to develop
git checkout develop
git merge --no-ff hotfix/1.0.1 -m "Merge hotfix/1.0.1 to develop"
git branch -d hotfix/1.0.1

# === VERIFY ===
# 18. View complete history
git log --oneline --graph --all --decorate

# 19. List all tags
git tag -l

# 20. Show current branches
git branch -a
```

#### Expected Branch Structure

```
* main (v1.0.1)
  └── Initial commit
  └── Release 1.0.0 (v1.0.0)
  └── Hotfix 1.0.1 (v1.0.1)

* develop
  └── Initialize develop branch
  └── Merge feature/user-authentication
  └── Merge feature/dashboard
  └── Merge release/1.0.0
  └── Merge hotfix/1.0.1
```

#### Verification Checklist

Two tags created (v1.0.0, v1.0.1)

Both main and develop branches exist

Hotfix merged to both main and develop

All feature branches deleted after merge

------------------------------------------------------------------------

### Exercise 4: Conventional Commits

**Objective**: Practice writing commit messages that follow the Conventional Commits specification.

**Skills Practiced**:

- Structured commit messages

- Semantic versioning awareness

- Clear change documentation

#### Task

Create a new repository and make commits using proper conventional commit format:

``` bash
# 1. Initialize repository
mkdir conventional-commits-demo
cd conventional-commits-demo
git init

# 2. Initial setup
echo "# Todo App" > README.md
git add README.md
git commit -m "chore: Initial project setup"

# 3. Add new feature
cat > todo.py << 'EOF'
todos = []

def add_todo(task):
    todos.append({"task": task, "done": False})
    return len(todos) - 1
EOF

git add todo.py
git commit -m "feat: Add todo creation functionality"

# 4. Add another feature with scope
cat >> todo.py << 'EOF'

def complete_todo(index):
    if 0 <= index < len(todos):
        todos[index]["done"] = True
        return True
    return False
EOF

git add todo.py
git commit -m "feat(todo): Add ability to mark todo as complete"

# 5. Fix a bug
cat >> todo.py << 'EOF'

def get_all_todos():
    return todos.copy()  # Return copy to prevent mutation
EOF

git add todo.py
git commit -m "fix: Return copy of todos list to prevent external mutation"

# 6. Add documentation
cat >> README.md << 'EOF'

## Features
- Create todos
- Mark todos as complete
- List all todos
EOF

git add README.md
git commit -m "docs: Add feature list to README"

# 7. Refactor code
cat > todo.py << 'EOF'
class TodoManager:
    def __init__(self):
        self._todos = []

    def add(self, task):
        self._todos.append({"task": task, "done": False})
        return len(self._todos) - 1

    def complete(self, index):
        if 0 <= index < len(self._todos):
            self._todos[index]["done"] = True
            return True
        return False

    def get_all(self):
        return self._todos.copy()
EOF

git add todo.py
git commit -m "refactor: Convert todo functions to TodoManager class"

# 8. Add tests
cat > test_todo.py << 'EOF'
from todo import TodoManager

def test_add_todo():
    manager = TodoManager()
    index = manager.add("Buy groceries")
    assert index == 0

def test_complete_todo():
    manager = TodoManager()
    manager.add("Buy groceries")
    assert manager.complete(0) == True
EOF

git add test_todo.py
git commit -m "test: Add unit tests for TodoManager"

# 9. Breaking change
cat > todo.py << 'EOF'
class TodoManager:
    def __init__(self, storage_backend="memory"):
        self._todos = []
        self._backend = storage_backend

    def add(self, task, priority="normal"):
        self._todos.append({
            "task": task,
            "done": False,
            "priority": priority
        })
        return len(self._todos) - 1

    def complete(self, index):
        if 0 <= index < len(self._todos):
            self._todos[index]["done"] = True
            return True
        return False

    def get_all(self):
        return self._todos.copy()
EOF

git add todo.py
git commit -m "feat(todo)!: Add priority support and storage backend

BREAKING CHANGE: TodoManager now requires storage_backend parameter
and add() method accepts optional priority parameter."

# 10. View history
git log --oneline
```

#### Expected Commit Types Used

| Commit | Type          | Description        |
|--------|---------------|--------------------|
| 1      | `chore`       | Initial setup      |
| 2      | `feat`        | New feature        |
| 3      | `feat(scope)` | Feature with scope |
| 4      | `fix`         | Bug fix            |
| 5      | `docs`        | Documentation      |
| 6      | `refactor`    | Code restructuring |
| 7      | `test`        | Adding tests       |
| 8      | `feat!`       | Breaking change    |

------------------------------------------------------------------------

### Exercise 5: Resolving Merge Conflicts

**Objective**: Learn to handle and resolve merge conflicts.

**Skills Practiced**:

- Understanding conflict markers

- Resolving conflicts manually

- Completing conflicted merges

#### Steps

``` bash
# 1. Initialize repository
mkdir conflict-demo
cd conflict-demo
git init

# 2. Create initial file
cat > greeting.py << 'EOF'
def greet(name):
    return f"Hello, {name}!"
EOF

git add greeting.py
git commit -m "feat: Add basic greeting function"

# 3. Create two branches that will conflict
git checkout -b feature/formal-greeting
cat > greeting.py << 'EOF'
def greet(name, formal=False):
    if formal:
        return f"Good day, {name}. How do you do?"
    return f"Hello, {name}!"
EOF

git add greeting.py
git commit -m "feat: Add formal greeting option"

# 4. Go back to main and make conflicting change
git checkout main
cat > greeting.py << 'EOF'
def greet(name, enthusiasm=1):
    exclamation = "!" * enthusiasm
    return f"Hello, {name}{exclamation}"
EOF

git add greeting.py
git commit -m "feat: Add enthusiasm level to greeting"

# 5. Try to merge - this will conflict!
git merge feature/formal-greeting
# CONFLICT! Auto-merge failed

# 6. View the conflict
cat greeting.py
# You'll see conflict markers:
# <<<<<<< HEAD
# ... (main's version)
# =======
# ... (feature branch's version)
# >>>>>>> feature/formal-greeting

# 7. Resolve the conflict by combining both features
cat > greeting.py << 'EOF'
def greet(name, formal=False, enthusiasm=1):
    """
    Generate a greeting for the given name.

    Args:
        name: Person's name to greet
        formal: Use formal greeting style
        enthusiasm: Number of exclamation marks (ignored if formal)
    """
    if formal:
        return f"Good day, {name}. How do you do?"
    exclamation = "!" * enthusiasm
    return f"Hello, {name}{exclamation}"
EOF

# 8. Mark as resolved and complete merge
git add greeting.py
git commit -m "Merge feature/formal-greeting: Combine formal and enthusiasm features"

# 9. Verify the merge
git log --oneline --graph

# 10. Test the merged code
python3 -c "from greeting import greet; print(greet('World', enthusiasm=3))"
python3 -c "from greeting import greet; print(greet('Sir', formal=True))"
```

#### Conflict Resolution Tips

``` diff
<<<<<<< HEAD
(your current branch's changes)
=======
(incoming branch's changes)
>>>>>>> feature-branch
```

1.  **Read both versions** carefully before deciding

2.  **Combine logic** when both changes are needed

3.  **Test thoroughly** after resolving

4.  **Use a merge tool** for complex conflicts: `git`` ``mergetool`

------------------------------------------------------------------------

### Exercise 6: Interactive Rebase

**Objective**: Clean up commit history using interactive rebase.

**Skills Practiced**:

- Squashing commits

- Reordering commits

- Editing commit messages

#### Steps

``` bash
# 1. Create a messy history
mkdir rebase-demo
cd rebase-demo
git init

echo "# Project" > README.md
git add README.md
git commit -m "Initial commit"

echo "line 1" > file.txt
git add file.txt
git commit -m "wip"

echo "line 2" >> file.txt
git add file.txt
git commit -m "more wip"

echo "line 3" >> file.txt
git add file.txt
git commit -m "still working"

echo "line 4" >> file.txt
git add file.txt
git commit -m "almost done"

echo "line 5" >> file.txt
git add file.txt
git commit -m "done!"

# 2. View messy history
git log --oneline
# Shows: done! -> almost done -> still working -> more wip -> wip -> Initial

# 3. Interactive rebase to clean up (last 5 commits)
git rebase -i HEAD~5

# In the editor, change the commits:
# pick abc1234 wip
# squash def5678 more wip
# squash ghi9012 still working
# squash jkl3456 almost done
# squash mno7890 done!

# 4. Update the commit message to something meaningful
# "feat: Add complete file content"

# 5. View clean history
git log --oneline
# Now shows: feat: Add complete file content -> Initial commit
```

#### Rebase Commands Reference

| Command  | Action                           |
|----------|----------------------------------|
| `pick`   | Keep commit as-is                |
| `reword` | Keep commit, edit message        |
| `edit`   | Pause to amend commit            |
| `squash` | Combine with previous commit     |
| `fixup`  | Like squash, but discard message |
| `drop`   | Remove commit entirely           |

Never rebase commits that have been pushed to a shared repository!

------------------------------------------------------------------------

### Summary Checklist

After completing all exercises, verify you can:

Initialize repositories and make commits

Create and merge branches

Follow Gitflow workflow for releases

Write conventional commit messages

Resolve merge conflicts

Use interactive rebase to clean history

Tag releases appropriately

------------------------------------------------------------------------

### Additional Challenges

1.  **Set up a remote**: Create a repository on GitHub/GitLab and practice push/pull operations

2.  **Collaborate with a partner**: Practice creating and reviewing pull requests

3.  **Automate with hooks**: Set up a pre-commit hook to lint commit messages

4.  **Cherry-pick exercise**: Practice moving specific commits between branches

## Review Questions

Test your understanding of Git concepts and workflows with these review questions.

------------------------------------------------------------------------

## Multiple Choice

### 1. What is the purpose of the staging area in Git?

- A\) To store backup copies of files

- B\) To select which changes to include in the next commit

- C\) To merge branches automatically

- D\) To resolve conflicts

Answer

**B) To select which changes to include in the next commit**

The staging area (index) allows you to selectively choose which changes go into your next commit, giving you fine-grained control over your commit history.

------------------------------------------------------------------------

### 2. In Gitflow, which branch should hotfixes be merged into?

- A\) Only `main`

- B\) Only `develop`

- C\) Both `main` and `develop`

- D\) Only the feature branch

Answer

**C) Both `main` and `develop`**

Hotfixes must be merged into both branches to ensure the fix is deployed to production (main) and also included in ongoing development (develop).

------------------------------------------------------------------------

### 3. What does the `--no-ff` flag do when merging?

- A\) Skips conflict resolution

- B\) Forces a merge commit even if fast-forward is possible

- C\) Deletes the source branch after merge

- D\) Merges without fetching remote changes

Answer

**B) Forces a merge commit even if fast-forward is possible**

The `--no-ff` flag creates a merge commit that preserves the history of the feature branch, making it easier to see when features were integrated.

------------------------------------------------------------------------

### 4. Which conventional commit type indicates a breaking change?

- A\) `break:`

- B\) `feat!:`

- C\) `major:`

- D\) `breaking:`

Answer

**B) `feat!:`**

The exclamation mark (`!`) after the type indicates a breaking change. You can also use a `BREAKING`` ``CHANGE:` footer in the commit body.

------------------------------------------------------------------------

### 5. What command shows the difference between staged and unstaged changes?

- A\) `git`` ``status`

- B\) `git`` ``log`

- C\) `git`` ``diff`

- D\) `git`` ``show`

Answer

**C) `git`` ``diff`**

- `git`` ``diff` shows unstaged changes

- `git`` ``diff`` ``--staged` shows staged changes

- `git`` ``diff`` ``HEAD` shows all changes since last commit

------------------------------------------------------------------------

## Scenario Questions

### 6. Feature Branch Conflict

You’re working on `feature/payment` and need to merge it to `main`. However, a teammate has already merged changes to `main` that conflict with your code.

**What steps should you take?**

Answer

1.  **Fetch latest changes**: `git`` ``fetch`` ``origin`

2.  **Rebase or merge main into your branch**:

    ``` bash
    git checkout feature/payment
    git rebase main  # or git merge main
    ```

3.  **Resolve conflicts** in each file (remove conflict markers)

4.  **Continue the rebase**: `git`` ``rebase`` ``--continue` (or commit if merging)

5.  **Test your code** to ensure the merge didn’t break anything

6.  **Push and create PR**: `git`` ``push`` ``origin`` ``feature/payment`

------------------------------------------------------------------------

### 7. Emergency Hotfix

Production is down due to a critical bug in version 2.1.0. You need to deploy a fix immediately, but the `develop` branch has incomplete features.

**Which workflow should you use?**

Answer

Use the **Hotfix workflow**:

1.  Create hotfix branch from `main`:

    ``` bash
    git checkout -b hotfix/2.1.1 main
    ```

2.  Fix the bug and commit:

    ``` bash
    git commit -m "fix: Resolve critical production issue"
    ```

3.  Bump version and merge to `main`:

    ``` bash
    git checkout main
    git merge --no-ff hotfix/2.1.1
    git tag -a v2.1.1 -m "Hotfix release"
    ```

4.  Merge back to `develop`:

    ``` bash
    git checkout develop
    git merge --no-ff hotfix/2.1.1
    ```

5.  Delete hotfix branch and deploy

------------------------------------------------------------------------

### 8. Messy Commit History

You have 5 commits with messages like “wip”, “fix typo”, “more changes”, “oops”, and “done”. Your team requires clean commit history before merging PRs.

**How do you clean this up?**

Answer

Use **interactive rebase** to squash commits:

``` bash
git rebase -i HEAD~5
```

In the editor, change all but the first `pick` to `squash`:

```
pick abc1234 wip
squash def5678 fix typo
squash ghi9012 more changes
squash jkl3456 oops
squash mno7890 done
```

Then write a proper commit message:

```
feat: Add payment processing module

- Implement payment gateway integration
- Add validation for card details
- Handle error responses
```

------------------------------------------------------------------------

## Command Identification

### 9. Match the command to its purpose:

| Command                               | Purpose |
|---------------------------------------|---------|
| `git`` ``stash`                       | ?       |
| `git`` ``cherry-pick`` ``<sha>`       | ?       |
| `git`` ``reset`` ``--hard`` ``HEAD~1` | ?       |
| `git`` ``reflog`                      | ?       |

Answer

| Command | Purpose |
|----|----|
| `git`` ``stash` | Temporarily save uncommitted changes |
| `git`` ``cherry-pick`` ``<sha>` | Apply a specific commit to current branch |
| `git`` ``reset`` ``--hard`` ``HEAD~1` | Discard the last commit and its changes |
| `git`` ``reflog` | View history of all Git operations |

------------------------------------------------------------------------

### 10. What commands would you use to:

1.  Create a new branch and switch to it

2.  See all branches (local and remote)

3.  Delete a merged branch

4.  Rename the current branch

Answer

``` bash
# 1. Create and switch to new branch
git checkout -b new-branch
# or
git switch -c new-branch

# 2. See all branches
git branch -a

# 3. Delete a merged branch
git branch -d branch-name
```

------------------------------------------------------------------------

## Short Answer

### 11. Explain the difference between `git`` ``merge` and `git`` ``rebase`

Answer

**Git Merge**:

- Creates a merge commit combining two branches

- Preserves complete history of both branches

- Non-destructive (doesn’t rewrite history)

- Results in a non-linear history

**Git Rebase**:

- Moves commits to begin on top of another branch

- Creates a linear, cleaner history

- Rewrites commit history (changes SHAs)

- Should not be used on shared/public branches

**When to use each**:

- **Merge**: For merging feature branches to main, preserving context

- **Rebase**: For updating feature branches with latest main changes, cleaning up local commits

------------------------------------------------------------------------

### 12. What makes a good Pull Request?

Answer

A good Pull Request should have:

1.  **Descriptive title** summarizing the change

2.  **Clear description** explaining what, why, and how

3.  **Small, focused scope** (ideally \< 400 lines)

4.  **Passing CI/CD checks**

5.  **Self-review completed** before requesting review

6.  **Linked issues** using keywords like “Closes \#123”

7.  **Screenshots/recordings** for UI changes

8.  **No merge conflicts** with target branch

9.  **Proper testing** documented or included

------------------------------------------------------------------------

### 13. Why should you use Conventional Commits?

Answer

Benefits of Conventional Commits:

1.  **Automatic changelog generation** - Tools can parse commits to create release notes

2.  **Semantic versioning** - Commit types determine version bumps:

    - `fix` → PATCH

    - `feat` → MINOR

    - `BREAKING`` ``CHANGE` → MAJOR

3.  **Clear communication** - Team members understand changes at a glance

4.  **Better searchability** - Easy to find specific types of changes

5.  **CI/CD automation** - Trigger different workflows based on commit type

------------------------------------------------------------------------

## Self-Assessment Checklist

Rate your confidence (1-5) on each skill:

| Skill                           | Confidence |
|---------------------------------|------------|
| Creating and switching branches | ⬜         |
| Staging and committing changes  | ⬜         |
| Resolving merge conflicts       | ⬜         |
| Using Gitflow workflow          | ⬜         |
| Writing conventional commits    | ⬜         |
| Interactive rebasing            | ⬜         |
| Creating effective PRs          | ⬜         |
| Reviewing others’ code          | ⬜         |

**Scoring**:

- **35-40**: Expert level - ready to lead Git workflows

- **25-34**: Proficient - comfortable with daily Git usage

- **15-24**: Intermediate - practice more complex scenarios

- **Below 15**: Beginner - review documentation and redo exercises
