---
trigger: model_decision
description: Comprehensive Bash scripting reference. Apply when writing or reviewing .sh/.bash scripts. Covers strict mode, quoting, error handling, functions, arrays, I/O, security hardening, and full anti-pattern catalog.
---

# Bash Scripting Best Practices

Synthesized from: Google Shell Style Guide, shellcheck canon, unofficial strict mode, progrium/bashstyle, and production DevOps patterns.

---

## 1. Strict Mode — Every Script, No Exceptions

```bash
#!/usr/bin/env bash
set -euo pipefail
[[ "${TRACE-0}" == "1" ]] && set -x
```

| Option | Long form | Effect |
|--------|-----------|--------|
| `set -e` | `set -o errexit` | Exit on non-zero command |
| `set -u` | `set -o nounset` | Error on unset variable access |
| `set -o pipefail` | — | Pipeline fails if any stage fails |
| `set -x` | `set -o xtrace` | Print each command before execution |

**`set -e` gotchas — it does NOT catch:**
- Failures inside `$(subshell)` assigned to a variable
- Commands in `if`/`while` conditions
- Left side of `||` or `&&`
- Commands in a group after `local` (because `local` returns 0)

For these cases, validate explicitly:
```bash
result=$(risky_cmd) || { echo "risky_cmd failed" >&2; exit 1; }
```

---

## 2. Variables

### Naming Convention

| Type | Scope | Convention | Example |
|------|-------|------------|---------|
| Exported / env | Global | `UPPER_CASE` | `APP_ENV` |
| Script-global | Script | `_UPPER_CASE` | `_CONFIG_FILE` |
| Local | Function | `lower_case` | `retry_count` |

### Quoting Rules

```bash
# Always quote expansions
echo "${var}"                     # safe: spaces preserved
echo "${var}.yml"                 # safe: concatenation
echo "${var:-default}"            # safe: default value
echo "${var:?must be set}"        # fail fast if unset/empty

# Only skip quotes in arithmetic context
(( count++ ))
(( total = a + b ))
```

### Constants

```bash
readonly MAX_RETRIES=3
readonly BASE_DIR="/opt/app"
# readonly variables cannot be unset — declare early
```

### The local + command substitution trap

```bash
# WRONG: local always returns 0, masking cmd failure
local output=$(risky_cmd)   # set -e won't fire here

# CORRECT
local output
output=$(risky_cmd)         # now set -e fires on failure
```

---

## 3. Error Handling

### Canonical Cleanup Pattern

```bash
WORK_DIR=""

cleanup() {
  local exit_code=$?
  [[ -d "${WORK_DIR}" ]] && rm -rf "${WORK_DIR}"
  exit "${exit_code}"
}
trap cleanup EXIT ERR INT TERM

WORK_DIR="$(mktemp -d)"
```

### Trap for Debugging

```bash
trap 'echo "Error on line ${LINENO}: ${BASH_COMMAND}" >&2' ERR
```

### Intentional Non-Zero Exits

```bash
# Allow a command to fail without triggering set -e
grep -q "pattern" file || true

# Or capture exit code manually
if ! some_cmd; then
  echo "some_cmd failed, continuing" >&2
fi
```

### Fail Fast on Empty Paths

```bash
# :? operator — exits with error if var is empty or unset
rm -rf "${TARGET_DIR:?TARGET_DIR must be set}"
```

---

## 4. Functions

### Declaration Style

```bash
# CORRECT: POSIX function syntax
deploy_service() {
  local service_name="$1"
  local env="${2:-staging}"
  local timeout="${3:-60}"

  echo "Deploying ${service_name} to ${env}..."
}

# NEVER: function keyword (deprecated in Bash)
function deploy_service {
  ...
}
```

### Complex Argument Naming

```bash
# For functions with many args, use declare for self-documentation
process_artifact() {
  declare artifact_path="$1" target_env="$2" dry_run="${3:-false}"
  ...
}
```

### Variadic Functions

```bash
push_images() {
  local registry="$1"; shift
  local tag="$1"; shift
  local images=("$@")

  for image in "${images[@]}"; do
    docker push "${registry}/${image}:${tag}"
  done
}
```

### Return Values

```bash
# Return data via stdout
get_pod_count() {
  kubectl get pods --no-headers | wc -l
}
count=$(get_pod_count)

# Return status via exit code
is_healthy() {
  curl -sf "http://localhost:8080/health" > /dev/null
}
if is_healthy; then echo "OK"; fi
```

### Library Guard

```bash
# At the end of every script that can be both sourced and run
[[ "$0" == "${BASH_SOURCE[0]}" ]] && main "$@"
```

---

## 5. Conditionals

```bash
# String comparison
[[ "$env" == "production" ]]
[[ "$name" != "default" ]]
[[ -z "$var" ]]          # empty string
[[ -n "$var" ]]          # non-empty string

# Numeric comparison — use -eq/-gt/-lt in [[ ]]
[[ "$count" -gt 0 ]]
[[ "$retries" -le 3 ]]

# Arithmetic — use (( )) without $
(( count > 0 ))
(( retries <= max ))

# File tests
[[ -f "$file" ]]         # exists and is regular file
[[ -d "$dir" ]]          # exists and is directory
[[ -x "$binary" ]]       # exists and is executable
[[ -r "$file" ]]         # exists and is readable

# Pattern matching (no quotes on pattern side)
[[ "$filename" == *.yml ]]
[[ "$url" =~ ^https?:// ]]

# Combined conditions
[[ -f "$file" && -r "$file" ]]
[[ "$env" == "prod" || "$env" == "production" ]]
```

**Never use `>` for numeric comparison in `[[ ]]`** — it's lexicographic:
```bash
# WRONG: "9" > "10" is TRUE lexicographically
[[ 9 > 10 ]]   # true — bug!

# CORRECT
[[ 9 -gt 10 ]]  # false — correct
(( 9 > 10 ))    # false — correct
```

---

## 6. Arrays & Iteration

### Declaring and Expanding

```bash
services=("api" "worker" "scheduler")

# Iterate all elements (handles spaces in elements)
for svc in "${services[@]}"; do
  echo "Starting ${svc}"
done

# Array length
echo "${#services[@]}"

# Slice
echo "${services[@]:1:2}"   # elements 1 and 2

# Append
services+=("gateway")
```

### Capturing Command Output

```bash
# CORRECT: readarray preserves spaces, newlines
readarray -t pods < <(kubectl get pods -o name)

# WRONG: word-splits on IFS
pods=($(kubectl get pods -o name))
```

### Safe Line Iteration

```bash
# Preserves leading whitespace, handles empty lines
while IFS= read -r line; do
  echo "Processing: ${line}"
done < <(some_cmd)

# Pipe version creates a subshell — variable changes lost outside
# WRONG for variable mutation:
some_cmd | while IFS= read -r line; do
  count=$(( count + 1 ))  # lost after loop
done
```

### Null-Delimited File Iteration

```bash
while read -rd $'\0' filepath; do
  process_file "${filepath}"
done < <(find . -name "*.conf" -print0)
```

### Optional Arguments as Arrays

```bash
# WRONG: empty string becomes an argument
opts="--verbose"
[[ "$debug" == true ]] || opts=""
cmd $opts   # passes "" if debug=false

# CORRECT: empty array = no args passed
opts=()
[[ "$verbose" == true ]] && opts+=(--verbose)
[[ -n "$log_file" ]] && opts+=(--log-file "$log_file")
cmd "${opts[@]}"
```

---

## 7. Input / Output

### printf vs echo

```bash
# echo -e is non-portable; \n behavior differs per shell/platform
echo -e "line1\nline2"   # avoid

# printf is portable and predictable
printf 'line1\nline2\n'
printf '%s\n' "$var"     # safe for vars starting with -
```

### Herestrings / Heredocs

```bash
# Herestring — avoid spawning echo subprocess
grep "pattern" <<<"${input_string}"

# Heredoc — unquoted tag = interpolation enabled
cat <<EOF
Host: ${HOSTNAME}
EOF

# Heredoc — single-quoted tag = no interpolation
cat <<'EOF'
Literal ${var} not expanded
EOF
```

### Reading Files

```bash
content=$(< file)        # bash builtin read — no cat subprocess
```

### Sudo + Redirect

```bash
# WRONG: only printf runs as root; > runs as current user
sudo printf "config" > /etc/app/config

# CORRECT: tee runs as root
printf "config" | sudo tee /etc/app/config > /dev/null
```

### Stderr

```bash
# Logging helpers
log()   { printf '[%s] %s\n' "$(date -u +%FT%T)" "$*"; }
warn()  { printf '[WARN] %s\n' "$*" >&2; }
error() { printf '[ERROR] %s\n' "$*" >&2; }
die()   { error "$*"; exit 1; }
```

---

## 8. Script Structure Template

```bash
#!/usr/bin/env bash
set -euo pipefail
[[ "${TRACE-0}" == "1" ]] && set -x

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "$0")"

usage() {
  cat <<EOF
Usage: ${SCRIPT_NAME} [OPTIONS] <required_arg>

Options:
  -e, --env ENV       Target environment (default: staging)
  -v, --verbose       Enable verbose output
  -h, --help          Show this help

Environment Variables:
  DEBUG               Set to 'true' for debug output
  APP_CONFIG          Path to config file
EOF
}

# Handle help before arg parsing
[[ "${1-}" =~ ^-*h(elp)?$ ]] && { usage; exit 0; }

# Globals
WORK_DIR=""
ENV="${APP_ENV:-staging}"
VERBOSE=false

cleanup() {
  local exit_code=$?
  [[ -d "${WORK_DIR}" ]] && rm -rf "${WORK_DIR}"
  exit "${exit_code}"
}
trap cleanup EXIT ERR INT TERM

parse_args() {
  while [[ $# -gt 0 ]]; do
    case "$1" in
      -e|--env)     ENV="$2"; shift 2 ;;
      -v|--verbose) VERBOSE=true; shift ;;
      --)           shift; break ;;
      -*)           die "Unknown option: $1" ;;
      *)            break ;;
    esac
  done
}

main() {
  parse_args "$@"
  WORK_DIR="$(mktemp -d)"

  [[ "${VERBOSE}" == true ]] && set -x

  # ... core logic here
}

main "$@"
```

---

## 9. Security

```bash
# Fail if path var is empty before destructive operations
rm -rf "${TARGET:?TARGET must be set}"

# Avoid eval; if unavoidable, whitelist input
case "${action}" in
  start|stop|restart) systemctl "${action}" app ;;
  *) die "Invalid action: ${action}" ;;
esac

# Mask secrets from xtrace
{
  set +x
  export DB_PASSWORD="${DB_PASSWORD:?must be set}"
  set -x
}

# Sanitize filenames
safe_name="${input//[^a-zA-Z0-9._-]/}"

# Temp files — never hardcoded
WORK_DIR="$(mktemp -d)"
TMP_FILE="$(mktemp)"
```

---

## 10. Portability Notes

- `readarray`/`mapfile` requires Bash 4+; macOS ships with Bash 3.2 (use `brew install bash` or avoid)
- Associative arrays (`declare -A`) require Bash 4.4+
- `[[ ]]` is Bash/zsh only (not POSIX `sh`) — acceptable if shebang is `bash`
- `$'...'` C-style strings (e.g., `$'\n'`, `$'\0'`) are Bash 4+ (also ksh/zsh)
- Avoid `/bin/sh` in shebang unless script must be POSIX portable

---

## 11. Linting Checklist

```bash
# 1. Syntax check
bash -n script.sh

# 2. ShellCheck — zero warnings policy
shellcheck script.sh
shellcheck --severity=style script.sh   # strictest mode

# 3. Inline disable (specific rule only, never globally)
# shellcheck disable=SC2086
ls $glob_pattern  # intentional glob

# 4. Pre-commit integration
# .pre-commit-config.yaml:
# - repo: https://github.com/koalaman/shellcheck-precommit
#   hooks:
#     - id: shellcheck
```

---

## 12. Anti-Pattern Reference

| Anti-Pattern | Fix | Reason |
|---|---|---|
| `` `cmd` `` | `$(cmd)` | Deprecated; not nestable |
| `function myfunc { }` | `myfunc() { }` | `function` keyword is deprecated |
| `[ ]` / `test` | `[[ ]]` | Single bracket has more footguns |
| `for i in $(cmd)` | `while IFS= read -r i; done < <(cmd)` | IFS word-splits output |
| `arr=($(cmd))` | `readarray -t arr < <(cmd)` | Word-splits on spaces |
| `local var=$(cmd)` | `local var; var=$(cmd)` | `local` always exits 0 |
| `echo "$x" \| cmd` | `cmd <<<"$x"` | Avoids subshell; vars persist |
| `$(cat file)` | `$(< file)` | UUOC (Useless Use of Cat) |
| `[[ $a > $b ]]` numeric | `[[ $a -gt $b ]]` | `>` is lexicographic in `[[ ]]` |
| `(( $var == 1 ))` | `(( var == 1 ))` | `$` breaks on unset |
| `cd dir && cmd && cd ..` | `( cd dir; cmd )` | Subshell; original dir always restored |
| Unquoted `$var` | `"${var}"` | Word splitting + glob expansion |
| `opts="--flag"; cmd "$opts"` | `opts=(); cmd "${opts[@]}"` | Empty string ≠ no argument |
| `/bin/bash` shebang | `#!/usr/bin/env bash` | Hardcoded path; not portable |
| `set -e` alone | `set -euo pipefail` | `set -e` is incomplete protection |
| `>&/dev/null 2>&1` | `&>/dev/null` | Bash shorthand; cleaner |
| `source ./lib.sh` (bare path) | `source "${SCRIPT_DIR}/lib.sh"` | Relative path depends on CWD |
| `if [ $# > 0 ]` | `[[ $# -gt 0 ]]` | `>` redirects in `[ ]` |
| `newvar="$oldvar"` (direct assign) | `newvar=$oldvar` | Quotes unnecessary in direct assign |
