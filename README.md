# ghcap

**Capability-based GitHub access for autonomous AI agents.**

ghcap provides a security wrapper around the GitHub CLI (`gh`) that gives AI agents like [Codex](https://openai.com/index/openai-codex/), [Claude Code](https://claude.ai/code), or [Sprites](https://sprites.dev) the ability to interact with GitHub repositories without ever exposing credentials directly.

## The Problem

When you give an AI agent access to your GitHub repositories, you typically have two bad options:

1. **Give the agent your credentials** — The agent can do anything: delete repos, push to main, access private data across all your repositories.

2. **Don't give access at all** — The agent can't clone, create PRs, or interact with your codebase meaningfully.

ghcap provides a third option: **capability-based access**.

## The Solution

Instead of handing the agent a token, ghcap:

- Stores your GitHub PAT in a root-only location the agent cannot read
- Provides an allowlisted wrapper that only permits specific, safe operations
- Presents a familiar `gh` interface to the agent (it doesn't know the difference)

The agent gets *capabilities* (clone, create PR, comment) without getting *credentials* (the token itself).

## Allowed Operations

| Command | Subcommands |
|---------|-------------|
| `gh repo` | `view`, `clone` |
| `gh pr` | `create`, `comment`, `view`, `list`, `checks`, `status` |
| `gh issue` | `create`, `comment`, `view`, `list` |

**Blocked:** `gh auth`, `gh api`, `gh repo delete`, `gh pr merge`, and everything else.

## Quick Start

### Prerequisites

Recommend to use https://sprites.dev for a quick set up. But below is included the minimum requirements if you don't use Sprites.

- Linux system (Ubuntu/Debian recommended)
- GitHub CLI (`gh`) installed
- A [fine-grained GitHub PAT](https://github.com/settings/tokens?type=beta) with appropriate permissions
- A non-root user account for your agent (e.g., `sprite`, `agent`)

### Sprites

Set up sprites.dev and access a machine. You get $30 free credits. This repository is not sponsored - though sponsorships from Fly.io would be welcomed :).


### Installation

#### Run the script as root:
```
# Run the init script as root
sudo bash scripts/init <agent_username>

# Enter your GitHub PAT when prompted
```

### Verify Installation

```bash
# As the agent user
sudo -u sprite bash -lc 'gh repo view owner/repo'
```

### Clone a Repository (as Agent)

```bash
sudo -u sprite bash -lc 'gh repo clone owner/repo ~/work/repo'
```

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│  Agent Process (unprivileged)                               │
│                                                             │
│  $ gh repo clone owner/repo                                 │
│       │                                                     │
│       ▼                                                     │
│  ~/.local/bin/gh  (shim script)                             │
│       │                                                     │
│       │  exec sudo -n /usr/local/bin/ghcap "$@"             │
│       ▼                                                     │
└───────┼─────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  ghcap (root-owned wrapper)                                 │
│                                                             │
│  1. Check command against allowlist                         │
│  2. Read token from /run/secrets/agent_github_pat           │
│  3. Execute real gh with GH_TOKEN in isolated env           │
│  4. Return result to agent                                  │
└─────────────────────────────────────────────────────────────┘
```

**Key security properties:**

- The agent never sees the PAT (it's read by root-owned ghcap)
- Commands are validated before execution
- Each invocation runs in a fresh, minimal environment
- Token file permissions: `600 root:root`

## GitHub PAT Permissions

When creating your fine-grained PAT, grant only:

| Permission | Access Level |
|------------|--------------|
| Contents | Read and write |
| Issues | Read and write |
| Metadata | Read-only (required) |
| Pull requests | Read and write |

**Recommended:** Scope the token to specific repositories only.

## Additional Security Layers

ghcap is one layer of defense. Consider also:

| Layer | Protection |
|-------|------------|
| **ghcap allowlist** | Limits which `gh` commands can run |
| **Fine-grained PAT** | Limits which repos/permissions the token has |
| **Branch protection** | Prevents direct pushes to main/prod (requires GitHub paid tier) |
| **PR reviews** | Requires human approval before merge |

## Configuration

Environment variables (set before running `init`):

| Variable | Default | Description |
|----------|---------|-------------|
| `SECRET_FILE` | `/run/secrets/agent_github_pat` | Token storage location |
| `CONF_FILE` | `/etc/ghcap.conf` | Config file location |
| `GHCAP_PATH` | `/usr/local/bin/ghcap` | Wrapper install path |

**Note:** `/run` is cleared on reboot. For persistence, set `SECRET_FILE=/etc/secrets/agent_github_pat`.

## Files Installed

| Path | Owner | Mode | Purpose |
|------|-------|------|---------|
| `/run/secrets/agent_github_pat` | root | 600 | Token storage |
| `/etc/ghcap.conf` | root | 600 | Points to token + real gh binary |
| `/usr/local/bin/ghcap` | root | 755 | Allowlisted wrapper |
| `/etc/sudoers.d/ghcap-<user>` | root | 440 | Passwordless sudo for ghcap only |
| `~/.local/bin/gh` | agent | 755 | Shim that calls sudo ghcap |

## Extending the Allowlist

Edit `/usr/local/bin/ghcap` to add commands. Example — adding `gh pr checkout`:

```bash
pr)
  case "${1:-}" in create|comment|view|list|checks|status|checkout) ;; *) echo "Denied"; exit 1 ;; esac
  ;;
```

## Use with AI Agent Platforms

### Codex / Claude Code

Include this in your system prompt or instructions:

```
Clone repositories using the preconfigured GitHub CLI:

  gh repo clone owner/repo ~/work/repo

Do not run `gh auth` or `gh api`. Do not attempt to read tokens or secrets.
```


## Troubleshooting

**"Denied" on a command you expected to work**
- Check the allowlist in `/usr/local/bin/ghcap`
- Verify the exact subcommand spelling

**"token file missing/empty"**
- Re-run `sudo bash scripts/init <user>` to reprovision the token
- Check if system rebooted (clears `/run`)

**"gh: command not found" as agent**
- Ensure `~/.local/bin` is on PATH: `echo $PATH`
- Run `hash -r` to clear command cache

## License

MIT

## Contributing

PRs welcome. Please ensure any changes maintain the security properties:
- Agent must never have read access to credentials
- Allowlist must be explicit (deny by default)
