<p align="center">
  <img src="assets/md-full-logo.png" alt="Majordomo" width="400">
</p>

# Majordomo SDLC

Majordomo SDLC takes an issue from a written request to a reviewed, approved pull request, with a coding agent doing the writing and your team keeping control of every step.

## How it works

Each issue goes through five stages:

**spec → plan → ut-cases → implementation → unit-test**

| Stage | What the agent produces |
|---|---|
| **spec** | a specification of what the issue asks for |
| **plan** | an implementation plan built from the spec |
| **ut-cases** | the unit test cases the code must pass |
| **implementation** | the code, committed to the issue's branch `sdlc-<id>`, with compiler, lint, test and coverage reports |
| **unit-test** | a report from running the tests |

For every stage:

1. **The agent writes it.** You answer any questions it asks.
2. **You review it.** A rubric review scores it from 1 to 5 and explains what to fix.
3. **You revise it** with your own feedback until you're happy with it.
4. **You sync it** to the Majordomo server, where your team, the code owners and the merge gate can see it.

When every stage is synced, you raise the pull request. A code owner approves the work on the Majordomo dashboard, and the PR can be merged.

## Two ways to run it

| | **Majordomo SDLC CLI** (`majordomo-sdlc`) | **Majordomo SDLC Claude plugin** (`md-sdlc`) |
|---|---|---|
| Where the agent runs | In a Docker container on your machine, one per issue | In your own Claude Code session |
| Coding agent | Claude Code, Codex or Copilot, as your admin set for the repo | Claude Code |
| You need | Docker and the CLI | Claude Code, Git, Python 3 and the GitHub CLI `gh` |
| Commands | `/plan <id>` at the `majordomo>` prompt | `/md-sdlc:plan` in Claude Code |

Both work on the same executions on the same Majordomo server, so you can switch between them on the same issue.

## Get started

Your Majordomo admin sets up the repo and gives you the Majordomo server URL, tenant ID and workspace ID. Then:

1. **Install one of them:**
   - [Majordomo SDLC CLI installation guide](documentation/md-sdlc-container-mode-installation.md)
   - [Majordomo SDLC Claude plugin installation guide](documentation/md-sdlc-plugin-inststallation.md)
2. **Read the [Developer manual](documentation/developer-manual.md).** It walks through every step, from creating the issue to merging the PR, for both the CLI and the plugin, and has the full command, settings and troubleshooting reference.

## Downloads

Builds are published as [releases](https://github.com/majordomoinc/majordomo-sdlc-public/releases) of this repo:

| Release | What's in it |
|---|---|
| [`cli-dev`](https://github.com/majordomoinc/majordomo-sdlc-public/releases/tag/cli-dev) | the CLI for Linux, macOS and Windows, the npm package, and `SHA256SUMS` |
| [`skills-latest`](https://github.com/majordomoinc/majordomo-sdlc-public/releases/tag/skills-latest) | the Claude plugin, `md-sdlc.zip`, and `SHA256SUMS` |

The installation guides say which file to download and how to install it.
