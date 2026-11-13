# Majordomo SDLC — Developer manual

## Contents

1. [Introduction](#1-introduction)
   - [1.1 Two ways to run it](#11-two-ways-to-run-it)
   - [1.2 Which one to use](#12-which-one-to-use)
2. [Getting started](#2-getting-started)
   - [2.1 Install](#21-install)
   - [2.2 How commands are written in this manual](#22-how-commands-are-written-in-this-manual)
   - [2.3 Using the CLI prompt](#23-using-the-cli-prompt)
   - [2.4 Using the plugin in Claude Code](#24-using-the-plugin-in-claude-code)
3. [The SDLC lifecycle](#3-the-sdlc-lifecycle)
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
4. [Working with the CLI and the plugin](#4-working-with-the-cli-and-the-plugin)
   - [4.1 Managing containers (CLI)](#41-managing-containers-cli)
   - [4.2 Running commands from scripts (CLI)](#42-running-commands-from-scripts-cli)
   - [4.3 Working in your checkout (plugin)](#43-working-in-your-checkout-plugin)
5. [Reference](#5-reference)
   - [5.1 Command reference](#51-command-reference)
   - [5.2 Settings reference](#52-settings-reference)
   - [5.3 Files and where they live](#53-files-and-where-they-live)
6. [FAQ](#6-faq)

---

## 1. Introduction

This guide is for developers who use Majordomo SDLC to take an issue from a written request to a reviewed, approved pull request.

You take each issue through five stages, **spec → plan → ut-cases → implementation → unit-test**: a coding agent writes each stage, you review and revise it, and you sync it to the **Majordomo server**, where the team, the code owners and the merge gate see it. Then you raise the pull request, a code owner approves the execution on the Majordomo dashboard, and the PR can be merged.

The repo itself is set up by a Majordomo admin, as described in the [Administrator manual](administrator-manual.md).

### 1.1 Two ways to run it

You can run Majordomo SDLC with the **Majordomo SDLC CLI** or the **Majordomo SDLC Claude plugin**. The lifecycle is the same in both, and so are most command names. What differs is where the agent runs.

| | **Majordomo SDLC CLI** (`majordomo-sdlc`) | **Majordomo SDLC Claude plugin** (`md-sdlc`) |
|---|---|---|
| Where the agent runs | In a Docker container on your machine, one per issue | In your own Claude Code session |
| Coding agent | Claude Code, Codex or Copilot, as your admin set for the repo | Claude Code, with the model your session uses |
| Where the repo is | A checkout the container clones (or one you point it at) | The clone you already have open |
| You need | Docker, the CLI, the agent's login | Claude Code, Git, Python 3, the GitHub CLI `gh` |
| How you type commands | At the `majordomo>` prompt: `/plan <id>` | In Claude Code: `/md-sdlc:plan` |
| Execution id | Always given | Left out when you're on the execution's branch |
| Pushes and the PR use | The repo token you give the container | Your own git credentials and `gh` login |

The CLI and the plugin work on the same executions on the same Majordomo server. There is one execution per repo and issue, whichever one started it.

### 1.2 Which one to use

- **The Claude plugin** if you already work in Claude Code and want the agent in your own editor session, on your own clone. Nothing runs in the background.
- **The CLI** if your repo uses Codex or Copilot, if you want each issue isolated in its own container, or if you want to drive the stages from scripts ([section 4.2](#42-running-commands-from-scripts-cli)).

Your admin may tell you which one your team uses.

## 2. Getting started

### 2.1 Install

Install, configure and sign in first, then come back here:

- **CLI:** the [Majordomo SDLC CLI installation guide](md-sdlc-container-mode-installation.md).
- **Claude plugin:** the [Majordomo SDLC Claude plugin installation guide](md-sdlc-plugin-inststallation.md).

Both use the same settings file (`~/.majordomo/.env`) and the same sign-in (`~/.majordomo/credentials.json`), so if you install both, signing in once is enough.

### 2.2 How commands are written in this manual

Each step in [section 3](#3-the-sdlc-lifecycle) shows the command for both the CLI and the plugin:

**CLI**
```
/plan <id>
```

**Plugin**
```
/md-sdlc:plan
```

Notes marked **CLI only** or **Plugin only** apply only to that one. Everything else applies to both.

The commands map one to one:

| Step | CLI | Plugin |
|---|---|---|
| Sign in / out | `/login`, `/logout` | `/md-sdlc:login`, `/md-sdlc:login logout`, `/md-sdlc:login whoami` |
| Start | `/start <repo-url> <issue>` | `/md-sdlc:start [repo-url] <issue>` |
| Run a stage | `/spec <id>`, `/plan <id>`, `/ut-cases <id>`, `/implement <id>`, `/unit-test <id>` | `/md-sdlc:spec`, `/md-sdlc:plan`, `/md-sdlc:ut-cases`, `/md-sdlc:implement`, `/md-sdlc:unit-test` |
| Review | `/review <id> [stage]` | `/md-sdlc:review [stage]` |
| Revise | `/feedback <id> [stage] <instructions>` | `/md-sdlc:feedback [stage] <instructions>` |
| Sync | `/sync <id> [stage]` | `/md-sdlc:sync [stage]` |
| Status | `/status <id>` | `/md-sdlc:status` |
| Raise the PR | `/raise-pr <id>` | `/md-sdlc:raise-pr` |

The CLI also has commands for its containers (`/issues`, `/stop`, `/remove`, `/reply`, `/cancel`). The full list is in [section 5.1](#51-command-reference).

### 2.3 Using the CLI prompt

**CLI only.** There is one prompt, `majordomo>`, for everything:

- **Commands start with `/`.** Press **Tab** to complete a command name. Up and Down arrows go through your history.
- **Stage commands name their execution:** `/plan <execution-id>`. The id is the **EXECUTION ID** column of `/issues`. Paste the full UUID; the CLI doesn't accept a prefix.
- **While a stage command runs, your terminal is connected to the container.** You see the agent's output live and can answer its questions. When the command finishes you're back at `majordomo>`.
  - **Ctrl-]** leaves early. The work carries on in the container.
  - **Ctrl-C** is ignored while a stage runs, so you can't kill the agent by accident.
- **At the prompt**, Ctrl-C or `/quit` asks whether to quit (`[y/N]`). Containers keep running after you quit.
- `/help` lists every command.

Tip: to avoid retyping the id, copy it from `/issues` once, or keep it in a shell variable and use the one-shot form ([section 4.2](#42-running-commands-from-scripts-cli)).

Some CLI output calls the Majordomo server by its service name, *north* (for example `north: all stages ready`).

### 2.4 Using the plugin in Claude Code

**Plugin only.** Run Claude Code at the root of your clone of the working repo.

- **Commands start with `/md-sdlc:`.** Type `/md-sdlc:` to see them in the autocomplete.
- **You can leave out the execution id** when you're on the execution's branch, `sdlc-<id>`. `/md-sdlc:start` checks that branch out for you. On any other branch, pass the id as the first argument: `/md-sdlc:plan <id>`.
- **The agent works in your session.** You see its work as it goes, and it asks its questions in the chat. Answer them there.
- **At the start of each session** the plugin creates `.majordomo/templates/` and `.majordomo/docs/` in the repo, from its built-in templates. It never overwrites files that are already there, so you can adapt the templates.
- **Commands only run when you type them.** Claude doesn't start a stage, a sync or a PR by itself.

---

## 3. The SDLC lifecycle

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

Note the issue **number** (e.g. `#42`). That's what you pass to start the execution.

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

The CLI and the plugin both accept a Jira key such as `PAY-123`, but neither fetches Jira content.

- **CLI only:** the agent only gets the key. For anything beyond a trivial task, use a GitHub issue, or be ready to answer the agent's questions and use `/feedback` on the spec.
- **Plugin only:** Claude asks you to paste the issue's title and description, and uses them like a GitHub issue's.

### 3.2 Step 2: Start an execution

**CLI**
```
/start <repo-url> <issue> [target-branch] [checkout-folder]
```

**Plugin** (in your clone of the repo)
```
/md-sdlc:start [repo-url] <issue> [target-branch]
```

| Argument | Required | Meaning |
|---|---|---|
| `<repo-url>` | CLI: yes. Plugin: no, it defaults to the clone's `origin` remote. | The working repo: `https://github.com/acme/payments`, `git@github.com:acme/payments.git`, a GitLab or Bitbucket URL. It must be the repo **as it is registered** on the Majordomo server. |
| `<issue>` | yes | A GitHub issue number (`42`; in the CLI, `"#42"` in quotes because `#` starts a shell comment), or a Jira key (`PAY-123`). |
| `[target-branch]` | no | The branch to base the work on and merge into. Default: the one in the repo's policy (usually `main`). **Fixed when the execution is created.** |
| `[checkout-folder]` | no, **CLI only** | Folder on your machine for the repo checkout. Default: `./<repo-name>-<issue>` under the folder you run the CLI from. It must be **empty** (the container clones into it) or **an existing clone of the same repo**. |

Starting gets the execution for that repo and issue from the Majordomo server, or creates it, and puts you on its working branch, **`sdlc-<id>`**. Every stage version that has been synced so far is restored.

#### Example

**CLI**
```
majordomo> /start https://github.com/acme/payments 42
Starting container for https://github.com/acme/payments #42
Pulling majordomo/sdlc-claude:1.4
.........
Execution 6736023f-fc8c-40de-b0ae-72b199b3b38e (created), target branch main, working branch sdlc-6736023f-fc8c-40de-b0ae-72b199b3b38e
sdlc-6736023f-fc8c-40de-b0ae-72b199b3b38e ready -- e.g. /plan 6736023f-fc8c-40de-b0ae-72b199b3b38e
```

**Plugin**
```
/md-sdlc:start 42
```
```
execution 6736023f-fc8c-40de-b0ae-72b199b3b38e (created) for 42, target branch main
...
Ready: execution id 6736023f-fc8c-40de-b0ae-72b199b3b38e, branch sdlc-6736023f-fc8c-40de-b0ae-72b199b3b38e. Next: /md-sdlc:spec 6736023f-fc8c-40de-b0ae-72b199b3b38e
```

**Plugin only:**
- The branch is checked out in your clone: the existing local branch, `origin/sdlc-<id>` if someone pushed it, or a new branch from `origin/<target-branch>`. It never resets or force-switches, so commit or stash your own work first.
- For a GitHub issue number, the issue is read with `gh`. If that fails you get `warning: couldn't fetch issue #42 with gh; continuing with the reference only.`, and the spec has only the issue number to go on. Check `gh auth status` and start again.

#### Variations

| You want to | Do |
|---|---|
| Work from a branch other than the policy's default | Give it as the target branch (`/start <repo> 42 release/2.3`, `/md-sdlc:start 42 release/2.3`). This works only the **first** time. If the execution already exists with another target branch, you're told which one it uses. |
| Continue an execution someone else started, or that you started on another machine, or with the other tool (CLI or plugin) | Start it again with the same repo and issue. You get the existing execution, and everything that was synced is restored. Unsynced work from elsewhere is not on the Majordomo server, so it doesn't come back. |
| **CLI only:** use a clone you already have | `/start <repo> 42 main ~/code/payments`. The folder must be a clone of the same repo. The container checks out `sdlc-<id>` in it, so commit or stash your own work there first. |
| **CLI only:** get a container back after `/remove` | `/start <repo> 42`, or just run any stage command on the id: the CLI recreates the container by itself. |
| **Plugin only:** go back to an execution you started in this clone | `git checkout sdlc-<id>` and carry on, or start it again. |

**CLI only:**
- `/start` refuses to create a second container for an execution that already has one on this machine. Use a stage command on it instead (e.g. `/status <id>`): a stopped container is started for you.
- If `/start` warns `not set in env/.env: REPO_TOKEN`, the container can't clone a private repo, read the issue, push or open the PR. Set the token, then `/remove` and `/start` again.

#### If the repo isn't found

If starting fails with `repo_not_registered`, `workspace_mismatch` or `unknown_tenant`, either `TENANT_ID` and `WORKSPACE_ID` don't match the repo, or the repo URL isn't exactly the one registered on the Majordomo server (case, `https` vs ssh, `.git`). Check them with your admin.

### 3.3 Step 3: Run a stage

Run the stages **in order**, starting with the spec.

**CLI**
```
majordomo> /spec 6736023f-fc8c-40de-b0ae-72b199b3b38e
sdlc-6736023f-... (acme/payments #42): /spec
Running spec ...
... the agent's output ...
spec v1 saved locally and its run recorded in north (files unsynced) -- /review spec, then /sync spec sends its files to north.
majordomo>
```

**Plugin**
```
/md-sdlc:spec
```
```
spec: run started ...
... the agent's work ...
spec v1 saved locally; its run is recorded in Majordomo (files unsynced). Next: /md-sdlc:review 6736023f-... spec
```

| Stage | CLI | Plugin | Artifact it writes |
|---|---|---|---|
| `spec` | `/spec <id>` | `/md-sdlc:spec` | `spec.md` |
| `plan` | `/plan <id>` | `/md-sdlc:plan` | `plan.md` |
| `ut-cases` | `/ut-cases <id>` | `/md-sdlc:ut-cases` | `unit-test-cases.md` |
| `implementation` | `/implement <id>` | `/md-sdlc:implement` | code changes, committed on `sdlc-<id>`, plus `implementation-summary.md` |
| `unit-test` | `/unit-test <id>` | `/md-sdlc:unit-test` | `test-report.md` (and `setup-report.md` if it had to set up a test environment) |

Artifacts go to `<checkout>/.majordomo/execution/<execution-id>/<stage>/`. Open them in your editor while you work. They are ordinary Markdown files. A stage can take several minutes.

**Plugin only:**
- **Run stages on the execution's branch.** On another branch, stage commands warn you (`warning: you are on main, not sdlc-<id> ...`), and `/md-sdlc:implement` refuses.
- **The implementation commit includes every change in your working tree**, except `.majordomo/` and `.env` files. Commit or stash unrelated work before `/md-sdlc:implement`. If you haven't set `git config user.email`, the commit is made as `Majordomo <majordomo@localhost>`.
- `/md-sdlc:implement` also writes compiler, lint, test and coverage reports to `.majordomo/execution/<id>/verification-reports/`. The implementation review uses them.

#### When the agent asks a question

Sometimes the agent needs an answer before it can continue.

**CLI only:**

```
Which currency rounding rule should partial refunds use: banker's rounding or half-up?
[the agent is asking a question -- type your answer to continue spec, or /cancel to abandon it]
```

- **Type your answer** and press Enter. The stage carries on. The question and answer count as one run.
- **Not ready to answer?** Press **Ctrl-]**. You're back at `majordomo>`, and the stage keeps waiting in the container. Later, run `/reply <id>` to reconnect and answer.
- **Want to drop the stage?** `/cancel <id>`. Nothing is saved. Run the stage command again to start over.

While a stage waits for an answer, other stage commands on that execution are refused (`The spec stage is waiting for your answer -- reply to it first.`).

**Plugin only:** Claude asks in the chat. Answer it there and the stage carries on. To drop the stage, press **Esc** and don't continue it: no version is saved. Run the stage command again to start over.

#### Regenerating a stage

Running a stage command again creates a **new** local version (`v2`, …). Only the **latest** version can be reviewed and synced. If later stages were built on the old version, they become out of date; see [section 3.10](#310-out-of-sync-stages).

- **CLI only:** the stage is generated from scratch. Earlier local versions are kept in the history.
- **Plugin only:** the agent starts from the existing file and revises it. Only the latest version's files are kept on disk.

### 3.4 Step 4: Review the stage

**CLI**
```
/review <id> [stage]
```

**Plugin**
```
/md-sdlc:review [stage]
```

The stage defaults to the last one you ran (in the plugin: the last stage, in order, that has an artifact).

```
majordomo> /review 6736023f-fc8c-40de-b0ae-72b199b3b38e spec
Reviewing spec v1 ...
... the reviewer's output ...
Review score: 4.3. spec v1 saved locally and its run recorded in north (files unsynced) -- /sync spec sends its files to north.
```

The review scores the artifact **as it is on disk now**, including any hand edits you made, writes the review to `review.md` next to the artifact, and shows the **score**. The plugin also lists the score for each criterion.

Read `review.md`. If the reviewer found real problems, fix them (section 3.5) and review again. You can review as often as you like. Each criterion is scored from 1 to 5, and the score is their weighted average, so it is also between 1 and 5. There's no minimum score: the score is information for you and your code owner, and doesn't block the sync.

Some reviews can't produce a score, for example when an input they need is missing. The review then says why it was skipped (in the plugin: **NOT AVAILABLE** with the reason). Without a score the stage can't be synced, so fix the cause and review again.

**A review is required.** Sync refuses a version that hasn't been reviewed. **A review must be current.** If you change the artifact after the review (by hand or with feedback), review it again before syncing. Otherwise the Majordomo server flags the stage as `review_stale` and the merge gate fails.

### 3.5 Step 5: Revise it (optional)

You have three ways to change a stage's output. Each one needs a fresh review afterwards.

#### Feedback: tell the agent what to change

**CLI**
```
/feedback <id> [stage] <instructions>
```

**Plugin**
```
/md-sdlc:feedback [stage] <instructions>
```

```
majordomo> /feedback 6736023f-fc8c-40de-b0ae-72b199b3b38e plan use the existing RefundLedger table instead of adding a new one, and split step 3 into a migration and a backfill
Revising plan v1 with your feedback ...
plan v2 saved locally ... -- /review plan, then /sync plan sends its files to north.
```

In the plugin, the same request is `/md-sdlc:feedback plan use the existing RefundLedger table ...`.

- The agent revises the artifact. The result is a **new local version**.
- The stage defaults to the last one you ran. Name it if you mean another one, for example `spec`.
- Write the instructions as you would to a colleague: specific and complete.
- For `implementation` and `unit-test`, feedback means "reject and redo with these changes". The code is changed and committed again.
- If the agent answers with a question instead of changing anything, answer it like any other stage question.
- Feedback can only revise a stage that has an artifact. Run the stage first.

#### Run the stage again

Running the stage command again also gives a new version ([section 3.3](#regenerating-a-stage)). Use this when the result is far off, or when an upstream stage changed.

#### Edit the file yourself

Open the artifact in your editor and change it in place (section 3.3 says where it is). Don't rename or move it.

### 3.6 Step 6: Sync it to the Majordomo server

**CLI**
```
/sync <id> [stage]
```

**Plugin**
```
/md-sdlc:sync [stage]
```

```
majordomo> /sync 6736023f-fc8c-40de-b0ae-72b199b3b38e spec
spec: synced v1 as north v1 (store_ref 3f9c1a2)
north: plan - missing
north: ut-cases - missing
...
```

The plugin prints the same lines with `Majordomo` in place of `north`, for example `spec: synced v1 as Majordomo v1 (store_ref 3f9c1a2)`.

| Form | Sends |
|---|---|
| with a stage | the latest local version of that stage |
| without a stage | the latest local version of **every** stage that isn't synced yet, in stage order |

Stage names accepted: `spec`, `plan`, `ut-cases` (or `ut`), `implementation` (or `implement`), `unit-test` (or `unittest`, `unit-tests`).

**Plugin only:** syncing `implementation` first pushes its commit to `sdlc-<id>` on origin, never forced (`implementation: pushed <sha> to sdlc-<id>`). If someone else pushed to that branch, the sync stops with `push_failed`: bring their commits into your branch, then run `/md-sdlc:implement`, review and sync again.

#### What sync checks

**Before sending anything**, it checks that each version to send:

- has been **reviewed** (`not_reviewed`: review it first);
- has all its files readable and under the size limit (`missing_file`, `file_too_large`).

If any stage fails these checks, **nothing** is sent.

**The Majordomo server then checks:**

| Rule | Refusal | What to do |
|---|---|---|
| **Order**: every earlier stage must already have a synced version | `ordering` (names the missing stage) | sync the earlier stage first |
| **Inputs**: the stage must have been built on exactly the upstream versions the Majordomo server holds | `input_mismatch` (names the upstream stage) | sync that upstream stage, or regenerate this one on top of the synced upstream |
| **Review present** | `review_missing`, `review_conflict` | review it again |
| **File integrity** | `hash_mismatch`, `file_too_large` | review and sync again; trim a file over the size limit (1 MiB by default) |

The first refused stage stops the ones after it. The stages before it stay synced.

After the sync, you see the Majordomo server's view of every stage that isn't ready yet, with the reason:

```
north: plan v1 out_of_sync - upstream_newer (based_on spec 1a2b3c4d5e6f != latest v2 9f8e7d6c5b4a)
north: ut-cases missing
```

`north: all stages ready` (plugin: `Majordomo: all stages ready`) means every stage is synced and consistent.

#### When the Majordomo server is unreachable

The events are queued on disk and the stage shows `queued (north unreachable); not synced yet` (plugin: `queued (Majordomo unreachable); not synced yet.`). The next sync sends the queued events first; the CLI also sends them on any run, the plugin on the next start, sign-in or stage. Nothing is lost, and resending never creates duplicates.

### 3.7 Step 7: Check the status

**CLI**
```
/status <id>
```

**Plugin**
```
/md-sdlc:status
```

Shows what **the Majordomo server** holds, next to your local versions. It's read-only: it never changes anything.

```
execution status : running
approval status  : not_approved
approvals        : 0 of 1 needed

stage           north (latest version)             local
spec            v2 score 4.3 at 2026-11-04T10:12Z  v3
plan            v1 score 4.1 at 2026-11-04T10:40Z  v1
ut-cases        -                                  v1  UNSYNCED (files not in north yet)
implementation  -
unit-test       -
```

| Line | Meaning |
|---|---|
| `execution status` | `running`, `stopped`, `failed` or `deleted` |
| `approval status` / `approvals` | approved or not, and how many approvers of the quorum |
| stage table | the Majordomo server's latest version and score per stage; your local latest version, marked `UNSYNCED` if the Majordomo server doesn't have it yet |

The plugin shows the approval on one line (`approval : not_approved (0 of 1 needed)`), adds the commit of the target branch the plan was built on (`plan target_sha`), and lists any out-of-sync stages under the table.

In the example, `spec` local is `v3` but the Majordomo server has `v2`: you regenerated or revised the spec and haven't synced it. Local and server version numbers are counted separately and don't need to match.

### 3.8 Step 8: Raise the pull request

When all five stages are synced:

**CLI**
```
majordomo> /raise-pr 6736023f-fc8c-40de-b0ae-72b199b3b38e
Pushed sdlc-6736023f-... (1 commit).
Opened PR: https://github.com/acme/payments/pull/118
```

**Plugin**
```
/md-sdlc:raise-pr
```

This pushes `sdlc-<id>` and opens a PR into the target branch. It **never force-pushes**, and if a PR is already open it reports that one (`PR already open: <url>`) instead of opening another.

Notes:

- Only **commits** are pushed. Uncommitted changes are reported and left out.
- It doesn't check review or sync state. You *can* raise the PR early, but the merge gate stays red until every stage is synced and approved.
- After later changes (e.g. a re-run implementation), raise the PR again to push the new commits. The PR updates; no new PR is opened.
- **CLI only:** the container pushes and opens the PR with your `REPO_TOKEN`.
- **Plugin only:** the push uses your own git credentials, and the PR is opened with `gh`. Git runs without prompting, so you need a credential helper or SSH agent that works without asking. If `gh` is missing or not signed in you get `no PR opened: ...`: install the GitHub CLI, run `gh auth login`, and raise the PR again. If the branch has no commits beyond the target branch, you get `nothing to raise`: run the implementation first.

### 3.9 Step 9: Approval and merge

The PR can only be merged once a **Majordomo user** (a code owner of the repo) approves the execution on the Majordomo dashboard. Neither the CLI nor the plugin has an approve command: ask your code owner to approve it.

When it's approved and every stage is synced, the PR's `sdlc-gate` check turns green and you merge the PR on GitHub as usual. The status command shows whether it's approved. If the gate stays red, a stage is probably out of sync: see [section 3.10](#310-out-of-sync-stages).

### 3.10 Out-of-sync stages

Each stage is built on the stages before it. If one of those changes after a later stage was built, the later stage is **out of sync**. You can still sync, but the PR can't be merged until it's fixed. The status command and the `sdlc-gate` check name the stage and the cause.

**The fix is almost always the same:** re-run the flagged stage, review it and sync it, then do the same for each later stage, in order. If the code changed, finish by raising the PR again.

| Cause | What it means | What to do |
|---|---|---|
| `upstream_newer` | An earlier stage was synced again after this one was built. | Re-run, review and sync this stage and every later one. |
| `upstream_out_of_sync` | An earlier stage is itself out of sync. | Fix the earliest flagged stage first. |
| `review_stale` / `review_not_latest` | The artifact changed after it was reviewed. | Review, then sync the stage. |
| `target_moved` | The target branch (e.g. `main`) moved after the plan was built. | Merge or rebase the target into `sdlc-<id>`, then re-run `plan` and every later stage. |
| `code_changed` / `branch_moved` | The PR has commits the synced implementation doesn't (a hand fix, a merge, GitHub's *Update branch*). | Commit your change on `sdlc-<id>` (or give feedback on `implementation`), review and sync the implementation, then re-run `unit-test` and raise the PR again. |
| `incomplete` | A stage was never synced. | Run, review and sync it. |

Tips:

- **The issue changed:** update the spec with feedback on `spec` describing the change, then re-run the later stages.
- **Don't re-review a stage that's already synced** unless you also mean to re-run the stages after it: syncing it again makes them `upstream_newer`.
- **Sync says `input_mismatch`:** sync the upstream stage first. If it still fails, re-run this stage.

---

## 4. Working with the CLI and the plugin

### 4.1 Managing containers (CLI)

**CLI only.**

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

### 4.2 Running commands from scripts (CLI)

**CLI only.** Every command also runs straight from your shell, without the prompt. Drop the leading `/`:

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

### 4.3 Working in your checkout (plugin)

**Plugin only.**

#### One branch per execution

Each execution has its own branch, `sdlc-<id>`, and its own folder, `.majordomo/execution/<id>/`. You can have several executions in one clone:

- **Switch** with `git checkout sdlc-<other-id>`. The plugin then works on that execution.
- **Or name the id** in the command (`/md-sdlc:status <id>`) to act on an execution without switching.

Stage commands on the wrong branch only warn, so check the branch before you run one. `/md-sdlc:implement` refuses on the wrong branch, so code never lands on another execution's branch.

#### What's committed and what isn't

- **Committed:** only the implementation's code changes, on `sdlc-<id>`.
- **Never committed:** everything under `.majordomo/`. The plugin doesn't add it to `.gitignore`, so it shows up as untracked files. To keep `git status` clean without changing the repo, add `.majordomo/` to `.git/info/exclude`.

#### Where your work lives

- **Synced versions** are on the Majordomo server. Starting the execution again, on any machine, restores them.
- **Unsynced local versions** live only in `.majordomo/execution/<id>/` in this clone. **Sync what you want to keep.**
- **Code commits** stay in your clone until syncing the implementation or raising the PR pushes them.

#### Leaving and coming back

Nothing runs in the background. Close Claude Code whenever you like. Next time, open Claude Code in the clone, check out `sdlc-<id>` and carry on.

#### On another machine

Sign in there, then run `/md-sdlc:start <issue>` in a clone of the repo. You get the same execution, its branch (if it was pushed) and every synced stage. Unsynced versions and unpushed commits stay on the first machine.

---

## 5. Reference

### 5.1 Command reference

In the CLI, `<id>` is the execution id (the EXECUTION ID column of `/issues`). In the plugin, `[id]` can be left out on the execution's branch and goes first when you give it. `[stage]` defaults to the last stage run.

#### Lifecycle commands

| CLI | Plugin | Does |
|---|---|---|
| `/login [--device] [--provider github\|bitbucket]` | `/md-sdlc:login [server-url] [--device] [--provider github\|bitbucket]` | sign in with your git account |
| `/logout` | `/md-sdlc:login logout` | remove the stored session |
| — | `/md-sdlc:login whoami` | show who is signed in, to which server, and until when |
| `/start <repo-url> <issue> [target-branch] [checkout-folder]` | `/md-sdlc:start [repo-url] <issue> [target-branch]` | get or create the execution and switch to its branch (CLI: start its container) |
| `/issues` | — | list executions started on this machine, with container state |
| `/stop <id>` | — | stop the container |
| `/remove <id>` | — | delete the container, drop it from `/issues` |
| `/help` | — | list commands (plugin: type `/md-sdlc:`) |
| `/quit` | — | leave the CLI (asks first; containers keep running) |

#### Stage commands

| CLI | Plugin | Does |
|---|---|---|
| `/spec <id>` | `/md-sdlc:spec [id]` | generate the spec |
| `/plan <id>` | `/md-sdlc:plan [id]` | generate the plan |
| `/ut-cases <id>` | `/md-sdlc:ut-cases [id]` | generate the unit-test cases |
| `/implement <id>` | `/md-sdlc:implement [id]` | implement and commit on `sdlc-<id>` |
| `/unit-test <id>` | `/md-sdlc:unit-test [id]` | run the tests and write the report |
| `/feedback <id> [stage] <instructions>` | `/md-sdlc:feedback [id] [stage] <instructions>` | the agent revises the stage as told: a new local version |
| `/review <id> [stage]` | `/md-sdlc:review [id] [stage]` | review and score the latest local version |
| `/sync <id> [stage]` | `/md-sdlc:sync [id] [stage]` | send the latest reviewed version(s) to the Majordomo server |
| `/raise-pr <id>` | `/md-sdlc:raise-pr [id]` | push `sdlc-<id>` and open (or find) its PR |
| `/status <id>` | `/md-sdlc:status [id]` | show the Majordomo server's view and your local versions |
| `/reply <id>` | — | reconnect to a stage waiting on your answer (plugin: answer in the chat) |
| `/cancel <id>` | — | abandon the stage waiting on your answer; nothing is saved (plugin: press Esc) |
| `/set-model <id> <model>` | — | use another model in this container (plugin: change your session's model) |
| `/set-github-token <id> <token>` | — | replace the container's repo token |

### 5.2 Settings reference

Read from the environment, then `./.env` (or `MAJORDOMO_ENV_FILE`), then `~/.majordomo/.env`. The first one that has a setting wins. The plugin reads `./.env` at the root of the repo, and reads the `MAJORDOMO_*` settings from the environment only.

| Setting | Used by | Default | Purpose |
|---|---|---|---|
| `BACKEND_API_URL` | both | — (required) | the Majordomo server's address, ending in `/sdlc/v1` |
| `TENANT_ID`, `WORKSPACE_ID` | both | — | tenant and workspace of the repo |
| `REPO_TOKEN` | CLI | — | the repo token used by the container |
| `COPILOT_GITHUB_TOKEN` | CLI | — | Copilot agent only |
| `AGENT_MODEL` | CLI | agent default | model for every stage run |
| `MAJORDOMO_ENV_FILE` | both | `./.env` | another settings file |
| `MAJORDOMO_HOME` | both | `~/.majordomo` | folder for the settings file and the sign-in |
| `MAJORDOMO_CREDENTIALS` | both | `~/.majordomo/credentials.json` | another file for the sign-in |
| `MAJORDOMO_TOKEN` | both | — | use this Majordomo server token instead of signing in (CI, tests) |
| `MAJORDOMO_NO_BROWSER` | both | — | always use device sign-in |
| `MAJORDOMO_LOGIN_TIMEOUT_MS` | both | 5 minutes | how long sign-in waits for the browser |
| `CLAUDE_CONFIG_DIR` | both | `~/.claude` | where the Claude Code login is (plugin: where it reads the session's token counts) |
| `CODEX_HOME` | CLI | `~/.codex` | where the Codex login is |

### 5.3 Files and where they live

#### On your machine

| Path | Used by | What |
|---|---|---|
| `~/.majordomo/.env` | both | your settings |
| `~/.majordomo/credentials.json` | both | the Majordomo server session from signing in |
| `~/.majordomo/issues.json` | CLI | executions started on this machine (`/issues`) |
| `~/.majordomo/outbox.jsonl` | CLI | lifecycle events waiting for the Majordomo server |
| `<checkout>/` | both | the repo, on branch `sdlc-<id>`. CLI default: `./<repo-name>-<issue>`. Plugin: your own clone. |

#### In the checkout

```
<checkout>/.majordomo/
├── execution/<execution-id>/
│   ├── execution.json                     the execution's repo, issue and branches (plugin)
│   ├── issue.json                         the issue, cached on first use
│   ├── state.json                         local versions and their sync state
│   ├── spec/            spec.md                  review.md  rubric.json
│   ├── plan/            plan.md                  review.md  rubric.json
│   ├── ut-cases/        unit-test-cases.md       review.md  rubric.json
│   ├── implementation/  implementation-summary.md review.md rubric.json
│   ├── unit-test/       test-report.md           review.md  rubric.json
│   ├── setup/           setup-report.md          (if the tests needed setup)
│   └── verification-reports/              compiler, lint, test and coverage reports (plugin)
├── templates/                             templates the skills use
├── docs/                                  reference docs the skills use (plugin)
└── outbox/                                events waiting for the Majordomo server
```

Edit artifacts in place; don't rename or move them. Don't edit `state.json` or `execution.json`.

---

## 6. FAQ

### Which coding agent will my repo use, and which login do I need?

- **CLI:** your admin picks the agent for each repo. You don't have to know it in advance: `/start` says which credential is missing before it pulls anything. The logins for each agent are in the [CLI installation guide, section 1](md-sdlc-container-mode-installation.md#1-before-you-start).
- **Plugin:** always Claude Code, in your own session, with whatever model the session uses. You only need your Claude Code login.

### Can anyone else see my work before I sync it?

No. Running, reviewing and revising a stage only change your checkout. Only syncing sends a stage's files to the Majordomo server, where the team, the dashboard and the merge gate can see them. The exception is the implementation's code: the plugin pushes it to `sdlc-<id>` when you sync the implementation, and both the CLI and the plugin push it when you raise the PR.

### Can I switch between the CLI and the plugin on the same issue?

Yes. There is one execution per repo and issue, so the CLI and the plugin work on the same one. Start it with the other one: you get the same execution and branch, and every synced stage is restored. Sync before you switch, because unsynced versions don't move. If the execution was started with the CLI, the plugin warns that it "was started in another mode"; that's expected.

### Do I lose my work if I remove a container?

**CLI only.** No for synced work, and no for your checkout. `/remove` deletes only the container: the execution on the Majordomo server and your checkout folder, with its local versions, stay. `/start <repo> <issue>` brings a container back and restores every synced version. See [section 4.1](#41-managing-containers-cli).

### Can I continue an execution on another machine?

Yes. Start the execution there with the same repo and issue: you get the same execution and everything that was synced. Work you didn't sync on the first machine isn't on the Majordomo server, so it doesn't come back.

### Can several people work on the same issue?

Yes. There is only **one execution per repo and issue**, so everyone works on the same one. When a teammate starts it on their machine, with the CLI or the plugin, they get the existing execution and its working branch `sdlc-<id>`, and every stage version that has been synced so far is restored.

What each person sees:

- **Synced work is shared. Local work isn't.** Each person has their own checkout (and, in the CLI, their own container). A stage you've run or reviewed but not synced is only on your machine. Once you sync it, everyone gets it the next time they start the execution.
- **The latest sync wins.** If two people sync the same stage, both versions are kept, and the one synced last becomes the stage's latest version. The dashboard, the status command and the merge gate use that one.
- **Stages have to fit together.** If a teammate syncs a new spec after you built the plan, your plan's sync is refused with `input_mismatch`, because your plan was built on the old spec. Re-run the plan so it's built on their spec, then review and sync it. If your plan was already synced, it becomes out of sync (`upstream_newer`) instead ([section 3.10](#310-out-of-sync-stages)).
- **Code goes through one branch.** Everyone pushes to the same `sdlc-<id>` branch and the same PR. Pushes are never forced, so if a teammate pushed first, your push is refused: pull their commits into your checkout, then push again.
- **Every run and sync is recorded under the person who did it**, so the dashboard shows who did what.

To avoid redoing each other's work, agree on who owns which stage, sync as soon as a stage is reviewed, and check the status before you start a stage to see what is already on the Majordomo server.

### Why is the merge gate still red when every stage is synced?

Usually the execution isn't approved yet, a stage is out of sync, a review is stale, or the target branch moved. The status command shows what is missing and why. See [section 3.9](#39-step-9-approval-and-merge) and [section 3.10](#310-out-of-sync-stages).
