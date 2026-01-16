# ghcap Security Model

This document explains the security architecture of ghcap and the threat model it addresses.

## Threat Model

### What We're Protecting Against

| Threat | Description |
|--------|-------------|
| **Credential exfiltration** | Agent reads the token and sends it elsewhere |
| **Privilege escalation** | Agent uses token for operations beyond its needs |
| **Accidental damage** | Agent runs destructive commands (delete repo, force push) |
| **Scope creep** | Agent accesses repositories outside its intended scope |

### What We're NOT Protecting Against

| Threat | Mitigation |
|--------|------------|
| **Malicious commits** | Code review, branch protection, CI checks |
| **Root compromise** | OS-level security, VM isolation |
| **Token theft from root** | Disk encryption, secure boot, HSM (out of scope) |

## Security Layers

### Layer 1: Token Isolation

The GitHub PAT is stored in a root-only file:

```
/run/secrets/agent_github_pat
  Owner: root:root
  Mode:  600 (rw-------)
```

The agent process runs as an unprivileged user (e.g., `sprite`) and cannot read this file directly.

### Layer 2: Command Allowlisting

The ghcap wrapper implements an explicit allowlist. Commands not on the list are denied:

```bash
# Allowed
repo view, clone
pr create, comment, view, list, checks, status
issue create, comment, view, list

# Explicitly blocked
auth, api

# Implicitly blocked (everything else)
repo delete, repo create, pr merge, workflow, ...
```

The allowlist is hardcoded in `/usr/local/bin/ghcap`. There's no configuration file an agent could modify.

### Layer 3: Environment Isolation

Each ghcap invocation runs in a clean environment:

```bash
env -i PATH=/usr/local/bin:/usr/bin:/bin HOME="$TMPHOME" \
  GH_TOKEN="$PAT" GITHUB_TOKEN="$PAT" \
  "$GH_BIN" "$subcmd" "$@"
```

- `env -i` clears inherited environment variables
- A fresh `$TMPHOME` is created and destroyed for each invocation
- Only essential PATH directories are included

This prevents:
- Environment variable leakage
- Persistent gh configuration/auth state
- Home directory snooping

### Layer 4: Sudo Restriction

The sudoers rule is narrowly scoped:

```
sprite ALL=(root) NOPASSWD: /usr/local/bin/ghcap
```

The agent can only sudo to run ghcap—nothing else. No shell access, no other commands.

## Attack Surface Analysis

### Can the agent read the token?

**No.** The token file is root-only (mode 600). The agent runs as an unprivileged user.

Attempted attack:
```bash
cat /run/secrets/agent_github_pat
# Permission denied

sudo cat /run/secrets/agent_github_pat
# sprite is not allowed to run 'cat' as root
```

### Can the agent run gh auth to see token status?

**No.** `gh auth` is explicitly blocked:

```bash
gh auth status
# Denied
```

### Can the agent use gh api for raw API calls?

**No.** `gh api` is explicitly blocked:

```bash
gh api /user
# Denied
```

### Can the agent delete a repository?

**No.** `gh repo delete` is not on the allowlist:

```bash
gh repo delete owner/repo
# Denied
```

### Can the agent merge a PR?

**No.** `gh pr merge` is not on the allowlist:

```bash
gh pr merge 123
# Denied
```

### Can the agent push to a protected branch?

**Depends on GitHub configuration.** ghcap doesn't prevent this, but GitHub branch protection rules do. This is a defense-in-depth measure.

### Can the agent access repositories not scoped to the PAT?

**No.** Fine-grained PATs are scoped to specific repositories. Attempting to access other repos will fail at the GitHub API level, regardless of ghcap.

### Can the agent modify the ghcap script?

**No.** ghcap is owned by root with mode 755. The agent cannot write to it:

```bash
echo "malicious code" >> /usr/local/bin/ghcap
# Permission denied
```

### Can the agent modify the sudoers rule?

**No.** Sudoers files are root-owned and mode 440.

## Trust Boundaries

```
┌─────────────────────────────────────────────────────────────┐
│                    Untrusted Zone                           │
│                                                             │
│   Agent Process (uid: sprite)                               │
│   - Can run arbitrary code                                  │
│   - Cannot read /run/secrets/*                              │
│   - Cannot sudo except to ghcap                             │
│                                                             │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ sudo /usr/local/bin/ghcap <args>
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    Control Plane                            │
│                                                             │
│   ghcap (uid: root)                                         │
│   - Validates command against allowlist                     │
│   - Reads token from secure storage                         │
│   - Executes allowed operations only                        │
│                                                             │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ gh <command> with GH_TOKEN
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    External Service                         │
│                                                             │
│   GitHub API                                                │
│   - Validates token                                         │
│   - Enforces repository scope                               │
│   - Enforces branch protection                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Comparison to Alternatives

### Alternative: Give agent the PAT directly

```
Agent → GH_TOKEN=xxx gh <any command> → GitHub
```

**Risks:**
- Agent can exfiltrate token
- Agent can run any gh command
- Token persists in agent's environment
- No audit point for operations

### Alternative: OAuth app with limited scopes

**Limitations:**
- OAuth scopes are coarse-grained
- Still requires token handling
- More complex setup
- Not suitable for CLI-based workflows

### Alternative: GitHub App with installation tokens

**Trade-offs:**
- Better for organization-wide access
- More complex to set up
- Requires webhook/callback infrastructure
- Installation tokens still need protection

ghcap is simpler and sufficient for single-machine, single-user agent deployments.

## Recommendations

1. **Use fine-grained PATs** scoped to specific repositories
2. **Enable branch protection** on main/production branches
3. **Require PR reviews** before merging
4. **Set token expiration** to limit exposure window
5. **Monitor ghcap usage** via logging (add to the wrapper)
6. **Run agents in isolated VMs** when possible
7. **Rotate tokens regularly** and re-run init

## Limitations

- ghcap trusts the real `gh` binary. A compromised gh binary could leak tokens.
- Token is stored on disk; disk access by root exposes it.
- ghcap doesn't validate command arguments beyond the subcommand level.
- No rate limiting is enforced by ghcap (GitHub's rate limits apply).

For higher-security environments, consider:
- Hardware security modules (HSMs) for token storage
- Network-level isolation for agent VMs
- More granular argument validation in ghcap
