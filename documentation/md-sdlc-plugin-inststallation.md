# Majordomo SDLC Claude plugin — Installation guide

## Contents

1. [Before you start](#1-before-you-start)
2. [Get the plugin](#2-get-the-plugin)
3. [Install it in Claude Code](#3-install-it-in-claude-code)
4. [Configure the Majordomo server](#4-configure-the-majordomo-server)
5. [Sign in](#5-sign-in)
6. [Check the installation](#6-check-the-installation)
7. [Next steps](#7-next-steps)
8. [Upgrade and uninstall](#8-upgrade-and-uninstall)

---

## 1. Before you start

You need the following.

| What | Why | Check |
|---|---|---|
| **Claude Code**, signed in | The plugin runs inside it. | `claude --version` |
| **Git** (on Windows: Git for Windows, which includes Git Bash) | Execution branches, commits and pushes. Claude Code on Windows also needs Git Bash to run the plugin's start-up hook. | `git --version` |
| **Python 3** (standard library only, nothing to `pip install`) | The plugin's helper script (`mdsync.py`) does the bookkeeping and the sync. | `python --version`, or `python3 --version`, or `py -3 --version` on Windows |
| **GitHub CLI `gh`**, signed in | Reads the issue and opens the pull request. | `gh auth status` |
| A **local clone** of the working repo | The plugin works in your checkout. | — |
| The **plugin zip**, downloaded from the Majordomo release page | See [section 2](#2-get-the-plugin). | — |
| From your Majordomo admin: the **server URL**, and the repo's **tenant ID** and **workspace ID** | See [section 4](#4-configure-the-majordomo-server). | — |

The working repo must already be registered on the Majordomo server by an admin (see the [Administrator manual](administrator-manual.md)). If it isn't, `/md-sdlc:start` fails with `repo_not_registered`.

## 2. Get the plugin

The plugin is the zip file **`md-sdlc.zip`**, published on the **Majordomo skills plugins** release (tag `skills-latest`) at [github.com/majordomoinc/majordomo-sdlc-public](https://github.com/majordomoinc/majordomo-sdlc-public/releases/tag/skills-latest). That release is replaced whenever a new version is built, so this link always serves the latest plugin:

```
https://github.com/majordomoinc/majordomo-sdlc-public/releases/download/skills-latest/md-sdlc.zip
```

Unzip it into a folder that will **stay where it is**. Claude Code installs the plugin from this folder, so don't unzip it into Downloads or a temp folder that gets cleaned up. The commands below download and unzip in one go.

**macOS / Linux**

```bash
mkdir -p ~/.majordomo/plugins
curl -fL -o /tmp/md-sdlc.zip https://github.com/majordomoinc/majordomo-sdlc-public/releases/download/skills-latest/md-sdlc.zip
unzip /tmp/md-sdlc.zip -d ~/.majordomo/plugins/md-sdlc
```

**Windows (PowerShell)**

```powershell
New-Item -ItemType Directory -Force "$HOME\.majordomo\plugins" | Out-Null
Invoke-WebRequest https://github.com/majordomoinc/majordomo-sdlc-public/releases/download/skills-latest/md-sdlc.zip -OutFile "$env:TEMP\md-sdlc.zip"
Expand-Archive "$env:TEMP\md-sdlc.zip" -DestinationPath "$HOME\.majordomo\plugins\md-sdlc"
```

The release also has a `SHA256SUMS` file if you want to verify the download.

The folder should now look like this:

```
md-sdlc/
  .claude-plugin/
    marketplace.json      <- the folder Claude Code is pointed at must contain this
  plugin/
    .claude-plugin/plugin.json
    hooks/  scripts/  skills/  assets/
```

`.claude-plugin` is a hidden folder. On macOS run `ls -a` to see it. In Windows Explorer, turn on *View → Show → Hidden items*.

## 3. Install it in Claude Code

Open Claude Code (any folder works) and run these two commands. Use the absolute path of the folder from [section 2](#2-get-the-plugin).

**macOS / Linux**

```
/plugin marketplace add ~/.majordomo/plugins/md-sdlc
/plugin install md-sdlc@majordomo-marketplace
```

**Windows**

```
/plugin marketplace add C:\Users\<you>\.majordomo\plugins\md-sdlc
/plugin install md-sdlc@majordomo-marketplace
```

- The first command registers the folder as a plugin source called **`majordomo-marketplace`**.
- The second command installs the plugin from it. When asked for a scope, choose **user** to have it in every project, or **project** / **local** to have it only in the current repo.

**Restart Claude Code** (or start a new session) so the commands load.

You can also do this from the menu: run `/plugin`, open **Marketplaces → Add marketplace**, enter the path, then install **md-sdlc** from **Discover**.

## 4. Configure the Majordomo server

The plugin needs three settings. Ask your Majordomo admin for the values.

| Setting | Required | Value |
|---|---|---|
| `BACKEND_API_URL` | yes, unless you pass the URL to `/md-sdlc:login` (see [section 5](#5-sign-in)) | The Majordomo server's API address. It ends in `/sdlc/v1`, for example `https://majordomo.example.com/sdlc/v1`. |
| `TENANT_ID` | yes for the hosted Majordomo server | The UUID of the tenant the repo is registered in. |
| `WORKSPACE_ID` | yes for the hosted Majordomo server | The UUID of the workspace the repo is registered in. |

Set both `TENANT_ID` and `WORKSPACE_ID`, or neither. The helper refuses one without the other, and refuses any value that isn't a UUID. A local development server needs neither.

Each setting is read from the first place that has it:

1. Environment variables.
2. **`./.env`** at the root of the working repo.
3. **`~/.majordomo/.env`** (Windows: `%USERPROFILE%\.majordomo\.env`). Set it once here to use it in every repo.
4. For `BACKEND_API_URL` only: the URL saved with your last login.

```ini
# ~/.majordomo/.env
BACKEND_API_URL=https://majordomo.example.com/sdlc/v1
TENANT_ID=00000000-0000-0000-0000-000000000000
WORKSPACE_ID=00000000-0000-0000-0000-000000000000
```

Use one `KEY=value` per line. `#` starts a comment.

If you put these in the repo's `./.env`, make sure `.env` is in `.gitignore`.

This is the same settings file the `majordomo-sdlc` CLI uses. If you've already set up the CLI, there's nothing more to do. The plugin doesn't need `REPO_TOKEN`: it uses your own `git` and `gh` credentials.

## 5. Sign in

Open Claude Code **at the root of the working repo** and run:

```
/md-sdlc:login
```

If you didn't set `BACKEND_API_URL`, pass the URL instead. It is saved with the login, so you only pass it once:

```
/md-sdlc:login https://majordomo.example.com/sdlc/v1
```

- **Browser sign-in (default).** A browser tab opens at GitHub. Approve the sign-in, then return to Claude Code. The command waits up to 5 minutes.
- **Device sign-in.** Use this over ssh, on a machine without a browser, or when the browser flow doesn't work:

  ```
  /md-sdlc:login --device
  ```

  Claude prints a URL and a code. Open the URL on any device, enter the code, then tell Claude you're done.

The hosted Majordomo server supports GitHub sign-in only.

Your session is stored in `~/.majordomo/credentials.json`. This is the same file the `majordomo-sdlc` CLI's `/login` writes, so signing in with either one signs you in for both. The token is never printed.

Other login commands:

| Command | What it does |
|---|---|
| `/md-sdlc:login whoami` | Shows who is signed in, to which server, and until when. |
| `/md-sdlc:login logout` | Signs out and deletes the stored session. |

The session belongs to its tenant. If you change `TENANT_ID`, sign in again.

## 6. Check the installation

1. Run `/plugin` and open **Installed**. **md-sdlc** is listed and enabled.
2. Type `/md-sdlc:`. The autocomplete lists the plugin's commands: `login`, `start`, `spec`, `plan`, `ut-cases`, `implement`, `unit-test`, `review`, `feedback`, `sync`, `status` and `raise-pr`. The underlying stage and review skills (`md-specification`, `md-planning`, …) are hidden from the menu on purpose. What each command does is in the [Developer manual, section 5.1](developer-manual.md#51-command-reference).
3. Run `/md-sdlc:login whoami`. It shows your user and the server URL.
4. In the working repo, check that `.majordomo/templates/` and `.majordomo/docs/` exist. The plugin creates them from its built-in templates at the start of each session, and never overwrites files that are already there.

## 7. Next steps

You're set up. The [Developer manual](developer-manual.md) covers the rest, for both the plugin and the CLI: creating the issue, starting an execution with `/md-sdlc:start`, running, reviewing and syncing each stage, and raising the PR. Start with [section 2.4, Using the plugin in Claude Code](developer-manual.md#24-using-the-plugin-in-claude-code).

## 8. Upgrade and uninstall

### Upgrade to a new zip

To move to the latest `md-sdlc.zip` from the release page, first remove the installed plugin:

```
/plugin uninstall md-sdlc@majordomo-marketplace
/plugin marketplace remove majordomo-marketplace
```

Delete the old folder, download and unzip the new zip into the same place ([section 2](#2-get-the-plugin)), then run the two commands from [section 3](#3-install-it-in-claude-code) again and restart Claude Code. Check the installed version in `/plugin` → **Installed**.

Your settings (`~/.majordomo/.env`), your login (`~/.majordomo/credentials.json`) and your executions (`.majordomo/execution/` in the repo) aren't affected.

### Uninstall

```
/plugin uninstall md-sdlc@majordomo-marketplace
/plugin marketplace remove majordomo-marketplace
```

Then delete the unzipped folder. To also sign out, run `/md-sdlc:login logout` before uninstalling.
