#!/usr/bin/env bash

set -euo pipefail

# ------------------------------------------------------------------------------
# CLI Flag Handling
# ------------------------------------------------------------------------------
FORMAT="text" # Options: text, tag-only, json
CREATE_TAG=false

for arg in "$@"; do
  case "$arg" in
    --tag-only)
      FORMAT="tag-only"
      ;;
    --json)
      FORMAT="json"
      ;;
    -c|--create)
      CREATE_TAG=true
      ;;
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

# ------------------------------------------------------------------------------
# 1. Check if HEAD is ALREADY tagged
# ------------------------------------------------------------------------------
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

# ------------------------------------------------------------------------------
# 2. Locate the Latest Semver Tag
# ------------------------------------------------------------------------------
# Tries git describe first, then falls back to sorting tags matching 'v*' or numbers
LATEST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || \
             git tag -l "v*" "0*" "1*" "2*" "3*" "4*" "5*" "6*" "7*" "8*" "9*" --sort=-v:refname 2>/dev/null | head -n 1 || \
             echo "v0.0.0")

if [[ -z "$LATEST_TAG" ]]; then
  LATEST_TAG="v0.0.0"
fi

# Strip leading 'v' for numeric processing
VERSION="${LATEST_TAG#v}"

# Validate semver format (X.Y.Z)
if [[ ! "$VERSION" =~ ^([0-9]+)\.([0-9]+)\.([0-9]+)$ ]]; then
  echo "Error: Tag '$LATEST_TAG' is not a valid semver (e.g., v1.2.3 or 1.2.3)" >&2
  exit 1
fi

MAJOR="${BASH_REMATCH[1]}"
MINOR="${BASH_REMATCH[2]}"
PATCH="${BASH_REMATCH[3]}"

# ------------------------------------------------------------------------------
# 3. Retrieve Commits Since Last Tag
# ------------------------------------------------------------------------------
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

# ------------------------------------------------------------------------------
# 4. Analyze Commits for Bump Level (Conventional Commits)
# ------------------------------------------------------------------------------
BUMP="patch"
REASON="Contains fixes, refactors, or general maintenance commits."

# A) MAJOR: Check summary lines for `BREAKING CHANGE:` or scope breakers like `feat(api)!:`
BREAKING_SUMMARY=$(echo "$COMMIT_LOGS" | grep -E 'BREAKING CHANGE:|^[a-f0-9]+ [a-z]+(\([^\)]+\))?!:' | head -n 1 || true)

# B) MAJOR: Check multi-line commit BODIES for `BREAKING CHANGE:` footers
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
  # C) MINOR: Check for features (`feat:` or `feat(scope):`)
  FEAT_COMMIT=$(echo "$COMMIT_LOGS" | grep -E '^[a-f0-9]+ feat(\([^\)]+\))?:' | head -n 1 || true)
  if [[ -n "$FEAT_COMMIT" ]]; then
    BUMP="minor"
    REASON="Found new feature -> $FEAT_COMMIT"
  else
    # D) PATCH: Check for fixes (`fix:` or `fix(scope):`)
    FIX_COMMIT=$(echo "$COMMIT_LOGS" | grep -E '^[a-f0-9]+ fix(\([^\)]+\))?:' | head -n 1 || true)
    if [[ -n "$FIX_COMMIT" ]]; then
      BUMP="patch"
      REASON="Found fix -> $FIX_COMMIT"
    fi
  fi
fi

# ------------------------------------------------------------------------------
# 5. Calculate Next Version Tag
# ------------------------------------------------------------------------------

# Handle initial repositories (0.0.0 / v0.0.0) as a minor bump vs patch
if [[ ("$MAJOR" -eq 0 && "$MINOR" -eq 0 && "$PATCH" -eq 0) && "$BUMP" == "patch" ]]; then
  BUMP="minor"
  REASON="Initial release baseline (no previous tags found)."
fi

NEW_MAJOR="$MAJOR"
NEW_MINOR="$MINOR"
NEW_PATCH="$PATCH"

case "$BUMP" in
  major)
    NEW_MAJOR=$((MAJOR + 1))
    NEW_MINOR=0
    NEW_PATCH=0
    ;;
  minor)
    NEW_MINOR=$((MINOR + 1))
    NEW_PATCH=0
    ;;
  patch)
    NEW_PATCH=$((PATCH + 1))
    ;;
esac

PREFIX=""
if [[ "$LATEST_TAG" =~ ^v ]]; then
  PREFIX="v"
fi

NEXT_TAG="${PREFIX}${NEW_MAJOR}.${NEW_MINOR}.${NEW_PATCH}"

# ------------------------------------------------------------------------------
# 5. Calculate Next Version Tag
# ------------------------------------------------------------------------------

# First, override BUMP if baseline is 0.0.0
if [[ ("$MAJOR" -eq 0 && "$MINOR" -eq 0 && "$PATCH" -eq 0) && "$BUMP" == "patch" ]]; then
  BUMP="minor"
  REASON="Initial release baseline (no previous tags found)."
fi

# Initialize variables
NEW_MAJOR="$MAJOR"
NEW_MINOR="$MINOR"
NEW_PATCH="$PATCH"

# Calculate based on the updated BUMP
case "$BUMP" in
  major)
    NEW_MAJOR=$((MAJOR + 1))
    NEW_MINOR=0
    NEW_PATCH=0
    ;;
  minor)
    NEW_MINOR=$((MINOR + 1))
    NEW_PATCH=0
    ;;
  patch)
    NEW_PATCH=$((PATCH + 1))
    ;;
esac

PREFIX=""
if [[ "$LATEST_TAG" =~ ^v ]]; then
  PREFIX="v"
fi

NEXT_TAG="${PREFIX}${NEW_MAJOR}.${NEW_MINOR}.${NEW_PATCH}"

# ------------------------------------------------------------------------------
# 6. Render Output
# ------------------------------------------------------------------------------
case "$FORMAT" in
  tag-only)
    echo "$NEXT_TAG"
    ;;
  json)
    # Escape quotes in REASON string for clean JSON output
    SAFE_REASON=$(echo "$REASON" | sed 's/"/\\"/g')
    printf '{"status":"ok","current":"%s","suggested":"%s","bump":"%s","reason":"%s"}\n' \
      "$LATEST_TAG" "$NEXT_TAG" "$BUMP" "$SAFE_REASON"
    ;;
  text)
    echo "Current tag:  $LATEST_TAG"
    echo "Suggested:    $NEXT_TAG ($BUMP bump)"
    echo "Reason:       $REASON"
    ;;
esac

# ------------------------------------------------------------------------------
# 7. Interactive Tag Creation (-c / --create)
# ------------------------------------------------------------------------------
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
