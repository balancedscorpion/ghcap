# ghcap Setup Guide

This guide walks through setting up ghcap for secure AI agent access to GitHub.

## Prerequisites

Before starting, ensure you have:

- A Linux system (Ubuntu 22.04+ or Debian 12+ recommended)
- Root/sudo access
- GitHub CLI (`gh`) installed
- A dedicated user account for your AI agent

### Installing GitHub CLI

```bash
# Ubuntu/Debian
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install gh
```

### Creating an Agent User

```bash
sudo useradd -m -s /bin/bash sprite
```

## Step 1: Create a Fine-Grained GitHub PAT

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens**
2. Click **Generate new token**
3. Configure the token:

| Setting | Value |
|---------|-------|
| Token name | `agent-ghcap` (or similar) |
| Expiration | 90 days (or your preference) |
| Repository access | **Only select repositories** → choose specific repos |

4. Under **Permissions → Repository permissions**, set:

| Permission | Access |
|------------|--------|
| Contents | Read and write |
| Issues | Read and write |
| Metadata | Read-only (required) |
| Pull requests | Read and write |

5. Click **Generate token** and copy it immediately (you won't see it again)

## Step 2: Install ghcap

```bash
# Clone the repository
git clone https://github.com/yourorg/ghcap.git
cd ghcap

# Make init executable
chmod +x scripts/init

# Run as root
sudo bash scripts/init sprite
```

When prompted, paste your GitHub PAT. The input is hidden.

You should see output like:

```
OK:
  Token stored (root-only): /run/secrets/agent_github_pat
  Config (root-only):       /etc/ghcap.conf
  Wrapper installed:        /usr/local/bin/ghcap
  Sudoers rule:             /etc/sudoers.d/ghcap-sprite
  Agent shim installed:     /home/sprite/.local/bin/gh
  Using gh binary:          /usr/bin/gh
```

## Step 3: Verify Installation

Test that the agent can use allowed commands:

```bash
# Switch to agent user and test
sudo -u sprite bash -lc 'gh repo view owner/repo'
```

Verify that blocked commands are denied:

```bash
sudo -u sprite bash -lc 'gh auth status'
# Should output: Denied
```

## Step 4: Clone a Repository

```bash
sudo -u sprite bash -lc 'mkdir -p ~/work && gh repo clone owner/repo ~/work/repo'
```

## Persistent Token Storage

By default, the token is stored in `/run/secrets/`, which is cleared on system reboot. For persistent storage:

```bash
# Set environment variable before running init
sudo SECRET_FILE=/etc/secrets/agent_github_pat bash scripts/init sprite
```

## Configuring AI Agents

### For Codex / Claude Code

Add to your system prompt or agent instructions:

```
You have access to GitHub via the `gh` CLI. Use it to:
- Clone repositories: gh repo clone owner/repo ~/work/repo
- Create PRs: gh pr create --title "..." --body "..."
- View PR status: gh pr view 123
- Comment on issues: gh issue comment 123 --body "..."

Do NOT run:
- gh auth (authentication is handled automatically)
- gh api (direct API access is not permitted)
```

### For Sprites.dev

Configure your Sprites instance to use the agent user (`sprite`) and include the clone instructions in your agent's startup prompt.

## Extending Permissions

To add commands to the allowlist, edit `/usr/local/bin/ghcap`:

```bash
# Example: Allow gh pr checkout
pr)
  case "${1:-}" in create|comment|view|list|checks|status|checkout) ;; *) echo "Denied"; exit 1 ;; esac
  ;;
```

## Adding Branch Protection (Recommended)

As an additional security layer, configure GitHub branch protection:

1. Go to **Repository → Settings → Branches**
2. Click **Add branch protection rule**
3. Set branch name pattern: `main` (or `master`, `prod`, etc.)
4. Enable:
   - Require a pull request before merging
   - require approvals (set to 1+)
   - Dismiss stale PR approvals when new commits are pushed

This ensures that even if an agent creates a PR, a human must approve it before merge.

## Troubleshooting

### "Denied" on expected commands

Check the allowlist in `/usr/local/bin/ghcap`. The command and subcommand must exactly match.

### "token file missing/empty"

The token may have been cleared (system reboot if using `/run`). Re-run:

```bash
sudo bash scripts/init sprite
```

### "gh: command not found"

Ensure `~/.local/bin` is in PATH:

```bash
echo $PATH | grep -q '.local/bin' || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Permission denied running sudo

Check that the sudoers file was created:

```bash
sudo cat /etc/sudoers.d/ghcap-sprite
# Should show: sprite ALL=(root) NOPASSWD: /usr/local/bin/ghcap
```

## Uninstalling

```bash
sudo rm /usr/local/bin/ghcap
sudo rm /etc/ghcap.conf
sudo rm /etc/sudoers.d/ghcap-sprite
sudo rm /run/secrets/agent_github_pat  # or wherever you stored it
rm ~/.local/bin/gh  # as the agent user
```
