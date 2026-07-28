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

2. **Save the script** as `git-suggest-version` inside `~/.local/bin/`:
   ```bash
   cat << 'EOF' > ~/.local/bin/git-suggest-version
   #!/usr/bin/env bash

   set -euo pipefail

   # Flags
   FORMAT="text"
   CREATE_TAG=false

   for arg in "$@"; do
     case "$arg" in
       --tag-only) FORMAT="tag-only" ;;
       --json) FORMAT="json" ;;
       -c|--create) CREATE_TAG=true ;;
       -h|--help)
         echo "Usage: git suggest-version [OPTIONS]"
         echo ""
         echo "Options:"
         echo "  --tag-only    Output only the suggested tag string"
         echo "  --json        Output result as a JSON object"
         echo "  -c, --create  Prompt to create the Git tag locally if an upgrade is recommended"
         echo "  -h, --help    Show this help message"
         exit 0
         ;;
       *)
         echo "Error: Unknown argument '$arg'" >&2
         exit 1
         ;;
     esac
   done

   EXACT_TAG=$(git tag --points-at HEAD 2>/dev/null | head -n 1 || true)
   if [[ -n "$EXACT_TAG" ]]; then
     if [[ "$FORMAT" == "tag-only" ]]; then
       echo "$EXACT_TAG"
     elif [[ "$FORMAT" == "json" ]]; then
       printf '{"status":"already_tagged","current":"%s","suggested":"%s","bump":"none","reason":"HEAD is already tagged"}\n' "$EXACT_TAG" "$EXACT_TAG"
     else
       echo "Current commit is already tagged as '$EXACT_TAG'. No upgrade recommended."
     fi
     exit 0
   fi

   LATEST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || \
                git tag -l "v*" "0*" "1*" "2*" "3*" "4*" "5*" "6*" "7*" "8*" "9*" --sort=-v:refname 2>/dev/null | head -n 1 || \
                echo "v0.0.0")

   [[ -z "$LATEST_TAG" ]] && LATEST_TAG="v0.0.0"
   VERSION="${LATEST_TAG#v}"

   if [[ ! "$VERSION" =~ ^([0-9]+)\.([0-9]+)\.([0-9]+)$ ]]; then
     echo "Error: Tag '$LATEST_TAG' is not a valid semver (e.g., v1.2.3 or 1.2.3)" >&2
     exit 1
   fi

   MAJOR="${BASH_REMATCH[1]}"
   MINOR="${BASH_REMATCH[2]}"
   PATCH="${BASH_REMATCH[3]}"

   if git rev-parse "$LATEST_TAG" >/dev/null 2>&1; then
     RANGE="${LATEST_TAG}..HEAD"
   else
     RANGE="HEAD"
   fi

   COMMIT_LOGS=$(git log "$RANGE" --oneline 2>/dev/null || true)

   if [[ -z "$COMMIT_LOGS" ]]; then
     if [[ "$FORMAT" == "tag-only" ]]; then
       echo "$LATEST_TAG"
     elif [[ "$FORMAT" == "json" ]]; then
       printf '{"status":"no_commits","current":"%s","suggested":"%s","bump":"none","reason":"No new commits found"}\n' "$LATEST_TAG" "$LATEST_TAG"
     else
       echo "No new commits found since tag '$LATEST_TAG'."
     fi
     exit 0
   fi

   BUMP="patch"
   REASON="Contains fixes, refactors, or general maintenance commits."

   BREAKING_SUMMARY=$(echo "$COMMIT_LOGS" | grep -E 'BREAKING CHANGE:|^[a-f0-9]+ [a-z]+(\([^\)]+\))?!:' | head -n 1 || true)
   BREAKING_BODY=""
   if [[ -z "$BREAKING_SUMMARY" ]]; then
     BREAKING_BODY=$(git log "$RANGE" | grep -E '^\s*BREAKING CHANGE:' | head -n 1 || true)
   fi

   if [[ -n "$BREAKING_SUMMARY" ]]; then
     BUMP="major"
     REASON="Found breaking change -> $BREAKING_SUMMARY"
   elif [[ -n "$BREAKING_BODY" ]]; then
     BUMP="major"
     REASON="Found BREAKING CHANGE footer in commit body."
   else
     FEAT_COMMIT=$(echo "$COMMIT_LOGS" | grep -E '^[a-f0-9]+ feat(\([^\)]+\))?:' | head -n 1 || true)
     if [[ -n "$FEAT_COMMIT" ]]; then
       BUMP="minor"
       REASON="Found new feature -> $FEAT_COMMIT"
     else
       FIX_COMMIT=$(echo "$COMMIT_LOGS" | grep -E '^[a-f0-9]+ fix(\([^\)]+\))?:' | head -n 1 || true)
       if [[ -n "$FIX_COMMIT" ]]; then
         BUMP="patch"
         REASON="Found fix -> $FIX_COMMIT"
       fi
     fi
   fi

   NEW_MAJOR="$MAJOR"
   NEW_MINOR="$MINOR"
   NEW_PATCH="$PATCH"

   case "$BUMP" in
     major) NEW_MAJOR=$((MAJOR + 1)); NEW_MINOR=0; NEW_PATCH=0 ;;
     minor) NEW_MINOR=$((MINOR + 1)); NEW_PATCH=0 ;;
     patch) NEW_PATCH=$((PATCH + 1)) ;;
   esac

   PREFIX=""
   [[ "$LATEST_TAG" =~ ^v ]] && PREFIX="v"
   NEXT_TAG="${PREFIX}${NEW_MAJOR}.${NEW_MINOR}.${NEW_PATCH}"

   case "$FORMAT" in
     tag-only) echo "$NEXT_TAG" ;;
     json)
       SAFE_REASON=$(echo "$REASON" | sed 's/"/\"/g')
       printf '{"status":"ok","current":"%s","suggested":"%s","bump":"%s","reason":"%s"}\n' \
         "$LATEST_TAG" "$NEXT_TAG" "$BUMP" "$SAFE_REASON"
       ;;
     text)
       echo "Current tag:  $LATEST_TAG"
       echo "Suggested:    $NEXT_TAG ($BUMP bump)"
       echo "Reason:       $REASON"
       ;;
   esac

   if [[ "$CREATE_TAG" == true && "$FORMAT" == "text" ]]; then
     echo ""
     read -p "Create annotated tag $NEXT_TAG locally? [y/N] " -n 1 -r
     echo ""
     if [[ $REPLY =~ ^[Yy]$ ]]; then
       git tag -a "$NEXT_TAG" -m "Release $NEXT_TAG ($REASON)"
       echo "Successfully created tag $NEXT_TAG."
     else
       echo "Skipped tag creation."
     fi
   fi
   EOF
   ```

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
