# git-suggest-version

A lightweight, zero-dependency Git subcommand that analyzes commit logs since your last Semantic Version (SemVer) tag and recommends the next logical version bump based on [Conventional Commits](https://www.conventionalcommits.org/).

---

## Key Features

- **Conventional Commit Parsing:** Automatically detects `feat:`, `fix:`, `refactor:`, `chore:`, and breaking changes.
- **Breaking Change Detection:** Catches `BREAKING CHANGE:` headers/footers as well as scope breaker syntax (e.g., `feat(api)!:` or `fix!:`).
- **Multi-Line Body Inspection:** Scans full commit bodies for `BREAKING CHANGE:` footers, ensuring breaking updates aren't missed even if not in the summary line.
- **Tagged HEAD Awareness:** Immediately alerts you if the current `HEAD` is already tagged, preventing redundant bumps.
- **Git Native Integration:** Installs as a `git-*` binary so you can execute it natively as `git suggest-version`.
- **Interactive Mode:** Optionally prompt to create an annotated local Git tag on the fly (`-c` / `--create`).
- **CI/CD Ready:** Offers structured JSON (`--json`) and raw string output (`--tag-only`) for automated deployment workflows.

---

## Installation

### macOS & Linux

1. **Create local bin directory** (if it doesn't already exist) and place the script:
   ```bash
   mkdir -p ~/.local/bin
   ```

2. **Save the script** as `git-suggest-version` inside `~/.local/bin/`

3. **Make executable:**
   ```bash
   chmod +x ~/.local/bin/git-suggest-version
   ```

4. **Ensure `~/.local/bin` is in your PATH** (Zsh default on macOS):
   ```bash
   echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
   source ~/.zshrc
   ```

---

## How It Works

`git-suggest-version` inspects the commit messages between your last SemVer tag (`vX.Y.Z` or `X.Y.Z`) and `HEAD`.

| Bump Level | Conventional Commit Pattern | Example |
| :--- | :--- | :--- |
| **MAJOR** | Contains `BREAKING CHANGE:` header/footer or `!` scope breaker | `feat(api)!: drop deprecated v1 endpoints` |
| **MINOR** | Starts with `feat:` or `feat(scope):` | `feat(auth): add OAuth2 provider` |
| **PATCH** | Starts with `fix:`, `fix(scope):`, or default fallback | `fix(db): resolve connection pool timeout` |

---

## Usage & CLI Reference

Run the command from within **any Git repository**:

```bash
git suggest-version [OPTIONS]
```

### Options

| Flag | Description |
| :--- | :--- |
| `-c`, `--create` | Prompts to create an annotated local Git tag if a version bump is recommended. |
| `--tag-only` | Outputs **only** the target tag string (`v1.3.0`). Useful for script piping. |
| `--json` | Outputs execution result as structured JSON. |
| `-h`, `--help` | Displays help information. |

---

## Usage Examples

### 1. Default Terminal Output
```bash
$ git suggest-version
Current tag:  v1.2.3
Suggested:    v1.3.0 (minor bump)
Reason:       Found new feature -> a1b2c3d feat(auth): add token refresh
```

### 2. Interactive Tag Creation
```bash
$ git suggest-version -c
Current tag:  v1.2.3
Suggested:    v1.3.0 (minor bump)
Reason:       Found new feature -> a1b2c3d feat(auth): add token refresh

Create annotated tag v1.3.0 locally? [y/N] y
Successfully created tag v1.3.0.
```

### 3. Pipeline Automation (`--tag-only`)
```bash
# Tag the repository and push to remote in a single command
git tag $(git suggest-version --tag-only) && git push origin --tags
```

### 4. Structured Output (`--json`)
```bash
$ git suggest-version --json
{"status":"ok","current":"v1.2.3","suggested":"v1.3.0","bump":"minor","reason":"Found new feature -> a1b2c3d feat(auth): add token refresh"}
```

---

## CI/CD Workflow Example (GitHub Actions)

You can use `git-suggest-version` in GitHub Actions to auto-tag releases on merge to `main`:

```yaml
name: Auto Tag Release

on:
  push:
    branches:
      - main

jobs:
  tag:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Full history required to inspect tags and commits

      - name: Calculate Next Version
        id: semver
        run: |
          chmod +x ./scripts/git-suggest-version
          TAG=$(./scripts/git-suggest-version --tag-only)
          echo "tag=$TAG" >> $GITHUB_OUTPUT

      - name: Create and Push Tag
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git tag ${{ steps.semver.outputs.tag }}
          git push origin ${{ steps.semver.outputs.tag }}
```

---

## License

MIT License. Free to use, modify, and distribute across all your projects.
