# agentnanny

Stop clicking "Allow" on every Claude Code permission prompt.

agentnanny auto-approves the tools you trust (file reads, shell commands, etc.) while still blocking the ones you don't (force-push, rm -rf, DROP TABLE). Each terminal gets its own rules. Permissions expire automatically.

**Before:** Claude asks permission for every file read, every grep, every test run. You click "Allow" hundreds of times a day, or use `--dangerously-skip-permissions` and lose all safety.

**After:** Pick a profile, start working. Safe operations auto-approve. Dangerous commands still prompt.

## Quick Start

```bash
git clone https://github.com/syd-ppt/agentnanny.git
cd agentnanny
python agentnanny.py install                                   # one-time setup
python agentnanny.py run safe-dev -- claude -p "refactor auth"  # done
```

`safe-dev` auto-approves reading/writing files and safe shell commands (ls, cat, grep, find) for 8 hours, then cleans up. Claude Code behaves normally for everything else.

## Profiles

Profiles bundle groups, deny patterns, and TTL under a name. Five ship built-in:

| Profile | Groups | Deny | TTL | Use case |
|---|---|---|---|---|
| **safe-dev** | filesystem, safe-shell | — | 8h | Interactive development (default) |
| **full-dev** | filesystem, shell, network | rm -rf, DROP TABLE, force-push | 8h | Trusted local work |
| **reviewer** | read-only, review-shell | — | 4h | Code review sessions |
| **overnight** | filesystem, shell, network | force-push, reset --hard | 12h | Long-running autonomous tasks |
| **ci-runner** | filesystem, shell | curl\|sh, wget\|sh, WebFetch, WebSearch | 1h | Automated/headless runs |

```bash
python agentnanny.py profiles   # list all available profiles
```

### Using profiles

**Wrap a command** (recommended — creates session, runs command, auto-cleans up):

```bash
python agentnanny.py run safe-dev -- claude -p "refactor the auth module"
python agentnanny.py run overnight -- claude -p "migrate all tests to pytest"
python agentnanny.py run reviewer -- claude   # interactive review session
```

**Override profile defaults** with CLI flags (flags merge on top):

```bash
# safe-dev + extra deny rule + shorter TTL
python agentnanny.py run safe-dev -d "Bash(curl*)" --ttl 2h -- claude -p "do stuff"

# reviewer + network access
python agentnanny.py run reviewer -g network -- claude
```

**Interactive terminal session** (for persistent shell use):

```bash
eval $(python agentnanny.py activate safe-dev)
claude   # work interactively
eval $(python agentnanny.py deactivate)
```

**Modify a running session** (add groups, tools, or deny patterns without restarting):

```bash
python agentnanny.py extend -g network          # add network group to current session
python agentnanny.py extend -t WebFetch -d "Bash(curl*)"  # add tool + deny pattern
```

### Shell aliases

Add to `~/.bashrc` or `~/.zshrc`:

```bash
# ── agentnanny ──────────────────────────────────────────────────────────
_AN="$(command -v agentnanny.py 2>/dev/null || echo /path/to/agentnanny.py)"
nanny-on()  { eval $(python "$_AN" activate "$@"); }
nanny-off() { eval $(python "$_AN" deactivate); }
alias nanny-safe='nanny-on safe-dev'
alias nanny-full='nanny-on full-dev'
alias nanny-night='nanny-on overnight'
alias nanny-status='python "$_AN" status'
alias nanny-log='python "$_AN" log'
```

```bash
nanny-safe                           # filesystem + safe-shell, 8h
nanny-full                           # filesystem + shell + network, 8h
nanny-night                          # 12h with force-push/hard-reset denied
nanny-on full-dev -g pydev --ttl 4h  # profile + extras
nanny-off                            # deactivate current session
nanny-status                         # check hook/session status
```

## Install

```bash
python agentnanny.py install                    # register hooks in ~/.claude/settings.json
python agentnanny.py trust /path/to/project     # pre-trust a directory (optional)
```

`install` registers two hooks: a `PermissionRequest` hook for policy enforcement, and a `PostToolUse` hook for context window pressure monitoring.

Verify: `python agentnanny.py status`

Uninstall: `python agentnanny.py uninstall`

### Requirements

- Python 3.10+ (stdlib only, no dependencies)
- tmux (daemon mode only)

## How It Works

agentnanny registers a `PermissionRequest` hook with Claude Code. When Claude wants to use a tool, the hook evaluates the request against the active session policy.

```
Claude Code fires PermissionRequest
        │
        ▼
  Global deny list ──── match ──▶ DENY
        │ no match
        ▼
  Active scope? ──── no ──▶ PASSTHROUGH (normal Claude prompt)
        │ yes
        ▼
  Session deny ──── match ──▶ DENY
        │ no match
        ▼
  Session allow ──── match ──▶ ALLOW
        │ no match
        ▼
     PASSTHROUGH (normal Claude prompt)
```

A separate `PostToolUse` hook monitors context window pressure and warns when usage exceeds configurable thresholds (default: 60% warning, 75% critical).

With no active scope (no `run` or `activate`), the permission hook does nothing — Claude Code behaves normally.

## Configuration

Configuration layers, each overriding the previous:

| Layer | File | Scope |
|---|---|---|
| 1. Built-in defaults | (in code) | Profiles and groups work without any config |
| 2. Script-adjacent | `config.toml` | Next to agentnanny.py |
| 3. User config | `~/.config/agentnanny/config.toml` | Personal preferences (all projects) |
| 4. Project config | `.agentnanny.toml` | Per-repo, version-controllable |
| 5. Environment vars | `AGENTNANNY_*` | Runtime overrides |

Scaffold a project config: `python agentnanny.py init`

### Groups

Groups bundle related tools under a name:

| Group | Tools |
|---|---|
| `read-only` | Read, Glob, Grep |
| `write` | Write, Edit |
| `filesystem` | Read, Write, Edit, Glob, Grep |
| `shell` | Bash |
| `safe-shell` | Bash(ls\*), Bash(cat\*), Bash(head\*), Bash(grep\*), Bash(find\*) |
| `review-shell` | Bash(git log\*), Bash(git diff\*), Bash(git show\*), Bash(git blame\*) |
| `network` | WebFetch, WebSearch |
| `all` | .\* (everything) |

```bash
python agentnanny.py list-groups   # show all configured groups (builtin + custom)
```

Add custom groups in any config layer:

```toml
[groups]
dev = ["Bash(pytest*)", "Bash(uv*)", "Bash(npm*)"]
```

### Custom profiles

```toml
[profiles.my-project]
groups = ["filesystem", "shell", "dev"]
deny = ["Bash(rm -rf*)"]
ttl = "4h"
```

Then: `python agentnanny.py run my-project -- claude`

### Deny patterns

Two levels, same syntax:

**Global** (config file — every session):

```toml
[hooks]
deny = ["Bash(rm -rf*)", "Bash(git push --force*)", "Bash(DROP TABLE*)"]
```

**Per-session** (CLI flag — one session):

```bash
python agentnanny.py run safe-dev -d "Bash(rm*),Bash(curl*)" -- claude
```

**Pattern syntax:**

| Pattern | Matches |
|---|---|
| `Bash` | Any Bash tool call |
| `Bash(rm*)` | Bash commands starting with `rm` |
| `Bash(rm -rf*)` | Bash commands starting with `rm -rf` |
| `Bash(curl*\|wget*)` | Bash commands starting with `curl` or `wget` |
| `Write(/etc/*)` | Write calls to paths under `/etc/` |
| `WebFetch(*evil.com*)` | WebFetch calls containing `evil.com` |
| `.*Fetch.*` | Regex against tool name |
| `.*` | Everything |

Inside `Tool(pattern)`, `*` matches anything and `?` matches a single character. Use `|` for alternation.

### Project config (.agentnanny.toml)

Create in your repo root (or run `python agentnanny.py init`):

```toml
[hooks]
deny = ["Bash(terraform destroy*)", "Bash(aws iam delete*)"]

[groups]
infra = ["Bash(terraform*)", "Bash(aws*)"]

[profiles.infra-dev]
groups = ["filesystem", "safe-shell", "infra"]
deny = ["Bash(terraform destroy*)"]
ttl = "4h"
```

Commit to share with your team. Everyone with agentnanny installed picks up the project rules.

### Full config.toml reference

```toml
[hooks]
deny = []
# allow = ["Bash", "Read"]   # explicit allow list (legacy, used when no scope is active)

[context]
warn_percent = 60              # context window warning threshold (PostToolUse hook)
critical_percent = 75          # context window critical threshold

[daemon]
session = "claude"
poll_interval = 0.3
cooldown_seconds = 2.0
dry_run = false

[logging]
audit_log = "/tmp/agentnanny.log"
level = "actions"              # "actions" or "all"
max_size_bytes = 10485760      # 10MB, then rotate
backup_count = 3               # keep 3 rotated backups
```

### Environment variables

| Variable | Purpose |
|---|---|
| `AGENTNANNY_SCOPE` | Session scope ID (set by `activate`, read by hook) |
| `AGENTNANNY_SESSION` | Override tmux session name |
| `AGENTNANNY_DENY` | Override global deny list (comma-separated) |
| `AGENTNANNY_LOG` | Override audit log path |
| `AGENTNANNY_DRY_RUN` | Set `1` to log without acting |

## Observability

```bash
python agentnanny.py status              # hook install, daemon, active sessions
python agentnanny.py sessions            # list active session policies
python agentnanny.py explain [scope_id]  # inspect a session policy in detail
python agentnanny.py list-groups         # show all configured groups
python agentnanny.py test-policy Bash -i '{"command":"rm -rf /"}' # dry-run policy check
python agentnanny.py log                 # tail the audit log (last 50 entries)
python agentnanny.py log -n 100 -f json --tool Bash --action denied  # filtered JSON output
python agentnanny.py log -f table        # formatted table output
```

### Session management

```bash
python agentnanny.py sessions            # list active sessions
python agentnanny.py explain             # inspect current session (from AGENTNANNY_SCOPE)
python agentnanny.py extend -g network   # add groups/tools/deny to current session
python agentnanny.py prune               # remove expired session files
```

## tmux Daemon (Fallback)

For WSL/headless environments where the hook mechanism doesn't cover all prompts, the tmux daemon polls pane buffers and auto-responds to permission widgets.

```bash
python agentnanny.py watch [session-name]   # start
python agentnanny.py stop                    # stop
```

Detects: permission prompts (selects "allow for project" or "yes"), trust prompts, "Continue?" prompts, collapsed transcripts.

## All Commands

| Command | Description |
|---|---|
| `install` | Register hooks in `~/.claude/settings.json` |
| `uninstall` | Remove hooks from `~/.claude/settings.json` |
| `trust [dir]` | Pre-trust a directory |
| `run <profile> -- <cmd>` | Run command with session-scoped permissions |
| `activate [profile]` | Create a session policy (prints `export` command) |
| `deactivate [scope_id]` | Remove a session policy |
| `extend [scope_id]` | Add groups, tools, or deny patterns to a session |
| `profiles` | List available profiles |
| `list-groups` | List all configured groups |
| `sessions` | List active session policies |
| `explain [scope_id]` | Inspect a session policy in detail |
| `test-policy <tool>` | Dry-run policy evaluation |
| `status` | Show hook + daemon status |
| `log` | Tail the audit log (supports `--format`, `--tool`, `--action`, `-n`) |
| `prune` | Remove expired session files |
| `init` | Create `.agentnanny.toml` in current directory |
| `watch [session]` | Start tmux daemon |
| `stop` | Stop tmux daemon |
