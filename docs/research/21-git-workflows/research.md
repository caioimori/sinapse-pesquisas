# Git Workflows & GitHub Collaboration -- Deep Research (Part 1 of 2)

> **Research Depth:** DEEP DIVE (Level 3)
> **Source Credibility Target:** Tier 1-3
> **Context:** SINAPSE framework (sinapse-ai v9.2.0) -- 2 human devs (Caio + Matheus) + 186 AI agents
> **Date:** 2026-04-10
> **Researcher:** Prism (research-orqx) via WebSearch

---

## Table of Contents

1. [Git Fundamentals & Internals](#1-git-fundamentals--internals)
2. [Branching Strategies -- Deep Comparison](#2-branching-strategies--deep-comparison)
3. [Branch Naming Conventions](#3-branch-naming-conventions)
4. [Commit Conventions & History](#4-commit-conventions--history)
5. [Pull Request Best Practices](#5-pull-request-best-practices)
6. [Code Review Practices](#6-code-review-practices)
7. [GitHub Features Deep Dive](#7-github-features-deep-dive)
8. [Git Hooks & Automation](#8-git-hooks--automation)
9. [Conflict Resolution & Recovery](#9-conflict-resolution--recovery)
10. [Key People, Books & Sources](#10-key-people-books--sources)

---

## 1. Git Fundamentals & Internals

### 1.1 The Object Model

Git is, at its core, a **content-addressable filesystem** -- a key-value store where every piece of data is identified by a SHA-1 (or SHA-256 in newer versions) hash of its contents. The entire repository history is built from just four object types.

#### The Four Object Types

| Object   | Purpose                                      | Identified By             |
|----------|----------------------------------------------|---------------------------|
| **Blob** | Stores file content (no filename, no mode)   | SHA-1 of content          |
| **Tree** | Represents a directory listing               | SHA-1 of entries          |
| **Commit** | Points to a tree + metadata + parent(s)    | SHA-1 of all data         |
| **Tag**  | Named, permanent label on a commit           | SHA-1 of tag data         |

**Blob:** A blob holds the raw contents of a file. It does NOT store the filename or permissions -- those belong to the tree that references the blob. Two files with identical content anywhere in the repo share the same blob object.

```
$ git hash-object -w myfile.txt
af1349b9f5f9a1a6a0404dea36dcc9499bcb25c9
```

**Tree:** A tree is a list of entries, each containing a file mode (100644 for file, 040000 for directory, 100755 for executable, 120000 for symlink), an object type (blob or tree), a SHA-1 hash, and a filename.

```
$ git cat-file -p main^{tree}
100644 blob a906cb2a4a904a152e80877d4088654daad0c859   README.md
040000 tree 99f1a6d12cb4b6f19c8655fca46c3ecf317074e0   src
100644 blob 47c6340d6459e05787f644c2447d2595f5d3a54b   package.json
```

**Commit:** A commit object contains: a pointer to a top-level tree (the snapshot), zero or more parent commits, author info (name, email, timestamp), committer info (can differ from author), and a commit message.

```
$ git cat-file -p HEAD
tree 6ef19b41225c5369f1c104d45d8d85efa9b057b9
parent a11bef06a3f659402fe7563abf99ad00de2209e6
author Caio Imori <caio@example.com> 1712700000 -0300
committer Caio Imori <caio@example.com> 1712700000 -0300

feat: add installer module
```

**Tag:** An annotated tag is a full object containing the tagged object's SHA, the object type, the tag name, tagger info, and a message. Lightweight tags are just refs (not objects).

#### How Objects Connect -- The DAG

```
                    tag v1.0
                       |
commit C3 ---------> tree T3 -------> blob B1 (README.md)
  |                     |-----------> blob B4 (index.js)  [changed]
  | parent              |-----------> tree T3a (src/)
  v
commit C2 ---------> tree T2 -------> blob B1 (README.md)
  |                     |-----------> blob B3 (index.js)
  | parent              |-----------> tree T2a (src/)
  v
commit C1 ---------> tree T1 -------> blob B1 (README.md)
                        |-----------> blob B2 (index.js)
```

Key properties of the DAG (Directed Acyclic Graph):
- **Directed:** Every edge points from child to parent (newer to older)
- **Acyclic:** No commit can be its own ancestor (no loops)
- **Immutable:** Changing any object changes its hash, breaking all references to it
- **Cascading integrity:** If you tamper with a blob, the tree hash changes, the commit hash changes, and every descendant commit hash changes

### 1.2 Refs and HEAD

**Refs** are human-readable names that point to SHA-1 hashes. They live in `.git/refs/`:

```
.git/refs/
  heads/           # Local branches
    main           # Contains SHA of latest commit on main
    feature/auth   # Contains SHA of latest commit on feature/auth
  remotes/
    origin/
      main         # Last known state of remote main
  tags/
    v1.0.0         # Points to tag object or commit
```

**HEAD** is a special ref that points to the current branch (or directly to a commit in "detached HEAD" state):

```
# Normal state -- HEAD points to a branch
$ cat .git/HEAD
ref: refs/heads/main

# Detached HEAD -- points directly to a commit
$ cat .git/HEAD
a11bef06a3f659402fe7563abf99ad00de2209e6
```

**Symbolic refs:** HEAD is typically a "symbolic ref" -- it points to another ref, not directly to a commit. `git symbolic-ref HEAD` shows what branch you are on.

### 1.3 The Three Trees of Git

Git manages three "trees" (data structures) during normal operation:

| Tree              | Location      | Purpose                              |
|-------------------|---------------|--------------------------------------|
| **Working Directory** | Filesystem | Files you see and edit               |
| **Staging Area (Index)** | `.git/index` | Next commit snapshot (staged changes) |
| **HEAD**          | `.git/HEAD`   | Last commit snapshot                 |

```
Working Dir    Index (Stage)     HEAD (Repo)
    |               |                |
    |-- git add --->|                |
    |               |-- git commit ->|
    |<----------- git checkout ------|
    |<- git restore |                |
    |               |<-- git reset --|
```

### 1.4 Merge vs Rebase Internals

#### Three-Way Merge

A three-way merge uses three points: the two branch tips and their **merge base** (most recent common ancestor).

```
         A---B---C  feature
        /
   D---E---F---G    main

After git merge feature (from main):

         A---B---C  feature
        /         \
   D---E---F---G---M  main  (M = merge commit)
```

How it works internally:
1. Git finds the merge base (commit E) using `git merge-base main feature`
2. Git diffs merge-base..main (changes on main: F, G)
3. Git diffs merge-base..feature (changes on feature: A, B, C)
4. Git applies both sets of changes to the merge base
5. If no overlap: creates merge commit M with two parents (G and C)
6. If overlap: marks conflicts, pauses for human resolution

**Fast-forward merge** occurs when the target branch has no new commits since the source branched off:

```
   D---E---A---B---C  feature
        ^
        main

After git merge feature (from main) -- fast-forward:

   D---E---A---B---C  main, feature
```

No merge commit is created; main simply moves its pointer forward. Use `--no-ff` to force a merge commit even when fast-forward is possible (useful for preserving branch history).

#### Rebase

Rebase replays commits from one branch onto another, creating NEW commit objects with different parents (and therefore different SHA hashes):

```
         A---B---C  feature
        /
   D---E---F---G    main

After git rebase main (from feature):

                  A'--B'--C'  feature
                 /
   D---E---F---G              main
```

Internally:
1. Git identifies commits to replay (A, B, C -- commits in feature not in main)
2. Git resets feature to tip of main (G)
3. Git cherry-picks each commit in order: A onto G = A', B onto A' = B', C onto B' = C'
4. Each new commit has the same diff as the original but a different parent and SHA

**Interactive rebase** (`git rebase -i`) allows reordering, squashing, editing, dropping, or rewording commits before replay.

#### When to Use Each

| Scenario                           | Use Merge          | Use Rebase         |
|------------------------------------|---------------------|--------------------|
| Public/shared branches             | YES                 | DANGEROUS          |
| Preserving full history            | YES                 | NO                 |
| Linear, clean history              | NO                  | YES                |
| Feature branch before PR           | --                  | YES (onto main)    |
| Main/develop integration           | YES (--no-ff)       | NO                 |
| Solo developer, local cleanup      | --                  | YES                |

**Golden Rule of Rebase:** Never rebase commits that have been pushed to a shared remote and that others may have based work on.

### 1.5 Cherry-Pick

Cherry-pick applies the diff introduced by a specific commit onto the current branch:

```
$ git cherry-pick abc1234

         A---B---C  feature
        /
   D---E---F---B'   main  (B' has same diff as B, different SHA)
```

Use cases:
- Applying a hotfix from a release branch to main
- Selectively pulling specific changes without merging entire branches
- Backporting fixes to older release branches

### 1.6 Reflog

The reflog records every change to HEAD and branch tips -- even commits that are no longer reachable from any branch. It is a local safety net.

```
$ git reflog
a1b2c3d HEAD@{0}: commit: feat: add auth module
f4e5d6c HEAD@{1}: checkout: moving from feature to main
9876abc HEAD@{2}: commit (amend): fix: typo in config
1234def HEAD@{3}: rebase (finish): returning to refs/heads/feature
```

Reflog entries expire after 90 days (reachable) or 30 days (unreachable) by default. This means you have at least 30 days to recover from almost any git mistake.

Key recovery commands:
```bash
# Find the commit you lost
git reflog

# Recover a deleted branch
git checkout -b recovered-branch HEAD@{5}

# Undo a bad rebase
git reset --hard HEAD@{3}

# See reflog for a specific branch
git reflog show feature/auth
```

### 1.7 Bisect

`git bisect` performs a binary search through commit history to find which commit introduced a bug:

```bash
git bisect start
git bisect bad                 # Current commit is broken
git bisect good v1.0.0         # v1.0.0 was working

# Git checks out a commit halfway between good and bad
# Test it, then tell git:
git bisect good   # or
git bisect bad

# Repeat until git identifies the first bad commit
# git bisect reset when done
```

With N commits between good and bad, bisect finds the culprit in O(log N) steps. For 1,000 commits, that is approximately 10 tests.

Automated bisect:
```bash
git bisect start HEAD v1.0.0
git bisect run npm test
# Git automatically runs the test suite on each candidate commit
```

---

## 2. Branching Strategies -- Deep Comparison

### 2.1 Git Flow (Vincent Driessen, 2010)

The original structured branching model, designed for software with explicit release cycles.

```
        hotfix
        __|__
       /     \
main  *---*---*-----------*-------*---*
       \       \         / \     /
develop *---*---*---*---*   *---*
             \     /   \       /
     feature  *---*     *---*--
              feat/a    feat/b
```

**Long-lived branches:**
- `main` -- Production-ready code only. Every commit is a release.
- `develop` -- Integration branch. All features merge here first.

**Short-lived branches:**
- `feature/*` -- Branch from develop, merge back to develop
- `release/*` -- Branch from develop when ready, merge to main AND develop
- `hotfix/*` -- Branch from main for emergency fixes, merge to main AND develop

| Aspect          | Detail                                                |
|-----------------|-------------------------------------------------------|
| **Complexity**  | HIGH -- 5 branch types, strict rules                  |
| **Team size**   | 5-50+ developers                                      |
| **Release cadence** | Scheduled releases (weekly, monthly, quarterly)   |
| **CI/CD fit**   | LOW -- long-lived branches delay integration          |
| **Best for**    | Mobile apps, desktop software, versioned APIs         |
| **Avoid when**  | Continuous deployment, small teams, web SaaS           |

**Pros:** Clear release management, parallel version support, good for regulated environments.
**Cons:** Complex, slow integration, merge hell between develop and features, Driessen himself acknowledged it is not ideal for web apps with continuous delivery.

### 2.2 GitHub Flow (Scott Chacon)

Simplified model designed for continuous deployment. Only two concepts: `main` and feature branches.

```
main  *---*---*---*---*---*---*---*
       \     /     \     /
        *---*       *---*
        feat/a      feat/b
        (PR+review) (PR+review)
```

**Rules:**
1. `main` is always deployable
2. Branch from main for any work (descriptive name)
3. Commit to your branch, push regularly
4. Open a Pull Request for discussion
5. After review and CI pass, merge to main
6. Deploy immediately after merge

| Aspect          | Detail                                                |
|-----------------|-------------------------------------------------------|
| **Complexity**  | VERY LOW -- 1 rule: branch from main, PR back         |
| **Team size**   | 1-20 developers                                       |
| **Release cadence** | Continuous (every merge = potential deploy)        |
| **CI/CD fit**   | EXCELLENT -- designed for it                          |
| **Best for**    | Web apps, SaaS, APIs with CD pipelines                |
| **Avoid when**  | Multiple versions in production, strict release gates  |

**Pros:** Simple, fast, encourages small PRs, minimal overhead.
**Cons:** No release branch concept, requires excellent CI/CD, no support for multiple production versions.

### 2.3 GitLab Flow

A middle ground between Git Flow and GitHub Flow, adding environment branches.

```
main      *---*---*---*---*---*
           \     /
            *---*  (feature, merged via MR)

pre-prod  *---*---*  (cherry-pick or merge from main)

prod      *---*       (deploy from pre-prod)
```

**Variants:**
- **Environment branches:** main -> pre-production -> production
- **Release branches:** main -> release/1.0, release/2.0 (for versioned software)

| Aspect          | Detail                                                |
|-----------------|-------------------------------------------------------|
| **Complexity**  | MEDIUM -- adds environment layers to GitHub Flow      |
| **Team size**   | 5-50 developers                                       |
| **Release cadence** | Flexible (continuous or scheduled)                |
| **CI/CD fit**   | HIGH -- environment branches map to deploy targets    |
| **Best for**    | Teams needing staging gates before production         |
| **Avoid when**  | Very small teams, pure CD without staging             |

**Pros:** Clear promotion path (dev -> staging -> prod), supports both CD and release models.
**Cons:** More complex than GitHub Flow, can drift if environment branches diverge.

### 2.4 Trunk-Based Development (Google, Meta, Microsoft)

All developers commit to a single branch ("trunk" / "main") with very short-lived feature branches (< 2 days) or direct commits.

```
main  *-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*
       \ /   \ /       \ /
        *     *         *
       (1d)  (hours)   (1d)
       short-lived feature branches
```

**Key practices:**
- Feature branches live less than 1-2 days (ideally hours)
- Feature flags hide incomplete work
- CI runs on every commit to trunk
- No long-lived branches
- Code review happens before merge (or post-commit for very trusted devs)

| Aspect          | Detail                                                |
|-----------------|-------------------------------------------------------|
| **Complexity**  | LOW (branching) / HIGH (supporting infra)             |
| **Team size**   | Any (Google: 30,000+ devs on monorepo)                |
| **Release cadence** | Continuous, on-demand                             |
| **CI/CD fit**   | MAXIMUM -- designed for CI/CD                         |
| **Best for**    | High-performing teams, monorepos, CD-native           |
| **Avoid when**  | Teams without feature flags, weak CI, junior-heavy    |

**DORA data (2024):** Elite performers (who typically practice trunk-based development or very short-lived branches) deploy multiple times per day, have lead times under one day, change failure rates around 5%, and restore service in under one hour. The gap with low performers is 182x in deployment frequency and 127x in change lead times (Source: DORA Report 2024).

**Pros:** Fastest integration, minimal merge conflicts, proven at massive scale (Google, Meta, Microsoft), aligns with DORA elite performance.
**Cons:** Requires feature flags, robust CI, high test coverage, discipline.

### 2.5 Release Flow (Microsoft)

Microsoft's model for large-scale projects with release branches but trunk-based development for daily work.

```
main  *-*-*-*-*-*-*-*-*-*-*-*-*-*-*
       \ /     |           |
        *      |           |
      (topic)  |           |
               v           v
          release/1.0   release/2.0
              *-*-*        *-*
            (hotfix)     (hotfix)
```

**Rules:**
1. Use `main` for all active development (trunk-based)
2. Create short-lived topic branches for changes
3. PR into main with code review
4. At release time, create `release/X.Y` from main
5. Cherry-pick hotfixes from main to release branch (never merge back)

| Aspect          | Detail                                                |
|-----------------|-------------------------------------------------------|
| **Complexity**  | MEDIUM                                                |
| **Team size**   | 10-100+ developers                                    |
| **Release cadence** | Periodic releases with hotfix capability           |
| **CI/CD fit**   | HIGH for main, moderate for release branches          |
| **Best for**    | Products with release trains (VS Code, .NET, Azure)   |
| **Avoid when**  | Pure CD with no versioning needs                      |

### 2.6 Ship / Show / Ask

A human-centric model that categorizes changes by the level of review needed:

```
main  *---*---*---*---*---*---*
       |       \     /
       |        *---*     <-- "Ask" (PR, needs review before merge)
       |
       *---*              <-- "Show" (PR, merge immediately, review async)
       |
       * (direct push)   <-- "Ship" (no PR needed, trivial change)
```

| Category | When                               | PR? | Review Before Merge? |
|----------|------------------------------------|-----|---------------------|
| **Ship** | Trivial, low-risk (typo, config)   | No  | No                  |
| **Show** | Medium changes, informational      | Yes | No (merge first)    |
| **Ask**  | Significant, needs discussion      | Yes | Yes                 |

| Aspect          | Detail                                                |
|-----------------|-------------------------------------------------------|
| **Complexity**  | LOW                                                   |
| **Team size**   | 2-15 (high-trust teams)                               |
| **CI/CD fit**   | HIGH                                                  |
| **Best for**    | Mature, high-trust teams with good test coverage      |
| **Avoid when**  | Compliance requirements, junior-heavy teams            |

### 2.7 Forking Workflow

Standard for open source. Each contributor has their own fork (full repo copy).

```
upstream/main  *---*---*---*---*---*
                    ^         ^
                    |         |
contributor/main    |    *---*---* (PR from fork)
                    |
contributor2/main   *---* (PR from fork)
```

**Flow:**
1. Fork the upstream repository
2. Clone your fork locally
3. Add upstream as a remote
4. Create feature branch on your fork
5. Push to your fork
6. Open PR from your fork to upstream
7. Maintainers review and merge

| Aspect          | Detail                                                |
|-----------------|-------------------------------------------------------|
| **Complexity**  | MEDIUM (extra remote management)                      |
| **Team size**   | Unlimited (open source scale)                         |
| **CI/CD fit**   | MEDIUM -- CI runs on fork PRs                         |
| **Best for**    | Open source, untrusted contributors                   |
| **Avoid when**  | Internal teams with repo access                       |

### 2.8 OneFlow

Adam Ruka's simplification of Git Flow, eliminating the `develop` branch.

```
main  *---*---*-------*---*---*
       \     / \     /
        *---*   *---*
       feature  release/1.0
                 |
                 * (hotfix applied here)
```

**Rules:**
- Only `main` is long-lived
- Features branch from and merge to `main`
- Releases branch from `main`, get stabilized, then merge back
- Hotfixes go on the release branch, then cherry-pick to main

| Aspect          | Detail                                                |
|-----------------|-------------------------------------------------------|
| **Complexity**  | MEDIUM (simpler than Git Flow)                        |
| **Team size**   | 5-30 developers                                       |
| **CI/CD fit**   | MEDIUM-HIGH                                           |
| **Best for**    | Teams outgrowing Git Flow, wanting simplicity         |
| **Avoid when**  | Pure CD workflows                                     |

### 2.9 Comparative Table

| Strategy        | Complexity | Team Size | Release Cadence | CI/CD Fit  | Long-Lived Branches |
|-----------------|------------|-----------|-----------------|------------|---------------------|
| Git Flow        | HIGH       | 5-50+     | Scheduled        | LOW        | main, develop       |
| GitHub Flow     | VERY LOW   | 1-20      | Continuous       | EXCELLENT  | main only           |
| GitLab Flow     | MEDIUM     | 5-50      | Flexible         | HIGH       | main + env branches |
| Trunk-Based     | LOW*       | Any       | Continuous       | MAXIMUM    | main only           |
| Release Flow    | MEDIUM     | 10-100+   | Periodic         | HIGH       | main + release/*    |
| Ship/Show/Ask   | LOW        | 2-15      | Continuous       | HIGH       | main only           |
| Forking         | MEDIUM     | Unlimited | Varies           | MEDIUM     | main per fork       |
| OneFlow         | MEDIUM     | 5-30      | Periodic         | MEDIUM-HIGH| main only           |

*Trunk-Based is low complexity in branching but requires high supporting infrastructure (feature flags, CI, tests).

---

## 3. Branch Naming Conventions

### 3.1 Standard Prefix Patterns

| Prefix       | Purpose                          | Example                        |
|--------------|----------------------------------|--------------------------------|
| `feat/`      | New feature                      | `feat/user-auth`               |
| `fix/`       | Bug fix                          | `fix/login-redirect`           |
| `hotfix/`    | Emergency production fix         | `hotfix/payment-crash`         |
| `chore/`     | Maintenance, deps, tooling       | `chore/update-deps`            |
| `docs/`      | Documentation only               | `docs/api-reference`           |
| `refactor/`  | Code restructuring               | `refactor/auth-module`         |
| `test/`      | Adding/fixing tests              | `test/payment-edge-cases`      |
| `ci/`        | CI/CD pipeline changes           | `ci/add-deploy-stage`          |
| `release/`   | Release preparation              | `release/2.1.0`                |
| `experiment/`| Exploratory, may be discarded    | `experiment/new-cache-layer`   |

### 3.2 Full Pattern Format

```
<user-prefix>/<type>/<short-description>

Examples:
caio/feat/installer-ux
soier/fix/agent-config
dev/refactor/auth-module
```

For the SINAPSE context (2 devs + AI agents):
```
caio/<type>/<description>      # Caio's branches
soier/<type>/<description>     # Matheus's branches
dev/<type>/<description>       # Unknown/AI branches
agent/<agent-id>/<description> # AI agent branches (optional)
```

### 3.3 Naming Rules

| Rule                          | Good                     | Bad                       |
|-------------------------------|--------------------------|---------------------------|
| Use lowercase                 | `feat/add-auth`          | `Feat/Add-Auth`           |
| Use hyphens (not underscores) | `fix/login-bug`          | `fix/login_bug`           |
| Keep short (< 50 chars)       | `feat/user-auth`         | `feat/implement-the-new-user-authentication-system` |
| Include ticket/story ID       | `feat/SIN-42-user-auth`  | `feat/user-auth`          |
| No special characters         | `feat/oauth2`            | `feat/oauth2.0!`          |
| No spaces                     | `feat/my-feature`        | `feat/my feature`         |

### 3.4 Automated Enforcement

**Branch protection regex (GitHub):**
```
^(feat|fix|hotfix|chore|docs|refactor|test|ci|release|experiment)\/[a-z0-9][a-z0-9\-]*$
```

**With user prefix:**
```
^(caio|soier|dev)\/(feat|fix|hotfix|chore|docs|refactor|test|ci|release)\/.+$
```

**Pre-push hook enforcement:**
```bash
#!/bin/bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
PATTERN="^(caio|soier|dev)/(feat|fix|hotfix|chore|docs|refactor|test|ci|release)/.+$"

if [[ ! "$BRANCH" =~ $PATTERN ]] && [[ "$BRANCH" != "main" ]]; then
  echo "ERROR: Branch name '$BRANCH' does not match pattern."
  echo "Expected: <user>/<type>/<description>"
  exit 1
fi
```

**GitHub Rulesets** (newer than branch protection rules) support regex patterns for branch naming enforcement at the repository or organization level.

---

## 4. Commit Conventions & History

### 4.1 Conventional Commits Specification (v1.0.0)

The Conventional Commits spec (https://www.conventionalcommits.org) provides a lightweight convention for creating explicit commit history.

**Format:**
```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

**Structural elements:**

| Element        | Required? | Purpose                                               |
|----------------|-----------|-------------------------------------------------------|
| `type`         | YES       | Category of change                                    |
| `scope`        | NO        | Module/component affected (in parentheses)            |
| `description`  | YES       | Short imperative summary (< 72 chars)                 |
| `body`         | NO        | Detailed explanation of what and why                   |
| `footer`       | NO        | Breaking changes, issue references, co-authors         |
| `!` after type | NO        | Indicates breaking change                             |

**Core types (mandated by spec):**

| Type    | SemVer Impact | When to Use                            |
|---------|---------------|----------------------------------------|
| `feat`  | MINOR         | New feature for the user               |
| `fix`   | PATCH         | Bug fix for the user                   |

**Extended types (Angular convention, widely adopted):**

| Type       | SemVer Impact | When to Use                             |
|------------|---------------|-----------------------------------------|
| `docs`     | None          | Documentation only                       |
| `style`    | None          | Formatting, whitespace, semicolons       |
| `refactor` | None          | Code restructuring, no feature/fix       |
| `perf`     | None (PATCH)  | Performance improvement                  |
| `test`     | None          | Adding or correcting tests               |
| `build`    | None          | Build system, dependencies               |
| `ci`       | None          | CI configuration and scripts             |
| `chore`    | None          | Maintenance, tooling                     |
| `revert`   | Varies        | Reverting a previous commit              |

**Breaking changes:**
```
feat(api)!: remove deprecated /v1 endpoints

BREAKING CHANGE: The /v1/* endpoints have been removed.
Migrate to /v2/* before upgrading.
```

The `BREAKING CHANGE:` footer (or the `!` shorthand) triggers a MAJOR version bump in semantic versioning.

**Examples:**
```
feat(auth): add OAuth2 login with Google

fix(parser): handle empty input without crash
Closes #423

docs: update API reference for v2 endpoints

refactor(core)!: rename Config to Settings across codebase

BREAKING CHANGE: All imports of Config must be updated to Settings.

chore(deps): bump eslint from 8.x to 9.x

ci: add Node 22 to test matrix

perf(query): add index on user_id for 3x faster lookups
```

### 4.2 Toolchain: Commitlint + Husky + Semantic Release + Commitizen

#### Commitlint

Validates commit messages against the Conventional Commits spec.

```bash
# Install
npm install --save-dev @commitlint/cli @commitlint/config-conventional

# Configuration (commitlint.config.js)
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'perf', 'test', 'build', 'ci', 'chore', 'revert'
    ]],
    'scope-case': [2, 'always', 'kebab-case'],
    'subject-max-length': [2, 'always', 72],
    'body-max-line-length': [2, 'always', 100],
  },
};
```

#### Husky v9+

Manages Git hooks from package.json. v9 simplified the API significantly.

```bash
# Install
npm install --save-dev husky
npx husky init

# Creates .husky/ directory with a sample pre-commit hook

# Add commit-msg hook for commitlint
echo 'npx --no -- commitlint --edit "$1"' > .husky/commit-msg

# Add pre-commit hook for lint-staged
echo 'npx lint-staged' > .husky/pre-commit
```

**Husky v9 changes from v8:**
- `npx husky init` replaces `npx husky install`
- Hook files are plain shell scripts (no more `npx husky add`)
- Simpler directory structure
- Automatic `.husky/_/husky.sh` bootstrapper

#### lint-staged

Runs linters/formatters ONLY on staged files (not the whole codebase).

```json
// package.json
{
  "lint-staged": {
    "*.{js,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,yml}": ["prettier --write"],
    "*.css": ["stylelint --fix", "prettier --write"]
  }
}
```

#### Semantic Release

Fully automated version management and package publishing based on commit messages.

```bash
npm install --save-dev semantic-release

# .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    "@semantic-release/npm",
    "@semantic-release/github",
    "@semantic-release/git"
  ]
}
```

**How it determines versions:**

| Commit Type                    | Version Bump | Example                    |
|-------------------------------|--------------|----------------------------|
| `fix:` / `perf:`              | PATCH        | 1.2.3 -> 1.2.4            |
| `feat:`                       | MINOR        | 1.2.4 -> 1.3.0            |
| `BREAKING CHANGE` / `!`       | MAJOR        | 1.3.0 -> 2.0.0            |
| `docs:` / `chore:` / etc.     | No release   | --                         |

**CI integration (GitHub Actions):**
```yaml
name: Release
on:
  push:
    branches: [main]
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: npm ci
      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

#### Commitizen

Interactive CLI that guides developers through creating properly formatted commit messages.

```bash
npm install --save-dev commitizen cz-conventional-changelog

# package.json
{
  "config": {
    "commitizen": {
      "path": "cz-conventional-changelog"
    }
  }
}

# Usage: instead of `git commit`, run:
npx cz
# or add script: "commit": "cz"
```

### 4.3 Signed Commits (GPG/SSH)

Signed commits prove that a commit was actually created by the claimed author.

**GPG signing:**
```bash
# Generate GPG key
gpg --full-generate-key

# Configure Git to use it
git config --global user.signingkey YOUR_KEY_ID
git config --global commit.gpgsign true

# Sign a commit
git commit -S -m "feat: signed commit"

# Verify
git log --show-signature
```

**SSH signing (Git 2.34+, simpler):**
```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

**GitHub verification:** Add your public key (GPG or SSH) to GitHub Settings -> SSH and GPG keys. Verified commits show a "Verified" badge on GitHub.

### 4.4 Atomic Commits

An atomic commit is a single, self-contained unit of change that:
- Does ONE thing (single logical change)
- Passes all tests on its own
- Can be reverted without side effects
- Has a clear, descriptive message

**Anti-patterns:**
- "Fix stuff" -- multiple unrelated fixes in one commit
- "WIP" -- incomplete state committed
- "Add feature X and fix bug Y and update docs" -- three things in one commit

**Practice:** Use `git add -p` (patch mode) to stage specific hunks within a file, splitting changes across multiple atomic commits.

### 4.5 Squash vs Merge vs Rebase Strategies

| Strategy              | History Shape | When to Use                        |
|-----------------------|---------------|-------------------------------------|
| **Merge commit**      | Branchy       | Default, preserves full history     |
| **Squash and merge**  | Linear        | Clean main, messy feature branches  |
| **Rebase and merge**  | Linear        | Clean main, clean feature branches  |

```
# Merge commit -- preserves all commits + adds merge commit
main: A---B---C---M
              \  /
feature:       D---E

# Squash and merge -- all feature commits become one
main: A---B---C---DE'  (single squashed commit)

# Rebase and merge -- feature commits replayed linearly
main: A---B---C---D'---E'
```

**GitHub PR merge options:**

| Option            | Result                                      | Best For                      |
|-------------------|---------------------------------------------|-------------------------------|
| Create merge commit | Merge commit + all feature commits visible | Full history, traceability    |
| Squash and merge  | Single commit on main                       | Clean history, WIP branches   |
| Rebase and merge  | Feature commits replayed linearly           | Already-clean feature commits |

**Recommendation for SINAPSE:** Squash-and-merge as default (AI agents may create many incremental commits), with merge commits for major features where history matters.

---

## 5. Pull Request Best Practices

### 5.1 Optimal PR Size

Research consistently shows that smaller PRs produce better outcomes:

| PR Size (lines changed) | Review Time | Defect Rate Impact               |
|--------------------------|-------------|----------------------------------|
| < 50 lines               | ~15 min     | Fast but higher revert rate      |
| 50-200 lines             | ~30 min     | Optimal sweet spot               |
| 200-400 lines            | ~60 min     | Good, 40% fewer defects vs large |
| 400-600 lines            | ~90 min     | Acceptable upper bound           |
| 600-1000 lines           | ~2-3 hours  | Quality drops significantly      |
| 1000+ lines              | ~4+ hours   | 70% lower defect detection rate  |

Key findings from research:
- **Google:** Recommends PRs under 200 lines of changed code
- **Microsoft:** PRs under 300 lines received 60% more thorough reviews; automated warnings for PRs over 400 lines reduced post-merge defects by 35%
- **Graphite:** The ideal PR is around 50 lines; 50-line changes are reviewed and merged ~40% faster than 250-line changes
- **Industry consensus:** After 60-90 minutes of review, defect detection rate drops sharply; each additional 100 lines adds ~25 minutes of review time

(Sources: Graphite research, Microsoft engineering data, Google eng-practices)

### 5.2 PR Templates

GitHub supports PR templates via `.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## Summary
<!-- What does this PR do? Why? -->

## Type of Change
- [ ] feat: New feature
- [ ] fix: Bug fix
- [ ] refactor: Code restructuring
- [ ] docs: Documentation
- [ ] test: Tests
- [ ] chore: Maintenance

## Story Reference
<!-- Link to story file: docs/stories/X.Y.story.md -->

## Changes Made
<!-- Bullet list of specific changes -->

## Testing
- [ ] Unit tests pass
- [ ] Lint passes
- [ ] Manual testing done (describe below)

## Screenshots
<!-- If UI changes, include before/after -->

## Checklist
- [ ] Self-reviewed my code
- [ ] Added/updated tests
- [ ] Updated documentation
- [ ] No secrets in code
- [ ] PR is < 400 lines (or justified why larger)
```

### 5.3 CODEOWNERS

The `CODEOWNERS` file (in `.github/`, root, or `docs/`) automatically assigns reviewers based on file paths:

```
# .github/CODEOWNERS

# Global owners (fallback)
*                           @caio-imori @soier

# Framework core -- requires both owners
.sinapse-ai/core/           @caio-imori @soier
.sinapse-ai/constitution.md @caio-imori @soier

# Agent definitions
.sinapse-ai/development/    @caio-imori

# Frontend
packages/web/               @soier
packages/ui/                @soier

# Backend
packages/api/               @caio-imori

# Documentation
docs/                       @caio-imori @soier

# CI/CD
.github/                    @caio-imori
```

When branch protection requires CODEOWNERS review, a PR cannot merge until at least one owner of every changed file path has approved.

### 5.4 Draft PRs

Draft PRs signal "work in progress -- not ready for review":

```bash
gh pr create --draft --title "feat: new auth module" --body "WIP"

# Convert to ready when done:
gh pr ready 123
```

Use cases:
- Get early CI feedback
- Share progress with team
- Start async discussion before code is complete
- Prevent accidental merging of incomplete work

### 5.5 Auto-Merge

GitHub auto-merge allows a PR to merge automatically once all required checks pass:

```bash
gh pr merge 123 --auto --squash
```

Requirements:
- Branch protection must be enabled
- All required status checks must pass
- All required reviews must be approved
- No merge conflicts

### 5.6 Merge Queues

GitHub Merge Queue (GA since 2023) ensures that PRs are tested against the latest main before merging, preventing the "semantic conflict" problem.

**Problem it solves:**
```
PR-A passes CI (tested against main at commit X)
PR-B passes CI (tested against main at commit X)
PR-A merges (main is now at commit Y)
PR-B merges -- but was never tested against Y!
   -> Potential breakage even though both PRs passed CI individually
```

**How merge queue works:**
1. Developer clicks "Merge when ready"
2. PR enters the queue
3. GitHub creates a temporary merge group (PR + latest main + other queued PRs)
4. CI runs against the merge group
5. If CI passes, PR merges. If it fails, PR is removed from queue

**Configuration:**
- Build concurrency: 1-100 parallel groups
- Merge method: merge, squash, or rebase
- Minimum/maximum group size
- Status check timeout

### 5.7 Stacked PRs

Stacked PRs break large changes into a chain of dependent, small PRs:

```
main
  |
  +-- PR #1: Add auth types        (50 lines)
       |
       +-- PR #2: Add auth service (80 lines)
            |
            +-- PR #3: Add auth UI (120 lines)
```

**Benefits:**
- Each PR is small and reviewable
- Reviewers can approve incrementally
- Work is not blocked waiting for review of a monolithic PR
- Aligns with trunk-based development

**Tools:**

| Tool       | Type    | Key Feature                                |
|------------|---------|--------------------------------------------|
| **Graphite** | CLI + Web | Full stack management, `gt stack submit`  |
| **ghstack** | CLI     | Facebook's tool, pushes to separate branches |
| **git-town** | CLI   | High-level commands for branch management  |
| **Sapling** | VCS     | Meta's Git-compatible VCS with native stacking |
| **spr**    | CLI     | Submit PRs from a single branch             |

**Graphite workflow:**
```bash
gt branch create feat/auth-types
# ... make changes ...
gt commit create -m "feat: add auth types"

gt branch create feat/auth-service  # stacks on top
# ... make changes ...
gt commit create -m "feat: add auth service"

gt stack submit  # creates/updates all PRs in the stack
gt stack sync    # rebase entire stack on latest main
```

---

## 6. Code Review Practices

### 6.1 Google's Code Review Guidelines

Google's engineering practices documentation (https://google.github.io/eng-practices/) is the most comprehensive public code review guide. Key principles:

**The Standard of Code Review:**
- A reviewer should approve a CL (changelist) if it definitely improves the overall code health of the codebase, even if it is not "perfect"
- There is no "perfect" code -- only "better" code
- A CL that improves the maintainability, readability, or understandability of the system should not be delayed for days because it is not "perfect"
- Reviewers should not require authors to polish every tiny piece

**Speed Guidelines:**
- **Maximum response time:** One business day
- **Average review turnaround at Google:** ~4 hours
- A typical CL gets multiple rounds of review within a single day
- If you are in the middle of a focused task, finish that task before reviewing, but respond by end of day
- Speed of code review is optimized for the speed at which a **team** can produce a product, not individual speed

**What to Look For:**
1. **Design:** Is the change well-designed? Does it belong in this codebase?
2. **Functionality:** Does the code do what the author intended? Is it good for users?
3. **Complexity:** Can the code be understood quickly? Will other developers create bugs when using it?
4. **Tests:** Does the code have correct, well-designed, useful automated tests?
5. **Naming:** Are names clear, descriptive, and following conventions?
6. **Comments:** Are comments clear and useful? Do they explain WHY, not WHAT?
7. **Style:** Does the code follow style guidelines?
8. **Documentation:** Did the author update relevant documentation?

### 6.2 Review Speed vs Thoroughness

| Factor              | Optimize for Speed          | Optimize for Thoroughness    |
|---------------------|-----------------------------|------------------------------|
| PR size             | Small (< 200 lines)         | Large PRs get split          |
| Review time         | < 4 hours turnaround        | Take the time needed         |
| Blocking threshold  | Only for correctness/safety | Also for style/design        |
| Automated checks    | Maximum automation (lint, test, security) | Human catches what tools miss |
| Context             | Read the diff only          | Check out, run locally       |

**Finding the balance:** The research suggests that teams should automate everything that CAN be automated (formatting, linting, type checking, test coverage, security scanning) so that human reviewers can focus on what machines cannot: design quality, business logic correctness, and maintainability.

### 6.3 Automated Code Review Tools

| Tool            | Type        | What It Checks                                 |
|-----------------|-------------|------------------------------------------------|
| **CodeRabbit**  | AI-powered  | Logic, design, security, performance, tests    |
| **SonarQube**   | Static      | Code smells, bugs, vulnerabilities, duplicates |
| **Danger.js**   | Scriptable  | Custom rules (PR size, changelog, labels)      |
| **ESLint**      | Linter      | JavaScript/TypeScript code quality             |
| **Prettier**    | Formatter   | Code formatting consistency                    |
| **Semgrep**     | SAST        | Security patterns, custom rules                |
| **Snyk**        | Dependency  | Vulnerable dependencies                        |
| **Dependabot**  | Dependency  | Outdated dependencies, security advisories     |
| **Renovate**    | Dependency  | Automated dependency updates                   |

**CodeRabbit integration (relevant to SINAPSE):**
CodeRabbit uses AI to provide contextual code review comments on PRs. It can catch logic errors, suggest improvements, and identify security issues that traditional linters miss. It integrates with GitHub as a PR reviewer.

### 6.4 Review Comment Patterns

Standardizing review language reduces friction:

| Prefix         | Meaning                                           | Blocking? |
|----------------|---------------------------------------------------|-----------|
| `nit:`         | Nitpick, minor style preference                   | NO        |
| `suggestion:`  | Alternative approach, consider this                | NO        |
| `question:`    | Need clarification, not blocking                   | NO        |
| `issue:`       | Must be fixed before merge                         | YES       |
| `blocker:`     | Critical problem, blocks merge                     | YES       |
| `praise:`      | Good work, highlighting excellent code             | NO        |
| `thought:`     | Observation for future consideration               | NO        |
| `todo:`        | Should be addressed, can be a follow-up PR         | MAYBE     |

**Example review comments:**
```
nit: Could use a more descriptive variable name here.

suggestion: Consider using Map instead of Object for this lookup -- O(1) guaranteed.

issue: This endpoint has no rate limiting. Must add before merge.
See: https://example.com/rate-limiting-guide

blocker: service_role key is exposed in client-side code. This is a security vulnerability.

praise: Excellent error handling here. The fallback logic is clean and well-tested.
```

### 6.5 Review Metrics to Track

| Metric                  | Target              | Why                                    |
|-------------------------|---------------------|----------------------------------------|
| Time to first review    | < 4 hours           | Unblocks the author                    |
| Review cycles           | <= 2 rounds         | Efficiency                             |
| PR size                 | < 400 lines         | Quality                                |
| Review turnaround       | < 1 business day    | Team velocity                          |
| Merge time (open->merge)| < 24 hours          | Flow efficiency                        |
| Defects found in review | Track trend          | Review effectiveness                   |

---

## 7. GitHub Features Deep Dive

### 7.1 Branch Protection Rules

Classic branch protection (per-branch configuration):

| Rule                                    | What It Does                                    |
|-----------------------------------------|-------------------------------------------------|
| Require pull request before merging     | No direct pushes to protected branch            |
| Require approvals (1-6)                 | N reviewers must approve before merge           |
| Dismiss stale reviews                   | Re-review required after new commits            |
| Require review from CODEOWNERS          | Owners of changed files must approve            |
| Require status checks to pass           | CI must pass before merge                       |
| Require branches to be up to date       | Branch must be rebased on latest target          |
| Require signed commits                  | Only verified (GPG/SSH) commits                 |
| Require linear history                  | Only squash or rebase merges allowed            |
| Require merge queue                     | PRs go through merge queue                      |
| Restrict who can push                   | Limit push access to specific users/teams       |
| Allow force pushes                      | (DANGEROUS) Permit history rewriting            |
| Allow deletions                         | Allow branch deletion                           |

**Recommended configuration for SINAPSE:**
```
main branch:
  - Require pull request: YES
  - Required approvals: 1 (Caio or Matheus)
  - Dismiss stale reviews: YES
  - Require CODEOWNERS: YES
  - Require status checks: YES (lint, test, build)
  - Require up to date: YES (via merge queue)
  - Require merge queue: YES
  - Require linear history: YES (squash merge)
  - Allow force push: NO
  - Allow deletion: NO
```

### 7.2 Repository Rulesets (Newer, More Powerful)

Rulesets (GA 2023) are the next generation of branch protection. Key advantages over classic branch protection:

| Feature                    | Branch Protection | Rulesets            |
|----------------------------|-------------------|---------------------|
| Scope                      | Single branch     | Pattern-based (glob)|
| Organization-level         | NO                | YES                 |
| Bypass list                | Admin only        | Configurable        |
| Tag protection             | Separate feature  | Integrated          |
| Import/export              | NO                | YES (JSON)          |
| Merge queue config         | Separate          | Integrated          |
| Multiple rulesets per branch| NO               | YES (layered)       |

**Ruleset configuration example:**
```json
{
  "name": "main-protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/heads/main"],
      "exclude": []
    }
  },
  "rules": [
    { "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true
      }
    },
    { "type": "required_status_checks",
      "parameters": {
        "required_status_checks": [
          { "context": "ci/lint" },
          { "context": "ci/test" },
          { "context": "ci/build" }
        ],
        "strict_required_status_checks_policy": true
      }
    },
    { "type": "merge_queue",
      "parameters": {
        "merge_method": "squash",
        "min_entries_to_merge": 1,
        "max_entries_to_merge": 5,
        "check_response_timeout_minutes": 30
      }
    }
  ],
  "bypass_actors": [
    { "actor_type": "DeployKey", "bypass_mode": "always" }
  ]
}
```

### 7.3 GitHub Actions for Branch Management

**Auto-label PRs by changed files:**
```yaml
# .github/workflows/labeler.yml
name: Label PRs
on: [pull_request_target]
jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/labeler@v5
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}

# .github/labeler.yml
framework:
  - changed-files:
    - any-glob-to-any-file: '.sinapse-ai/**'
docs:
  - changed-files:
    - any-glob-to-any-file: 'docs/**'
```

**Auto-assign reviewers:**
```yaml
name: Auto Assign
on: [pull_request]
jobs:
  assign:
    runs-on: ubuntu-latest
    steps:
      - uses: kentaro-m/auto-assign-action@v2
        with:
          configuration-path: '.github/auto-assign.yml'
```

**PR size check:**
```yaml
name: PR Size Check
on: [pull_request]
jobs:
  size:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check PR size
        run: |
          ADDITIONS=$(gh pr view ${{ github.event.number }} --json additions -q '.additions')
          DELETIONS=$(gh pr view ${{ github.event.number }} --json deletions -q '.deletions')
          TOTAL=$((ADDITIONS + DELETIONS))
          if [ "$TOTAL" -gt 600 ]; then
            echo "::warning::PR has $TOTAL changed lines. Consider splitting."
          fi
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 7.4 Environments

GitHub Environments provide deployment targets with protection rules:

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy-staging:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to staging"

  deploy-production:
    needs: deploy-staging
    environment:
      name: production
      url: https://myapp.com
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to production"
```

**Environment protection rules:**
- Required reviewers (N people must approve the deployment)
- Wait timer (delay deployment by N minutes)
- Deployment branches (only allow deployments from specific branches)
- Environment secrets (secrets scoped to the environment)

### 7.5 GitHub CLI (gh)

Essential commands for workflow automation:

```bash
# PRs
gh pr create --title "feat: X" --body "desc" --base main
gh pr list --state open
gh pr view 123
gh pr merge 123 --squash --auto
gh pr review 123 --approve
gh pr checks 123

# Issues
gh issue create --title "Bug: X" --body "desc" --label bug
gh issue list --assignee @me
gh issue close 123

# Releases
gh release create v1.0.0 --generate-notes
gh release list

# Actions
gh run list
gh run view 12345
gh run rerun 12345

# Repository
gh repo clone owner/repo
gh repo view --web
gh api repos/{owner}/{repo}/pulls/123/comments
```

### 7.6 Dependabot

Automated dependency updates:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 10
    reviewers:
      - "caio-imori"
    labels:
      - "dependencies"
    groups:
      eslint:
        patterns:
          - "eslint*"
          - "@typescript-eslint/*"
      testing:
        patterns:
          - "jest*"
          - "@testing-library/*"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### 7.7 Secret Scanning and Code Scanning

**Secret scanning** (enabled by default on public repos, available for private repos with GitHub Advanced Security):
- Scans for tokens from 100+ service providers
- Alerts when secrets are found in commits
- Push protection: blocks pushes containing secrets BEFORE they reach the repo

**Code scanning** (CodeQL):
```yaml
# .github/workflows/codeql.yml
name: CodeQL Analysis
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Weekly Monday 6am

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    strategy:
      matrix:
        language: ['javascript-typescript']
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
      - uses: github/codeql-action/analyze@v3
```

---

## 8. Git Hooks & Automation

### 8.1 Client-Side Hooks

Git hooks are scripts that run automatically at specific points in the Git workflow. They live in `.git/hooks/`.

| Hook              | Trigger                         | Common Use                              |
|-------------------|---------------------------------|-----------------------------------------|
| `pre-commit`      | Before commit is created        | Lint, format, test staged files         |
| `prepare-commit-msg` | After default msg, before editor | Add ticket ID to message             |
| `commit-msg`      | After message entered           | Validate message format (commitlint)    |
| `post-commit`     | After commit is created         | Notifications, logging                  |
| `pre-push`        | Before push to remote           | Full test suite, branch name validation |
| `post-checkout`   | After checkout/switch           | Install deps, rebuild                   |
| `pre-rebase`      | Before rebase starts            | Warn if rebasing shared branch          |
| `post-merge`      | After merge completes           | Install deps if package.json changed    |
| `pre-auto-gc`     | Before automatic gc             | Rarely used                             |

### 8.2 Husky v9 Setup (Complete)

```bash
# 1. Install
npm install --save-dev husky

# 2. Initialize
npx husky init
# Creates .husky/ directory with sample pre-commit hook

# 3. Pre-commit hook (lint-staged)
echo 'npx lint-staged' > .husky/pre-commit

# 4. Commit-msg hook (commitlint)
echo 'npx --no -- commitlint --edit "$1"' > .husky/commit-msg

# 5. Pre-push hook (tests + branch validation)
cat > .husky/pre-push << 'EOF'
npm test
BRANCH=$(git rev-parse --abbrev-ref HEAD)
if [ "$BRANCH" = "main" ]; then
  echo "ERROR: Direct push to main is not allowed."
  exit 1
fi
EOF
```

**Directory structure after setup:**
```
.husky/
  _/
    husky.sh        # Bootstrapper (auto-generated)
  pre-commit        # Runs lint-staged
  commit-msg        # Runs commitlint
  pre-push          # Runs tests + branch validation
```

### 8.3 Lefthook (Alternative to Husky)

Lefthook is a Git hooks manager written in Go. It is faster than Husky due to parallel execution and compiled binary.

```yaml
# lefthook.yml
pre-commit:
  parallel: true
  commands:
    lint:
      glob: "*.{js,ts,tsx}"
      run: npx eslint --fix {staged_files}
      stage_fixed: true
    format:
      glob: "*.{js,ts,tsx,json,md,yml}"
      run: npx prettier --write {staged_files}
      stage_fixed: true
    typecheck:
      run: npx tsc --noEmit

commit-msg:
  commands:
    validate:
      run: npx commitlint --edit {1}

pre-push:
  commands:
    test:
      run: npm test
    branch-check:
      run: |
        BRANCH=$(git rev-parse --abbrev-ref HEAD)
        if [ "$BRANCH" = "main" ]; then
          echo "Direct push to main blocked."
          exit 1
        fi
```

**Husky vs Lefthook comparison:**

| Feature           | Husky v9           | Lefthook           |
|-------------------|--------------------|--------------------|
| Language          | JavaScript/Shell    | Go binary          |
| Execution         | Sequential          | Parallel           |
| Configuration     | Shell scripts       | YAML               |
| Performance       | Good                | Better (50%+ faster)|
| Node.js required  | YES                | NO                 |
| Monorepo support  | Manual              | Built-in           |
| Staged files      | Via lint-staged     | Built-in `{staged_files}` |
| Community         | Larger              | Growing            |

### 8.4 Custom Hooks for SINAPSE-Specific Needs

**Secret scanning hook (pre-commit):**
```bash
#!/bin/bash
# Scan staged files for secrets

PATTERNS=(
  'SUPABASE_SERVICE_ROLE_KEY'
  'sk-[a-zA-Z0-9]{20,}'
  'ghp_[a-zA-Z0-9]{36}'
  'npm_[a-zA-Z0-9]{36}'
  'PRIVATE KEY'
  'password\s*=\s*["\x27][^"\x27]+'
)

FILES=$(git diff --cached --name-only --diff-filter=ACM)

for pattern in "${PATTERNS[@]}"; do
  MATCHES=$(echo "$FILES" | xargs grep -lnE "$pattern" 2>/dev/null)
  if [ -n "$MATCHES" ]; then
    echo "BLOCKED: Potential secret detected matching pattern: $pattern"
    echo "Files: $MATCHES"
    exit 1
  fi
done
```

**Branch naming enforcement (pre-push):**
```bash
#!/bin/bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
VALID="^(caio|soier|dev)/(feat|fix|hotfix|chore|docs|refactor|test|ci|release)/.+$"

if [[ "$BRANCH" == "main" ]]; then
  echo "BLOCKED: Direct push to main. Use a PR."
  exit 1
fi

if [[ ! "$BRANCH" =~ $VALID ]]; then
  echo "BLOCKED: Branch '$BRANCH' does not follow naming convention."
  echo "Expected: <user>/<type>/<description>"
  echo "Example: caio/feat/new-feature"
  exit 1
fi
```

**Story reference in commits (commit-msg):**
```bash
#!/bin/bash
MSG_FILE=$1
MSG=$(cat "$MSG_FILE")

# Allow merge commits and reverts
if echo "$MSG" | grep -qE "^(Merge|Revert)"; then
  exit 0
fi

# Check for conventional commit format
if ! echo "$MSG" | grep -qE "^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?!?: .+"; then
  echo "BLOCKED: Commit message does not follow Conventional Commits."
  echo "Expected: <type>(<scope>): <description>"
  echo "Got: $MSG"
  exit 1
fi
```

---

## 9. Conflict Resolution & Recovery

### 9.1 Merge Strategies

Git provides several merge strategies:

| Strategy             | Command                      | When to Use                          |
|----------------------|------------------------------|--------------------------------------|
| **recursive** (default) | `git merge`               | Standard merge, handles renames      |
| **ort** (Git 2.33+)  | `git merge` (new default)   | Faster recursive, better renames     |
| **ours**             | `git merge -s ours`          | Keep our version entirely            |
| **theirs**           | `git merge -X theirs`        | Keep their version on conflicts      |
| **octopus**          | `git merge A B C`            | Merge 3+ branches (no conflicts)     |

**Conflict resolution commands:**
```bash
# See conflicted files
git status

# See all conflicts with markers
git diff --check

# Accept ours for a file
git checkout --ours path/to/file

# Accept theirs for a file
git checkout --theirs path/to/file

# Use mergetool
git mergetool

# After resolving all conflicts
git add .
git commit
```

### 9.2 git rerere (Reuse Recorded Resolution)

`rerere` = "reuse recorded resolution." It remembers how you resolved conflicts and automatically applies the same resolution if the same conflict occurs again.

```bash
# Enable globally
git config --global rerere.enabled true

# How it works:
# 1. First time you resolve a conflict, git records the resolution
# 2. Next time the SAME conflict appears (e.g., during rebase), git auto-resolves it
# 3. You still need to verify and `git add`, but the resolution is pre-applied

# See recorded resolutions
git rerere status

# Forget a specific resolution
git rerere forget path/to/file

# Clear all recorded resolutions
git rerere gc
```

This is particularly valuable for teams that frequently rebase feature branches -- the same conflicts that appear during rebase get auto-resolved.

### 9.3 Stash

Git stash temporarily saves uncommitted changes:

```bash
# Stash current changes (tracked files)
git stash

# Stash with a descriptive message
git stash push -m "WIP: auth module refactor"

# Stash including untracked files
git stash push -u -m "WIP: including new files"

# List stashes
git stash list
# stash@{0}: On feature/auth: WIP: auth module refactor
# stash@{1}: On main: WIP: config changes

# Apply most recent stash (keeps it in stash list)
git stash apply

# Apply specific stash
git stash apply stash@{1}

# Apply and remove from stash list
git stash pop

# Drop a specific stash
git stash drop stash@{1}

# Create a branch from stash (useful when stash conflicts with current branch)
git stash branch new-branch-name stash@{0}
```

### 9.4 Reflog for Recovery

The reflog is git's "undo history." Every move of HEAD or a branch tip is recorded.

**Common recovery scenarios:**

```bash
# 1. Recover from bad rebase
git reflog
# Find the commit BEFORE the rebase started
git reset --hard HEAD@{5}

# 2. Recover deleted branch
git reflog
# Find the last commit on the deleted branch
git checkout -b recovered-branch abc1234

# 3. Recover from bad reset
git reflog
git reset --hard HEAD@{2}

# 4. Find a lost commit (e.g., after amend)
git reflog
# The original commit (before amend) is still there
git cherry-pick abc1234

# 5. Recover from accidental force push (local reflog)
git reflog origin/main
git push origin HEAD@{1}:main --force-with-lease
```

### 9.5 git bisect for Bug Hunting

```bash
# Manual bisect
git bisect start
git bisect bad HEAD           # Current state is broken
git bisect good v1.0.0        # This version was working
# Git checks out middle commit
# Test it...
git bisect good               # This commit is fine
# Git narrows the range...
git bisect bad                 # This commit is broken
# Continue until found
git bisect reset               # Return to original state

# Automated bisect
git bisect start HEAD v1.0.0
git bisect run npm test
# Git runs tests on each candidate, finds the breaking commit automatically

# Skip a commit that can't be tested
git bisect skip

# View bisect log
git bisect log
```

### 9.6 Cherry-Pick for Selective Recovery

```bash
# Apply a single commit from another branch
git cherry-pick abc1234

# Apply multiple commits
git cherry-pick abc1234 def5678

# Apply a range (exclusive start, inclusive end)
git cherry-pick abc1234..ghi9012

# Cherry-pick without committing (stage only)
git cherry-pick --no-commit abc1234

# If conflicts arise during cherry-pick
git cherry-pick --continue   # After resolving
git cherry-pick --abort      # Cancel
```

### 9.7 Disaster Recovery Playbook

| Disaster                      | Recovery Command                                        |
|-------------------------------|---------------------------------------------------------|
| Committed to wrong branch     | `git reset HEAD~1` then checkout correct branch          |
| Pushed secrets to remote      | Rotate secrets immediately, `git filter-repo`, force push|
| Accidentally deleted branch   | `git reflog` + `git checkout -b name SHA`                |
| Bad merge to main             | `git revert -m 1 MERGE_SHA` (creates revert commit)     |
| Lost commits after rebase     | `git reflog` + `git reset --hard HEAD@{N}`               |
| Force push overwrote remote   | Find SHA from collaborator or reflog, force push back    |
| Detached HEAD with changes    | `git checkout -b save-my-work` (creates branch from HEAD)|
| Corrupted repository          | `git fsck` to find issues, re-clone as last resort       |

**Critical: Force push recovery**
```bash
# If someone force-pushed and overwrote your commits:
# 1. Check your local reflog
git reflog origin/main

# 2. Find the commit before the force push
# 3. Force push the correct state back
git push origin HEAD@{1}:main --force-with-lease

# Prevention: ALWAYS use --force-with-lease instead of --force
# It fails if the remote has commits you have not fetched
```

---

## 10. Key People, Books & Sources

### 10.1 Key People

| Person                | Contribution                                                    |
|-----------------------|-----------------------------------------------------------------|
| **Linus Torvalds**    | Creator of Git (2005), designed the DAG object model            |
| **Junio C Hamano**    | Git maintainer since 2005, responsible for most of Git's evolution |
| **Scott Chacon**      | Co-founder of GitHub, author of Pro Git, created GitHub Flow    |
| **Vincent Driessen**  | Created Git Flow (2010), the first formal branching model       |
| **Martin Fowler**     | Feature Flags, Continuous Integration advocacy                  |
| **Jez Humble**        | Co-author of Continuous Delivery, DORA metrics co-creator       |
| **Nicole Forsgren**   | Lead researcher of DORA/Accelerate, PhD                         |
| **Gene Kim**          | Co-author of Accelerate, The Phoenix Project, DevOps advocate   |
| **Paul Hammant**      | Trunk-based development advocate, trunkbaseddevelopment.com     |
| **Adam Ruka**         | Created OneFlow as Git Flow alternative                         |
| **Rouan Wilsenach**   | Created Ship/Show/Ask pattern (2020)                            |
| **Derrick Stolee**    | Microsoft, Git performance (sparse-checkout, partial clone)     |

### 10.2 Essential Books

| Book                                        | Author(s)                     | Key Topic                        |
|---------------------------------------------|-------------------------------|----------------------------------|
| **Pro Git** (free online)                   | Scott Chacon, Ben Straub      | Comprehensive Git reference      |
| **Accelerate**                              | Forsgren, Humble, Kim         | DORA metrics, elite performance  |
| **Continuous Delivery**                     | Jez Humble, David Farley      | CD pipeline, trunk-based dev     |
| **The Phoenix Project**                     | Gene Kim et al.               | DevOps narrative                 |
| **Release It!**                             | Michael Nygard                | Production resilience            |
| **Git Internals** (free PDF)                | Scott Chacon                  | Deep dive into Git's object model|
| **Version Control with Git** (3rd ed.)      | Prem Kumar Ponuthorai         | O'Reilly, practical Git          |

### 10.3 Key Online Sources

| Source                                       | URL                                                    | What It Covers                |
|----------------------------------------------|--------------------------------------------------------|-------------------------------|
| Pro Git Book                                 | https://git-scm.com/book/en/v2                          | Everything Git                |
| Google Eng Practices                         | https://google.github.io/eng-practices/                 | Code review guidelines        |
| Conventional Commits                         | https://www.conventionalcommits.org/en/v1.0.0/          | Commit message spec           |
| Trunk Based Development                      | https://trunkbaseddevelopment.com/                      | TBD patterns and practices    |
| DORA Reports                                 | https://dora.dev/                                       | DevOps performance research   |
| Atlassian Git Tutorials                      | https://www.atlassian.com/git/tutorials                 | Git workflows explained       |
| GitHub Docs                                  | https://docs.github.com/                                | GitHub features reference     |
| Graphite Blog                                | https://graphite.com/blog/                              | Stacked PRs, PR size research |
| Semantic Release                             | https://github.com/semantic-release/semantic-release    | Automated versioning          |
| Git Flow Original Post                       | https://nvie.com/posts/a-successful-git-branching-model/| Vincent Driessen's model      |

### 10.4 DORA Metrics Reference (2024 Report)

The DORA (DevOps Research and Assessment) framework measures software delivery performance across four key metrics, with a fifth added in 2025:

| Metric                    | Elite              | High               | Medium             | Low                |
|---------------------------|--------------------|---------------------|--------------------|---------------------|
| Deployment Frequency      | On-demand (multi/day)| Weekly-Monthly    | Monthly-Biannually | Biannually-Annually|
| Lead Time for Changes     | < 1 day            | 1 day - 1 week     | 1 week - 1 month  | 1-6 months          |
| Change Failure Rate       | 0-5%               | 5-10%               | 10-15%             | 15-64%              |
| Time to Restore Service   | < 1 hour           | < 1 day             | 1 day - 1 week    | 1 week - 1 month   |
| **Rework Rate** (new 2025)| Low                | --                  | --                 | High                |

**Key 2024 finding:** Elite performers are 2x more likely to exceed organizational goals related to profitability, productivity, and customer satisfaction. AI adoption improves throughput but increases delivery instability (DORA 2025).

(Sources: Octopus Deploy DORA guide, DORA Report 2024, RedMonk analysis)

---

## Sources

### Web Sources Consulted
- [Conventional Commits Specification](https://www.conventionalcommits.org/en/v1.0.0/)
- [DORA Metrics -- Octopus Deploy](https://octopus.com/devops/metrics/dora-metrics/)
- [DORA Report 2024 -- RedMonk](https://redmonk.com/rstephens/2024/11/26/dora2024/)
- [The Ideal PR is 50 Lines Long -- Graphite](https://graphite.com/blog/the-ideal-pr-is-50-lines-long)
- [PR Size Impact on Code Review Quality -- Propel](https://www.propelcode.ai/blog/pr-size-impact-code-review-quality-data-study)
- [Trunk-Based Development vs Git Flow -- Toptal](https://www.toptal.com/developers/software/trunk-based-development-git-flow)
- [Trunk-Based Development vs Gitflow -- Mergify](https://mergify.com/blog/trunk-based-development-vs-gitflow-which-branching-model-actually-works/)
- [GitHub Merge Queue Docs](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue)
- [GitHub Rulesets Docs](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets)
- [Google Code Review Speed](https://google.github.io/eng-practices/review/reviewer/speed.html)
- [Google Code Review Standard](https://google.github.io/eng-practices/review/reviewer/standard.html)
- [Code Reviews at Google -- Dr. Michaela Greiler](https://www.michaelagreiler.com/code-reviews-at-google/)
- [Semantic Release -- GitHub](https://github.com/semantic-release/semantic-release)
- [Husky + lint-staged Guide 2025 -- DEV Community](https://dev.to/_d7eb1c1703182e3ce1782/git-hooks-with-husky-and-lint-staged-the-complete-setup-guide-for-2025-53ji)
- [Lefthook vs Husky -- DEV Community](https://dev.to/quave/lefthook-benefits-vs-husky-and-how-to-use-30je)
- [Lefthook vs Husky -- Edopedia](https://www.edopedia.com/blog/lefthook-vs-husky/)
- [Stacked PRs -- Graphite](https://graphite.com/blog/stacked-prs)
- [Stacked PRs Guide -- Awesome Code Reviews](https://www.awesomecodereviews.com/best-practices/stacked-prs/)
- [Git Internals -- Git SCM Book](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [Git Internals Deep Dive -- DEV Community](https://dev.to/__whyd_rf/a-deep-dive-into-git-internals-blobs-trees-and-commits-1doc)
- [GitHub Actions Workflows -- DevOps Training Institute](https://www.devopstraininginstitute.com/blog/top-15-github-actions-workflows-for-automation)
- [Branch Protection as Code -- TheAIOps Blog](https://theaiops.blog/2025/07/10/Automating_Branch_Protection/)
- [Dependabot Configuration -- GitHub Docs](https://docs.github.com/en/code-security/dependabot)
- [GitHub Secret Scanning -- GitHub Docs](https://docs.github.com/en/code-security/secret-scanning)
- [Conventional Commits Cheatsheet -- GitHub Gist](https://gist.github.com/qoomon/5dfcdf8eec66a051ecd85625518cfd13)
- [PR Size Metric -- EM Tools](https://www.em-tools.io/engineering-metrics/pull-request-size)
- [Microsoft Engineering PR Data -- Propel](https://www.propelcode.ai/blog/pr-size-impact-code-review-quality-data-study)

---

# PART 2 — Advanced Workflows, Ecosystem & Recommendations

---

## 11. CI/CD Integration with Git

### 11.1 Branch-Based Deployments

The dominant pattern in modern CI/CD ties deployment targets directly to Git branches
and events. Each branch maps to an environment, creating a predictable promotion path.

```
Branch/Event            Environment         URL Pattern
─────────────────────────────────────────────────────────────
PR opened/updated  -->  Preview / Ephemeral  pr-42.preview.app.com
develop (push)     -->  Staging              staging.app.com
main (push)        -->  Production           app.com
tag v*.*.* (push)  -->  Release artifact     npm registry / Docker Hub
```

**Preview environments** (Vercel, Netlify, Render, Railway) spin up a full deploy for
every PR. This lets reviewers click a live URL instead of pulling the branch locally.

### 11.2 Feature Flags + Trunk-Based Development

Feature flags decouple **deployment** from **release**. Code ships to production
continuously, but user-facing behavior is toggled independently.

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│  Commit to   │────>│  Deploy to   │────>│  Flag OFF:   │
│  main        │     │  Production  │     │  Hidden from │
│              │     │  (always)    │     │  users       │
└─────────────┘     └──────────────┘     └──────┬───────┘
                                                 │
                                          Flag turned ON
                                                 │
                                          ┌──────▼───────┐
                                          │  Feature      │
                                          │  visible to   │
                                          │  users        │
                                          └──────────────┘
```

**Key providers:** LaunchDarkly, Unleash, Flagsmith, GrowthBook, PostHog, ConfigCat.

Benefits:
- Merge incomplete features without user exposure
- Eliminate long-lived branches and integration hell
- Test in production with percentage rollouts (1% -> 10% -> 50% -> 100%)
- Instant kill switch if metrics degrade

In 2025, developers have shifted from relying solely on staging environments to testing
directly in production with feature flags, reducing integration risk and enabling true
continuous delivery.
(Source: FeatBit 2025 analysis, Harness Feature Flags documentation)

### 11.3 GitHub Actions Workflows

The three critical trigger points for CI/CD:

```yaml
# .github/workflows/ci.yml
name: CI Pipeline
on:
  pull_request:
    branches: [main]          # Run on every PR
  push:
    branches: [main]          # Run on merge to main
    tags: ['v*.*.*']          # Run on version tags

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test -- --coverage

  deploy-preview:
    if: github.event_name == 'pull_request'
    needs: test
    # Deploy to preview environment...

  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: test
    environment: production    # Requires approval
    # Deploy to production...

  publish-npm:
    if: startsWith(github.ref, 'refs/tags/v')
    needs: test
    permissions:
      id-token: write          # OIDC for npm trusted publishing
    # npm publish with provenance...
```

### 11.4 Status Checks as Merge Gates

Required status checks block merging until CI passes. Configure in GitHub:

| Check                  | Purpose                          | Blocking? |
|------------------------|----------------------------------|-----------|
| `test`                 | Unit + integration tests         | YES       |
| `lint`                 | Code style enforcement           | YES       |
| `typecheck`            | TypeScript compilation           | YES       |
| `build`                | Verify production build succeeds | YES       |
| `coverage`             | Minimum coverage threshold       | OPTIONAL  |
| `security/secret-scan` | Detect leaked credentials        | YES       |
| `coderabbit`           | AI code review                   | OPTIONAL  |

### 11.5 Deployment Environments & Approval Gates

GitHub Actions environments support:
- **Required reviewers** -- humans must approve before deploy proceeds
- **Wait timer** -- configurable delay (e.g., 15 minutes for production)
- **Branch restrictions** -- only `main` can deploy to production
- **Environment secrets** -- scoped credentials per environment

```yaml
deploy-production:
  environment:
    name: production
    url: https://app.com
  runs-on: ubuntu-latest
  steps:
    - run: echo "Deploying to production..."
```

### 11.6 Rollback Strategies

| Strategy               | Speed    | Risk     | When to use                    |
|------------------------|----------|----------|--------------------------------|
| `git revert` + redeploy | Minutes | Low      | Bad code merged, need to undo  |
| Deploy previous tag    | Minutes  | Low      | Tag-based releases             |
| Feature flag kill       | Seconds | Lowest   | Flag-gated features            |
| Database rollback      | Variable | High     | Schema migrations (rare)       |
| Blue-green swap        | Seconds  | Low      | Infrastructure-level rollback  |

**Best practice:** Always prefer `git revert` over `git reset --hard` in shared
branches. Revert creates a new commit that undoes changes, preserving history.

```bash
# Revert the last merge commit
git revert -m 1 HEAD
git push origin main
# CI automatically deploys the reverted state
```

### 11.7 GitOps -- Git as Source of Truth for Infrastructure

GitOps extends the Git workflow to infrastructure and Kubernetes deployments.
The desired state of the entire system lives in Git; operators reconcile
the live cluster to match.

**ArgoCD vs Flux comparison (2025):**

| Dimension        | ArgoCD                          | Flux                            |
|------------------|---------------------------------|---------------------------------|
| Architecture     | Centralized server + UI         | Kubernetes-native controllers   |
| UI               | Rich built-in web dashboard     | Third-party (Weave GitOps)      |
| Multi-cluster    | Native support                  | Requires additional setup       |
| RBAC             | Built-in with SSO               | Kubernetes RBAC                 |
| Complexity       | Higher (more features)          | Lower (composable CRDs)         |
| Best for         | Platform teams, large orgs      | K8s-native teams, minimal setup |
| Adoption (2025)  | Market leader (~65% share)      | Strong in CNCF ecosystem        |

(Sources: Zignuts 2025 comparison, Spacelift analysis, Akuity GitOps best practices)

**GitOps flow:**

```
Developer commits K8s manifests to git
        │
        ▼
  Git repository (source of truth)
        │
        ▼
  ArgoCD / Flux watches repo
        │
        ▼
  Detects drift from desired state
        │
        ▼
  Automatically reconciles cluster
        │
        ▼
  Cluster matches git state
```

---

## 12. Monorepo Git Strategies

### 12.1 Monorepo vs Polyrepo -- Data-Driven Tradeoffs

Benchmark data from Faros AI across 320 engineering teams reveals significant
differences:

| Metric                    | Monorepo          | Polyrepo          |
|---------------------------|-------------------|-------------------|
| Median PR cycle time      | 19 hours          | 2 hours           |
| 90th percentile PR cycle  | 5-10+ days        | Tighter, more predictable |
| CI build time (optimized) | 18% faster*       | Baseline          |
| Coordination overhead     | Low               | High              |
| Tooling complexity        | High              | Low               |

*Teams migrating from polyrepo to well-tooled monorepo saw 18% reduction in CI build
times due to optimized caching and atomic dependency management.
(Source: Faros AI benchmark data, Developers.dev internal data 2024-2025)

**The fundamental trade:** Monorepos trade high tooling complexity for low coordination
cost. Polyrepos trade low tooling complexity for high coordination cost. The question
is where you want to pay the complexity tax.

**Who uses monorepos:** Google (billions of lines), Meta, Microsoft (Windows), Uber,
Airbnb, Twitter/X, Stripe.

**Who uses polyrepos:** Amazon (service-oriented), Netflix (microservices-first),
Spotify (autonomous squads).

**Hybrid approach (most common in 2025):** Centralized monorepo for core libraries and
shared services + polyrepos for highly decoupled, independent applications.

### 12.2 Monorepo Tooling Comparison (2026)

| Tool      | Language   | Philosophy            | Best For           | Cache     |
|-----------|------------|-----------------------|--------------------|-----------|
| Turborepo | Rust       | Minimal task runner   | 5-15 packages      | Remote    |
| Nx        | TS -> Rust | Full framework        | 15-200+ packages   | Nx Cloud  |
| Lerna     | TS         | Publishing focus      | npm workspace mgmt | Limited   |
| Moon      | Rust       | Language-agnostic     | Multi-language      | Built-in  |
| Bazel     | Java/Starlark | Hermetic builds    | 500+ packages      | Remote    |

**Performance benchmarks (2025-2026):**
- Turborepo: 3x faster than Nx on 10-package test projects
- Nx: 7x faster than Turborepo in large-scale (100+) open-source benchmarks
- Verdict: Turborepo wins on simplicity; Nx wins on scale
- Both migrating to Rust: Turborepo completed (2023), Nx targeting 2025

(Sources: PkgPulse 2026 comparison, vsavkin/large-monorepo benchmark, daily.dev analysis)

**Recommendation for small-medium projects (< 15 packages):**

```jsonc
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "test": {
      "dependsOn": ["build"]
    },
    "lint": {},
    "typecheck": {}
  }
}
```

### 12.3 Sparse Checkout for Monorepos

When a developer only works on one package in a large monorepo:

```bash
# Clone without checking out files
git clone --no-checkout --filter=blob:none https://github.com/org/monorepo.git
cd monorepo

# Initialize sparse checkout
git sparse-checkout init --cone

# Only check out the packages you need
git sparse-checkout set packages/my-app packages/shared-lib

# Result: only those directories are populated
```

**Performance impact (2025 data):**
- Chromium repo (60.9 GB): standard clone 95 min -> optimized 6 min 41 sec (93% faster)
- GitLab website (8.9 GB): standard clone 6 min 23 sec -> optimized 6.49 sec (98.3% faster)
(Source: Medium Zoe analysis 2025, GitHub blog on sparse checkout)

### 12.4 CODEOWNERS per Package

```
# .github/CODEOWNERS
# Global
*                           @org/core-team

# Per-package ownership
packages/api/               @org/backend-team
packages/web/               @org/frontend-team
packages/shared-lib/        @org/platform-team
packages/cli/               @caio-imori
docs/                       @org/docs-team
```

### 12.5 Affected-Only CI

Only test and build packages that actually changed:

```yaml
# With Turborepo
- run: npx turbo run test --filter='...[origin/main]'

# With Nx
- run: npx nx affected --target=test --base=origin/main

# With custom script (package.json workspaces)
- run: |
    CHANGED=$(git diff --name-only origin/main...HEAD | grep '^packages/' | cut -d/ -f2 | sort -u)
    for pkg in $CHANGED; do
      npm -w packages/$pkg test
    done
```

---

## 13. Solo Developer Workflow

### 13.1 Simplified Branching

Solo developers do not need GitFlow, GitHub Flow, or any formal branching strategy.
The minimal effective workflow:

```
main (always deployable)
  │
  ├── feat/new-feature    (short-lived, merge back quickly)
  ├── fix/broken-thing    (hotfix, merge same day)
  └── experiment/idea     (exploratory, may be abandoned)
```

**Rules for solo:**
1. `main` is always deployable
2. Feature branches for anything that takes > 1 commit
3. Direct commits to main for trivial fixes (typos, config tweaks)
4. No PRs required (but useful for self-review of complex changes)
5. Tag releases with semantic versions

### 13.2 When Solo Is Enough vs When to Add Complexity

| Signal                                   | Action                        |
|------------------------------------------|-------------------------------|
| Single developer, single project         | main + feature branches       |
| Need to track releases                   | Add version tags              |
| Users report bugs in specific versions   | Add changelog                 |
| Second developer joins                   | Add PRs + branch protection   |
| CI/CD pipeline added                     | Add required status checks    |
| Publishing to npm / deploying SaaS       | Add release automation        |
| Team grows to 3+                         | Consider GitHub Flow formally |

### 13.3 Self-Review Practices

Even solo, reviewing your own code catches bugs:

```bash
# Before merging, review the full diff
git diff main..feat/my-feature

# Use git log to see commit narrative
git log --oneline main..feat/my-feature

# Let the PR sit for a few hours ("fresh eyes" effect)
# GitHub Drafts are perfect for this
gh pr create --draft --title "feat: new feature" --body "Self-review PR"
```

**Automated CI for solo developers:**
- Pre-commit hooks (lint, format, typecheck)
- GitHub Actions on push (test, build)
- Dependabot for dependency updates
- CodeQL for security scanning

### 13.4 Version Tagging & Changelog Generation

**Tool comparison for automated releases:**

| Tool                        | Approach                | Best For               |
|-----------------------------|-------------------------|------------------------|
| `semantic-release`          | Fully automated from CI | npm packages, SaaS     |
| `commit-and-tag-version`    | Local control, no push  | Solo devs who want review |
| `auto-changelog`            | Generate from git tags  | Simple projects        |
| `conventional-changelog-cli`| CLI-based generation    | Custom workflows       |
| `release-please` (Google)   | GitHub App, Release PRs | Google ecosystem       |
| `changesets`                 | Monorepo-friendly       | Multi-package repos    |

(Sources: conventional-changelog GitHub, standard-version GitHub, cookpete/auto-changelog)

**Recommended solo workflow with `commit-and-tag-version`:**

```bash
# 1. Write code with conventional commits
git commit -m "feat: add dark mode support"
git commit -m "fix: correct color contrast ratio"

# 2. When ready to release
npx commit-and-tag-version
# Bumps version in package.json
# Updates CHANGELOG.md
# Creates git tag

# 3. Review the release locally
git log --oneline -5
cat CHANGELOG.md | head -30

# 4. Push when satisfied
git push --follow-tags origin main
```

**Fully automated with `semantic-release`:**

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## 14. Small Team Collaboration (2-5 People)

### 14.1 Recommended Workflow: GitHub Flow with Conventions

For teams of 2-5, GitHub Flow with added conventions strikes the right balance
between safety and speed:

```
main (protected, requires PR + 1 approval)
  │
  ├── caio/feat/installer-ux     (Caio's feature)
  ├── soier/fix/agent-config     (Matheus's fix)
  ├── dev/feat/new-api           (shared feature branch)
  └── release/v9.3.0             (release candidate, if needed)
```

**Branch protection rules (minimum for small teams):**
- Require PR before merging
- Require 1 approval (the other person)
- Require status checks to pass
- Require branches to be up to date before merging
- Require linear history (squash or rebase, no merge commits)

### 14.2 Communication Patterns

| Integration           | Purpose                              | Tool              |
|-----------------------|--------------------------------------|--------------------|
| PR notifications      | Alert reviewer when PR is ready      | GitHub + Slack bot |
| Deploy notifications  | Alert team when deploy happens       | GitHub Actions     |
| Review reminders      | Nudge if review sits > 4 hours       | GitHub Actions     |
| Conflict alerts       | Warn when two people edit same file  | CODEOWNERS + bot   |
| Daily standup         | Async status via PR descriptions     | GitHub Projects    |

**GitHub Actions for review reminders:**

```yaml
name: Review Reminder
on:
  schedule:
    - cron: '0 14 * * 1-5'  # 2 PM weekdays
jobs:
  remind:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const prs = await github.rest.pulls.list({
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: 'open',
            });
            for (const pr of prs.data) {
              const reviews = await github.rest.pulls.listReviews({...});
              if (reviews.data.length === 0) {
                // Send Slack notification or add comment
              }
            }
```

### 14.3 Code Ownership Without Bureaucracy

For small teams, heavy CODEOWNERS files create bottlenecks. Instead:

| Pattern                    | Implementation                     |
|----------------------------|------------------------------------|
| Soft ownership             | "Caio usually handles CLI, Matheus handles API" (informal) |
| Review rotation            | Alternate who reviews (avoid bus factor = 1) |
| Knowledge sharing PRs      | Tag the other person on unfamiliar areas |
| Pair review for critical   | Both review security, auth, data model changes |

### 14.4 Release Management for Small Teams

| Release Style       | When to Use                      | Complexity |
|---------------------|----------------------------------|------------|
| Continuous (no tags)| SaaS, internal tools             | Lowest     |
| Tag-based releases  | npm packages, libraries          | Low        |
| Release branches    | Multiple versions in production  | Medium     |
| Release trains      | Scheduled releases (weekly)      | Medium     |

### 14.5 The SINAPSE Case: 2 Humans + AI Agents

SINAPSE presents a unique collaboration challenge:

```
Collaborators:
  - Caio Imori (product builder, not git expert)
  - Matheus Soier (developer)
  - 186 AI agents (autonomous, create branches, commit code)
  - AI agents organized in 18 specialized squads

Git challenges:
  1. AI agents need deterministic branch naming (avoid collisions)
  2. AI agents must never push to main directly
  3. Human developers need git complexity abstracted away
  4. Multiple agents may work on overlapping files
  5. Commit attribution must distinguish human from AI
  6. Secret scanning is critical (AI may inadvertently include tokens)
```

The current SINAPSE `safe-collaboration` rules address these:
- Auto-branch with prefix (caio/, soier/, dev/)
- Auto-sync (fetch + pull on session start)
- Auto-resolve (simple conflicts handled by agent)
- Auto-PR (with reviewer assignment)
- Secret scanning before every commit
- Users never run git commands manually

---

## 15. Open Source Project Workflow

### 15.1 The Forking Model

The standard open source contribution workflow:

```
Contributor                          Maintainer
──────────                           ──────────
1. Fork repo on GitHub
2. Clone fork locally
3. Create feature branch
4. Make changes + commits
5. Push to fork
6. Open PR from fork -> upstream   -->  7. Review PR
                                         8. Request changes (optional)
9. Push fixes                      -->  10. Approve + merge
                                         11. Delete fork branch (optional)
```

**Advantages:**
- Contributors never need write access to the official repo
- Maintainers control what enters the codebase
- Scale to thousands of contributors (Linux kernel model)

(Sources: Atlassian forking workflow guide, GitHub Docs contributing to projects)

### 15.2 CONTRIBUTING.md Best Practices

Essential sections:

```markdown
# Contributing to ProjectName

## Quick Start
1. Fork the repository
2. Create your branch: `git checkout -b feat/my-feature`
3. Make your changes
4. Run tests: `npm test`
5. Commit with conventional format: `git commit -m "feat: add feature"`
6. Push to your fork: `git push origin feat/my-feature`
7. Open a Pull Request

## Development Setup
- Node.js >= 22
- npm >= 10
- Run `npm install` to install dependencies

## Coding Standards
- We use ESLint + Prettier (run `npm run lint`)
- All code must pass TypeScript strict mode
- 80%+ test coverage required for new code

## Commit Convention
We follow [Conventional Commits](https://conventionalcommits.org):
- `feat:` New feature
- `fix:` Bug fix
- `docs:` Documentation only
- `chore:` Maintenance

## Pull Request Guidelines
- Keep PRs focused (one feature/fix per PR)
- Target 50-200 lines changed
- Include tests for new functionality
- Update relevant documentation
- Link related issues: `Closes #123`

## Code of Conduct
[Link to CODE_OF_CONDUCT.md]
```

### 15.3 Issue & PR Templates

**Issue templates (`.github/ISSUE_TEMPLATE/bug_report.yml`):**

```yaml
name: Bug Report
description: Report a bug
labels: [bug, triage]
body:
  - type: textarea
    id: description
    attributes:
      label: Describe the bug
    validations:
      required: true
  - type: textarea
    id: reproduction
    attributes:
      label: Steps to reproduce
    validations:
      required: true
  - type: input
    id: version
    attributes:
      label: Version
    validations:
      required: true
  - type: dropdown
    id: os
    attributes:
      label: Operating System
      options: [Windows, macOS, Linux]
```

**Labels taxonomy:**

| Category   | Labels                                       |
|------------|----------------------------------------------|
| Type       | `bug`, `feature`, `enhancement`, `docs`      |
| Priority   | `P0-critical`, `P1-high`, `P2-medium`, `P3-low` |
| Status     | `triage`, `confirmed`, `in-progress`, `blocked` |
| Effort     | `good-first-issue`, `help-wanted`, `complex` |
| Area       | `cli`, `core`, `agents`, `docs`, `ci`        |

### 15.4 CLA (Contributor License Agreement)

**CLA-bot options:**
- **CLA Assistant** (GitHub App) -- most popular, free
- **CLA Lite** -- lightweight GitHub Action
- **DCO (Developer Certificate of Origin)** -- simpler alternative, sign-off per commit

```
# DCO: Contributors sign off each commit
git commit -s -m "feat: add new feature"
# Adds: Signed-off-by: Name <email>
```

### 15.5 Release Management for Open Source

```
Commit to main
     │
     ▼
semantic-release / release-please runs
     │
     ├── Determines version bump (major/minor/patch)
     ├── Updates CHANGELOG.md
     ├── Creates GitHub Release with notes
     ├── Creates git tag (v9.3.0)
     └── Publishes to npm registry
```

### 15.6 npm Publish with Trusted Publishing (2025+)

As of July 2025, npm supports OIDC trusted publishing, eliminating long-lived tokens.
Classic npm tokens were permanently deprecated on December 9, 2025.

```yaml
# .github/workflows/publish.yml
name: Publish to npm
on:
  push:
    tags: ['v*.*.*']
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write          # Required for OIDC
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci
      - run: npm test
      - run: npm publish
        # No NPM_TOKEN needed -- OIDC handles auth
        # Provenance is generated automatically
```

**Requirements:**
- npm CLI >= 11.5.1, Node >= 22.14.0
- Configure trust relationship in npm web UI (link GitHub repo to npm package)
- `id-token: write` permission in workflow

**Limitations:**
- Private repositories do not generate provenance attestations
- Only GitHub Actions and GitLab CI/CD supported as trusted publishers

(Sources: npm Docs trusted publishing, GitHub Changelog July 2025, Socket.dev analysis)

---

## 16. Git for AI-Assisted Development

### 16.1 AI Agents Committing Code

The standard convention for AI-generated commits uses `Co-Authored-By`:

```
feat: implement dark mode toggle

Implement theme switching with system preference detection
and localStorage persistence.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
```

This convention:
- Preserves human accountability (the human is the author)
- Credits the AI tool transparently
- Enables filtering AI-assisted commits in analytics
- Is supported natively by GitHub's contributor graph

### 16.2 Branch Management with Multiple AI Agents

When multiple agents operate in parallel, branch collisions are a real risk:

```
Collision-proof naming scheme:

  {user-prefix}/{type}/{agent-id}-{short-desc}

Examples:
  caio/feat/pixel-dark-mode        (developer agent)
  caio/fix/litmus-test-coverage    (quality gate agent)
  dev/docs/sage-api-reference      (research agent)
  dev/feat/tensor-schema-migration (data engineer agent)
```

**Rules for AI agent branches:**
1. Always include agent ID in branch name (avoid agent-to-agent collision)
2. Never reuse branch names (append timestamp or ticket ID if needed)
3. Always branch from latest main (fetch + pull before branching)
4. One concern per branch (never mix features)
5. Short-lived: merge or close within 24 hours

### 16.3 Review Workflow When AI Generates PRs

```
AI Agent creates branch + commits
        │
        ▼
AI Agent opens Draft PR
        │
        ▼
Automated CI runs (tests, lint, typecheck)
        │
        ▼
AI review pass (CodeRabbit, Graphite Diamond)
        │
        ▼
Human reviews (focus on: intent, architecture, security)
        │
        ▼
Human approves OR requests changes
        │
        ▼
Merge to main
```

**What humans should focus on when reviewing AI code:**
- Does this change match the intended behavior?
- Are there security implications the AI missed?
- Does the code follow project conventions?
- Is the approach aligned with architecture decisions?
- Are there edge cases the AI did not consider?

**What humans can skip (AI is reliable at):**
- Syntax correctness
- Import organization
- Basic formatting
- Test structure (if tests pass)

### 16.4 Git Safety Nets for Autonomous Agents

| Safety Net                    | Implementation                            |
|-------------------------------|-------------------------------------------|
| Branch protection on main     | GitHub branch rules (no direct push)      |
| Required CI checks            | All tests must pass before merge          |
| Secret scanning               | Pre-commit hook + GitHub secret scanning  |
| File path validation          | Hook rejects writes to protected paths    |
| Commit message validation     | commitlint + conventional commits         |
| Max PR size                   | Bot warns if PR > 400 lines               |
| Required human approval       | At least 1 human must approve every PR    |
| Audit trail                   | Co-Authored-By on every AI commit         |

### 16.5 The SINAPSE Model

SINAPSE implements a unique git workflow where:

```
┌─────────────────────────────────────────────────────┐
│                   SINAPSE Git Model                  │
│                                                      │
│  Agents (186)          Humans (2)                    │
│  ┌──────────┐          ┌──────────┐                  │
│  │ @developer│          │ Caio     │                  │
│  │ @architect│          │ Matheus  │                  │
│  │ @analyst  │          │          │                  │
│  │ ...       │          │          │                  │
│  └─────┬────┘          └─────┬────┘                  │
│        │                     │                        │
│   Can: branch, commit       Can: branch, commit      │
│   Cannot: push              Cannot: push (usually)   │
│        │                     │                        │
│        └────────┬────────────┘                        │
│                 │                                     │
│          ┌──────▼──────┐                              │
│          │  @devops     │  EXCLUSIVE push authority   │
│          │  (Pipeline)  │                              │
│          └──────┬──────┘                              │
│                 │                                     │
│          git push + gh pr create                      │
│                 │                                     │
│          ┌──────▼──────┐                              │
│          │   GitHub     │                              │
│          │   (main)     │                              │
│          └─────────────┘                              │
└─────────────────────────────────────────────────────┘
```

**Key design decisions:**
1. **Single push authority** -- Only @devops can push to remote, preventing race conditions
2. **Auto-branch** -- Agents create branches with deterministic naming
3. **Auto-sync** -- Every session starts with `git fetch` + pull
4. **Secret scanning** -- Pre-commit hook blocks credentials
5. **Auto-PR** -- After push, PR is created with reviewer assignment
6. **Human gate** -- Every PR requires human approval before merge

### 16.6 AI Coding Tools and Git (2025-2026)

| Tool         | Git Integration                                | Autonomous? |
|--------------|------------------------------------------------|-------------|
| Claude Code  | Full git access, hooks, sub-agents, Co-Authored-By | Yes         |
| GitHub Copilot Agent | Opens draft PRs from issues            | Yes         |
| OpenAI Codex | Sandbox environment, submits PRs for review   | Yes         |
| Cursor       | Agent mode with terminal access               | Semi        |
| Windsurf     | Cascade agent with file editing               | Semi        |
| Aider        | CLI, auto-commits with conventional format    | Yes         |

GitHub now allows assigning issues to Copilot, Claude, or Codex -- agents submit
draft PRs that fit into existing review workflows.
(Sources: Builder.io Codex vs Claude Code comparison, GitHub Agent HQ blog, Faros AI 2026 review)

### 16.7 Risks of AI-Assisted Git

| Risk                             | Mitigation                                    |
|----------------------------------|-----------------------------------------------|
| Hallucinated file paths          | CI build verification catches missing files   |
| Context window limits            | Agent loses track of changes in large PRs     |
| Merge conflicts from parallel agents | Deterministic branch naming + frequent sync |
| Leaked secrets in generated code | Pre-commit secret scanning hooks              |
| Inconsistent commit messages     | commitlint enforcement                        |
| Over-broad changes               | Max PR size warnings + scope discipline       |
| Wrong base branch                | Hook validates branch target before push      |

---

## 17. Advanced Git Techniques

### 17.1 git worktree (Multiple Working Directories)

Work on multiple branches simultaneously without stashing:

```bash
# Create a worktree for a feature branch
git worktree add ../my-repo-feat feat/new-feature

# Create a worktree for a hotfix
git worktree add ../my-repo-hotfix fix/urgent-bug

# List all worktrees
git worktree list
# /home/user/my-repo              abc1234 [main]
# /home/user/my-repo-feat         def5678 [feat/new-feature]
# /home/user/my-repo-hotfix       ghi9012 [fix/urgent-bug]

# Remove when done
git worktree remove ../my-repo-feat
```

**When to use worktrees:**
- Reviewing a PR while actively working on another branch
- Running long tests on one branch while coding on another
- Comparing behavior between branches side by side
- CI/CD scripts that need multiple branch checkouts

### 17.2 git submodule vs git subtree

| Dimension         | Submodule                       | Subtree                         |
|-------------------|---------------------------------|---------------------------------|
| Storage           | Reference (pointer)             | Full copy in repo               |
| Clone behavior    | Requires `--recurse-submodules` | Works with normal clone         |
| History           | Separate (external repo)        | Merged into main repo           |
| Update workflow   | `git submodule update`          | `git subtree pull`              |
| Learning curve    | Higher                          | Lower                           |
| CI/CD setup       | Extra init step needed          | No extra setup                  |
| Repo size impact  | Minimal                         | Larger (includes full code)     |
| Best for          | Pinned external dependencies    | Vendored libraries              |

**2025 recommendation:** Prefer subtree for most cases. Submodules are warranted only
when you need to pin an exact commit of a frequently-updated external dependency and
team members are comfortable with the additional git workflow.
(Sources: Atlassian git subtree tutorial, GitProtect.io comparison, tiendu.github.io analysis)

### 17.3 git notes & git archive

```bash
# Attach metadata to a commit without changing it
git notes add -m "Reviewed by security team" HEAD

# View notes
git log --show-notes

# Create a tarball of the repo at a specific tag
git archive --format=tar.gz --prefix=myproject-v1.0/ v1.0.0 > release.tar.gz

# Archive only a subdirectory
git archive HEAD packages/cli/ | tar -x -C /tmp/cli-only/
```

### 17.4 Partial Clone, Shallow Clone, Sparse Checkout

Three complementary strategies for large repos:

```
┌─────────────────────────────────────────────┐
│           Full Clone (default)               │
│  All history + all files + all blobs         │
│  Size: 100% of repo                          │
│                                              │
│  ┌───────────────────────────────────┐       │
│  │      Partial Clone                │       │
│  │  All history + file tree,         │       │
│  │  but blobs downloaded on demand   │       │
│  │  Size: ~5-20% of repo            │       │
│  │                                   │       │
│  │  ┌─────────────────────────┐      │       │
│  │  │    Sparse Checkout      │      │       │
│  │  │  Only specified dirs    │      │       │
│  │  │  appear in working tree │      │       │
│  │  │  Size: varies           │      │       │
│  │  └─────────────────────────┘      │       │
│  └───────────────────────────────────┘       │
│                                              │
│  ┌───────────────────────────────────┐       │
│  │      Shallow Clone                │       │
│  │  Only recent N commits            │       │
│  │  No full history                  │       │
│  │  Size: ~1-5% of repo             │       │
│  │  Best for: CI one-shot builds     │       │
│  └───────────────────────────────────┘       │
└─────────────────────────────────────────────┘
```

```bash
# Partial clone (download blobs on demand)
git clone --filter=blob:none https://github.com/org/large-repo.git

# Shallow clone (only last 1 commit)
git clone --depth 1 https://github.com/org/large-repo.git

# Combine partial + sparse for maximum efficiency
git clone --no-checkout --filter=blob:none https://github.com/org/large-repo.git
cd large-repo
git sparse-checkout init --cone
git sparse-checkout set src/my-module
git checkout main
```

### 17.5 Git LFS (Large File Storage)

Store large binary files (images, videos, models, datasets) outside the git repo:

```bash
# Install LFS
git lfs install

# Track file types
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "assets/videos/*"

# This creates/updates .gitattributes
cat .gitattributes
# *.psd filter=lfs diff=lfs merge=lfs -text
# *.zip filter=lfs diff=lfs merge=lfs -text

# Commit .gitattributes first
git add .gitattributes
git commit -m "chore: configure Git LFS for binary assets"

# Then add and commit large files normally
git add assets/hero.psd
git commit -m "feat: add hero image asset"
```

**When to use LFS:**
- Binary files > 10 MB that change frequently
- Design assets (PSD, Figma exports, videos)
- ML model weights, datasets
- Build artifacts that must be versioned

**When NOT to use LFS:**
- Text files (git handles these efficiently)
- Small images (< 1 MB, infrequent changes)
- Files you rarely change (one-time additions are fine without LFS)

### 17.6 git blame -w -M (Advanced Blame)

```bash
# Standard blame (noisy, shows whitespace/formatting changes)
git blame src/utils.ts

# Ignore whitespace changes (-w)
git blame -w src/utils.ts

# Detect moved/copied code within the file (-M)
git blame -w -M src/utils.ts

# Detect code moved from other files (-C)
git blame -w -C -C src/utils.ts

# Show blame for a specific line range
git blame -w -L 42,60 src/utils.ts
```

---

## 18. Tools & Ecosystem

### 18.1 Git GUIs

| Tool         | Platform        | Price        | Best Feature                     |
|--------------|-----------------|--------------|----------------------------------|
| GitKraken    | Win/Mac/Linux   | Free/Paid    | Visual branch graph, intuitive   |
| Sourcetree   | Win/Mac         | Free         | Atlassian integration            |
| Fork         | Win/Mac         | $49.99       | Fast, clean UI, conflict editor  |
| lazygit      | Terminal         | Free (OSS)   | Keyboard-driven, fast            |
| tig          | Terminal         | Free (OSS)   | ncurses-based, lightweight       |
| GitHub Desktop | Win/Mac       | Free         | Simplest for beginners           |
| VS Code Git  | All             | Free         | Integrated, extensions available |
| Tower        | Win/Mac         | $69/yr       | Professional, undo feature       |

### 18.2 GitHub CLI (gh) Essential Commands

```bash
# Authentication
gh auth login
gh auth status

# Pull Requests
gh pr create --title "feat: X" --body "## Summary\n..."
gh pr list --state open
gh pr view 42
gh pr checkout 42          # Check out a PR branch locally
gh pr review 42 --approve
gh pr merge 42 --squash --delete-branch

# Issues
gh issue create --title "Bug: X" --label bug
gh issue list --label "P0-critical"
gh issue close 42

# Repository
gh repo clone org/repo
gh repo fork org/repo --clone
gh repo view --web          # Open in browser

# Actions / CI
gh run list
gh run view 12345
gh run watch 12345          # Stream logs live
gh run rerun 12345 --failed # Rerun only failed jobs

# API (powerful escape hatch)
gh api repos/{owner}/{repo}/pulls/42/reviews
gh api graphql -f query='{ viewer { login } }'

# Extensions
gh extension install dlvhdr/gh-dash    # PR dashboard in terminal
gh extension install seachicken/gh-poi # Clean stale branches
```

### 18.3 Graphite (Stacked PRs)

Graphite is the market leader for stacked PR workflows on GitHub.

```bash
# Install
npm install -g @withgraphite/graphite-cli

# Create a stack of dependent changes
gt branch create feat-auth-types
# ... make changes ...
gt commit create -m "feat: add auth type definitions"

gt branch create feat-auth-logic
# ... make changes building on types ...
gt commit create -m "feat: implement auth logic"

gt branch create feat-auth-ui
# ... make UI changes building on logic ...
gt commit create -m "feat: add auth UI components"

# Submit the entire stack as chained PRs
gt stack submit

# Keep stack synced with main
gt stack sync
```

**Impact data (2025-2026):**
- Shopify: 33% more PRs merged per developer after adoption
- Asana: engineers saved 7 hours weekly, shipped 21% more code, cut median PR size by 11%
- Graphite Diamond (AI reviewer): provides immediate, actionable feedback

(Sources: Graphite blog, DEV Community Graphite guide, CodePulseHQ 2026 comparison)

### 18.4 Better Diffs

| Tool        | Type           | What It Does                              |
|-------------|----------------|-------------------------------------------|
| `delta`     | Syntax-aware   | Line-based diff with syntax highlighting  |
| `difftastic`| Structural     | AST-aware diff (understands code structure)|

```bash
# Install delta
cargo install git-delta
# or: brew install git-delta

# Configure as default pager
git config --global core.pager delta
git config --global interactive.diffFilter 'delta --color-only'
git config --global delta.navigate true
git config --global delta.side-by-side true

# Install difftastic
cargo install difftastic
# or: brew install difftastic

# Use for structural diffs
GIT_EXTERNAL_DIFF=difft git diff
GIT_EXTERNAL_DIFF=difft git log -p
```

### 18.5 Security & Pre-commit

```bash
# git-crypt: encrypt specific files in the repo
git-crypt init
echo "secrets/** filter=git-crypt diff=git-crypt" >> .gitattributes
git-crypt add-gpg-user USER_ID

# pre-commit framework (Python-based, language-agnostic)
pip install pre-commit
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: detect-private-key
      - id: check-added-large-files
        args: ['--maxkb=500']

  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.27.0
    hooks:
      - id: commitizen
```

---

## 19. Metrics & Git Health

### 19.1 DORA Metrics Deep Dive (2025 Benchmarks)

The DORA (DevOps Research and Assessment) framework defines four key metrics.
In 2025, DORA moved from "Elite/High/Medium/Low" buckets to percentile
distributions, with the top 15% used as the elite benchmark.

| Metric                     | Top 15% (Elite) | Median          | Bottom 15%      |
|----------------------------|-----------------|-----------------|------------------|
| Deployment Frequency       | Multiple/day    | Weekly-Monthly  | < Monthly        |
| Change Lead Time           | < 1 day         | 1-7 days        | > 1 month        |
| Change Failure Rate        | < 4%            | 10-15%          | > 30%            |
| Failed Deploy Recovery     | < 1 hour        | 1-7 days        | > 1 month        |

**Key 2025 insight:** Only 16.2% of organizations deploy on-demand (multiple times/day),
while 21.9% deploy between daily and weekly.

(Sources: DORA.dev metrics guide, Octopus Deploy DORA 2024/25 report, RDEL #115 2025 benchmarks,
Multitudes DORA guide, LinearB DORA metrics blog)

### 19.2 PR-Level Metrics

**Data from LinearB's 2025 Engineering Benchmarks (6.1M+ pull requests, 3,000 teams):**

| Metric                 | Elite        | Average      | Poor           |
|------------------------|-------------|--------------|----------------|
| PR Cycle Time          | < 1 day     | 7 days       | > 14 days      |
| Pickup Time (to review)| < 2 hours   | 4 days       | > 7 days       |
| Review Time            | < 4 hours   | 4 days       | > 7 days       |
| PR Size (lines)        | < 100       | 200-400      | > 1,000        |
| Time to Merge          | < 1 day     | 5 days       | > 10 days      |

**PR cycle time breakdown (average):**
- Coding: ~2 days
- Pickup (waiting for reviewer): ~1 day
- Review: ~4 days (the largest bottleneck)
- Deploy: < 1 day

**Key finding:** PR Size is the single most significant driver of engineering velocity.
Smaller PRs are easier to review, less risky to merge, and produce faster cycle times.

**New 2025 metric -- PR Maturity:** Measures how well-prepared a PR is when submitted
for review (description quality, test coverage, linked issues). Higher maturity
correlates with faster review cycles.

(Sources: LinearB 2025 Engineering Benchmarks, LinearB cycle time docs)

### 19.3 Code Churn & Bus Factor

**Code churn:** Percentage of recently changed code that is changed again within
2-3 weeks. High churn (> 25%) indicates:
- Unclear requirements
- Poor initial implementation
- Excessive refactoring
- Scope creep

**Bus factor:** Number of people who would need to be "hit by a bus" before the
project stalls. Calculate by analyzing commit distribution:

```bash
# Quick bus factor estimate
git shortlog -sn --no-merges | head -10
#    452  Caio Imori
#     38  Matheus Soier
#     12  dependabot[bot]
# Bus factor = 1 (Caio has 90%+ of commits)
```

**Improving bus factor:**
- Rotate code reviewers (everyone reviews everything)
- Pair programming sessions
- Document architectural decisions (ADRs)
- Avoid single-person knowledge silos

### 19.4 Engineering Metrics Platforms (2025)

| Platform          | Focus                              | DORA?  | Pricing        |
|-------------------|------------------------------------|--------|----------------|
| LinearB           | Team productivity, cycle time      | Yes    | Free tier + paid |
| Sleuth            | Deployment-centric, DORA accuracy  | Yes    | Free tier + paid |
| Pluralsight Flow  | Individual + team analytics        | Yes    | Enterprise     |
| Jellyfish         | Engineering investment, ROI        | Yes    | Enterprise     |
| Swarmia           | Developer experience + delivery    | Yes    | Team tier      |
| Faros AI          | Engineering intelligence platform  | Yes    | Enterprise     |
| GitHub Insights   | Built-in (limited)                 | Partial| Enterprise     |
| Haystack          | PR analytics, review insights      | Partial| Free + paid    |

(Sources: LinearB blog, Axify LinearB alternatives comparison, Jellyfish Swarmia alternatives)

---

## 20. SINAPSE-Specific Recommendations

Based on the complete research across all 19 preceding sections, here are the optimal
git workflow recommendations for SINAPSE, considering its unique constraints.

### 20.1 Context Summary

```
Team Composition:
  - 2 humans: Caio (product builder), Matheus (developer)
  - 186 AI agents across 18 squads
  - 1 npm package: sinapse-ai (published to registry)
  
Current Rules (safe-collaboration):
  - Auto-branch with prefix (caio/, soier/, dev/)
  - Auto-sync on session start
  - Auto-resolve simple conflicts
  - Auto-PR with reviewer assignment
  - Secret scanning pre-commit
  - @devops exclusive push authority
```

### 20.2 Recommended Branch Strategy: Modified GitHub Flow

GitHub Flow is the correct base strategy for SINAPSE. Do NOT use GitFlow (too complex
for 2 humans), trunk-based (too risky without comprehensive test suite), or release
branches (single npm package does not need them).

```
main (protected, always deployable)
  │
  ├── caio/feat/{description}      Human: Caio
  ├── soier/feat/{description}     Human: Matheus
  ├── agent/{squad}/{agent-id}/{description}   AI agent
  └── release/v{X.Y.Z}            Release candidate (optional, for major versions)
```

**Proposed branch naming refinement:**

Current naming (`dev/feat/...`) does not distinguish which AI agent or squad created
the branch. Recommended update:

```
# Current (ambiguous for AI agents)
dev/feat/new-feature

# Proposed (includes squad and agent for traceability)
agent/core/pixel/feat-dark-mode
agent/research/sage/docs-api-reference
agent/data/tensor/fix-schema-migration
```

### 20.3 Commit Convention Recommendations

Keep the current Conventional Commits standard. Add these SINAPSE-specific rules:

```
# Human commits
feat: implement dark mode toggle [Story 5.2]

# AI agent commits (always include Co-Authored-By)
feat: implement dark mode toggle [Story 5.2]

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>

# AI agents should also include agent ID in commit body
Agent: @developer (Pixel)
Squad: core
```

### 20.4 CI/CD Pipeline Design

```yaml
# Recommended pipeline structure for SINAPSE
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
    tags: ['v*.*.*']

jobs:
  # ── GATE 1: Quality ──────────────────────
  quality:
    runs-on: ubuntu-latest
    steps:
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test -- --coverage
      - run: npx gitleaks detect --source . --no-banner

  # ── GATE 2: Build ────────────────────────
  build:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

  # ── GATE 3: Publish (tags only) ──────────
  publish:
    if: startsWith(github.ref, 'refs/tags/v')
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci
      - run: npm publish   # OIDC trusted publishing, no token needed
      - run: |
          # Create GitHub Release
          gh release create ${{ github.ref_name }} \
            --title "${{ github.ref_name }}" \
            --generate-notes
```

### 20.5 Release Process

**For minor/patch releases (most common):**

```bash
# 1. Agent (or human) ensures all tests pass on main
# 2. @devops creates and pushes tag
git tag -a v9.3.0 -m "Release v9.3.0: dark mode + bug fixes"
git push origin v9.3.0

# 3. GitHub Actions automatically:
#    - Runs full test suite
#    - Publishes to npm with OIDC + provenance
#    - Creates GitHub Release with auto-generated notes

# 4. Users install: npm install sinapse-ai@9.3.0
```

**For major releases (breaking changes):**

```bash
# 1. Create release branch for stabilization
git checkout -b release/v10.0.0

# 2. Only bug fixes go to this branch
# 3. When stable, merge to main + tag
git checkout main
git merge release/v10.0.0
git tag -a v10.0.0 -m "Release v10.0.0: major architecture overhaul"
git push origin main --follow-tags

# 4. Delete release branch
git branch -d release/v10.0.0
```

### 20.6 npm Publish Automation

Migrate to OIDC trusted publishing (mandatory since Dec 2025):

1. Configure trust relationship in npm web UI:
   - Package: `sinapse-ai`
   - Repository: `caio-imori/sinapse-ai`
   - Workflow: `.github/workflows/publish.yml`
   - Environment: `npm-publish` (optional, adds approval gate)

2. Remove classic npm tokens from GitHub Secrets (they no longer work)

3. Use the workflow template from section 20.4 above

### 20.7 Specific Improvement Recommendations

Based on the research, here are concrete improvements for SINAPSE:

| # | Current State                          | Recommended Change                           | Priority |
|---|----------------------------------------|----------------------------------------------|----------|
| 1 | Branch naming: `dev/feat/...`          | `agent/{squad}/{agent-id}/feat-...`          | HIGH     |
| 2 | No CI pipeline                         | Add GitHub Actions (quality + build + publish)| HIGH    |
| 3 | Manual npm publish                     | OIDC trusted publishing on tag push          | HIGH     |
| 4 | No changelog generation                | Add `semantic-release` or `commit-and-tag-version` | MEDIUM |
| 5 | No PR size limits                      | Bot warning at 400+ lines                    | MEDIUM   |
| 6 | No DORA tracking                       | Add LinearB (free tier) or GitHub Insights   | LOW      |
| 7 | Secret scan pre-commit only            | Add GitHub Secret Scanning (repo level)      | HIGH     |
| 8 | No CODEOWNERS                          | Add per-package ownership                    | MEDIUM   |
| 9 | No PR template                         | Add `.github/pull_request_template.md`       | LOW      |
| 10| No issue templates                     | Add bug report + feature request templates   | LOW      |

### 20.8 Scaling Considerations

As SINAPSE grows beyond 2 human developers:

| Team Size    | Git Workflow Change Needed                              |
|--------------|---------------------------------------------------------|
| 2 humans     | Current setup is sufficient (GitHub Flow + conventions) |
| 3-5 humans   | Add CODEOWNERS, review rotation, async review reminders |
| 6-10 humans  | Consider Graphite for stacked PRs, add DORA tracking   |
| 10+ humans   | Evaluate merge queue, formalize release train           |
| Monorepo?    | Current single-package structure works. If packages grow to 5+, add Turborepo |

---

## Part 2 Sources

### Web Sources Consulted
- [Monorepo vs Polyrepo Benchmark Data -- Faros AI](https://www.faros.ai/blog/monorepo-vs-polyrepo-benchmark-data)
- [Monorepo vs Polyrepo 2025 -- DEV Community](https://dev.to/md-afsar-mahmud/monorepo-vs-polyrepo-which-one-should-you-choose-in-2025-g77)
- [Monorepo vs Polyrepo Guide 2026 -- ZTABS](https://ztabs.co/blog/monorepo-vs-polyrepo-guide)
- [Turborepo vs Nx 2026 -- PkgPulse](https://www.pkgpulse.com/blog/turborepo-vs-nx-monorepo-2026)
- [Monorepo 2026: Turborepo vs Nx vs Bazel -- daily.dev](https://daily.dev/blog/monorepo-turborepo-vs-nx-vs-bazel-modern-development-teams)
- [Large Monorepo Benchmark -- vsavkin/GitHub](https://github.com/vsavkin/large-monorepo)
- [DORA Metrics Guide -- dora.dev](https://dora.dev/guides/dora-metrics-four-keys/)
- [DORA Metrics 2024/25 -- Octopus Deploy](https://octopus.com/devops/metrics/dora-metrics/)
- [2025 DORA Benchmarks -- RDEL Substack](https://rdel.substack.com/p/rdel-115-what-are-the-2025-benchmarks)
- [DORA Metrics Full Guide -- Multitudes](https://www.multitudes.com/blog/dora-metrics)
- [DORA Metrics in Age of AI -- Future Processing](https://www.future-processing.com/blog/dora-devops-metrics/)
- [2025 Engineering Benchmarks (6.1M PRs) -- LinearB](https://linearb.io/blog/2025-engineering-benchmarks-insights)
- [PR Cycle Time Calculation -- LinearB](https://linearb.helpdocs.io/article/0vif1ihmgc-how-is-cycle-time-calculated)
- [LinearB Alternatives -- Axify](https://axify.io/blog/linearb-alternatives)
- [Feature Flags + Trunk-Based Development 2025 -- FeatBit](https://www.featbit.co/articles2025/trunk-based-development-feature-flags-2025)
- [Feature Flags for GitHub Actions -- LaunchDarkly](https://launchdarkly.com/blog/ready-set-actions-flag-evaluations-for-github/)
- [Feature Flags + TBD -- Harness](https://developer.harness.io/docs/feature-flags/get-started/trunk-based-development/)
- [ArgoCD vs Flux 2025 -- Zignuts](https://www.zignuts.com/blog/argo-cd-vs-flux-cd--comparison)
- [Flux vs ArgoCD -- Spacelift](https://spacelift.io/blog/flux-vs-argo-cd)
- [GitOps Best Practices -- Akuity](https://akuity.io/blog/gitops-best-practices-whitepaper)
- [Codex vs Claude Code -- Builder.io](https://www.builder.io/blog/codex-vs-claude-code)
- [AI Coding Agents 2026 -- Faros AI](https://www.faros.ai/blog/best-ai-coding-agents-2026)
- [GitHub Agent HQ -- GitHub Blog](https://github.blog/news-insights/company-news/pick-your-agent-use-claude-and-codex-on-agent-hq/)
- [Graphite Stacked PRs Guide -- DEV Community](https://dev.to/semgrep/a-guide-to-using-graphites-stacked-prs-for-github-users-5c47)
- [Code Review Tools 2026 -- CodePulseHQ](https://codepulsehq.com/guides/code-review-platforms-comparison)
- [Stacked Diffs -- Pragmatic Engineer](https://newsletter.pragmaticengineer.com/p/stacked-diffs)
- [Git Sparse Checkout -- GitHub Blog](https://github.blog/open-source/git/bring-your-monorepo-down-to-size-with-sparse-checkout/)
- [30GB Repo Sparse Checkout -- Medium](https://medium.com/@quicksilversel/the-30gb-repository-that-taught-me-about-git-sparse-checkout-2a1075d841d8)
- [Partial Clone and Shallow Clone -- GitHub Blog](https://github.blog/open-source/git/get-up-to-speed-with-partial-clone-and-shallow-clone/)
- [Git Subtree Tutorial -- Atlassian](https://www.atlassian.com/git/tutorials/git-subtree)
- [Git Subtree vs Submodule -- GitProtect.io](https://gitprotect.io/blog/managing-git-projects-git-subtree-vs-submodule/)
- [Git Worktree + Subtree Guide -- tiendu.github.io](https://tiendu.github.io/2025/03/01/git-worktree-subtree-eng.html)
- [npm Trusted Publishing -- npm Docs](https://docs.npmjs.com/trusted-publishers/)
- [npm OIDC Generally Available -- GitHub Changelog](https://github.blog/changelog/2025-07-31-npm-trusted-publishing-with-oidc-is-generally-available/)
- [npm OIDC Trusted Publishing -- Socket.dev](https://socket.dev/blog/npm-trusted-publishing)
- [Conventional Changelog -- GitHub](https://github.com/conventional-changelog/conventional-changelog)
- [auto-changelog -- GitHub](https://github.com/cookpete/auto-changelog)
- [Forking Workflow -- Atlassian](https://www.atlassian.com/git/tutorials/comparing-workflows/forking-workflow)
- [Contributing to Open Source -- GitHub Docs](https://docs.github.com/en/get-started/exploring-projects-on-github/contributing-to-a-project)
- [Semantic Release + GitHub Actions -- GitBook](https://semantic-release.gitbook.io/semantic-release/recipes/ci-configurations/github-actions)
