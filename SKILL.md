---
name: skill-design-philosophy
description: >
  Design philosophy for creating outstanding Claude Code skills.
  Use when: creating a new skill, improving an existing skill, designing script interfaces.
user-invocable: false
disable-model-invocation: false
---

# Skill Design Philosophy

Philosophy and principles only. Toolchain-specific quirks and implementation details belong in their respective skill docs (e.g., [php-cli-builder](../php-cli-builder/SKILL.md)).

**Goal**: Create skills that eliminate errors by design. Users should never encounter tool quirks—scripts handle everything internally.

Official docs: https://code.claude.com/docs/en/skills.md

For project structure, global access patterns, and setup: [skill-project-structure-setup.md](skill-project-structure-setup.md)

---

## Core Principles

### 1. Zero-Error by Design
The skill should make it impossible to fail due to underlying tool quirks. If glab needs `--description` not `--body`, the script uses the right flag. If URLs need encoding, the script encodes them. Users never see the complexity.

### 2. Intuitive Interfaces
```bash
# Good: Reads naturally, obvious what it does
gitlab-mr-create publicala/farfalla "Fix bug" --description "Details"

# Bad: Requires knowing glab internals
glab mr create --repo publicala/farfalla --title "Fix bug" --description "Details" --target-branch main --yes
```

### 3. Predictable Failure Modes
All errors should be:
- **Catchable**: Exit code != 0
- **Parseable**: JSON format when possible
- **Actionable**: Tell user exactly what's wrong

```bash
# Good error
{"error": "Invalid project format. Use: org/project"}

# Bad error
Error: something went wrong
```

### 4. Validate Early, Fail Fast
Check inputs before doing any work:
```bash
# Check dependencies
command -v glab >/dev/null 2>&1 || { echo '{"error": "glab not installed"}' >&2; exit 1; }

# Validate format
[[ "$PROJECT" =~ ^[^/]+/[^/]+$ ]] || { echo '{"error": "Invalid project format"}' >&2; exit 1; }

# Validate types
[[ "$ID" =~ ^[0-9]+$ ]] || { echo '{"error": "ID must be a number"}' >&2; exit 1; }
```

### 5. Sensible Defaults
Users shouldn't need to specify obvious things:
- Target branch defaults to `main`
- Filters default to the most common case (`open` not `all`)
- Output format defaults to JSON (parseable)

### 6. Consistent Output

**Principle**: Parseable data is JSON. Debug info is plain text. Errors always go to stderr.

Specific rules:
- **Data operations**: Return JSON to stdout
- **Logs/traces**: Return plain text (it's already text)
- **Validation errors**: JSON to stderr, exit 1
- **API errors (404, etc)**: JSON to stderr, exit 1 (don't leak raw API errors)
- **Usage help**: Plain text to stderr, exit 0
- **Success with side effects**: JSON with result info

### 7. Scripts Are Self-Contained

Each script should work standalone without external package dependencies or build steps.

**Bash skills:**
- No shared `lib.sh` or common includes
- Copy patterns rather than abstracting them
- Each script validates its own inputs and handles its own errors

**TypeScript (Bun) skills:**
- A local `lib.ts` within the skill is fine for shared infra (auth, HTTP client, validation, flag parsing)
- No npm packages beyond Bun builtins - use `fetch`, `Bun.spawn`, etc.
- No build step - scripts run directly via `bun run`
- Each script still has its own usage/help and argument parsing

---

## Anti-Patterns to Avoid

Set the right mindset before writing scripts.

### Don't Leak Complexity
```bash
# Bad: User needs to know about URL encoding
glab api projects/org%2Frepo/merge_requests/123

# Good: Script handles it
gitlab-mr-view org/repo 123
```

### Don't Use Wrong Flags
```bash
# Bad: --body doesn't exist
glab mr create --body "text"

# Good: Script uses --description internally
gitlab-mr-create project "title" --description "text"
```

### Don't Require Memorization
```bash
# Bad: Which filter is default? What are the options?
gitlab-mr-discussions project 123 ???

# Good: Defaults to most common, shows options in help
gitlab-mr-discussions project 123           # defaults to 'open'
gitlab-mr-discussions project 123 resolved  # explicit
```

### Don't Mix Output Formats
```bash
# Bad: Sometimes JSON, sometimes plain text, sometimes nothing
script1  # returns JSON
script2  # returns "Success!"
script3  # returns nothing

# Good: Consistent contract
script1  # returns JSON
script2  # returns JSON with status
script3  # returns JSON (empty array if no results)
```

---

## Script Template

```bash
#!/bin/bash
# [One-line description]
# Usage: skill-resource-action <required> [optional]
#
# [What quirks this handles internally]

set -e

# 1. Check dependencies
command -v tool >/dev/null 2>&1 || { echo '{"error": "tool not installed"}' >&2; exit 1; }
command -v jq >/dev/null 2>&1 || { echo '{"error": "jq not installed"}' >&2; exit 1; }

# 2. Usage check
if [[ $# -lt 1 ]]; then
  echo 'Usage: script-name <arg> [options]' >&2
  exit 1
fi

# 3. Parse arguments
ARG="$1"
shift

# 4. Validate inputs
if [[ ! "$ARG" =~ ^expected-pattern$ ]]; then
  echo '{"error": "Invalid format. Expected: pattern"}' >&2
  exit 1
fi

# 5. Set defaults
OPTION="${OPTION:-default}"

# 6. Parse options
while [[ $# -gt 0 ]]; do
  case "$1" in
    --option) OPTION="$2"; shift 2 ;;
    *) echo "{\"error\": \"Unknown option: $1\"}" >&2; exit 1 ;;
  esac
done

# 7. Do the work (handle quirks internally)
# - Encode URLs, use correct flags, escape special chars, etc.

# 8. Return consistent JSON
jq -n --arg result "$RESULT" '{"result": $result}'
```

> **TypeScript skills** follow the same structure (shebang, usage, validate, defaults, work, output) but use a shared `lib.ts` for auth, HTTP, and validation. See `~/.claude/skills/gitlab/scripts/` for reference.

### Handling API Errors

API calls can fail (404, 401, 500). Don't let raw errors leak through:

```bash
# Capture API response and check for errors
RESPONSE=$(glab api "endpoint" 2>&1) || {
  # Check if it's an HTTP error
  if echo "$RESPONSE" | grep -q "HTTP [0-9]"; then
    jq -n --arg msg "$RESPONSE" '{"error": $msg}' >&2
  else
    jq -n --arg msg "API request failed" '{"error": $msg}' >&2
  fi
  exit 1
}

# Validate response is valid JSON before processing
if ! echo "$RESPONSE" | jq -e . >/dev/null 2>&1; then
  jq -n '{"error": "Invalid API response"}' >&2
  exit 1
fi

# Now safe to process
echo "$RESPONSE" | jq '{id, name}'
```

### Pagination

GitLab API paginates by default (20 items). For complete results:

```bash
# Use --paginate for full results (careful with large datasets)
glab api "endpoint" --paginate | jq '...'

# Or set per_page for bounded queries
glab api "endpoint?per_page=100" | jq '...'
```

---

## Checklist for New Skills

### Before Building
- [ ] Identify the underlying tool's quirks and pain points
- [ ] List common operations users need
- [ ] Design intuitive command signatures

### For Each Script
- [ ] Dependency checks at the top
- [ ] Input validation before any work
- [ ] Sensible defaults for optional params
- [ ] Internal handling of: encoding, escaping, correct flags
- [ ] JSON output (or plain text if that's the data type)
- [ ] JSON errors to stderr
- [ ] Usage message on wrong args
- [ ] Made executable (`chmod +x`)

### Documentation
- [ ] SKILL.md with clear usage examples
- [ ] Comment in each script explaining what quirks it handles
- [ ] `reference.md` or `references/` directory if raw API/tool docs would be useful

### Testing
- [ ] Happy path works
- [ ] Validation errors return clean JSON
- [ ] Edge cases handled (empty results, special chars)

---

## Log Challenges During Development

When building a new skill, document challenges in a log file at `~/.claude/skills/{skill-name}-implementation-log.md`.

Include:
- What went wrong
- Workarounds applied
- Root causes identified

These logs inform future workflow improvements. Review logs periodically to update skill templates and documentation.

Example prompt to give Claude during skill development:
> "If you face any challenges, document them in `~/.claude/skills/{skill-name}-implementation-log.md` so we can later improve the workflow."

---

## Validate External Assumptions

Skills that depend on external systems (APIs, HTML structure, third-party tools) should verify assumptions before building:

1. **Inspect before coding**: Use browser/API tools to understand actual structure
2. **Test assumptions in isolation**: Verify selectors, endpoints, flags work as expected
3. **Build incrementally**: Use `--dry-run` or `--limit N` to validate before full execution
4. **Prefer resilient patterns**: Attribute-based selectors over structural ones, explicit flags over implicit behavior

External systems change. Skills should fail gracefully with actionable errors when assumptions break.

---

## Example: GitLab Skill

The `gitlab` skill demonstrates these principles:

- Scripts covering MR and pipeline operations
- **Zero quirks exposed**: URL encoding, `--description` vs `--body`, `--yes` flags
- **Consistent JSON**: All data operations return parseable output
- **Validation**: Project format (`org/project`), numeric IDs
- **Sensible defaults**: `open` discussions, `main` target branch, `all` pipeline jobs

Reference implementation: `~/.claude/skills/gitlab/`

---

## SPEC-Driven Development

For non-trivial skills, start with a collaborative SPEC.md:

1. **Create skill directory** with SPEC.md
2. **Draft spec collaboratively** - architecture, schemas, flow, decisions
3. **Implement from spec** - spec serves as source of truth
4. **Keep SPEC.md** - retain as historical artifact, not deleted post-implementation

### Why Keep SPEC.md

- Documents original design intent
- Shows evolution of thinking
- Useful for future refactoring
- Serves as onboarding context

### Example

See `~/.claude/skills/ai-eval-research/SPEC.md` for reference.
