# Majordomo SDLC CLI — Installation guide

## Contents

1. [Before you start](#1-before-you-start)
2. [Install the CLI](#2-install-the-cli)
3. [Configure it](#3-configure-it)
4. [Sign in](#4-sign-in)
5. [Next steps](#5-next-steps)

---

The Majordomo SDLC CLI, `majordomo-sdlc`, starts a Docker container on your machine for each issue, and a coding agent (Claude Code, Codex or Copilot) works on your repo inside it. This guide installs and configures the CLI. Using it, from creating the issue to merging the PR, is in the [Developer manual](developer-manual.md).

## 1. Before you start

Check each of these once per machine.

| You need | Why | How to check |
|---|---|---|
| **Docker** (Docker Desktop on Windows/macOS, Docker Engine on Linux), running | every execution runs in a container | `docker info` prints server details, no error |
| **Disk space**: ~12 GB free | the agent image is large | — |
| **A GitHub (or Bitbucket) account** with access to the working repo | you sign in to the Majordomo server with it | you can open the repo in the browser |
| **A repo token** (`REPO_TOKEN`) | the container clones, reads the issue, commits, pushes and opens the PR with it | see below |
| **The agent's login**, depending on the agent the Majordomo server names for your repo | the agent in the container uses it | see below |
| **The Majordomo server's address**, your **tenant id** and **workspace id** | every call goes there | ask your Majordomo admin |
| **The repo set up for SDLC** by your admin | `/start` is refused for repos that aren't set up | ask your Majordomo admin. See [What your admin sets up](#what-your-admin-sets-up) below. |

**Repo token.** A GitHub fine-grained personal access token on the working repo, with:

| Permission | Access |
|---|---|
| Contents | Read and write (clone, commit, push) |
| Pull requests | Read and write (open the PR) |
| Issues | Read (fetch the issue text) |
| Metadata | Read |

A classic token with the `repo` scope also works. Create it under *GitHub → Settings → Developer settings → Personal access tokens*.

**The agent's login.** Your admin can tell you which agent your repo uses. If you're not sure, run `/start`: it says which credential is missing before it pulls anything.

| Agent | What to do on this machine |
|---|---|
| `claude-code` | Install Claude Code and run `claude` once, then log in. The CLI reads `~/.claude/.credentials.json`. **Not supported on macOS yet**: there the login is kept in the Keychain, not in that file. |
| `codex` | Install the Codex CLI and run `codex login` (ChatGPT login or API key). The CLI reads `~/.codex/auth.json` (`CODEX_HOME` overrides the folder). |
| `copilot` | Create a fine-grained GitHub token with the **Copilot Requests** permission and set it as `COPILOT_GITHUB_TOKEN` ([section 3](#3-configure-it)). |

### What your admin sets up

You don't set up the repo yourself: a Majordomo admin does it once per repo ([Administrator manual](administrator-manual.md)). If you see either of these, send the message to your admin:

- `/start` is refused with `409 repo_not_registered` or `policy_missing`.
- Every `/sync` ends with `queued (north unreachable)`. (*north* is the Majordomo server's service name in CLI output.)

## 2. Install the CLI

Pick one of the two ways. Both give you the `majordomo-sdlc` command.

### Option A: standalone binary (no Node or Python needed)

The files are on the **cli-dev** release of [majordomo-sdlc-public](https://github.com/majordomoinc/majordomo-sdlc-public):

| Platform | File |
|---|---|
| Linux / WSL (x64) | `majordomo-sdlc-linux-x64` |
| Linux (ARM64) | `majordomo-sdlc-linux-arm64` |
| macOS (Apple Silicon) | `majordomo-sdlc-macos-arm64` |
| macOS (Intel) | `majordomo-sdlc-macos-x64` |
| Windows (x64) | `majordomo-sdlc-windows-x64.exe` |

Download from the [`cli-dev` release](https://github.com/majordomoinc/majordomo-sdlc-public/releases/tag/cli-dev) page, or with the GitHub CLI `gh`.

**Linux / WSL / macOS**

```bash
gh release download cli-dev --repo majordomoinc/majordomo-sdlc-public --pattern 'majordomo-sdlc-linux-x64'
chmod +x majordomo-sdlc-linux-x64
sudo mv majordomo-sdlc-linux-x64 /usr/local/bin/majordomo-sdlc     # or ~/.local/bin, anywhere on PATH
majordomo-sdlc --help
```

On macOS, a file downloaded with a browser is quarantined. Clear it once:
`xattr -d com.apple.quarantine /usr/local/bin/majordomo-sdlc`. Files fetched with `gh` aren't affected.

**Windows (PowerShell)**

```powershell
gh release download cli-dev --repo majordomoinc/majordomo-sdlc-public --pattern 'majordomo-sdlc-windows-x64.exe'
mkdir -Force $HOME\bin; Move-Item majordomo-sdlc-windows-x64.exe $HOME\bin\majordomo-sdlc.exe
[Environment]::SetEnvironmentVariable('Path', "$env:Path;$HOME\bin", 'User')   # once; open a new terminal after
majordomo-sdlc --help
```

The first time, SmartScreen may warn about an unsigned app: *More info → Run anyway*.

To upgrade, replace the file with a newer one.

### Option B: npm package (machines with Node ≥ 18)

```bash
gh release download cli-dev --repo majordomoinc/majordomo-sdlc-public --pattern 'md-sdlc-*.tgz'
npm install -g ./md-sdlc-0.1.0.tgz          # the version in the file name may differ
majordomo-sdlc --help
```

Upgrade by installing a newer `.tgz` the same way. `npm uninstall -g md-sdlc` removes it.

## 3. Configure it

### Where settings come from

The CLI reads each setting from the first place that has it:

1. **Environment variables.**
2. **`./.env`** in the folder you run the CLI from, or the file named by `MAJORDOMO_ENV_FILE`.
3. **`~/.majordomo/.env`** (Windows: `%USERPROFILE%\.majordomo\.env`). The per-user default: set it once and it works from any folder.


### The settings you need

```ini
# ~/.majordomo/.env

# The Majordomo server's address (ask your admin; it ends in /sdlc/v1).
BACKEND_API_URL=https://majordomo.example.com/sdlc/v1

# The tenant and workspace your repo is registered in (ask your admin).
TENANT_ID=00000000-0000-0000-0000-000000000000
WORKSPACE_ID=00000000-0000-0000-0000-000000000000

# The repo token from section 1.
REPO_TOKEN=github_pat_...

# Only if the Majordomo server uses the Copilot agent for your repo.
# COPILOT_GITHUB_TOKEN=github_pat_...

# Optional: the model for every stage run.
# AGENT_MODEL=
```

Format: one `KEY=value` per line. `export KEY=value` and quoted values work, and `#` starts a comment.

| Setting | Required | Notes |
|---|---|---|
| `BACKEND_API_URL` | yes | The Majordomo server's address, from your admin. It ends in `/sdlc/v1`. |
| `TENANT_ID`, `WORKSPACE_ID` | yes | UUIDs. Without them the Majordomo server answers `422 scope_required`. |
| `REPO_TOKEN` | yes | `/start` warns if it is missing. Without it the container can't clone a private repo, read the issue or push. |
| `COPILOT_GITHUB_TOKEN` | Copilot repos only | |
| `AGENT_MODEL` | no | Leave it unset to use the agent's default. |

All settings are in the [Developer manual, section 5.2](developer-manual.md#52-settings-reference).

**Keep these files out of git.** They hold tokens.

**Windows:** if you use environment variables instead of a file (`setx REPO_TOKEN "..."`), open a **new** terminal afterwards. Terminals that are already open don't see the change.

## 4. Sign in

Start the CLI and sign in with your git account:

```
$ majordomo-sdlc
Majordomo CLI -- one prompt for everything. /help for commands.
majordomo> /login
```

**Browser sign-in (default).** Your browser opens at GitHub. Approve, and the browser shows that you can close it. The CLI prints:

```
Logged in as **USER** via github. Session valid until 11/5/2026, 6:12:44 PM.
```

**Device sign-in.** Use this over ssh, on a machine without a browser, or when the browser flow doesn't work:

```
majordomo> /login --device
```

The CLI shows a URL and a code. Open the URL on any device, enter the code and approve. The CLI waits until you're done. It switches to this flow by itself over ssh and on Linux without a display.

**Bitbucket.** `/login --provider bitbucket`.

**When the session expires**, the Majordomo server answers `401` and the CLI tells you to sign in again. Run `/login`. Running and stopped containers pick up the new token automatically.

`/logout` removes the token from this machine.

## 5. Next steps

You're set up. The [Developer manual](developer-manual.md) covers the rest: using the prompt, creating the issue, starting an execution, running and reviewing each stage, and raising the PR.
