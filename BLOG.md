# Capability, Not Credential: Secure GitHub Access for AI Agents

*How to give autonomous AI agents the power to work with your code without handing them the keys to the kingdom.*

---

## The Rise of Autonomous Coding Agents

Tools like OpenAI Codex, Claude Code, and Sprites are changing how we write software. Instead of writing every line yourself, you describe what you want, and an AI agent writes the code, runs tests, creates pull requests, and iterates based on feedback.

But there's a problem. These agents need access to your repositories to be useful. And the moment you give an AI agent your GitHub credentials, you've created a security liability that scales with the agent's autonomy.

## The Credential Problem

The simplest way to give an AI agent GitHub access is `gh auth login`. The agent authenticates interactively (or via a token), and the GitHub CLI stores credentials in `~/.config/gh/`. From that point on, the agent can run any `gh` command—including `gh auth token`, which prints the credential in plain text.

This is a problem. Classic PATs (personal access tokens) are scoped by permission type (repo, workflow, admin) but apply to *every repository you can access*. Give an agent a classic PAT with `repo` scope, and it can read and write to your private repositories, your employer's repositories, any organization you belong to.

### Fine-grained PATs: A Partial Solution

GitHub's fine-grained PATs improve this significantly. You can scope a token to specific repositories and grant granular permissions: read-only contents, issues but not PRs, metadata only. This limits the blast radius—if the token leaks, attackers can only access what the token permits.

For AI agents, a fine-grained PAT scoped to a single repository with only Contents, Issues, Pull Requests, and Metadata permissions is a reasonable starting point.

But there's still a problem.

### The Exposure Risk

Even with a perfectly scoped fine-grained PAT, if the agent *has* the token, the agent can *leak* the token.

AI coding agents write code. They execute shell commands. They make HTTP requests. They create files. Any of these actions could inadvertently—or through prompt injection—expose the token:

- The agent writes a debug script that logs environment variables (including `GH_TOKEN`) to a file that gets committed
- The agent crafts a curl request and echoes the command, token included, to stdout
- The agent creates a GitHub issue with diagnostic info that happens to include the token
- A malicious prompt tricks the agent into posting credentials to an external URL

This isn't hypothetical paranoia. LLMs can be manipulated. They make mistakes. They follow instructions—including instructions hidden in files they're asked to process.

If your repository is sensitive, you don't want the token in the agent's address space at all. You want the agent to have *capabilities* without having *credentials*.

## A Third Way: Capability-Based Access

The solution is borrowed from operating system security: instead of giving the agent credentials, give it *capabilities*.

A capability is a limited permission to perform a specific action. The agent doesn't hold the credential—it holds access to a controlled interface that can perform certain operations on its behalf.

Here's the key insight: **the agent doesn't need to authenticate itself. It needs to perform specific tasks.**

## How ghcap Works

ghcap implements this pattern for GitHub CLI access. The architecture has three components:

**1. Root-owned token storage**

The GitHub PAT is stored in a file that only root can read (`/run/secrets/agent_github_pat`). The agent process, running as an unprivileged user, cannot access this file directly.

**2. An allowlisted wrapper**

A root-owned script (`/usr/local/bin/ghcap`) reads the token and executes GitHub CLI commands—but only if they're on an explicit allowlist:

```
Allowed:
  gh repo view, clone
  gh pr create, comment, view, list, checks, status
  gh issue create, comment, view, list

Blocked:
  gh auth (would reveal token status)
  gh api (raw API access, too powerful)
  gh repo delete (obviously)
  gh pr merge (requires human review)
  Everything else (deny by default)
```

**3. A transparent shim**

The agent sees a normal `gh` command in its PATH. Under the hood, it's a shim that calls `sudo ghcap`. From the agent's perspective, GitHub CLI "just works"—it doesn't know or care that there's a security layer.

```
Agent runs:  gh repo clone owner/repo
     ↓
Shim runs:   sudo ghcap repo clone owner/repo
     ↓
ghcap:       Checks allowlist → "repo clone" is allowed
             Reads token from root-only file
             Executes real gh with token injected
     ↓
Result:      Repo cloned successfully
```

## Defense in Depth

ghcap is one layer in a multi-layer security model:

| Layer | What it Protects Against |
|-------|-------------------------|
| **ghcap allowlist** | Agent running dangerous commands (delete, merge, auth) |
| **Fine-grained PAT** | Token accessing repos beyond its scope |
| **Repository selection** | Token scoped to specific repos, not all repos |
| **Branch protection** | Direct pushes to main/prod branches |
| **PR review requirements** | Code merged without human approval |

No single layer is perfect. Together, they create a security posture where an AI agent can be productive without being dangerous.

## Why This Matters at Scale

When you're running one agent on one repository, manual oversight is feasible. But the value proposition of AI agents is scale: dozens of agents working on hundreds of repositories, creating PRs faster than humans can review them.

At that scale, the credential model breaks down:

- You can't manually review every command an agent runs
- A leaked or misused token affects all repositories it can access  
- Revoking access means regenerating tokens across all agents
- Audit trails become meaningless when everything runs under one credential

The capability model scales differently:

- Agents are limited to safe operations by design
- A compromised agent can only do what the allowlist permits
- Access can be tuned per-agent by adjusting their allowlist
- Every operation goes through a single chokepoint that can be logged

## Implementation Details

The ghcap init script handles all the setup:

1. Prompts for your GitHub PAT (hidden input)
2. Stores it root-only at `/run/secrets/agent_github_pat`
3. Creates a config file pointing to the token and the real `gh` binary
4. Installs the ghcap wrapper with the hardcoded allowlist
5. Adds a sudoers rule allowing the agent user to run only ghcap
6. Installs a `gh` shim in the agent's PATH

After running init, the agent can use `gh` normally for allowed operations:

```bash
# These work
gh repo clone owner/repo ~/work/repo
gh pr create --title "Fix bug" --body "Description"
gh issue comment 123 --body "Working on this"

# These are denied
gh auth status          # Denied
gh api /user            # Denied
gh repo delete owner/repo  # Denied
```

## Operational Considerations

**Token persistence:** By default, the token lives in `/run`, which is cleared on reboot. For persistent setups, point `SECRET_FILE` to `/etc/secrets/agent_github_pat` before running init.

**Multiple agents:** Each agent user gets their own shim and sudoers rule, but they can share the same ghcap wrapper and token (if they should have identical access).

**Extending the allowlist:** Edit `/usr/local/bin/ghcap` to add commands. The pattern is explicit: if it's not in the case statement, it's denied.

**Logging:** Add logging to ghcap for audit trails:

```bash
echo "$(date -Is) $USER: ghcap $*" >> /var/log/ghcap.log
```

## The Broader Pattern

ghcap solves a specific problem (GitHub access), but the pattern applies broadly:

- **Database access:** Agents get query capabilities, not connection strings
- **Cloud APIs:** Agents call through a proxy that allowlists specific operations
- **File systems:** Agents work in sandboxed directories, not full filesystem access
- **Network:** Agents access specific endpoints, not arbitrary network access

The principle is consistent: give agents the minimum capability they need to do their job, mediated through a control plane you own.

## Getting Started

The code is available at [github.com/yourorg/ghcap](https://github.com/yourorg/ghcap).

```bash
# Install
git clone https://github.com/yourorg/ghcap.git
cd ghcap
sudo bash scripts/init <agent_username>

# Test
sudo -u <agent_username> bash -lc 'gh repo view owner/repo'
```

Create a fine-grained PAT with minimal permissions (Contents, Issues, Pull Requests, Metadata), scoped to only the repositories your agent needs. Add branch protection rules as a second line of defense.

Then let your agents work—with the confidence that they can't do more than you've explicitly allowed.

---

*ghcap is open source under the MIT license. Contributions welcome.*