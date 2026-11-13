# Majordomo SDLC — Developer manual

## Contents

1. [Introduction](#1-introduction)
2. [Getting started](#2-getting-started)
   - [2.1 Before you start](#21-before-you-start)
   - [2.2 Install the CLI](#22-install-the-cli)
   - [2.3 Configure it](#23-configure-it)
   - [2.4 Sign in](#24-sign-in)
   - [2.5 Using the prompt](#25-using-the-prompt)
3. [SDLC lifecycle commands](#3-sdlc-lifecycle-commands)
   - [3.1 Step 1: Create the issue](#31-step-1-create-the-issue)
   - [3.2 Step 2: Start an execution](#32-step-2-start-an-execution)
   - [3.3 Step 3: Run a stage](#33-step-3-run-a-stage)
   - [3.4 Step 4: Review the stage](#34-step-4-review-the-stage)
   - [3.5 Step 5: Revise it (optional)](#35-step-5-revise-it-optional)
   - [3.6 Step 6: Sync it to the Majordomo server](#36-step-6-sync-it-to-the-majordomo-server)
   - [3.7 Step 7: Check the status](#37-step-7-check-the-status)
   - [3.8 Step 8: Raise the pull request](#38-step-8-raise-the-pull-request)
   - [3.9 Step 9: Approval and merge](#39-step-9-approval-and-merge)
   - [3.10 Out-of-sync stages](#310-out-of-sync-stages)
   - [3.11 Managing containers](#311-managing-containers)
   - [3.12 Running commands from scripts](#312-running-commands-from-scripts)
   - [3.13 Command reference](#313-command-reference)
   - [3.14 Settings reference](#314-settings-reference)
   - [3.15 Files and where they live](#315-files-and-where-they-live)
4. [FAQ](#4-faq)

---

## 1. Introduction

This guide is for developers who use the `majordomo-sdlc` CLI to take a GitHub issue from a written request to a reviewed, approved pull request.

For each issue, the CLI starts a Docker container on your machine in which a coding agent (Claude Code, Codex or Copilot) works on your repo. You take the issue through five stages, **spec → plan → ut-cases → implementation → unit-test**: the agent writes each stage, you review and revise it, and you sync it to the **Majordomo server**, where the team, the code owners and the merge gate see it. Then you raise the pull request, a code owner approves the execution on the Majordomo dashboard, and the PR can be merged.

The repo itself is set up by a Majordomo admin, as described in the [Administrator manual](administrator-manual.md). Some CLI output calls the Majordomo server by its service name, *north* (for example `north: all stages ready`).

## 2. Getting started

### 2.1 Before you start

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
| `copilot` | Create a fine-grained GitHub token with the **Copilot Requests** permission and set it as `COPILOT_GITHUB_TOKEN` (section 2.3). |

#### What your admin sets up

You don't set up the repo yourself: a Majordomo admin does it once per repo ([Administrator manual](administrator-manual.md)). If you see either of these, send the message to your admin:

- `/start` is refused with `409 repo_not_registered` or `policy_missing`.
- Every `/sync` ends with `queued (north unreachable)`.

### 2.2 Install the CLI

Pick one of the two ways. Both give you the `majordomo-sdlc` command.

#### Option A: standalone binary (no Node or Python needed)

The files are on the repo's Releases page, release **cli-dev**:

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

#### Option B: npm package (machines with Node ≥ 18)

```bash
gh release download cli-dev --repo majordomoinc/majordomo-sdlc-public --pattern 'md-sdlc-*.tgz'
npm install -g ./md-sdlc-0.1.0.tgz          # the version in the file name may differ
majordomo-sdlc --help
```

Upgrade by installing a newer `.tgz` the same way. `npm uninstall -g md-sdlc` removes it.

### 2.3 Configure it

#### Where settings come from

The CLI reads each setting from the first place that has it:

1. **Environment variables.**
2. **`./.env`** in the folder you run the CLI from, or the file named by `MAJORDOMO_ENV_FILE`.
3. **`~/.majordomo/.env`** (Windows: `%USERPROFILE%\.majordomo\.env`). The per-user default: set it once and it works from any folder.


#### The settings you need

```ini
# ~/.majordomo/.env

# The Majordomo server's address (ask your admin; it ends in /sdlc/v1).
BACKEND_API_URL=https://north.example.com/sdlc/v1

# The tenant and workspace your repo is registered in (ask your admin).
TENANT_ID=00000000-0000-0000-0000-000000000000
WORKSPACE_ID=00000000-0000-0000-0000-000000000000

# The repo token from section 2.1.
REPO_TOKEN=github_pat_...

# Only if north uses the Copilot agent for your repo.
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

All settings are in [section 3.14](#314-settings-reference).

**Keep these files out of git.** They hold tokens.

**Windows:** if you use environment variables instead of a file (`setx REPO_TOKEN "..."`), open a **new** terminal afterwards. Terminals that are already open don't see the change.

### 2.4 Sign in

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

### 2.5 Using the prompt

There is one prompt, `majordomo>`, for everything:

- **Commands start with `/`.** Press **Tab** to complete a command name. Up and Down arrows go through your history.
- **Stage commands name their execution:** `/plan <execution-id>`. The id is the **EXECUTION ID** column of `/issues`. Paste the full UUID; the CLI doesn't accept a prefix.
- **While a stage command runs, your terminal is connected to the container.** You see the agent's output live and can answer its questions. When the command finishes you're back at `majordomo>`.
  - **Ctrl-]** leaves early. The work carries on in the container.
  - **Ctrl-C** is ignored while a stage runs, so you can't kill the agent by accident.
- **At the prompt**, Ctrl-C or `/quit` asks whether to quit (`[y/N]`). Containers keep running after you quit.
- `/help` lists every command.

Tip: to avoid retyping the id, copy it from `/issues` once, or keep it in a shell variable and use the one-shot form ([section 3.12](#312-running-commands-from-scripts)).

---


## 3. SDLC lifecycle commands

### 3.1 Step 1: Create the issue

Every execution starts from an issue. The **spec** stage is generated from the issue's title and body, and every later stage builds on the spec, so the issue is the most important input you write.

#### Create a GitHub issue

In the browser: open the working repo → **Issues** → **New issue**.

Or with the GitHub CLI:

```bash
gh issue create --repo acme/payments \
  --title "Refunds: support partial refunds on card payments" \
  --body-file issue.md
```

Note the issue **number** (e.g. `#42`). That's what you pass to `/start`.

#### What to write

| Section | Example |
|---|---|
| **Goal**: what should be possible afterwards, and for whom | "Support agents can refund part of a card payment from the admin API." |
| **Context**: where this lives in the code, related features | "Refunds live in `payments/refunds/`. Full refunds already exist (`POST /refunds`)." |
| **Requirements**: one per line, testable | "Amount must be > 0 and ≤ the remaining refundable amount." |
| **Out of scope** | "No UI changes. No refunds for bank transfers." |
| **Acceptance criteria** | "A payment of 100 can be refunded 30 then 70. A third refund is rejected with 409." |
| **Constraints** | "Don't change the public response shape of `GET /payments/{id}`." |

The more precise the issue, the fewer questions the agent asks and the fewer revisions you need. If the issue changes after you've run the spec, see [section 3.10](#310-out-of-sync-stages).

#### Jira keys

`/start` also accepts a Jira key such as `PAY-123`. The container **can't fetch Jira content yet**: the agent only gets the key. For anything beyond a trivial task, use a GitHub issue, or be ready to answer the agent's questions and use `/feedback` on the spec.

### 3.2 Step 2: Start an execution

```
/start <repo-url> <issue> [target-branch] [checkout-folder]
```

| Argument | Required | Meaning |
|---|---|---|
| `<repo-url>` | yes | The working repo: `https://github.com/acme/payments`, `git@github.com:acme/payments.git`, a GitLab or Bitbucket URL. |
| `<issue>` | yes | A GitHub issue number (`42`, or `"#42"` in quotes because `#` starts a shell comment), or a Jira key (`PAY-123`). |
| `[target-branch]` | no | The branch to base the work on and merge into. Default: the one in the repo's policy (usually `main`). **Fixed when the execution is created.** |
| `[checkout-folder]` | no | Folder on your machine for the repo checkout. Default: `./<repo-name>-<issue>` under the folder you run the CLI from. It must be **empty** (the container clones into it) or **an existing clone of the same repo**. |

#### Example

```
majordomo> /start https://github.com/acme/payments 42
Starting container for https://github.com/acme/payments #42
Pulling majordomo/sdlc-claude:1.4
.........
Execution 6736023f-fc8c-40de-b0ae-72b199b3b38e (created), target branch main, working branch sdlc-6736023f-fc8c-40de-b0ae-72b199b3b38e
sdlc-6736023f-fc8c-40de-b0ae-72b199b3b38e ready -- e.g. /plan 6736023f-fc8c-40de-b0ae-72b199b3b38e
```

#### Variations

| You want to | Do |
|---|---|
| Work from a branch other than the policy's default | `/start <repo> 42 release/2.3`. This works only the **first** time. If the execution already exists with another target branch, `/start` refuses and tells you which one it uses. |
| Use a clone you already have | `/start <repo> 42 main ~/code/payments`. The folder must be a clone of the same repo. The container checks out `sdlc-<id>` in it, so commit or stash your own work there first. |
| Continue an execution someone else started, or that you started on another machine | `/start <repo> 42`. It returns the existing execution, and the container restores everything that was synced. Unsynced work from the other machine is not on the Majordomo server, so it doesn't come back. |
| Get a container back after `/remove` | `/start <repo> 42`, or just run any stage command on the id: the CLI recreates the container by itself. |

`/start` refuses to create a second container for an execution that already has one on this machine. Use a stage command on it instead (e.g. `/status <id>`): a stopped container is started for you.

If `/start` warns `not set in env/.env: REPO_TOKEN`, the container can't clone a private repo, read the issue, push or open the PR. Set the token, then `/remove` and `/start` again.

### 3.3 Step 3: Run a stage

Run the stages **in order**, starting with the spec:

```
majordomo> /spec 6736023f-fc8c-40de-b0ae-72b199b3b38e
sdlc-6736023f-... (acme/payments #42): /spec
Running spec ...
... the agent's output ...
spec v1 saved locally and its run recorded in north (files unsynced) -- /review spec, then /sync spec sends its files to north.
majordomo>
```

| Command | Stage | Artifact it writes |
|---|---|---|
| `/spec <id>` | `spec` | `spec.md` |
| `/plan <id>` | `plan` | `plan.md` |
| `/ut-cases <id>` | `ut-cases` | `unit-test-cases.md` |
| `/implement <id>` | `implementation` | code changes, committed on `sdlc-<id>`, plus `implementation-summary.md` |
| `/unit-test <id>` | `unit-test` | `test-report.md` (and `setup-report.md` if it had to set up a test environment) |

Artifacts go to `<checkout>/.majordomo/execution/<execution-id>/<stage>/`. Open them in your editor while you work. They are ordinary Markdown files. A stage can take several minutes.

#### When the agent asks a question

Sometimes the agent needs an answer before it can continue:

```
Which currency rounding rule should partial refunds use: banker's rounding or half-up?
[the agent is asking a question -- type your answer to continue spec, or /cancel to abandon it]
```

- **Type your answer** and press Enter. The stage carries on. The question and answer count as one run.
- **Not ready to answer?** Press **Ctrl-]**. You're back at `majordomo>`, and the stage keeps waiting in the container. Later, run `/reply <id>` to reconnect and answer.
- **Want to drop the stage?** `/cancel <id>`. Nothing is saved. Run the stage command again to start over.

While a stage waits for an answer, other stage commands on that execution are refused (`The spec stage is waiting for your answer -- reply to it first.`).

#### Regenerating a stage

Running a stage command again creates a **new** local version from scratch (`v2`, …). Earlier local versions are kept in the history, but only the **latest** can be reviewed and synced. If later stages were built on the old version, they become out of date; see [section 3.10](#310-out-of-sync-stages).

### 3.4 Step 4: Review the stage

```
/review <id> [stage]
```

The stage defaults to the last one you ran.

```
majordomo> /review 6736023f-fc8c-40de-b0ae-72b199b3b38e spec
Reviewing spec v1 ...
... the reviewer's output ...
Review score: 8.6. spec v1 saved locally and its run recorded in north (files unsynced) -- /sync spec sends its files to north.
```

`/review` reviews the artifact **as it is on disk now**, including any hand edits you made, writes the review to `review.md` next to the artifact, and shows the **score**.

Read `review.md`. If the reviewer found real problems, fix them (section 3.5) and review again. You can run `/review` as often as you like.

Some reviews need to ask you questions and can't run from the CLI. `/review` tells you when it skips one.

**A review is required.** `/sync` refuses a version that hasn't been reviewed. **A review must be current.** If you change the artifact after the review (by hand or with `/feedback`), run `/review` again before syncing. Otherwise the Majordomo server flags the stage as `review_stale` and the merge gate fails.

### 3.5 Step 5: Revise it (optional)

You have three ways to change a stage's output. Each one needs a fresh `/review` afterwards.

#### `/feedback`: tell the agent what to change

```
/feedback <id> [stage] <instructions>
```

```
majordomo> /feedback 6736023f-fc8c-40de-b0ae-72b199b3b38e plan use the existing RefundLedger table instead of adding a new one, and split step 3 into a migration and a backfill
Revising plan v1 with your feedback ...
plan v2 saved locally ... -- /review plan, then /sync plan sends its files to north.
```

- The agent revises the artifact. The result is a **new local version**.
- The stage defaults to the last one you ran. Name it if you mean another one: `/feedback <id> spec ...`.
- Write the instructions as you would to a colleague: specific and complete.
- For `implementation` and `unit-test`, feedback means "reject and redo with these changes". The code is changed and committed again.
- If the agent answers with a question instead of changing anything, answer it like any other stage question.

#### Run the stage again

`/plan <id>` again generates a new version from scratch. Use this when the result is far off, or when an upstream stage changed.

#### Edit the file yourself

Open the artifact in your editor and change it in place (section 3.3 says where it is). Don't rename or move it.

### 3.6 Step 6: Sync it to the Majordomo server

```
/sync <id> [stage]
```

```
majordomo> /sync 6736023f-fc8c-40de-b0ae-72b199b3b38e spec
spec: synced v1 as north v1 (store_ref 3f9c1a2)
north: plan - missing
north: ut-cases - missing
...
```

| Form | Sends |
|---|---|
| `/sync <id> <stage>` | the latest local version of that stage |
| `/sync <id>` | the latest local version of **every** stage that isn't synced yet, in stage order |

Stage names accepted: `spec`, `plan`, `ut-cases` (or `ut`), `implementation` (or `implement`), `unit-test` (or `unittest`, `unit-tests`).

#### What `/sync` checks

**Before sending anything**, it checks that each version to send:

- has been **reviewed** (`not_reviewed`: run `/review` first);
- has all its files readable and under the size limit (`missing_file`, `file_too_large`).

If any stage fails these checks, **nothing** is sent.

**The Majordomo server then checks:**

| Rule | Refusal | What to do |
|---|---|---|
| **Order**: every earlier stage must already have a synced version | `ordering` (names the missing stage) | sync the earlier stage first |
| **Inputs**: the stage must have been built on exactly the upstream versions the Majordomo server holds | `input_mismatch` (names the upstream stage) | sync that upstream stage, or regenerate this one on top of the synced upstream |
| **Review present** | `review_missing`, `review_conflict` | run `/review` again |
| **File integrity** | `hash_mismatch`, `file_too_large` | `/review` and sync again; trim a file over the size limit (1 MiB by default) |

The first refused stage stops the ones after it. The stages before it stay synced.

After the sync, the CLI prints the Majordomo server's view of every stage that isn't ready yet, with the reason:

```
north: plan v1 out_of_sync - upstream_newer (based_on spec 1a2b3c4d5e6f != latest v2 9f8e7d6c5b4a)
north: ut-cases missing
```

`north: all stages ready` means every stage is synced and consistent.

#### When the Majordomo server is unreachable

The events are queued on disk and the stage shows `queued (north unreachable); not synced yet`. The next `/sync` (or any run) sends the queued events first. Nothing is lost, and resending never creates duplicates.

### 3.7 Step 7: Check the status

```
/status <id>
```

Shows what **the Majordomo server** holds, next to your local versions. It's read-only: it never changes anything.

```
execution status : running
approval status  : not_approved
approvals        : 0 of 1 needed

stage           north (latest version)             local
spec            v2 score 8.6 at 2026-11-04T10:12Z  v3
plan            v1 score 8.1 at 2026-11-04T10:40Z  v1
ut-cases        -                                  v1  UNSYNCED (files not in north yet)
implementation  -
unit-test       -
```

| Line | Meaning |
|---|---|
| `execution status` | `running`, `stopped`, `failed` or `deleted` |
| `approval status` / `approvals` | approved or not, and how many approvers of the quorum |
| stage table | the Majordomo server's latest version and score per stage; your local latest version, marked `UNSYNCED` if the Majordomo server doesn't have it yet |

In the example, `spec` local is `v3` but the Majordomo server has `v2`: you regenerated or revised the spec and haven't synced it. Note that local and server version numbers are counted separately and don't need to match.

### 3.8 Step 8: Raise the pull request

When all five stages are synced:

```
majordomo> /raise-pr 6736023f-fc8c-40de-b0ae-72b199b3b38e
Pushed sdlc-6736023f-... (1 commit).
Opened PR: https://github.com/acme/payments/pull/118
```

`/raise-pr` pushes `sdlc-<id>` and opens a PR into the target branch. It **never force-pushes**, and if a PR is already open it reports that one instead of opening another.

Notes:

- Only **commits** are pushed. Uncommitted changes are reported (`note: there are uncommitted changes ...`) and left out.
- `/raise-pr` doesn't check review or sync state. You *can* raise the PR early, but the merge gate stays red until every stage is synced and approved.
- After later changes (e.g. a re-run `/implement`), run `/raise-pr` again to push the new commits. The PR updates; no new PR is opened.

### 3.9 Step 9: Approval and merge

The PR can only be merged once a **Majordomo user** (a code owner of the repo) approves the execution on the Majordomo dashboard. There is no approve command in the CLI: ask your code owner to approve it.

When it's approved and every stage is synced, the PR's `sdlc-gate` check turns green and you merge the PR on GitHub as usual. `/status <id>` shows whether it's approved. If the gate stays red, a stage is probably out of sync: see [section 3.10](#310-out-of-sync-stages).

### 3.10 Out-of-sync stages

Each stage is built on the stages before it. If one of those changes after a later stage was built, the later stage is **out of sync**. You can still sync, but the PR can't be merged until it's fixed. `/status <id>` and the `sdlc-gate` check name the stage and the cause.

**The fix is almost always the same:** re-run the flagged stage, `/review` it and `/sync` it, then do the same for each later stage, in order. If the code changed, finish with `/raise-pr`.

| Cause | What it means | What to do |
|---|---|---|
| `upstream_newer` | An earlier stage was synced again after this one was built. | Re-run, review and sync this stage and every later one. |
| `upstream_out_of_sync` | An earlier stage is itself out of sync. | Fix the earliest flagged stage first. |
| `review_stale` / `review_not_latest` | The artifact changed after it was reviewed. | `/review`, then `/sync` the stage. |
| `target_moved` | The target branch (e.g. `main`) moved after the plan was built. | Merge or rebase the target into `sdlc-<id>`, then re-run `plan` and every later stage. |
| `code_changed` / `branch_moved` | The PR has commits the synced implementation doesn't (a hand fix, a merge, GitHub's *Update branch*). | Commit your change on `sdlc-<id>` (or use `/feedback <id> implementation ...`), review and sync the implementation, then re-run `unit-test` and `/raise-pr`. |
| `incomplete` | A stage was never synced. | Run, review and sync it. |

Tips:

- **The issue changed:** update the spec with `/feedback <id> spec <the change>`, then re-run the later stages.
- **Don't re-review a stage that's already synced** unless you also mean to re-run the stages after it: syncing it again makes them `upstream_newer`.
- **`/sync` says `input_mismatch`:** sync the upstream stage first. If it still fails, re-run this stage.

### 3.11 Managing containers

#### List your executions

```
majordomo> /issues
REPO           ISSUE  EXECUTION ID                          CONTAINER                                  STATE    STARTED
acme/payments  42     6736023f-fc8c-40de-b0ae-72b199b3b38e  sdlc-6736023f-fc8c-40de-b0ae-72b199b3b38e  running  2026-11-04T09:58:12Z
acme/payments  37     0b1e...                               sdlc-0b1e...                               exited   2026-11-01T14:03:40Z
```

`/issues` lists the executions **started on this machine**, with their container state.

| STATE | Meaning |
|---|---|
| `running` | ready |
| `exited` / `created` | stopped. Any stage command starts it again. |
| `gone` | the container was removed. Any stage command (or `/start`) recreates it; `/remove <id>` drops the row. |
| `-` | Docker isn't reachable |

#### Stop, start, remove

| Command | Effect | Your work |
|---|---|---|
| `/stop <id>` | stops the container (frees CPU and memory) | kept. Any stage command on the id starts it again. |
| `/remove <id>` | deletes the container and drops it from `/issues` | The **execution on the Majordomo server is untouched**. Your checkout folder stays on disk, with the local versions. `/start <repo> <issue>` brings a container back. |

#### Where your work lives

- **Synced versions** are on the Majordomo server. Any container for the execution, on any machine, restores them on start.
- **Unsynced local versions** live only in your checkout folder (`.majordomo/execution/<id>/`). They survive `/stop` and `/remove` because the folder is on your machine, but they aren't on any other machine. **Sync what you want to keep.**
- **Code commits** are on `sdlc-<id>` in your checkout until `/raise-pr` pushes them.

#### Leaving and coming back

`/quit` (or closing the terminal) leaves containers running. Next time, run `majordomo-sdlc`, `/issues` to find the id, and carry on with any stage command. If you left a stage waiting on a question, `/reply <id>` reconnects you to it.

#### Session settings for one container

These change settings for one container until it is recreated:

| Command | Effect |
|---|---|
| `/set-model <id> <model>` | use another model for the following runs |
| `/set-github-token <id> <token>` | replace the repo token |

### 3.12 Running commands from scripts

Every command also runs straight from your shell, without the prompt. Drop the leading `/`:

```bash
majordomo-sdlc start https://github.com/acme/payments 42
majordomo-sdlc issues
majordomo-sdlc plan   6736023f-fc8c-40de-b0ae-72b199b3b38e
majordomo-sdlc review 6736023f-fc8c-40de-b0ae-72b199b3b38e plan
majordomo-sdlc sync   6736023f-fc8c-40de-b0ae-72b199b3b38e plan
majordomo-sdlc help
```

The arguments are the same as at the prompt. Drop the slash because Git Bash on Windows turns a leading `/word` into a Windows path.

A handy pattern:

```bash
ID=6736023f-fc8c-40de-b0ae-72b199b3b38e
for s in plan ut-cases; do
  majordomo-sdlc "$s" "$ID" && majordomo-sdlc review "$ID" "$s" && majordomo-sdlc sync "$ID" "$s" || break
done
```

**Exit status:**

| Code | Meaning |
|---|---|
| `0` | success |
| `1` | error, or unknown command |
| `2` | a stage command with the wrong arguments |
| other | the status of the command inside the container |

`start`, `stop`, `remove` and `login` print a usage line but exit `0` on wrong arguments.

Stage commands still connect to the container's terminal, so run them from a real terminal: the agent may ask a question. For CI or tests, set `MAJORDOMO_TOKEN` to a pre-issued token instead of running `/login`.

---


### 3.13 Command reference

`<id>` is the execution id (the EXECUTION ID column of `/issues`). `[stage]` defaults to the last stage run.

#### Lifecycle commands

| Command | Does |
|---|---|
| `/login [--device] [--provider github\|bitbucket]` | sign in with your git account |
| `/logout` | remove the stored session |
| `/start <repo-url> <issue> [target-branch] [checkout-folder]` | get or create the execution, start its container |
| `/issues` | list executions started on this machine, with container state |
| `/stop <id>` | stop the container |
| `/remove <id>` | delete the container, drop it from `/issues` |
| `/help` | list commands |
| `/quit` | leave the CLI (asks first; containers keep running) |

#### Stage commands

| Command | Does |
|---|---|
| `/spec <id>` | generate the spec |
| `/plan <id>` | generate the plan |
| `/ut-cases <id>` | generate the unit-test cases |
| `/implement <id>` | implement and commit on `sdlc-<id>` |
| `/unit-test <id>` | run the tests and write the report |
| `/feedback <id> [stage] <instructions>` | the agent revises the stage as told: a new local version |
| `/review <id> [stage]` | review and score the latest local version |
| `/sync <id> [stage]` | send the latest reviewed version(s) to the Majordomo server |
| `/raise-pr <id>` | push `sdlc-<id>` and open (or find) its PR |
| `/status <id>` | show the Majordomo server's view and your local versions |
| `/reply <id>` | reconnect to a stage waiting on your answer |
| `/cancel <id>` | abandon the stage waiting on your answer; nothing is saved |

### 3.14 Settings reference

Read from the environment, then `./.env` (or `MAJORDOMO_ENV_FILE`), then `~/.majordomo/.env`.

| Setting | Default | Purpose |
|---|---|---|
| `BACKEND_API_URL` | — (required) | the Majordomo server's address, ending in `/sdlc/v1` |
| `TENANT_ID`, `WORKSPACE_ID` | — | tenant and workspace of the repo |
| `REPO_TOKEN` | — | the repo token used by the container |
| `COPILOT_GITHUB_TOKEN` | — | Copilot agent only |
| `AGENT_MODEL` | agent default | model for every stage run |
| `MAJORDOMO_ENV_FILE` | `./.env` | another settings file |
| `MAJORDOMO_TOKEN` | — | use this Majordomo server token instead of `/login` (CI, tests) |
| `MAJORDOMO_NO_BROWSER` | — | always use device sign-in |
| `CLAUDE_CONFIG_DIR` | `~/.claude` | where the Claude Code login is |
| `CODEX_HOME` | `~/.codex` | where the Codex login is |

### 3.15 Files and where they live

#### On your machine

| Path | What |
|---|---|
| `~/.majordomo/.env` | your settings |
| `~/.majordomo/credentials.json` | the Majordomo server session token from `/login` |
| `~/.majordomo/issues.json` | executions started on this machine (`/issues`) |
| `~/.majordomo/outbox.jsonl` | lifecycle events waiting for the Majordomo server |
| `<checkout>/` | the repo, on branch `sdlc-<id>` (default `./<repo-name>-<issue>`) |

#### In the checkout

```
<checkout>/.majordomo/
├── execution/<execution-id>/
│   ├── issue.json                         the issue, cached on first use
│   ├── state.json                         local versions and their sync state
│   ├── spec/            spec.md                  review.md  rubric.json
│   ├── plan/            plan.md                  review.md  rubric.json
│   ├── ut-cases/        unit-test-cases.md       review.md  rubric.json
│   ├── implementation/  implementation-summary.md review.md rubric.json
│   ├── unit-test/       test-report.md           review.md  rubric.json
│   └── setup/           setup-report.md          (if the tests needed setup)
├── templates/                             templates the skills use
└── outbox/                                container events waiting for north
```

Edit artifacts in place; don't rename or move them. Don't edit `state.json`.

## 4. FAQ

### Which coding agent will my repo use, and which login do I need?

Your admin picks the agent for each repo. You don't have to know it in advance: `/start` says which credential is missing before it pulls anything. The logins for each agent are in [section 2.1](#21-before-you-start).

### Can anyone else see my work before I sync it?

No. Running, reviewing and revising a stage only change your checkout. Only `/sync` sends a stage's files to the Majordomo server, where the team, the dashboard and the merge gate can see them.

### Do I lose my work if I remove a container?

No for synced work, and no for your checkout. `/remove` deletes only the container: the execution on the Majordomo server and your checkout folder, with its local versions, stay. `/start <repo> <issue>` brings a container back and restores every synced version. See [section 3.11](#311-managing-containers).

### Can I continue an execution on another machine?

Yes. Run `/start <repo> <issue>` there: it returns the same execution and restores everything that was synced. Work you didn't sync on the first machine isn't on the Majordomo server, so it doesn't come back.

### Can several people work on the same issue?

Yes. There is only **one execution per repo and issue**, so everyone works on the same one. When a teammate runs `/start <repo> <issue>` on their machine, they get the existing execution and its working branch `sdlc-<id>`, and their container restores every stage version that has been synced so far.

What each person sees:

- **Synced work is shared. Local work isn't.** Each person has their own container and checkout. A stage you've run or reviewed but not synced is only on your machine. Once you `/sync` it, everyone gets it the next time their container starts.
- **The latest sync wins.** If two people sync the same stage, both versions are kept, and the one synced last becomes the stage's latest version. The dashboard, `/status` and the merge gate use that one.
- **Stages have to fit together.** If a teammate syncs a new spec after you built the plan, your `/sync plan` is refused with `input_mismatch`, because your plan was built on the old spec. Re-run the plan so it's built on their spec, then review and sync it. If your plan was already synced, it becomes out of sync (`upstream_newer`) instead ([section 3.10](#310-out-of-sync-stages)).
- **Code goes through one branch.** Everyone pushes to the same `sdlc-<id>` branch and the same PR. `/raise-pr` never force-pushes, so if a teammate pushed first, your push is refused: pull their commits into your checkout, then push again.
- **Every run and sync is recorded under the person who did it**, so the dashboard shows who did what.

To avoid redoing each other's work, agree on who owns which stage, sync as soon as a stage is reviewed, and run `/status <id>` before you start a stage to see what is already on the Majordomo server.

### Why is the merge gate still red when every stage is synced?

Usually the execution isn't approved yet, a stage is out of sync, a review is stale, or the target branch moved. `/status <id>` shows what is missing and why. See [section 3.9](#39-step-9-approval-and-merge) and [section 3.10](#310-out-of-sync-stages).
