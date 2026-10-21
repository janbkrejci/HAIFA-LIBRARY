---
name: factory
description: >-
  Use when a coding agent sets up HAIFA on a machine (check, library), adds HAIFA
  to an existing Git repository or takes over a repo that already has it,
  manages its agents and workflows, turns a plan into a project/step/task
  backlog, drives factory tasks, or upgrades HAIFA. Do not use for general
  coding without HAIFA, unpacking the first HAIFA bundle (install.sh), or
  multi-repository operations.
---

# HAIFA skill: driving the `factory` CLI (aifactory 0.1.472)

This guide is generated from the installed code by `factory --skill`.
Save its Markdown output as an agent's `SKILL.md`; `factory --skill --json`
returns the same guide inside the JSON envelope. Compare `factory --version`
with aifactory 0.1.472: whenever the version changes, regenerate this skill
with `factory --skill` and replace the saved copy.

## Prerequisites

Start in the existing Git repository with `factory check --json`. The check only
reads; even a failed check can return `data.findings`. Read each finding's `action`
and `fix`, together with `data.state` and `data.action`, before making changes:

| State | Action | Next step |
| --- | --- | --- |
| `none` (repo without factory) | `init` | Add factory to the existing repo below. |
| `uncommitted` (configuration only in the working tree) | `config_commit` | Review and commit configuration using the Commit configuration procedure, then check again. |
| `installed` (`.factory/config.yaml` in base) | none | Factory is installed. Agents and workflows the repo lacks come from the library. |
| `unsupported` (sssf `adws/` without `.factory/config.yaml`) | none | HAIFA does not take over sssf. Tell the user; do not convert it. |

Resolve machine findings (Git, provider CLI, chosen harness CLI and credentials)
using their `fix`; do not run tasks until error findings are resolved. See Check for
the payload details.

## Calling convention

- Start with `factory check --json` in a repository you have not worked in yet: it says
  (only by reading) whether factory will run there and on this machine, and what to fix.
  On a new machine run it anywhere, even outside a repository (`factory` without a command
  prints "První spuštění: factory check").
- Always pass `--json`. stdout then carries exactly one JSON document, the envelope below;
  engine narration and tracebacks go to stderr. Parse stdout only.
- Most commands take `--repo PATH` (default: the git repository of the current directory).
- Runs read the configuration (`.factory/`) and the backlog from the **base commit**, not
  from the working tree. Commit (and push) backlog and `.factory/` edits before you run a
  task; `factory config status --json` lists the uncommitted `.factory/` changes a run
  would not see. Commit them with `factory config commit --json` (see Commit
  configuration) and the backlog with `factory backlog commit --json`.

### The envelope

Success:

```json
{"ok": true, "data": {"...": "command payload"}, "error": null, "warnings": []}
```

Failure:

```json
{
  "ok": false,
  "data": null,
  "error": {"code": "unknown_task", "message": "...", "path": null, "id": "X-T01", "issues": []},
  "warnings": []
}
```

Rules (every command follows them, tests enforce them):

- The top-level keys are exactly `ok`, `data`, `error`, `warnings`.
- `ok` is true exactly when the exit code is 0. With `ok: true`, `error` is `null`.
- With `ok: false`, `error` is an object with exactly `code` (string), `message` (non-empty
  string), `path` (string or null), `id` (string or null) and `issues` (list of objects).
- With `ok: false`, `data` may still be filled when a result exists: a failed run,
  `backlog check` or `workflow check` with problems, a missing harness.
- `error.code` is always one of the error codes listed below.
- `error.issues` lists the problems that caused the error (backlog, workflow, roles or
  config issues). Each has at least `code` and `message` (config issues have
  `code: "invalid_config"` and a `path`).
- `warnings` is always a list of strings. Read them; they are not errors.
- Exit codes:
  - `0`: ok.
  - `1`: the command ran and found a problem: validation issues, a failed run, a missing
    harness, a pull request that could not be opened, a write that would break the backlog.
  - `2`: the command could not do its job: bad input, configuration, state, or a provider
    error. Nothing was changed.
- `--help` and `--version` print plain text, not an envelope.

## Commands

### Switch the local roster

`factory config roster show --json` lists effective local agent harness/model/thinking.
`factory config roster set codex --dry-run --json` previews all-Codex
(`gpt-6.1-sol`, medium); omit `--dry-run` to write locally. Use
`factory config roster set claude --json` for all-Claude (`claude-opus-5-5`, medium).
A whole-roster preset clears existing per-agent harness/model/thinking overrides,
retaining purposes, prompts, writes and other settings. For a mixed roster, set Claude
first, then `factory config roster set codex --agent builder --json`; a reviewer can
be set with `factory config roster set claude --agent reviewer --json`.
`--harness`, `--model`, `--thinking` override preset values; all commands accept `--repo`.
Review the returned diff, then use the existing `factory config commit` flow. Runs read
committed base; workflow step overrides remain authoritative.

Every command below accepts `--json`.

### factory check

```
factory check [--repo PATH] [--offline] [--json]
```

Check, only by reading, whether factory will run in this repository and on this machine, and what to fix. Reports the state of the repo read from base (installed, unsupported, uncommitted or none) with its action (config_commit or init, or null) and findings {code, scope, severity, message, fix, action}: scope 'repo' is fixed and committed to base, 'machine' is fixed locally, 'library' is fixed in the library in $HAIFA_HOME; severity is error, warning or info; action names the fix (init, config_commit, config_pull) or is null. Outside a git repository (without --repo) only the machine and the library are checked (in_repo false). Ahead/behind compare base and the library with local remote-tracking refs (no fetch). Exits 0 without errors, 1 with an error (checks_failed), 2 when --repo is not a git repository.

- `--repo` `PATH`: repository to check (default: the current git repo)
- `--offline`: skip calls to the hosting and the harness logins (gh auth status, az account show, claude auth status, codex login status, pi auth check, pi --list-models)
- `--json`: print machine-readable JSON

### factory init

```
factory init [--repo PATH] [--base BRANCH] [--provider {local,github,azure}] [--azure-org ORG] [--azure-project PROJECT] [--azure-repo REPO] [--docs-root DIR] [--base-changes {direct,pr}] [--base-pr-merge {manual,auto}] [--dry-run] [--commit] [--pr] [--expect DIGEST] [-m TEXT] [--force] [--json]
```

Install factory into a git repository that has none: .factory/config.yaml, the backlog directory and the .gitignore lines. Agents, workflows, skills and extensions are not copied: runs take them from the library in the home directory (the seed in the package without a library); a copy in .factory/ wins. Three modes. Without --dry-run and --commit the files are written to the working tree, existing files are skipped unless --force and nothing is committed. --dry-run returns the plan: files with content, blockers, warnings, digest and the detected remote, base, provider and harness CLIs (detected); it writes nothing. --commit makes the same plan one commit on base (pushed before base moves; --pr opens a pull request from factory-init/<n> instead); --expect DIGEST refuses a plan that changed (plan_changed). Blockers: already_installed (config.yaml in base), existing_config (an sssf installation; HAIFA does not take it over), config_not_committed (run factory config commit), dirty_paths, invalid_plan, not_on_base, run_in_progress, base_behind, base_diverged. Nothing is written to the library.

- `--repo` `PATH`: repository to install into (default: the current git repo)
- `--base` `BRANCH`: base branch of the runs (default: the remote HEAD with --dry-run/--commit, else the current branch)
- `--provider` `{local,github,azure}`: git provider (default: from the remote URL with --dry-run/--commit, else local) (one of `local`, `github`, `azure`)
- `--azure-org` `ORG`: Azure DevOps organization
- `--azure-project` `PROJECT`: Azure DevOps project
- `--azure-repo` `REPO`: Azure DevOps repository
- `--docs-root` `DIR`: root of the factory documents with backlog/, specs/ and docs/ (default: factory)
- `--base-changes` `{direct,pr}`: how factory changes base from now on: direct commits (default) or a pull request per change; pr also makes --commit open a pull request (one of `direct`, `pr`)
- `--base-pr-merge` `{manual,auto}`: factory's own pull requests: left to a reviewer (default) or approved where allowed and merged at once (GitHub with --admin) (one of `manual`, `auto`)
- `--dry-run`: only return the plan; write nothing
- `--commit`: commit the plan to base in one commit
- `--pr`: with --commit: open a pull request instead
- `--expect` `DIGEST`: with --commit: refuse unless the plan has this digest
- `-m`, `--message` `TEXT`: with --commit: commit message
- `--force`: overwrite existing files
- `--json`: print machine-readable JSON

### factory try

```
factory try [DIR] [--json]
```

Create the sample repository: a small Python library with tests (python -m unittest discover -s tests -t .) and a backlog of three tasks of growing size (DEMO-S01-T01..T03). The folder (default: demo in the HAIFA home) gets git init with base main, .factory/config.yaml with git_provider local (no GitHub, no gh), the .gitignore lines and one commit; the repository is registered in the dashboard. A folder that exists and is not empty is not changed: its path is returned with created false (a git repository there is registered). git needs user.name and user.email (git_identity_missing); a path that is a file is invalid_value.

- `DIR`: folder of the sample repository (default: $HAIFA_HOME/demo)
- `--json`: print machine-readable JSON

### factory task add

```
factory task add STEP TITLE [--id CODE] [--slug SLUG] [--workflow NAME] [--writes [PATH ...]] [--depends-on ID [ID ...]] [--related ID [ID ...]] [--body TEXT] [--assignee {agent,human}] [--json] [--repo PATH]
```

Create a new todo task file in the directory of STEP. The backlog is validated before writing; a change that would break it is rejected.

- `STEP`: id of the step the task belongs to (required)
- `TITLE`: task title (required)
- `--id` `CODE`: short code, e.g. T05; the id is STEP-CODE (default: the next free)
- `--slug` `SLUG`: file name slug (default: from the title)
- `--workflow` `NAME`: workflow (default: inherited)
- `--writes` `[PATH ...]`: paths the task writes
- `--depends-on` `ID [ID ...]`: dependencies
- `--related` `ID [ID ...]`: related tasks
- `--body` `TEXT`: text of '## Zadání'
- `--assignee` `{agent,human}`: human: a task for a person, no run starts it (default: inherited, agent) (one of `agent`, `human`)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task edit

```
factory task edit ID [--title TEXT] [--status STATUS] [--workflow NAME] [--clear-workflow] [--writes [PATH ...]] [--clear-writes] [--auto-merge {on,off,inherit}] [--assignee {agent,human,inherit}] [--json] [--repo PATH]
```

Change title, status (todo|cancelled, or done for a person's task), workflow, writes, auto_merge or assignee of a task.

- `ID`: task id (required)
- `--title` `TEXT`: new title
- `--status` `STATUS`: todo or cancelled; done for a task with assignee human (a person did it)
- `--workflow` `NAME`: set the workflow
- `--clear-workflow`: remove the workflow (inherit it)
- `--writes` `[PATH ...]`: set writes (none = [])
- `--clear-writes`: remove writes (inherit)
- `--auto-merge` `{on,off,inherit}`: set auto_merge of the task (off, e.g. when it changes the guard or permissions) or remove it (inherit) (one of `on`, `off`, `inherit`)
- `--assignee` `{agent,human,inherit}`: human: the task is for a person and no run starts it; inherit removes it (one of `agent`, `human`, `inherit`)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task follow-up

```
factory task follow-up ID --body TEXT [--title TEXT] [--run] [--json] [--repo PATH]
```

Create a new todo task in the step of the done task ID with follows: ID, the own writes and workflow of ID (inherited values stay inherited) and TEXT as its '## Zadání'. The title defaults to 'Navazuje na ID: <title of ID>'. A task that is not done is refused (not_done). With --run the backlog is committed to base (factory backlog commit) and the new task is run (factory task run).

- `ID`: id of the done task to follow (required)
- `--body` `TEXT`: text of '## Zadání' of the new task (required)
- `--title` `TEXT`: title of the new task
- `--run`: commit the backlog to base and run the new task right away
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task link

```
factory task link ID [--depends-on ID [ID ...]] [--related ID [ID ...]] [--remove] [--json] [--repo PATH]
```

Add (or with --remove drop) depends_on and related references of a task.

- `ID`: task id (required)
- `--depends-on` `ID [ID ...]`: dependencies
- `--related` `ID [ID ...]`: related tasks
- `--remove`: remove the references instead
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task show

```
factory task show ID [--json] [--repo PATH]
```

Print one task.

- `ID`: task id (required)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task list

```
factory task list [--status STATE] [--project ID] [--step ID] [--json] [--repo PATH]
```

List tasks with their derived state (ready/blocked/done/cancelled).

- `--status` `STATE`: todo|done|cancelled|ready|blocked
- `--project` `ID`: only this project (id or directory)
- `--step` `ID`: only this step
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task runs

```
factory task runs ID [--json] [--repo PATH]
```

List every run of a task, newest first: state, failure_kind (why a run did not succeed: accept_not_met, nothing_to_do, quota, writes_violation, writes_needed, invalid_output, process_gone, stopped, blocked or other) and the error; a blocked run (it waits for blocking tasks) lists its blockers. --json gives each run with failure_kind and failure_detail.

- `ID`: task id (required)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task run

```
factory task run ID [--note TEXT] [--force] [--auto] [--harness NAME] [--model NAME] [--thinking {off,minimal,low,medium,high,xhigh,max,ultra}] [--member] [--started-by {manual,auto-continue,auto-resolve,auto-ci-fix}] [--json] [--repo PATH]
```

Run a task from the backlog in base in a new worktree <worktrees_dir>/<run-id> on a new branch factory/<task-id>-<n>. Agents may change only the task's writes and its spec/doc files inside the worktree; anything else, including any change in the main checkout, is rolled back and fails the phase. With --auto a run that opened its PR is followed by the next ready task (auto-continue). Without it auto_continue: true (from the index.md of the step or project) must hold for both the finished and the next task. With auto_merge: true a reviewed run's PR is merged first when it may be (see 'factory backlog auto-merge --help').

- `ID`: task id (required)
- `--note` `TEXT`: a note for the agents, appended to the task
- `--force`: run even if depends_on are not done in base; never overrides a running task
- `--auto`: after a successful run (PR opened) start the next ready task of the step, then of the project, whatever its auto_continue (auto-continue); a failure stops the chain. The kanban's queue order goes first and tasks excluded in the kanban are never started by the chain
- `--harness` `NAME`: run every agent of the roster on this harness for this run only (nothing is written); a switch without --model takes the harness's preset model. Step overrides of the workflow still win, so one workflow can mix harnesses. With --auto only the first run of the chain gets it
- `--model` `NAME`: model of every agent for this run only (like --harness)
- `--thinking` `{off,minimal,low,medium,high,xhigh,max,ultra}`: thinking level of every agent for this run only (like --harness) (one of `off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`, `ultra`)
- `--member`: ==SUPPRESS==
- `--started-by` `{manual,auto-continue,auto-resolve,auto-ci-fix}`: ==SUPPRESS== (one of `manual`, `auto-continue`, `auto-resolve`, `auto-ci-fix`; default `manual`)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task approve

```
factory task approve ID [--allow-writes] [--json] [--repo PATH]
```

Add a commit with status: done and a line in '## Běhy' to the task's pull request, wait for the CI checks of that head, merge it, bring base up to date with the remote and remove the branch's worktrees. After restacking dependents, delete the merged branch locally and on the remote unless an open stacked PR, a running run or a checkout still uses it. Deletion failures are warnings; deleted_branches lists deleted refs (remote refs prefixed by the remote name). Red checks start up to 2 ci-fix repair runs on the PR branch (started_by auto-ci-fix), each followed by a new wait; checks still red or running leave the PR open (auto_merge_error). Azure sends an approval vote after the final push and before merge; local and GitHub do not send a self-approval review. A ci-fix run that needs paths outside the task's writes leaves the PR open with writes_needed; --allow-writes adds those paths to the task's writes in base first.

- `ID`: task id (required)
- `--allow-writes`: after a ci-fix run that ended writes_needed: add the paths it needs to the task's writes (committed to base by base_changes), then approve again; the new ci-fix runs may write them. Paths in protected_files or in the backlog are never added
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task return

```
factory task return ID --note TEXT [--json] [--repo PATH]
```

Start a new run on the same branch with the note in the prompt; the pull request is updated.

- `ID`: task id (required)
- `--note` `TEXT`: what to change, appended to the task (required)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task resolve

```
factory task resolve ID [--json] [--repo PATH]
```

Start a new run on the branch of the task's pull request: rebase it onto the current base; on a conflict an agent changes only the conflicted files. Then the suite runs, the branch is force-pushed (with a lease) and the pull request updated. When the run fails, the branch stays as it was before the rebase.

- `ID`: task id (required)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task publish

```
factory task publish ID [--json] [--repo PATH]
```

For the task's last succeeded run that has no pull request (its push or PR failed at the end of 'task run'): push the branch and open the pull request with the body from the run, then finish auto-merge and auto-continue as after 'task run'. An open PR the hosting already has for the branch is adopted and its body updated. Without a succeeded run, with a known PR or while a run is in progress it fails with exit 2 and changes nothing.

- `ID`: task id (required)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task clean

```
factory task clean [--json] [--repo PATH]
```

Remove worktrees of runs whose PR is merged or closed, and of abandoned runs (a newer run of the task exists). Also delete local and remote branches of merged PRs, including older merges, unless an open stacked PR, a running run or a checkout still uses them. Keep closed and abandoned branches. Deletion failures are warnings; deleted_branches lists deleted local and remote refs.

- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task stop

```
factory task stop ID [--run RUN_ID] [--json] [--repo PATH]
```

Send SIGTERM to the coding agents of the task's running run and then to the run's process (SIGKILL after a timeout). The run is marked stopped; its worktree and branch stay (factory task clean removes them).

- `ID`: task id (required)
- `--run` `RUN_ID`: stop exactly this run (it must belong to the task)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task pause

```
factory task pause ID [--run RUN_ID] [--json] [--repo PATH]
```

Ask the task's running run to pause. The phase in progress (an agent, a test) runs to its end, then the run waits as paused before the next phase; its process stays alive. factory task resume lets it go on, factory task stop stops it.

- `ID`: task id (required)
- `--run` `RUN_ID`: pause exactly this run (it must belong to the task)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task resume

```
factory task resume ID [--run RUN_ID] [--json] [--repo PATH]
```

Resume the task's paused run: it goes on with its next phase in the same worktree and branch. A pause that is still pending (pausing) is dropped.

- `ID`: task id (required)
- `--run` `RUN_ID`: resume exactly this run (it must belong to the task)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory task retry

```
factory task retry ID [--run RUN_ID] [--note TEXT] [--from PHASE] [--auto] [--allow-writes] [--harness NAME] [--model NAME] [--thinking {off,minimal,low,medium,high,xhigh,max,ultra}] [--json] [--repo PATH]
```

Continue the task's newest failed, stopped or aborted run in its own worktree and branch, as the next attempt of the same run. Phases already done are not repeated and every agent rejoins its session. Base is pulled from the remote first; when it moved, the branch is rebased onto it (a conflict is resolved like factory task resolve) and the tests run again before any review. Configuration, backlog and workflow come from the current base.

- `ID`: task id (required)
- `--run` `RUN_ID`: retry exactly this run (it must belong to the task)
- `--note` `TEXT`: a note for the agents of the continuing phases
- `--from` `PHASE`: continue from this phase (a phase name like review_2 or a step like review) instead of the one that failed; the phases after it run again
- `--auto`: after a successful retry (PR opened) start the next ready task whatever its auto_continue, like factory task run --auto. Without it the retry goes on like task run: auto_merge merges the PR and auto_continue starts the next ready task
- `--allow-writes`: after a writes_violation or writes_needed: add the paths written outside writes or reported in needs_writes to the task's writes (committed to base by base_changes), then retry from the failed phase with the paths in the Retry note. Paths in protected_files, in the backlog or in the main checkout are never added. If base lacks the new writes (writes_not_committed), commit the backlog, then factory task retry ID
- `--harness` `NAME`: continue with every agent on this harness (e.g. after a quota failure); it replaces the run's harness override, the switch is recorded in the run and its PR. Without --model the computer's model for the harness, else its preset model
- `--model` `NAME`: model of every agent for the retry (with --harness)
- `--thinking` `{off,minimal,low,medium,high,xhigh,max,ultra}`: thinking level of every agent for the retry (with --harness) (one of `off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`, `ultra`)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory backlog check

```
factory backlog check [--json] [--repo PATH]
```

Load the backlog and report duplicate ids, unknown references, dependency cycles, invalid states and missing fields. Exits 1 on problems.

- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory backlog list

```
factory backlog list [--json] [--repo PATH] [--status {todo,done,cancelled,ready,blocked,human}] [--project ID]
```

Print the backlog tree with task states (ready/blocked derived from depends_on) and progress of every container.

- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)
- `--status` `{todo,done,cancelled,ready,blocked,human}`: show only tasks in this state (one of `todo`, `done`, `cancelled`, `ready`, `blocked`, `human`)
- `--project` `ID`: show only this project (id or directory)

### factory backlog sync

```
factory backlog sync [--json] [--repo PATH]
```

Refresh the states of task PRs; for tasks whose PR was merged outside factory and that are not done in base, open one PR that sets status: done and adds a line to '## Běhy'. The commit goes only to its own branch factory-sync/<n>, never to base, and the sync PR is not merged. Closed PRs leave the task alone.

- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory backlog auto-continue

```
factory backlog auto-continue ID [--on | --off | --inherit] [--json] [--repo PATH]
```

Write (--on/--off) or remove (--inherit) 'auto_continue' in the index.md of a project or step. With auto_continue: true, a task run that opened its PR is followed by the next ready task that has auto_continue: true too; a chain never starts a task with it off. The change is written to the working tree; runs read the backlog from base, so commit it first.

- `ID`: project or step id (required)
- `--on` (exclusive: --on|--off|--inherit (one required))
- `--off` (exclusive: --on|--off|--inherit (one required))
- `--inherit`: remove the field, inherit it from the parent (exclusive: --on|--off|--inherit (one required))
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory backlog auto-merge

```
factory backlog auto-merge ID [--on | --off | --inherit] [--json] [--repo PATH]
```

Write (--on/--off) or remove (--inherit) 'auto_merge' in the index.md of a project or step (a task sets it with 'factory task edit ID --auto-merge'). With auto_merge: true, a succeeded run whose last review approved and whose PR merges cleanly with no red checks is approved and merged like 'factory task approve'; a workflow without a review phase is never merged automatically. Auto-continue then starts the next task only after the merge. The change is written to the working tree; runs read the backlog from base, so commit it first.

- `ID`: project or step id (required)
- `--on` (exclusive: --on|--off|--inherit (one required))
- `--off` (exclusive: --on|--off|--inherit (one required))
- `--inherit`: remove the field, inherit it from the parent (exclusive: --on|--off|--inherit (one required))
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory backlog commit

```
factory backlog commit [-m TEXT] [--json] [--repo PATH]
```

Commit every change under the backlog root <docs_root>/backlog (new, edited and deleted task files) to base, so runs see it. Other changes in the checkout are left alone. The main checkout must be on base and the backlog must be valid. With a remote, the commit is pushed.

- `-m`, `--message` `TEXT`: commit message
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory backlog add

```
factory backlog add [PARENT] --id CODE --title TEXT [--body TEXT] [--json] [--repo PATH]
```

Create a project (without PARENT) or a step of project PARENT: the directory <code>-<slug> with an index.md holding id and title (a step's directory drops the project's prefix: M01-S03 -> S03-<slug>). The id matches the task id pattern, is unique, and a step's id starts with its project's id and '-'. The backlog is validated before writing; a change that would break it is rejected. Nothing is committed (factory backlog commit).

- `PARENT`: project id for a new step (none: a project)
- `--id` `CODE`: short code, e.g. M01 or S10; a step's id is PARENT-CODE (required)
- `--title` `TEXT`: short title (required)
- `--body` `TEXT`: description in index.md
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory backlog edit

```
factory backlog edit ID [--title TEXT] [--workflow NAME] [--writes [PATH ...]] [--auto-continue {on,off}] [--auto-heal {on,off}] [--quota-switch [HARNESS ...]] [--quota-switch-back {on,off}] [--clear {harness,model,thinking,workflow,writes,auto_continue,auto_heal,quota_switch,quota_switch_back} [{harness,model,thinking,workflow,writes,auto_continue,auto_heal,quota_switch,quota_switch_back} ...]] [--json] [--repo PATH]
```

Change the title of a project or step and set or remove (--clear) the keys harness, model, thinking, workflow, writes, auto_continue, auto_heal, quota_switch, quota_switch_back of its index.md. Other keys and the description stay as they are. The backlog is validated before writing; nothing is committed.

- `ID`: project or step id (required)
- `--title` `TEXT`: new title
- `--workflow` `NAME`: set the workflow
- `--writes` `[PATH ...]`: set writes (none = [])
- `--auto-continue` `{on,off}`: set auto_continue (one of `on`, `off`)
- `--auto-heal` `{on,off}`: set auto_heal: the chain's automatic actions after a failure (default on) (one of `on`, `off`)
- `--quota-switch` `[HARNESS ...]`: project only: set quota_switch, the backup harnesses in order a run switches to when its harness runs out of quota (none = [], off)
- `--quota-switch-back` `{on,off}`: project only: set quota_switch_back: new runs go back up the chain once a harness has its limit back (one of `on`, `off`)
- `--clear` `{harness,model,thinking,workflow,writes,auto_continue,auto_heal,quota_switch,quota_switch_back} [{harness,model,thinking,workflow,writes,auto_continue,auto_heal,quota_switch,quota_switch_back} ...]`: remove keys (inherit them again): harness, model, thinking, workflow, writes, auto_continue, auto_heal, quota_switch, quota_switch_back (one of `harness`, `model`, `thinking`, `workflow`, `writes`, `auto_continue`, `auto_heal`, `quota_switch`, `quota_switch_back`)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory backlog propose

```
factory backlog propose INTENT [--project ID | --step ID] [--apply] [--json] [--repo PATH]
```

Run the read-only agent 'proposer' (prompts overridable in .factory/prompts/proposer/) in an isolated checkout of the base commit. It explores the repo and the backlog and proposes a new step of --project with its tasks, or new tasks of the existing --step. Without either it places them itself: in an existing step, a new step of an existing project, or a new project with a new step, with a one-sentence placement_reason. Per task it gives the title, the assignment with a 'Hotovo znamená' section, depends_on (also on existing tasks), writes and a workflow of the catalog. The proposal is validated like 'factory task add' (ids, dependencies, no cycles or self references, existing workflows); an invalid one goes back to the agent once, a second invalid one fails with proposal_invalid and writes nothing. Findings of the quality check of writes (files the assignment names, protected_files and the backlog, overlap of tasks that can run at the same time) go back in the same correction; the rest are shown in the preview. Without --apply only the preview (project ->) step -> tasks is shown; --apply writes the files and commits nothing (factory backlog commit).

- `INTENT`: what the new tasks should achieve (required)
- `--project` `ID`: propose a new step of this project (default: the agent chooses the place) (exclusive: --project|--step)
- `--step` `ID`: propose new tasks of this existing step (default: the agent chooses the place) (exclusive: --project|--step)
- `--apply`: write the proposal to the backlog (no commit)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory workflow check

```
factory workflow check FILE [--json] [--roles PATH] [--agents PATH]
```

Validate a workflow file against the role registry (.factory/roles.yaml or the packaged defaults) and the agent roster (.factory/agents.yaml) of the working tree. Nothing runs. Exits 1 when the workflow has problems, 2 when the registry or roster cannot be loaded.

- `FILE`: workflow YAML file (required)
- `--json`: print machine-readable JSON
- `--roles` `PATH`: role registry instead of .factory/roles.yaml
- `--agents` `PATH`: agent roster instead of .factory/agents.yaml

### factory config roster show

```
factory config roster show [--repo PATH] [--json]
```

show effective local agent settings

- `--repo` `PATH`: repository (default: current git repo)
- `--json`: print machine-readable JSON

### factory config roster set

```
factory config roster set [preset] [--agent NAME] [--harness NAME] [--model ID] [--thinking LEVEL] [--dry-run] [--repo PATH] [--json]
```

Switch presets: codex (gpt-6.1-sol, medium) or claude (claude-opus-5-5, medium). Explicit flags override the preset. A whole-roster preset clears per-agent harness/model/thinking overrides. Other settings stay. Changes remain local until factory config commit; workflow step overrides win.

- `preset` (one of `codex`, `claude`)
- `--agent` `NAME`: change only this agent
- `--harness` `NAME`: explicit harness override
- `--model` `ID`: explicit model override
- `--thinking` `LEVEL`: explicit thinking override
- `--dry-run`: validate and show diff only
- `--repo` `PATH`: repository (default: current git repo)
- `--json`: print machine-readable JSON

### factory config status

```
factory config status [--json] [--base REF]
```

List uncommitted changes in .factory/ against the base commit. Runs read the committed configuration, so these changes are ignored by them.

- `--json`: print machine-readable JSON
- `--base` `REF`: base ref (default: `base` in .factory/config.yaml)

### factory config show

```
factory config show [--json] [--base REF]
```

Load the configuration from the base commit, as a run does, and show it.

- `--json`: print machine-readable JSON
- `--base` `REF`: base ref (default: `base` in .factory/config.yaml)

### factory config commit

```
factory config commit [--dry-run] [--pr] [--expect DIGEST] [-m TEXT] [--json] [--repo PATH]
```

Commit the uncommitted shared configuration (the paths of config status: config.yaml, agents.yaml, roles.yaml, prompts/, workflows/, extensions/ and the repo skills in .claude/skills/ and .agents/skills/; never local.yaml) to base in one commit, built on base without a checkout; other staged and unstaged work stays. --dry-run returns the plan: files {path, action, diff, content}, blockers and digest. --expect DIGEST refuses a plan that changed since (plan_changed). -m/--message TEXT sets the commit message. The direct target needs the main checkout on base (not_on_base) and no run in progress (run_in_progress); with a remote, base is fetched first (base_behind, base_diverged), the commit is pushed without force (push_failed changes nothing) and only then the local base is fast-forwarded. --pr pushes the commit to a branch factory-config/<n> and opens a pull request; base and the checkout stay. Without changes: committed false.

- `--dry-run`: only print the plan
- `--pr`: open a pull request instead of committing to base
- `--expect` `DIGEST`: commit only if the plan still has this digest
- `-m`, `--message` `TEXT`: commit message
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory config pull

```
factory config pull [--json] [--repo PATH]
```

Fetch the remote base and fast-forward the local base to it when no run is in progress (run_in_progress). A diverged base (base_diverged), uncommitted changes in the checkout on base (dirty_base) or no remote (no_remote) are refused and nothing changes.

- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory config set

```
factory config set type agent [--harness NAME] [--model ID] [--thinking LEVEL] [--tools LIST] [--writes LIST] [--color HEX] [--dry-run] [--commit] [--pr] [--expect DIGEST] [-m TEXT] [--json] [--repo PATH]
```

Change only the bindings of an agent (harness, model, thinking, tools, writes, color) by a round trip that keeps comments and key order; the prompts stay. An agent the repo inherits from the library gets an entry that overrides only these keys. --tools and --writes take a comma list ('' = empty list, '-' removes the key; writes expand $specs_dir/ and $docs_dir/); --color #rrggbb ('-' removes it). An unknown thinking level or harness is invalid_value, an agent neither in the repo nor in the library unknown_item. Modes: --dry-run returns the plan (files {path, action, diff, content}, removed, bindings, blockers, validation, digest) and writes nothing; without an option the plan is written to the working tree (nothing committed; factory config commit commits it later; --expect DIGEST of a --dry-run refuses a changed plan with plan_changed; a run in progress blocks with run_in_progress); --commit [--pr] [--expect DIGEST] [-m TEXT] commits the plan of the tree of base in one commit (pushed before base moves; --pr opens a pull request from factory-config/<n>), --commit --dry-run shows that plan. The planned configuration must load and every workflow of the repo must pass its preflight (invalid_plan). Blockers: run_in_progress, invalid_plan (working tree); dirty_paths, invalid_plan, not_on_base, run_in_progress, base_behind, base_diverged (--commit); dirty_paths, invalid_plan (--pr). Nothing to change: changed false, exit 0.

- `type` (required; one of `agent`)
- `agent`: agent name (required)
- `--harness` `NAME`: harness (claude, codex, pi)
- `--model` `ID`: model id
- `--thinking` `LEVEL`: thinking level
- `--tools` `LIST`: comma list of tools
- `--writes` `LIST`: comma list of writable paths
- `--color` `HEX`: dashboard color #rrggbb
- `--dry-run`: only print the plan
- `--commit`: commit to base in one commit (default: write)
- `--pr`: with --commit: open a pull request instead
- `--expect` `DIGEST`: commit only if the plan still has this digest
- `-m`, `--message` `TEXT`: commit message (--commit)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository (default: the current git repo)

### factory config remove

```
factory config remove type name [--dry-run] [--commit] [--pr] [--expect DIGEST] [-m TEXT] [--json] [--repo PATH]
```

Remove the repo copy of an agent (roster entry, prompts), workflow, skill (both .claude/skills/ and .agents/skills/) or extension; the library item of the same name then applies. Without one, refused with in_use (data.used_by) while a workflow of the repo or roles.yaml names the agent, a backlog task in base names the workflow, or a roster agent binds the skill or extension. Modes: --dry-run returns the plan (files {path, action, diff, content}, removed, bindings, blockers, validation, digest) and writes nothing; without an option the plan is written to the working tree (nothing committed; factory config commit commits it later; --expect DIGEST of a --dry-run refuses a changed plan with plan_changed; a run in progress blocks with run_in_progress); --commit [--pr] [--expect DIGEST] [-m TEXT] commits the plan of the tree of base in one commit (pushed before base moves; --pr opens a pull request from factory-config/<n>), --commit --dry-run shows that plan. The planned configuration must load and every workflow of the repo must pass its preflight (invalid_plan). Blockers: run_in_progress, invalid_plan (working tree); dirty_paths, invalid_plan, not_on_base, run_in_progress, base_behind, base_diverged (--commit); dirty_paths, invalid_plan (--pr). Nothing to change: changed false, exit 0.

- `type` (required; one of `agent`, `workflow`, `skill`, `extension`)
- `name`: item name in the repo (required)
- `--dry-run`: only print the plan
- `--commit`: commit to base in one commit (default: write)
- `--pr`: with --commit: open a pull request instead
- `--expect` `DIGEST`: commit only if the plan still has this digest
- `-m`, `--message` `TEXT`: commit message (--commit)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository (default: the current git repo)

### factory config move

```
factory config move type name [--dry-run] [--commit] [--pr] [--expect DIGEST] [-m TEXT] [--json] [--repo PATH]
```

Write the repo copy of an agent (roster purpose and prompts), workflow, skill or extension into the library under its name (create or replace), then delete it from the repo; the repo inherits it from the library from then on. A moved agent keeps the agent.yaml defaults of its library item with the repo purpose; a new one takes its defaults from the roster entry. Its roster entry keeps only the bindings (an override), an entry with nothing else is removed. Order: the library is written first (library.lock, fetch, library_behind, library_diverged, library_dirty, push without force: a rejected push is push_failed and the repo is not touched), then the repo. Without a library: library_missing. data.library_plan has the library files, data.library_commit the library commit. The digest covers the library and the repo plan. Modes: --dry-run returns the plan (files {path, action, diff, content}, removed, bindings, blockers, validation, digest) and writes nothing; without an option the plan is written to the working tree (nothing committed; factory config commit commits it later; --expect DIGEST of a --dry-run refuses a changed plan with plan_changed; a run in progress blocks with run_in_progress); --commit [--pr] [--expect DIGEST] [-m TEXT] commits the plan of the tree of base in one commit (pushed before base moves; --pr opens a pull request from factory-config/<n>), --commit --dry-run shows that plan. The planned configuration must load and every workflow of the repo must pass its preflight (invalid_plan). Blockers: run_in_progress, invalid_plan (working tree); dirty_paths, invalid_plan, not_on_base, run_in_progress, base_behind, base_diverged (--commit); dirty_paths, invalid_plan (--pr). Nothing to change: changed false, exit 0.

- `type` (required; one of `agent`, `workflow`, `skill`, `extension`)
- `name`: item name in the repo (required)
- `--dry-run`: only print the plan
- `--commit`: commit to base in one commit (default: write)
- `--pr`: with --commit: open a pull request instead
- `--expect` `DIGEST`: commit only if the plan still has this digest
- `-m`, `--message` `TEXT`: commit message (--commit)
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository (default: the current git repo)

### factory harness check

```
factory harness check [--json] [--config PATH]
```

Verify each harness CLI used by the config is on PATH and print versions.

- `--json`: print machine-readable JSON
- `--config` `PATH`: HAIFA roster (.factory/agents.yaml); only the harnesses it uses are checked

### factory skills sync

```
factory skills sync [--repo PATH] [--json]
```

Write .agents/skills/ as an exact copy of .claude/skills/ (new, changed and deleted skills), so codex and pi read the same skills as claude. Edit skills in .claude/skills/ only. Writes the working tree only; commit the result. No symlinks. Exits 0 on success, 2 when the folder is not a git repository or the mirror cannot be written (skills_sync_failed).

- `--repo` `PATH`: repository to sync (default: the current git repo)
- `--json`: print machine-readable JSON

### factory library init

```
factory library init [--name N] [--remote URL] [--json]
```

Create the library ($HAIFA_HOME/library, overridden by HAIFA_LIBRARY) as a git repository from the packaged seed in one commit: library.yaml (format, id, name, min_factory_version, seed) and the seed agents, workflows and the factory skill. An existing library is refused (library_exists); git needs user.name and user.email (git_identity_missing). With --remote URL the commit is pushed to the empty remote before the library exists locally; a remote with any ref is refused (remote_not_empty) and a failed push (push_failed) leaves no library. The remote is stored as origin without user and password.

- `--name` `N`: name of the library (default: library)
- `--remote` `URL`: share the library through this empty remote
- `--json`: print machine-readable JSON

### factory library list

```
factory library list [--type {agent,workflow,skill,extension}] [--json]
```

List the items in HEAD of the library: type, name, version, short_version (8 characters), and commit, date and author of the last change.

- `--type` `{agent,workflow,skill,extension}`: list only items of this type (one of `agent`, `workflow`, `skill`, `extension`)
- `--json`: print machine-readable JSON

### factory library show

```
factory library show TYP JMENO [--version V] [--json]
```

Show an item in HEAD of the library (or at --version, a full version or a prefix of at least 4 hex digits): files {path, executable, size, content} and history {n, version, short_version, commit, date, author}, oldest first. unknown_item, unknown_version.

- `TYP`: item type (required; one of `agent`, `workflow`, `skill`, `extension`)
- `JMENO`: item name (required)
- `--version` `V`: show this version of the item
- `--json`: print machine-readable JSON

### factory library import

```
factory library import CESTA --type {agent,workflow,skill,extension} [--name N] [--dry-run] [--json]
```

Import a folder (agent, skill, extension) or file (workflow) from under the user's home directory (else outside_home, also for a symlink that leaves it) into the library. The item is validated (invalid_item with error.issues) and the plan {items, files with diffs, digest} is returned. Equal content is a no-op (committed false); other content under an existing name is a new version. Without --dry-run the write holds $HAIFA_HOME/library.lock, refuses a library with uncommitted changes (library_dirty), commits on HEAD without a checkout and fast-forwards the branch and working tree. A library with a remote sends the change as a pull request (data.pull_request), never a push to its branch.

- `CESTA`: folder or file to import (required)
- `--type` `{agent,workflow,skill,extension}`: type of the item (required; one of `agent`, `workflow`, `skill`, `extension`)
- `--name` `N`: item name (default: folder name or file stem)
- `--dry-run`: only print the plan
- `--json`: print machine-readable JSON

### factory library clone

```
factory library clone [URL] [--seed-fallback] [--branch B] [--json]
```

Clone the library from URL into the library path and return its status. Without URL the default library is cloned: $HAIFA_LIBRARY_URL, else https://github.com/janbkrejci/HAIFA-LIBRARY.git. An existing library is refused (library_exists), a failed clone is clone_failed and a remote without a valid library.yaml (format, id, name) is invalid_library; nothing is left behind. The remote is stored as origin without user and password. With --seed-fallback a failed clone (no network, no access) creates a local library from the seed instead (data.fallback true, with a warning).

- `URL`: git remote of the library (default: the HAIFA library)
- `--seed-fallback`: when the clone fails, create a local library from the seed
- `--branch` `B`: branch to clone (default: the remote HEAD)
- `--json`: print machine-readable JSON

### factory library status

```
factory library status [--fetch] [--json]
```

Show path, id, name, remote (without credentials), branch, ahead and behind against origin (from the last fetched refs; --fetch fetches first), last_fetch, uncommitted changes and min_factory_version against the installed factory (compatible). Without a remote, ahead and behind are null. sync_state is unshared (no remote), local (changes not sent), pending (every local change waits in a pull request), behind or synced; pull_requests lists the open pull requests sent from this library (--fetch asks GitHub which are still open).

- `--fetch`: fetch the remote first
- `--json`: print machine-readable JSON

### factory library pull

```
factory library pull [--json]
```

Fetch origin and fast-forward the library. Local commits the remote lacks (sent in pull requests) are rebased onto it; the merged ones drop out. A library with uncommitted changes (library_dirty) or whose rebase conflicts (library_diverged) is refused and nothing changes; without a remote it is no_remote.

- `--json`: print machine-readable JSON

### factory library push

```
factory library push [--json]
```

Fetch origin and send the local commits that are in no open pull request as one pull request against the library branch; the branch itself is never pushed. On GitHub the pull request comes from a branch in the library repo when you may write to it, else from your fork (gh repo fork); other remotes get only the branch haifa-library/... HAIFA never merges or approves it. A library behind the remote (library_behind: run factory library pull) or diverged from it (library_diverged) is refused; a rejected push is push_failed, a failed gh call pr_failed.

- `--json`: print machine-readable JSON

### factory library seed

```
factory library seed [--all | --take TYP/JMENO] [--dry-run] [--json]
```

Compare every item of the installed seed with HEAD of the library and the seed version recorded in library.yaml (seed). Missing in the library: create. Library head equal to the recorded version: update to the new seed version. Library head changed by the team: kept, the plan shows the diff (items[].diff); --take TYP/JMENO takes the seed version anyway (take). Then seed in library.yaml is set to the installed seed. Items the seed no longer has are not touched. The write goes like import: library.lock, library_dirty, fetch (library_behind, library_diverged), commit, push, fast-forward. A second run without a new seed commits nothing (committed false). A --take item the seed lacks is not_in_seed. --all replaces every seed item with the installed seed version, also items the team changed (take); items the seed lacks and the repos' .factory/ stay. Install and upgrade run seed --all. --all with --take is usage_error.

- `--all`: replace every seed item with the installed seed version, also items the team changed; install and upgrade run this (exclusive: --all|--take)
- `--take` `TYP/JMENO`: take the seed version of this team-changed item (repeatable) (exclusive: --all|--take)
- `--dry-run`: only print the plan
- `--json`: print machine-readable JSON

### factory learn scan

```
factory learn scan [--since DATE] [--json] [--repo PATH]
```

Read the objections of every task run of the repo from the trace DB (blocking items and unmet findings of review and test review envelopes, test triage of failed checks, revise phases) and let the read-only agent 'learner' (prompts overridable in .factory/prompts/learner/) group similar ones into lessons, in an isolated checkout of the base commit. A lesson has a short rule, where it belongs (CLAUDE.md, AGENTS.md, a skill of the repo or the prompt of a role), its occurrences (the distinct runs), the runs and snippets of the objections. A lesson seen in one run only is not proposed, nor one a person accepted (factory learn accept) or rejected (factory learn dismiss). Proposed lessons are saved in the trace DB; nothing in the repo is written.

- `--since` `DATE`: only objections from this date (YYYY-MM-DD) or ISO timestamp on
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory learn accept

```
factory learn accept LESSON [--project ID] [--json] [--repo PATH]
```

File a lesson of 'factory learn scan' as a task in the step LEARN of the repo's project (the step is created when it is missing): workflow build-review, writes on the lesson's target only (CLAUDE.md, AGENTS.md, .claude/skills/<name>/ and .agents/skills/<name>/, or .factory/prompts/<role>/) and the rule with the objections of the runs as evidence. The backlog change is committed to base; the task then runs the ordinary way up to a pull request. The factory writes no rule itself. The project is --project, else the one of the tasks whose runs objected, else the only project. Accepting the lesson again shows its task and files nothing new.

- `LESSON`: lesson id from 'factory learn scan' (required)
- `--project` `ID`: project whose LEARN step gets the task
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory learn dismiss

```
factory learn dismiss LESSON [--reason TEXT] [--json] [--repo PATH]
```

Mark a lesson of 'factory learn scan' rejected in the trace DB; later scans do not offer it (nor its objections) again.

- `LESSON`: lesson id from 'factory learn scan' (required)
- `--reason` `TEXT`: why the lesson is rejected
- `--json`: print machine-readable JSON
- `--repo` `PATH`: repository root with .factory/config.yaml (default: the current git repo)

### factory upgrade

```
factory upgrade BUNDLE [--dry-run] [--json]
```

Install a newer HAIFA from a bundle haifa-<version>.zip (a path, or an https or file URL; other schemes are invalid_url). The bundle is unpacked into a temporary directory, SHA256SUMS is verified (checksum_mismatch), the version is read from the wheel and compared with the installed one (up_to_date when not newer; a broken zip or a missing file is bundle_invalid). Then uv tool install --force <wheel> --constraints constraints.txt runs in the bundle (upgrade_failed, uv_missing). An editable install is refused (editable_install): update it with git pull. Restart a running dashboard afterwards. On Windows the running factory cannot replace its own files: the install runs from a detached script after this command exits (handed_off, with its log), and a running dashboard is refused (dashboard_running). After the install the new factory refreshes the library: library clone --seed-fallback when it is missing, then library seed --all (library_refreshed); a failure is a warning.

- `BUNDLE`: path or https/file URL of the zip (required)
- `--dry-run`: check the bundle and show the versions and the command; install nothing
- `--json`: print machine-readable JSON

### factory obs

```
factory obs [--repo PATH] [--port N] [--host ADDR] [--no-open] [--json]
```

serve the local dashboard (127.0.0.1 unless --host) and open it

- `--repo` `PATH`: register this repository and open it (any folder inside it): its backlog, or its Factory tab when the factory is not ready
- `--port` `N`: port (default: port in the registry dashboard.yaml, else 4700)
- `--host` `ADDR`: address to listen on (default: $HAIFA_DASH_HOST, else 127.0.0.1); 0.0.0.0 serves the whole network, with no login
- `--no-open`: do not open the browser
- `--json`: print the URL as a JSON envelope before serving

## Error codes

`error.code`, the exit code it comes with, and what it means:

| code | exit | meaning |
| --- | --- | --- |
| `backlog_invalid` | 1 | the backlog has problems (backlog check) or the write would break it (nothing written); see error.issues |
| `checks_failed` | 1 | factory check found errors; see data.findings |
| `harness_missing` | 1 | a harness CLI is missing or broken; see data.harnesses |
| `learn_failed` | 1 | learn scan: the agent reported that it cannot judge the objections |
| `lessons_invalid` | 1 | learn scan: the lessons stayed invalid after one correction (nothing saved) |
| `pr_failed` | 1 | the run succeeded but push or pull request failed (task run, task publish); see data.pr_error and retry with factory task publish |
| `proposal_failed` | 1 | backlog propose: the agent reported that it cannot propose |
| `proposal_invalid` | 1 | backlog propose: the proposal stayed invalid after one correction (nothing written); see error.issues |
| `run_failed` | 1 | the run ended failed or accept was not met; worktree and branch kept |
| `up_to_date` | 1 | upgrade: the bundle's version is not newer than the installed one; nothing installed |
| `update_busy` | 1 | dashboard update is installing; wait for its restart |
| `update_unavailable` | 1 | no newer installable release is available |
| `upgrade_failed` | 1 | upgrade: uv tool install failed (data.stdout, data.stderr); the old version stays |
| `workflow_invalid` | 1 | workflow check found problems; see error.issues |
| `missing_backlog_dir` | 1|2 | the backlog directory <docs_root>/backlog is missing where it cannot be created (otherwise it is created silently; 1 for backlog list/check, 2 for task commands) |
| `agent_unavailable` | 2 | the agent is missing from the roster or its harness is off |
| `already_installed` | 2 | factory init: .factory/config.yaml is committed |
| `already_running` | 2 | a run of this task is in progress |
| `approve_failed` | 2 | host approval failed or the pull request head changed; merge stopped |
| `az_devops_missing` | 2 | the az devops extension is not installed |
| `az_failed` | 2 | an az command failed |
| `az_missing` | 2 | the az CLI is not installed |
| `az_not_logged_in` | 2 | az is not logged in |
| `az_timeout` | 2 | an Azure CLI command exceeded its timeout; retry the operation |
| `bare_repo` | 2 | dashboard API: a bare repository has no working tree (HTTP 422) |
| `base_behind` | 2 | the remote base has commits the local base lacks (run factory config pull) |
| `base_diverged` | 2 | the local base and the remote base have diverged; nothing changed |
| `base_moved` | 2 | base moved on the remote during the operation |
| `branch_checked_out` | 2 | the branch is checked out in another worktree |
| `bundle_invalid` | 2 | upgrade: the bundle is missing, not a zip or lacks a file |
| `busy` | 2 | dashboard API: another apply or base pull runs in this repository, or another global library write is running (HTTP 409) |
| `chain_running` | 2 | the chain is still running and cannot be dismissed |
| `checksum_mismatch` | 2 | upgrade: a file of the bundle differs from SHA256SUMS |
| `clone_failed` | 2 | library clone: git clone failed |
| `commit_failed` | 2 | backlog/config commit or a library write: the commit failed |
| `config_not_committed` | 2 | factory init --commit: .factory/config.yaml is only in the working tree; run factory config commit |
| `conflict` | 2 | the pull request conflicts with base (run factory task resolve) |
| `conflicting_options` | 2 | two options that exclude each other were given |
| `cross_origin` | 2 | dashboard API: a write from another origin (Origin or Sec-Fetch-Site) was rejected (HTTP 403) |
| `dashboard_running` | 2 | upgrade on Windows: the running dashboard locks factory |
| `dirty_base` | 2 | the base checkout has uncommitted changes |
| `dirty_paths` | 2 | init --commit and config set/remove/move --commit: planned paths exist in the working tree with other content |
| `dirty_worktree` | 2 | the worktree has uncommitted changes |
| `download_failed` | 2 | upgrade: the bundle URL could not be downloaded |
| `duplicate_id` | 2 | the id is ambiguous or already used in the backlog |
| `editable_install` | 2 | upgrade: factory is an editable install from the repo; git pull |
| `existing_config` | 2 | factory init: the repo has an sssf installation (adws/adw_sssf_config/); HAIFA does not take it over |
| `fetch_failed` | 2 | git fetch failed |
| `file_exists` | 2 | the task file or the project/step directory already exists |
| `gh_failed` | 2 | a gh command failed |
| `gh_missing` | 2 | the gh CLI is not installed |
| `git_identity_missing` | 2 | git has no user.name or user.email |
| `human_task` | 2 | the task is for a person (assignee: human); mark it done when it is |
| `in_use` | 2 | config remove: the library lacks the item and a workflow, roles.yaml, an agent or a backlog task in base uses it; see data.used_by |
| `internal_error` | 2 | an unexpected exception; the traceback is on stderr |
| `invalid_config` | 2 | the configuration (.factory/, roster, provider) is invalid |
| `invalid_id` | 2 | the id does not match ^[A-Za-z0-9][A-Za-z0-9._-]*$ |
| `invalid_item` | 2 | library import: the item does not validate; see error.issues |
| `invalid_library` | 2 | library clone: the remote has no commit or no valid library.yaml (format, id, name) |
| `invalid_override` | 2 | task run --harness/--model/--thinking: unknown harness or thinking, or a model the harness rejects |
| `invalid_plan` | 2 | init --commit and config set/remove/move: the planned configuration does not load or a workflow fails its preflight; see error.issues |
| `invalid_roles` | 2 | the role registry cannot be loaded; see error.issues |
| `invalid_status` | 2 | the status (or status filter) is not allowed |
| `invalid_strategy` | 2 | merge_strategy is not squash or merge |
| `invalid_test_timeout` | 2 | the task's test_timeout (own or inherited) is not a whole number of seconds above 0 |
| `invalid_url` | 2 | upgrade: the bundle URL is neither https nor file |
| `invalid_value` | 2 | an argument value is not allowed (empty title, bad path, ...) |
| `invalid_workflow` | 2 | the task's workflow does not validate |
| `library_behind` | 2 | the library is behind its remote (run factory library pull); nothing changed |
| `library_dirty` | 2 | the library has uncommitted changes; commit or discard them |
| `library_diverged` | 2 | the library and its remote have diverged; nothing changed, nothing is forced |
| `library_exists` | 2 | library init: the library directory already exists |
| `library_missing` | 2 | there is no library yet (run factory library init); config move needs one |
| `linked_worktree` | 2 | dashboard API: a linked worktree; add the main checkout instead (HTTP 422) |
| `merge_failed` | 2 | the hosting did not merge the pull request |
| `missing_note` | 2 | task return needs a non-empty --note |
| `no_changes` | 2 | the command was given nothing to change |
| `no_commits` | 2 | dashboard API: the repository has no commit yet (HTTP 422) |
| `no_index` | 2 | the container has no index.md to write |
| `no_pr` | 2 | the task has no pull request |
| `no_remote` | 2 | config pull or library status/pull/push: no remote is configured |
| `no_succeeded_run` | 2 | task publish: the task has no succeeded run to publish |
| `no_workflow` | 2 | the task has no workflow (none set nor inherited) |
| `no_writes` | 2 | the task has no writes |
| `not_a_directory` | 2 | dashboard API: the path is not a directory (HTTP 422) |
| `not_a_repository` | 2 | the folder is not a git repository |
| `not_done` | 2 | task follow-up: only a done task can be followed up |
| `not_found` | 2 | dashboard API: no such endpoint (HTTP 404) |
| `not_git` | 2 | dashboard API: the folder is not in a git repository; HAIFA runs no git init |
| `not_in_seed` | 2 | library seed --take: the installed seed has no such item |
| `not_nothing_to_do` | 2 | dashboard Hotovo of a run: the run did not fail with nothing_to_do |
| `not_on_base` | 2 | backlog/config commit: the main checkout is not on the base branch |
| `not_writes_violation` | 2 | task retry/approve --allow-writes: the run did not fail with writes_violation or writes_needed (approve: the newest run is not a ci-fix run) |
| `nothing_to_allow` | 2 | task retry --allow-writes: no path writes can allow (protected, backlog, main checkout, or allowed already) |
| `outside_backlog` | 2 | the file is outside backlog_dir |
| `outside_home` | 2 | dashboard API: the folder is outside the user's home once symlinks are resolved (HTTP 403) |
| `path_not_found` | 2 | dashboard API: the path does not exist (HTTP 404) |
| `permission_denied` | 2 | dashboard API: the folder cannot be read (HTTP 403) |
| `picker_busy` | 2 | dashboard API: a folder dialog is already open (HTTP 409) |
| `picker_failed` | 2 | dashboard API: the folder dialog program failed (HTTP 502) |
| `picker_timeout` | 2 | dashboard API: the folder dialog was not answered in time (HTTP 504) |
| `picker_unavailable` | 2 | dashboard API: no graphical session or dialog program for a folder dialog (HTTP 503) |
| `plan_changed` | 2 | config commit/set/remove/move or init --expect: the plan's digest differs (move: also when the library HEAD moved); global library API plan changed; review the plan again |
| `port_in_use` | 2 | the dashboard port is already taken (factory obs --port N) |
| `pr_exists` | 2 | task publish: the task (or the run's branch) already has a pull request |
| `pr_not_open` | 2 | the task's pull request is not open |
| `project_required` | 2 | learn accept: the project cannot be told from the lesson's runs; give --project |
| `pull_failed` | 2 | config pull or library pull: git could not move the branch |
| `push_failed` | 2 | git push failed |
| `registry_invalid` | 2 | dashboard.yaml in the HAIFA home is invalid; fix it by hand, it is not overwritten |
| `remote_backlog_conflict` | 2 | local and shared backlog edits conflict, or the shared snapshot changed; preserve local edits, reload the backlog and retry |
| `remote_exists` | 2 | library remote: the library is already shared through a remote |
| `remote_not_empty` | 2 | library init --remote: the remote already has refs |
| `remote_run` | 2 | stop the shared run on the machine that started it |
| `repo_io_error` | 2 | a registered repository could not be read or written |
| `repo_missing` | 2 | dashboard API: the registered repository's folder is gone (HTTP 404) |
| `run_already_paused` | 2 | the run is already pausing or paused (factory task pause) |
| `run_in_progress` | 2 | a run with a live process is in progress in the repository (blocks base moves and config set/remove/move) |
| `run_not_archived` | 2 | only an archived run can be deleted |
| `run_not_paused` | 2 | the run is not paused, there is nothing to resume |
| `run_not_retryable` | 2 | task retry: the run is not failed, stopped or aborted, or it is a resolve or ci-fix run (after a ci-fix writes_needed: task approve --allow-writes) |
| `run_not_running` | 2 | the run is not running (finished, stopped or aborted) |
| `run_running` | 2 | the run is still running and cannot be archived |
| `run_worktree` | 2 | dashboard API: the path is a worktree of a task run (HTTP 422) |
| `self_ref` | 2 | a task cannot reference itself |
| `skills_sync_failed` | 2 | factory skills sync could not write .agents/skills |
| `stacked_on_open_pr` | 2 | the PR builds on a dependency's open PR; it is rebased onto base after that merge |
| `sync_on_base` | 2 | backlog sync refused to commit on the base branch |
| `task_blocked` | 2 | the task has unmet depends_on (a blocker a run reported); finish them first |
| `task_done` | 2 | task retry: the task's PR is merged or the task is done in base |
| `task_not_in_base` | 2 | the task is not committed in base |
| `task_running` | 2 | dashboard API: a running task cannot be edited or linked |
| `trace_db_locked` | 2 | the trace database is locked by another process |
| `trace_db_shared` | 2 | dashboard API: another registered repository uses the same trace DB (HTTP 409) |
| `unknown_base` | 2 | the base branch does not exist |
| `unknown_branch` | 2 | the branch does not exist |
| `unknown_chain` | 2 | no auto-continue chain with this id in the trace database |
| `unknown_container` | 2 | no project or step with this id |
| `unknown_intent` | 2 | dashboard API: the intent proposal expired or was closed |
| `unknown_item` | 2 | no item of this type and name in the library (config remove/move: no own copy in the repo; config set: the agent is neither in the repo nor in the library) |
| `unknown_lesson` | 2 | learn accept/dismiss: no lesson with this id |
| `unknown_phase` | 2 | no phase with this id in the run |
| `unknown_project` | 2 | no project with this id or directory |
| `unknown_repo` | 2 | dashboard API: no registered repository has this id (HTTP 404) |
| `unknown_run` | 2 | no run with this id in the trace database |
| `unknown_step` | 2 | no step with this id |
| `unknown_task` | 2 | no task with this id |
| `unknown_version` | 2 | the item has no such version in its history (or it is ambiguous) |
| `unknown_workflow` | 2 | the workflow is neither in .factory/workflows in base nor in the library (the seed without a library) |
| `unknown_workflow_advice` | 2 | dashboard API: workflow advice expired or not found |
| `unmet_dependencies` | 2 | depends_on are not done in base (use --force) |
| `unsupported_media_type` | 2 | dashboard API: a write body that is not application/json (HTTP 415) |
| `usage_error` | 2 | the arguments were rejected (unknown option, missing argument) |
| `uv_missing` | 2 | upgrade: uv is not on PATH |
| `worktree_failed` | 2 | the git worktree could not be created or used |
| `worktree_remove_failed` | 2 | a worktree could not be removed |
| `write_failed` | 2 | the task file could not be written |
| `writes_in_pr` | 2 | task retry --allow-writes: the writes change waits in an unmerged backlog PR |
| `writes_needed` | 2 | task approve: a ci-fix run needs paths outside the task's writes (listed); the PR stays open; task approve --allow-writes adds them and approves again |
| `writes_not_committed` | 2 | task retry --allow-writes: base lacks the new writes; commit the backlog, then retry |

Codes that appear in `error.issues[].code` (and in `data.issues[].code`), by source
(`check`: `data.findings[].code` of `factory check`; `item_origin`: the `origin` of an
agent, workflow, skill or extension in the dashboard, `repo` or `library`):

- backlog: `invalid_frontmatter`, `missing_field`, `invalid_field`, `missing_backlog_dir`, `misplaced_file`, `missing_index`, `misplaced_dir`, `invalid_status`, `duplicate_id`, `unknown_ref`, `id_prefix`, `cycle`
- workflow: `unknown_agent`, `invalid_agent`, `missing_file`, `bad_condition`, `bad_description`, `invalid_id`, `duplicate_id`, `invalid_step`, `unknown_key`, `missing_max`, `invalid_max`, `empty_repeat`, `unknown_step`, `unknown_harness`, `invalid_thinking`, `invalid_model`, `invalid_input`, `override_on_code_step`, `missing_id`, `missing_argv`, `invalid_timeout`, `test_without_plan`, `invalid_coverage`, `unknown_ref`, `unknown_field`, `not_a_mapping`, `missing_name`, `missing_steps`, `invalid_yaml`
- roles: `missing_output_type`, `unknown_output_type`, `invalid_output_type`, `invalid_gates`, `unknown_gate`, `bad_description`, `invalid_role`, `unknown_key`, `missing_agent`, `invalid_retries`, `invalid_aliases`, `unknown_code_step`, `invalid_code_step`, `missing_owner`, `not_a_mapping`, `missing_roles`, `missing_code_steps`, `missing_code_step`, `alias_conflict`, `missing_key`, `missing_file`, `invalid_yaml`
- proposal: `invalid_value`, `invalid_id`, `duplicate_id`, `self_ref`, `file_exists`, `unknown_workflow`, `missing_done`
- config: `invalid_config`
- item_origin: `repo`, `library`
- repo_state: `installed`, `unsupported`, `uncommitted`, `none`
- repo_action: `config_commit`, `init`
- library_seed: `create`, `update`, `unchanged`, `kept`, `take`
- library_status: `seed_update_available`
- library_sync: `unshared`, `local`, `pending`, `behind`, `synced`
- startup: `env_file_mode`
- library: `invalid_type`, `invalid_name`, `missing_item`, `missing_file`, `not_a_file`, `symlink`, `too_many_files`, `too_large`, `not_utf8`, `invalid_yaml`, `not_a_mapping`, `unknown_key`, `missing_purpose`, `invalid_defaults`, `unexpected_file`, `missing_skill_md`, `invalid_front_matter`, `name_mismatch`, `missing_description`, `missing_entry`
- check: `factory_missing`, `config_not_committed`, `repo_unsupported`, `repo_installed`, `base_missing`, `checkout_not_on_base`, `remote_missing`, `remote_base_missing`, `base_behind_remote`, `base_ahead_of_remote`, `base_config_invalid`, `worktree_config_invalid`, `config_uncommitted`, `base_setting_mismatch`, `backlog_missing`, `backlog_invalid`, `workflow_unset`, `workflow_unknown`, `workflow_invalid`, `gitignore_missing`, `skill_mirror_missing`, `skill_mirror_differs`, `skill_mirror_extra`, `harness_missing`, `disallowed_commands_unenforced`, `local_config_invalid`, `local_port_ignored`, `gh_missing`, `gh_login`, `az_missing`, `az_devops_missing`, `az_login`, `hosting_skipped`, `codex_not_isolated`, `pi_not_isolated`, `unsupported_platform`, `git_missing`, `git_identity_missing`, `uv_missing`, `node_missing`, `harness_login`, `pi_model_unknown`, `env_file_mode`, `env_override`, `factory_outdated`, `library_missing`, `library_dirty`, `library_behind`, `library_unpushed`, `seed_update_available`, `roles_full_copy`, `unknown_thinking`, `check_failed`

What to do with the common ones:

- `unmet_dependencies`: wait until the dependencies are done in base, or pass `--force`.
- `conflict`: run `factory task resolve ID --json`, then approve again.
- `backlog_invalid`: read `error.issues`, fix the files, run `factory backlog check --json`
  until `ok` is true.
- `already_running`: another run of the task is in progress; wait for it.
- `run_already_paused`, `run_not_paused`: see Procedures, Pause, resume and stop.
- `pr_failed`: the run succeeded and its branch exists; `data.pr_error` says why. Check the
  remote and the provider CLI (`gh`, `az`), then `factory task publish ID --json` (see
  Publish). A push that failed on the network was already retried a few times.
- `run_failed`: read `data.run.failure_kind` (`accept_not_met`, `nothing_to_do`, `quota`,
  `writes_violation`, `writes_needed`, `invalid_output`, `process_gone`, `stopped`,
  `blocked`, `other`), its
  facts in `data.run.failure_detail` and the text in `data.run.error`; `factory task runs ID
  --json` and `factory task show ID --json` give them for every run (a run recorded before
  the field reads as `other`, or `blocked` when its error starts with `blocked: `).
  `blocked` is no failure: an agent reported a blocking follow-up or task, the detail's
  `blockers` lists them; run them first, then retry. For `accept_not_met` the detail holds `accept`, the `values`
  of the fields it reads, the `rounds` of each `repeat`, the latest review `objections`
  and the test's `failed_checks`; `error` names the first of them in one sentence. The worktree and branch are kept. Continue the run
  where it ended with `factory task retry ID --note TEXT --json` (see Procedures, Retry), or
  start over with `factory task run ID --note TEXT --json`. A `nothing_to_do` run (the
  task asks only for a change of the backlog) is not retried: make that change with
  `factory task edit`, then the dashboard's Hotovo (`POST /api/runs/{id}/done`) marks the
  task done without changes (refused: `not_nothing_to_do`), or cancel the task. A builder
  that finds the work done already reports `already_done` with evidence; that run
  succeeds and the task is done without changes, with no PR.
- `run_not_retryable`, `task_done`: see Procedures, Retry.
- `invalid_config`: read `error.issues[].path` and `message`, fix `.factory/`, commit.
- `usage_error`: the arguments are wrong; see the command reference above.
- `checks_failed`: `factory check` found errors; read `data.findings` (see Procedures,
  Check), fix the `error` findings and run the check again.

## Backlog format

The factory documents live under one directory, `docs_root` in `.factory/config.yaml`
(default `factory`). It holds the backlog, the task specs and the task documentation, all
structured by project and step:

```
<docs_root>/backlog/<project>/<step>/<task>.md
<docs_root>/specs/<project>/<step>/<task-id>-<slug>.md
<docs_root>/docs/<project>/<step>/<task-id>-<slug>.md
```

- `<project>/<step>` are the directory names of the task's project and step in the backlog.
  `docs_root: .` puts `backlog/`, `specs/` and `docs/` at the repository root.
- Old keys `backlog_dir`, `backlog_dirs`, `specs_dir` and `docs_dir` in
  `.factory/config.yaml` and `specs_dir`, `docs_dir` and `workdir` in `index.md` or a task
  are ignored; saving the settings in the dashboard drops them from `config.yaml`.
- The levels come from `levels` (default `[project, step, task]`). Tasks live only in the
  directories of the second-to-last level (steps by default).
- Every container directory (project, step) has `index.md` with a YAML header:
  `id` and `title` are required; the optional inherited defaults are
  `harness, model, thinking, test_timeout, workflow, writes, auto_continue, auto_merge, auto_heal, quota_switch, quota_switch_back, assignee`. Tasks inherit them; the nearest level wins. A markdown body with the
  description follows the header.
- A task is one file named `<id>-<slug>.md` (slug: ASCII lowercase, e.g.
  `M01-S02-T01-endpoint.md`).

### Task header

YAML front matter; HAIFA writes the keys in this order: `id, title, status, assignee, workflow, depends_on, related, follows, writes, test_timeout, auto_continue, auto_merge, auto_heal`.

- `id`, `title` and `status` are required.
- `status` is one of `todo, done, cancelled`. Set `todo` or `cancelled` by hand
  (`factory task edit`); `done` is set only by `factory task approve` (or `backlog sync`).
- `depends_on, related, writes` are lists of strings.
- `depends_on` may reference tasks or steps (a step is done when all its tasks are done).
  Store only `depends_on`; "blocks" is computed. `related` is informational.
- `follows` is the id of the done task this one continues (a task, never inherited).
  `factory task follow-up ID --body TEXT [--title TEXT] [--run] --json` creates it in the
  step of ID with ID's own `writes` and `workflow`; a task that is not done is refused
  (`not_done`). `task show` lists the tasks that follow a task (`followed_by`).
- `writes` is the list of paths or globs the agents may change; anything else is rolled back.
  Own or inherited; when no level sets it, the whole repo (`**`), protected files excepted.
  An explicit empty list `[]` cannot run (`no_writes`).
- `workflow` names the workflow (own or inherited); when no level sets it, `simple-sdlc`.
  An explicit `null` means none (`no_workflow`).
- `test_timeout` is the shared time limit of the checks a workflow's `test` step runs, in
  whole seconds greater than 0 (own, or the nearest `index.md` above); without it `test_timeout` from
  `.factory/config.yaml`, else 600. An invalid value is reported by `factory backlog check`
  and stops `factory task run` with `invalid_test_timeout`.

The body has two sections:

- `## Zadání`: what to do, where, "done means", out of scope, hard constraints.
- `## Běhy` with `<!-- doplňuje HAIFA při schválení PR -->`; HAIFA adds one line per approved PR.

Ids match `^[A-Za-z0-9][A-Za-z0-9._-]*$`, are unique across the backlog and start with the
parent id plus `-` (`M01` -> `M01-S02` -> `M01-S02-T01`). The default task id is
`<step>-T<NN>` (next free number).

States: the file stores `todo, done, cancelled`; `ready` and `blocked` are derived from
`depends_on` for `todo` tasks; running, failed and in review live in the trace DB and the
hosting. List filters (`--status`): `todo, done, cancelled, ready, blocked, human`.

Example `index.md` of a project (its id is any short code, e.g. `M01` or `HAIFA`):

```markdown
---
id: M01
title: Core
workflow: plan-build-test
writes: [src/]
auto_continue: false
auto_merge: false
---

Core: the data model and the API.
```

Example `index.md` of a step:

```markdown
---
id: M01-S02
title: API
---

Read-only HTTP API over the model.
```

Example task file `backlog/M01-core/S02-api/M01-S02-T01-endpoint.md`:

```markdown
---
id: M01-S02-T01
title: Endpoint
status: todo
workflow: plan-build-test
depends_on: [M01-S01]
related: [M02-S01-T01]
writes: [src/api/, tests/api/]
---

## Zadání
Add GET /model returning the model as JSON.

Where: `src/api/`, `tests/api/`.

Done means:
- GET /model returns 200 with the model; a test covers it.
- The suite passes.

Out of scope: writing endpoints.

Hard constraints: do not change `src/model/`.

## Běhy
<!-- doplňuje HAIFA při schválení PR -->
```

### `.factory/config.yaml`

Shared settings are committed in `.factory/config.yaml`. Create them through
`factory init`,
review the detected base and provider, then edit the YAML to fit
this repository. Validate with `factory check --json` and
`factory backlog check --json`; review and commit using Commit configuration.
Runs read these files from base. Machine-local `trace_db` and `database_url`
belong in `.factory/local.yaml`, not here; the dashboard port is in its registry.

Every key, including nested entries, is generated from the configuration model.
`null` means unset, `[]` an empty list, and `required` has no default:

| Key | Default | Meaning |
| --- | --- | --- |
| `workdir` | `"."` | Working directory relative to the repository. |
| `docs_root` | `"factory"` | Root of the factory documents: backlog/, specs/ and docs/ under it, each structured project/step/task. |
| `levels` | `["project", "step", "task"]` | Ordered backlog hierarchy; the last level contains tasks. |
| `worktrees_dir` | `".factory/worktrees"` | Directory for isolated task worktrees, relative to the repository. |
| `base` | `"main"` | Base branch whose committed configuration and backlog runs read. |
| `remote` | `"origin"` | Git remote used for publishing and synchronizing base. |
| `git_provider` | `"local"` | Hosting provider: local (no PR hosting), github (gh), or azure (az). |
| `merge_strategy` | `"squash"` | Strategy for merging approved task pull requests. |
| `base_changes` | `"direct"` | How factory changes base (install, configuration, library items, backlog, uninstall): direct commits pushed to base, or pr, one pull request per change for a base that takes changes only through pull requests. |
| `base_pr_merge` | `"manual"` | Pull requests of factory's own base changes: manual leaves them to a reviewer; auto approves them where the hosting lets the author (Azure DevOps), merges them at once (GitHub with --admin when the branch protection requires it) and fast-forwards the local base. |
| `github_approver` | `null` | GitHub login that sends the approve review when a task PR is approved (git_provider github). It must be another account than the PR author, logged in to gh on the machine (gh auth login). Unset: no review is sent. |
| `test_timeout` | `null` | Shared time limit in positive whole seconds of the checks a test step runs; null means 600. Inherited backlog test_timeout overrides it. |
| `test_slots` | `1` | Maximum simultaneous test steps on this machine; others queue without consuming their timeout. |
| `stacked_prs` | `false` | A task whose dependencies wait only in open pull requests starts from the dependency's PR branch; its PR is rebased onto base and merged after theirs. |
| `max_parallel_runs` | `1` | Maximum concurrent task runs within one auto-continue chain. |
| `protected_files` | `[".factory/"]` | Paths protected from agents unless explicitly allowed by their writes; task scope still applies. |
| `generated` | `[]` | Tracked build outputs and rebuild commands used when resolving conflicts. |
| `azure` | `null` | Azure DevOps connection details; required when git_provider is azure. |
| `azure.organization` | `required` | Azure DevOps organization name or full URL. |
| `azure.project` | `required` | Azure DevOps project name. |
| `azure.repository` | `required` | Azure DevOps repository name. |
| `generated[].path` | `required` | Generated file or directory relative to the repository. |
| `generated[].command` | `required` | Command run from the repository root to rebuild this output after a resolve conflict. |
| `generated[].timeout` | `600` | Rebuild time limit in positive whole seconds. |

### Agents and prompts

Agents, workflows, skills and extensions are looked up in the repo first and in the
library second: `$HAIFA_HOME/library` at its HEAD, or the seed in the package when there
is no library (never both). A copy in the repo wins. A repo without `.factory/agents.yaml`
and `.factory/prompts/` runs with the library agents and their default bindings; the
roster `defaults` still apply to them. The library commit a run read is recorded in
`task_runs.library_commit` of the trace DB (null for the seed).

`.factory/agents.yaml` contains a `defaults` mapping and an `agents` list.
Each agent has a unique `name`. The name identifies its prompts and is
what a role's `agent` or a workflow step's `agent` selects. An agent with its prompts in
`.factory/prompts/<name>/` is the repo's own copy and takes nothing from the library. An
entry without prompts there overrides only the keys it sets of the library agent of that
name (for example its `harness` and `model`); prompts, purpose and the rest come from the
library.

| Key | Meaning |
| --- | --- |
| `name` | Required slot name in each agent entry. |
| `harness` | Explicit harness: `claude`, `codex`, `pi`; aliases `claude_code` = `claude`; never inferred from model. May be inherited from defaults. |
| `coding_agent` | Compatibility alias for harness; conflicting values are rejected. |
| `model` | Model accepted by the selected harness; may be inherited. |
| `thinking` | Reasoning level: `off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`, `ultra`; may be inherited. |
| `purpose` | Agent role description. |
| `color` | Dashboard lane color; may be inherited. |
| `tools` | Tool allowlist; null allows all tools supported by the harness. May be inherited. |
| `writes` | Allowed paths/globs: null unrestricted except protections, [] read-only. Task scope also applies. May be inherited. |
| `disallowed_commands` | Shell command patterns denied by the harness; may be inherited. Unsupported enforcement is reported. |
| `harness_engineering` | Paths to pi extensions; may be inherited, rejected for other harnesses. |

An agent the library does not have requires both `.factory/prompts/<name>/system.md` and
`.factory/prompts/<name>/user.md`; `prompt_engineering` is forbidden in this
roster because the paths are fixed. Put role instructions in system.md and the
request plus expected output in user.md. Variables are replaced literally using
double braces around their names:

| Variable | Value |
| --- | --- |
| `prompt` | Task header, allowed paths, output paths, assignment and operator note. |
| `previous_envelope` | Previous agent's Report JSON, or `(none)`. |
| `context_handoff_dir` | Absolute session handoff directory. |
| `task_id` | Current task ID. |
| `spec_path`, `doc_path` | Specification and documentation paths relative to the worktree. |
| `workdir` | Absolute task worktree path. |
| `test_result` | Latest test results as JSON. |
| `baseline` | Commit the run started from; `git diff <baseline>` is the whole change. |
| `previous_test_plan` | The plan the last test ran, `(none)` before the first. |
| `failed_test` | In a tester triage: the red test to judge; otherwise `(none)`. |
| `rebase_onto` | Target commit, only in conflict resolution runs. |

`<context_handoff_dir>` is an absolute path outside the repo. Write handoff files at exactly that path; never create a directory of the same name inside the repo or the worktree.

A workflow role step's `input` mapping supplies additional named variables with
JSON results of the selected earlier steps. Engine variables take precedence on
a collision; unknown variables remain unchanged. A workflow `test` step executes
code, not an agent prompt: it runs the checks the latest `test_plan` step chose.
Timeout and slots follow the Workflow format rules below.

To change a library agent for one repo, copy its prompts into
`.factory/prompts/<name>/` and edit them there. To share the repo's copy with every repo,
move it to the library: `factory config move agent <name> --commit --dry-run --json`
previews it, `factory config move agent <name> --commit --expect <digest> --json` writes
the library and deletes the copy from the repo in one commit (see Items of the repo).


## Workflow format

A workflow is YAML in `.factory/workflows/<name>.yaml` (read from base), falling back to the
library (without a library the packaged workflows: `build` (Implement the task in one shot and commit it, without tests or review); `build-review` (Build straight from the task, then have a reviewer confirm it is what was asked for, with revisions); `build-test` (Build straight from the task, repair until the checks are green, and commit only green code); `build-test-review` (Build straight from the task, test with repairs, review with revision — no separate plan or write-up); `ci-fix` (Repair what the PR's red CI checks reported, test, review, commit to the same PR); `document` (Write up a finished change from its diff, which code captures before the documenter reads it); `heal` (Run the whole suite and repair until it is green — each repair checked on what it touched, the end confirmed on the whole suite); `plan` (Turn a request into a written plan and stop there, so a human can read it first); `plan-build` (Plan the change, implement it and commit the result without running tests); `plan-build-test` (Plan and build, repair until the suite is green, and commit only green code); `plan-build-test-quality` (Plan and build, then lint, typecheck, build and test the whole repo with repairs, and commit only verified code); `quality` (Lint, typecheck, build and test the whole repo without changing it); `resolve` (Bring a conflicting pull request up to date with base and prove the suite is still green); `resolve-reviewed` (Resolve like `resolve`, then review a resolution the agent wrote; auto-merge resolves a conflicting PR with it); `scout` (Read-only recon that reports where things live without changing the repo); `simple-sdlc` (Plan, build, test with repairs, review with revision, then document — committing each work product separately)).

- Top-level keys: `name` (required), `description`, `steps` (required), `accept`.
- A step is one of:
  - a bare name: `- plan`
  - a name with options: `- commit: {when: test.passed, description: ...}`
  - a loop: `- repeat: {max: 3, until: test.passed, when: ...}` with `steps: [...]`
    (`max` required).
- Role steps are the roles of the registry (`.factory/roles.yaml`, else packaged):
  `plan` (= `planner`), `build` (= `builder`), `test_plan`, `test_review`, `scout`, `review` (= `reviewer`), `document` (= `documenter`), `ask`, `fix`, `revise`, `resolve`. Their options: `agent` (a name from `.factory/agents.yaml`; the step runs on it
  and keeps the role's output type and gates), `harness` (one of `claude`, `codex`, `pi`; aliases `claude_code` = `claude`), `model`,
  `thinking` (one of `off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`, `ultra`), `when`, `id`, `description` and `input`.
- Code steps are `test`, `commit`, `changes`, `command`, `rebase`, `rebuild`. `command` needs `id` and `argv` (a list) and takes an
  optional `timeout`. Code steps reject `agent`, `harness`, `model` and `thinking`.
- A `test_plan` role step (agent `tester`) reads the change since the `baseline` prompt variable and the
  repo, and returns `coverage` (`full`, `scoped` or `none`), `reason` and `checks`
  (`name`, `argv` list, optional `timeout` in seconds): enough checks to trust the
  change, nothing beyond. There is no configured test command; every repo, whatever its
  stack, gets its checks from the tester. The gate `checks_runnable` requires the first
  argv element of every check on PATH or as a file in the worktree.
- A later `test_plan` keeps every check of the previous plan (same argv) or names it in
  `dropped` with a reason (gate `plan_keeps_checks`). Workflows plan again after every
  fix and every revision, from the whole diff since `baseline`.
- A `test_plan` step may require a coverage: `coverage: full` (every check the repo has)
  or `coverage: scoped` (only the latest changes); gate `coverage_required`. The result of
  a `test` repeats the plan's coverage, so conditions can read `test.coverage == "full"`.
  The packaged `heal` workflow runs the whole suite, repairs until it is green (each repair
  checked scoped, then confirmed on the whole suite) and commits only on a green full run.
- A red `test` goes to the tester first (phase `triage`): `failure_cause: plan` (the
  checks themselves were wrong) replaces the plan and runs the test again without a
  repair round and without the builder, at most twice per run; `code` leaves the
  failure for `fix`.
- A test review (`test_review`) that requires tests the repo does not have yet: the
  revised plan (`plan_revision`) lists them in `missing_tests`, the builder writes them
  (phase `test_fix`, role `fix`, with the review as `previous_envelope`), the tester
  plans again and the new plan is reviewed; at most twice per run, then the run stops.
- A `test` step (also under another `id`, e.g. `retest`) runs the checks of the latest
  `test_plan` in order, without a shell, and runs every check even after one fails (only
  a failed check with `stop_on_fail: true` or an exhausted time limit stops the rest,
  which the result lists in `not_run` with the reason); a workflow with a
  `test` and no earlier `test_plan` is invalid (`test_without_plan`). `none` runs nothing
  and passes. The checks share one time limit: the task's `test_timeout`, else the
  nearest `index.md` `test_timeout`, else `test_timeout` from `.factory/config.yaml`,
  else 600 s; a check over the limit is stopped and the step fails (exit 124). The
  step's `test_plan` field (also in the `test_result` prompt variable) records coverage, reason, the commands
  and how many ran. Reviewers judge the plan: enough to trust the change, not wider than
  it needs.
- At most `test_slots` (`.factory/config.yaml`, whole number ≥ 1, default 1) `test`
  steps run their command on the machine at once; the others wait in a queue in the
  order they asked (`<HAIFA home>/test_slots`). The wait is traced as `test_slot` events
  (`state: waiting` with `ahead`, then `state: acquired` with `waited_seconds`), shows in
  the dashboard and does not count into `test_timeout`. A slot of a crashed, stopped or
  killed run frees itself.
- Conditions (`when`, `until`, `accept`) read `<step>.<field>` of an earlier step's result
  (e.g. `test.passed`); every step has `ran`. `accept` decides whether
  the run succeeded.
- Agents come from `.factory/agents.yaml` and roles from `.factory/roles.yaml`; a workflow
  names roles. `agent:` on a role step changes the agent for that step only;
  `factory workflow check` and the run's preflight verify that it is in the roster and
  accepts the step's `harness`, `model` and `thinking` (`unknown_agent` at
  `steps[N].<step>.agent`). `data.steps[].agent` shows each step's agent.
- `.factory/roles.yaml` without `code_steps` is an overlay of the packaged registry: an
  existing role changes only the fields it names (e.g. `review: {agent: critic}`), a new
  role needs `agent`, `output_type`, `gates` and `description`. An alias cannot be
  overridden; override its canonical name. A file with `code_steps` replaces the whole
  registry. Problems come as issues with `path` and `code` (`unknown_key`,
  `alias_conflict`, `missing_key`, ...):

  ```yaml
  roles:
    review: {agent: critic}
    audit:
      agent: auditor
      output_type: ReviewOutput
      gates: [artifacts_exist]
      description: Audit the change against the security checklist
  ```
- Validate with `factory workflow check FILE --json` (`data.issues`, `data.steps`).

Example:

```yaml
name: plan-build-test
description: Plan and build, repair until the suite is green, and commit only green code
steps:
  - plan
  - build:
      harness: codex
      thinking: high
  - test_plan
  - repeat: {max: 3, until: test.passed}
    steps:
      - test
      - fix
  - commit:
      when: test.passed
accept: test.passed
```

## Procedures

### Set up a machine

1. `factory check --json` anywhere (outside a repository only the machine and the library
   are checked). Resolve the `error` findings of `scope: machine` with their `fix`
   (Git identity, uv, the provider CLI, harness CLIs and their logins); see Check.
2. A `library_missing` finding: `factory library clone --json` clones the default team
   library (`$HAIFA_LIBRARY_URL`, else https://github.com/janbkrejci/HAIFA-LIBRARY.git);
   `factory library clone URL --json` another team library; `--seed-fallback` creates a
   local library from the seed when the clone fails (`data.fallback`, a warning). When this
   is the team's first machine with its own library, `factory library init [--name N]
   [--remote URL] --json` from the seed (see Library and Sharing the library). Ask the
   user which one; never create a second library for a team that has one.
3. `factory check --json` again until it reports no errors. Then add repositories (Add
   factory to an existing repo, Take over a repo with committed `.factory/`).

### Check

`factory check [--repo PATH] [--offline] --json` only reads; it changes nothing in the
repository, the trace DB, the library or the home directory, and never fetches.

1. `data.state` is the state of the repo, read from base, first match wins:
   - `installed`: `.factory/config.yaml` is in base. No action.
   - `unsupported`: an sssf installation in `adws/` without `.factory/config.yaml`. No
     action: HAIFA does not take over sssf.
   - `uncommitted`: the configuration exists only in the working tree. Action
     `config_commit`.
   - `none`: no factory. Action `init`.

   `data.action` is what that state calls for (`config_commit`, `init` or `null`). The
   finding of the state carries the same action: `factory_missing` (`init`),
   `config_not_committed` (`config_commit`), `repo_unsupported` (error for sssf `adws/`)
   and `repo_installed` (info, names the library commit or the seed the repo inherits
   from).
2. Outside a git repository (no `--repo`) only the machine and the library are checked:
   `data.in_repo` is `false` and `data.repo`, `data.state`, `data.action` and `data.base`
   are `null`. Inside a repository `data.in_repo` is `true`.
3. `data.findings` is a list of `{code, scope, severity, message, fix, action}`:
   - `scope: repo`: fix it in the repository and commit it to base (runs read base).
   - `scope: machine`: fix it on this machine (install a program, log in); nothing to commit.
   - `scope: library`: fix it in the library in `$HAIFA_HOME` (`factory library ...`).
   - `severity`: `error` (factory will not run), `warning` (it runs, but something is off),
     `info`.
   - `fix` says in words what to do. `action` names the fix (`init`, `config_commit`,
     `config_pull`) or is `null` for a manual fix.
     `config_commit` and `config_pull` are `factory config commit` and `factory config pull`
     (see Commit configuration); `init` is `factory init`.
4. Findings and their fixes, by group:
   - Installation: `factory_outdated` (machine, error: the library needs a newer factory;
     `factory upgrade`).
   - Platform and tools: `unsupported_platform` (only macOS, Linux, WSL and Windows),
     `git_missing`, `git_identity_missing` (`git config --global user.name/user.email`),
     `uv_missing` (install uv), `node_missing` (a roster with pi needs node).
   - Harnesses: `harness_missing` (error for a harness the roster uses; outside a
     repository `info` for each of claude, codex and pi), `harness_login` (`claude auth
     status`, `codex login status`, `pi auth check --model M --json --no-refresh` failed;
     log in with `claude auth login`, `codex login` or `pi auth login`),
     `pi_model_unknown` (the model is not in `pi --list-models` or matches several;
     register it in pi or fix `model` in `.factory/agents.yaml`), `codex_not_isolated`
     and `pi_not_isolated` (global files the harness loads).
   - Hosting by `git_provider`: `gh_missing`, `gh_login` (`gh auth login`), `az_missing`,
     `az_devops_missing`, `az_login` (`az login` or the PAT variable), `hosting_skipped`.
   - Library: `library_missing` (`factory library clone URL` or `factory library init`),
     `library_dirty` (commit or discard in the library), `library_behind` (`factory
     library pull`), `library_unpushed` (`factory library push`), both as of the last
     fetch, and `seed_update_available` (`factory library seed`).
   - Environment: `env_file_mode` (`$HAIFA_HOME/env` is not 0600; `chmod 600`),
     `env_override` (one of `CLAUDE_SAFE_MODE`, `CLAUDE_MCP_CONFIG`,
     `CLAUDE_PERMISSION_MODE`, `CODEX_SAFE_MODE`, `CODEX_SANDBOX`, `PI_SAFE_MODE` is set;
     unset it unless you want it).
   - Repo: `workflow_unknown` (the backlog names a workflow neither the repo nor the
     library has; add it or change the tasks), `roles_full_copy`
     (`.factory/roles.yaml` has `code_steps`; keep only the changed roles),
     and `unknown_thinking` (an agent's `thinking` is not a known level).
5. `data.ahead` / `data.behind` compare base with `refs/remotes/<remote>/<base>` as of the
   last `git fetch`; fetch first when you need the current state.
6. `--offline` skips the calls to the hosting and the harness logins (`gh auth status`,
   `az account show`, `claude auth status`, `codex login status`, `pi auth check`,
   `pi --list-models`); inside a repository the finding `hosting_skipped` says so.
7. Exit codes: 0 no errors (`ok: true`, warnings may still be listed), 1 errors
   (`checks_failed`, the report is in `data`), 2 only when `--repo` is not a git
   repository (`not_a_repository`).

### Add factory to an existing repo

`factory init` installs factory into a repository that has none. Install it into base with
one commit, after the user has seen the plan.

1. `factory check --json`: follow `data.action` and the findings' actions.
   `init` means a repo without factory. `config_commit` means configuration exists only
   in the working tree; commit it before continuing. State `installed` needs no init,
   and HAIFA does not take over a repo in state `unsupported`.
2. `factory init --dry-run --json [options]` writes nothing and returns the plan:
   - `data.files`: `path`, `action`, `diff` and `content`: `.factory/config.yaml`, the
     backlog `.gitkeep` and `.gitignore`. Agents and workflows are not copied; runs take
     them from the library.
   - `data.blockers`: `code`, `message` and `fix` when a command removes the blocker.
   - `data.warnings`: `code` and `message`.
   - `data.digest`, `data.exclude` and `data.validation`.
   - `data.detected`: `remote`, `base` (from `refs/remotes/<remote>/HEAD`, else the current
     branch), `provider` (`github`, `azure` with `azure.organization`, `project` and
     `repository`, or `local`) and `harnesses` (`installed` and `path` per harness CLI).
   Options:
   - `--base BRANCH` and `--provider local|github|azure`. Azure takes `--azure-org`,
     `--azure-project` and `--azure-repo`, each falling back to the remote URL.
   - `--docs-root DIR` (default `factory`). `$specs_dir/` and `$docs_dir/` in `writes`
     become `<docs_root>/specs/` and `<docs_root>/docs/`.
   - `--pr` plans a pull request; the direct-only blockers then do not apply.
   - `--base-changes pr` writes `base_changes: pr`: from now on every change factory makes
     to base (install, config commit, config set/remove/move, backlog commit, uninstall)
     goes through a pull request, this install too. `--base-pr-merge auto` writes
     `base_pr_merge: auto`: factory approves its own pull request where the hosting allows
     it, merges it at once (GitHub with `gh pr merge --admin`) and fast-forwards the local
     base (`data.merged`); a failed merge leaves the PR open with a warning. Ask the user
     before choosing either.
3. Show the user the files, the blockers, the warnings and the digest.
4. When the user agrees: `factory init --commit --expect <digest> [-m TEXT] --json`. It is
   the same plan as one commit on base. Repeat all options (base, directories and
   provider) from the preview; `--expect`
   validates the recomputed plan, it does not store those options. The commit is pushed before base moves, and the
   files reach the working tree only after the push (`data.commit`, `data.pushed`,
   `data.advanced`).
5. `plan_changed`: base or the plan changed since the preview. Run `--dry-run` again and
   show the user the new plan.
6. `push_failed`: nothing changed. The remote may refuse direct pushes. Ask the user, and
   only with their consent repeat it as `factory init --commit --pr --expect <digest>
   --json`. The commit then goes to `factory-init/<n>` with a pull request (`data.branch`,
   `data.pr`), and base and the checkout stay as they are.
7. Blockers (the commit refuses with the first one):
   - `already_installed`: `.factory/config.yaml` is committed; factory is installed.
   - `existing_config`: an sssf installation; HAIFA does not take it over.
   - `config_not_committed`: run `factory config commit`.
   - `dirty_paths`: the user moves or commits the listed files.
   - `invalid_plan`: see `data.validation.issues`, or `error.issues` on commit.
   - `not_on_base`, `run_in_progress`, `base_behind` and `base_diverged`: as in Commit
     configuration.
8. Warnings never block:
   - `harness_missing`: the harness CLI of an agent is not on PATH; install it before runs
     or switch the agent (Switch the local roster).
   - `gitignore_dirty`: `.gitignore` has uncommitted changes, so the runtime lines go to
     `info/exclude` (`data.exclude`).
   - `foreign_content`: the backlog, specs or docs directory already has files.
   - `remote_note`.
9. After the direct commit (or after the installation PR is merged and base is
   synchronized), verify `factory check --json` and `factory backlog check --json`.
   Resolve findings before creating or running tasks.
10. Without `--dry-run` and `--commit`, `factory init` writes the files to the working tree,
   skips existing files unless `--force`, and commits nothing.

### Take over a repo with committed `.factory/`

A repo where HAIFA is already installed (another machine ran `factory init`) needs no
installation on this machine.

1. Clone the repository with git and set up the machine (Set up a machine), with the
   same library as the team (`factory library clone URL --json`).
2. `factory check --json` in the clone: `data.state` is `installed`. Never run
   `factory init` there (`already_installed`). Resolve the machine findings (harnesses
   of the roster, logins, provider CLI).
3. Optionally switch the harnesses for this repo (Switch the local roster) and commit
   it with Commit configuration.
4. `factory obs --repo PATH` registers the repository in the dashboard and opens it.

### Commit configuration

`factory config commit` commits the uncommitted shared `.factory/` files (the paths of
`factory config status`; never `local.yaml`) to base in one commit. The commit is built on
base without a checkout: other staged and unstaged work in the checkout stays.

1. `factory config commit --dry-run --json` returns the plan: `data.files` (`path`,
   `action` create/modify/delete, `diff` against base, `content`), `data.blockers`
   (`code`, `message`) and `data.digest`. The digest covers base, its sha and every
   file's old blob and new content; not the target and not the message.
2. Review the plan, then `factory config commit --expect <digest> [-m TEXT] --json`.
   `plan_changed` means base or the files changed since: run `--dry-run` again.
3. Direct target: the main checkout must be on base (`not_on_base`), no run may be in
   progress (`run_in_progress`), and with a remote the remote base must not be ahead
   (`base_behind`: run `factory config pull --json`) nor diverged (`base_diverged`: fix it
   by hand). The commit is pushed without force; `push_failed` changed nothing. The local
   base moves only after the push (`data.pushed`, `data.advanced`); when that move fails,
   `warnings` say so and `factory config pull` catches up.
4. `--pr`: the commit goes to a new branch `factory-config/<n>`, which is pushed and gets a
   pull request (`data.branch`, `data.pr`); base and the checkout stay as they are.
5. Nothing to commit: `data.committed: false`.
6. `factory config pull --json` fast-forwards the local base to the remote base
   (`data.updated`, `data.before`, `data.after`). `base_diverged`, `dirty_base`,
   `run_in_progress` and `no_remote` change nothing.

### Plan -> backlog

There is no importer; converting a plan is this procedure (D12).

1. Read the plan (for example `implementation-plan` output: an index plus linked, sharded
   markdown files; follow the links).
2. Map every module of the plan to a project: a project directory
   `<docs_root>/backlog/<project-id>-<slug>/` with
   `index.md`, and every step to a step directory inside it with its own
   `index.md`. Create them with
   `factory backlog add --id CODE --title TEXT [--body TEXT] --json` (a project) and
   `factory backlog add PROJECT --id CODE --title TEXT --json` (a step). Give only the short
   code (`S03`); the id is always composed from the parent's id: `M01-S03`, in the directory
   `S03-<slug>`. Put shared defaults (`workflow`, `writes`, ...) into
   the project or step index with `factory backlog edit ID --workflow NAME --writes PATH ...
   --json` (`--clear KEY ...` removes a key; keys it does not know stay as they are). The
   project id is any short code; keep the module's code from the plan (e.g. `M01`).
   An invalid code (letters, digits, `.` and `_`, no `-`) or a duplicate is rejected with
   `invalid_id` or `backlog_invalid` (`error.issues`: `duplicate_id`) and nothing is written.
3. Add each task, in dependency order:
   `factory task add STEP TITLE [--id CODE] --writes PATH ... --depends-on ID ... --body TEXT --json`
   (`--id T05` makes `STEP-T05`; without it the next free `T<nn>`)
   (or write the task files directly in the format above). `--body` is the text of
   `## Zadání`: what, where, "done means", out of scope, hard constraints.
4. Add references later with `factory task link ID --depends-on ID ... --json`.
5. Run `factory backlog check --json` until `ok` is true, then `factory backlog list`.
6. `factory backlog commit --json` commits and pushes it to base; runs read the
   backlog from base.

Example with the default project → step → task hierarchy:

```bash
factory backlog add --id M01 --title "Core" --json
factory backlog edit M01 --workflow simple-sdlc --writes src/ tests/ --json
factory backlog add M01 --id S01 --title "API" --json
factory task add M01-S01 "Health endpoint" --id T01 --body "Add GET /health; cover HTTP 200 in tests/api." --json
factory backlog check --json
factory backlog commit --json
```

Put common settings at project level, narrower defaults on the step and only
exceptions on the task; the nearest value wins.

### Intent -> step with tasks

`factory backlog propose "INTENT" [--project ID | --step ID] [--apply] --json` lets the
read-only agent `proposer` turn a short intent into a proposal: a new step of the project
with its tasks, or new tasks of an existing step. Without `--project` and `--step` the
agent places the tasks itself: in an existing step whose goal covers the intent, in a new
step of the project the intent belongs to, or, when it belongs to no project, in a new
project with a new step (not in a backlog without a project level). It works in an isolated checkout of the
base commit and writes nothing. Every task has a title, an assignment ending with
`Hotovo znamená:`, `depends_on` (also on existing tasks), `writes` and a workflow of the
catalog. The proposal is validated like `factory task add`; an invalid one goes back to
the agent once, and a second invalid one fails with `proposal_invalid` (`error.issues`)
without writing. A quality check also looks at `writes`: they cover the files the
assignment names, stay out of `protected_files` and the backlog, and tasks that can run
at the same time do not overlap. Its findings go to the agent in the same single
correction; those left are in `data.findings` (`rule`, `task`, `message`) and in the
preview. Without `--apply` the command only shows the proposal (`data.project`: `id`,
`title`, `body`, `path`, `new`, or null without a project level; `data.step`;
`data.tasks`; `data.target`: the resolved `{project, step}`; `data.placement_reason`: why
the agent put it there, empty with `--project` or `--step`; `data.applied: false`); show
it to the user, with the placement, before applying. `--apply`
writes the step and task files and commits nothing: check and commit as in Plan -> backlog.
The agent's prompts are overridden in `.factory/prompts/proposer/`.

### Lessons from runs

`factory learn scan [--since DATE] --json` reads the objections of every task run of the
repo from the trace DB (blocking items and unmet findings of the review and test review,
test triage of failed checks, revise phases) and lets the read-only agent `learner` group
similar ones into lessons. Each of `data.lessons` has an `id`, a short `rule`, a `target`
(`CLAUDE.md`, `AGENTS.md`, `skill` or `prompt`, with `target_detail` naming the skill or
role), `occurrences` (the distinct runs), `runs` and `complaints` (snippets with their run).
A lesson seen in one run only is never proposed. Show the lessons to the user and let them
decide; never write a rule into the repo yourself.
`factory learn accept ID [--project ID] --json` files the lesson as a task in the step
`LEARN` of the repo's project (created when missing) with workflow `build-review`, `writes`
on the lesson's target only and the rule with the objections as evidence, and commits it to
`base`; `data.task` names it. Run it like any task (it ends in a pull request). The project
is `--project`, else the one of the tasks whose runs objected, else the only project
(`project_required` otherwise). Accepting again shows the same task (`data.existed`).
`factory learn dismiss ID [--reason TEXT] --json` rejects a lesson in the trace DB. No later
scan offers an accepted or dismissed lesson or its objections again. Objections that repeat
an accepted lesson are added to it instead (`data.repeats`: lesson `id` and the number
`added`); the dashboard's Factory screen (section Lekce) shows the lessons, scans, accepts
and dismisses them, and counts for an accepted lesson the runs with its objections before
and after its pull request was merged. The agent's prompts are overridden in
`.factory/prompts/learner/`.

### Run

1. `factory task list --status ready --json` lists the tasks that can start.
2. `factory task run ID [--note TEXT] [--auto] --json` runs one task in its own worktree on
   a new branch `factory/<task-id>-<n>` and opens a pull request. `--harness NAME`,
   `--model NAME` and `--thinking LEVEL` put every agent on them for this run only (step
   overrides of the workflow still win; with `--auto` only the first run gets them). An
   unknown harness or thinking level, or a model the harness rejects, is `invalid_override`.
3. Read `data.run.state`, `data.pr` (the pull request), `data.pr_error` and `data.chain`.
4. On failure (`run_failed`) the worktree and branch are kept; re-run with `--note` to tell
   the agents what to fix.
5. Auto-continue (`--auto`, or `auto_continue: true` in the project or step
   `index.md`, set with `factory backlog auto-continue ID --on|--off|--inherit`):
   after a run opens its PR, the next ready task of the step, then of the project, starts.
   Without `--auto` the chain continues only after a task with `auto_continue: true` and
   starts only tasks with `auto_continue: true`; `--auto` starts any task.
   The dashboard's kanban sets the order of ready tasks (taken first) and can exclude a
   task from auto-continue (skipped as `excluded`, with `--auto` too; it still runs when
   started by hand).
   Tasks it skips carry a reason (`human_task`, `no_workflow`, `waits_on_pr`, `blocked`, `in_review`, `running`, `cannot_start`, `writes_overlap`, `exclusive`, `excluded`, `no_slot`); the chain stops with
   `failed`, `exhausted`, `disabled`, `not_merged` (`data.chain.stop`, `data.chain.waiting`). The chain approves and
   merges nothing except through `auto_merge` (next point).
   Self-healing (`auto_heal`, inherited like `auto_continue`, default `true`; switch it off
   with `auto_heal: false` or `factory backlog edit ID --auto-heal off`): a chain that goes
   on acts on a failed run by its `failure_kind`. `quota` continues on the project's
   quota chain (`quota_switch: [HARNESS, ...]`, `quota_switch_back`, project-only, set with
   `factory backlog edit ID --quota-switch ... --quota-switch-back on|off`; it wins over any
   `fallback`), else once on the configured fallback harness (with `auto_heal: false` not
   even a single run switches),
   `process_gone` is retried once, `nothing_to_do` with the agent's reason marks the task
   done without changes and the chain goes on; every other kind stops the chain as before.
   Each action is in `data.chain.events` (`action`: `fallback`, `retry` or `close`, `ok`,
   `text`), in `task_chains.events` and on the dashboard's Overview.
6. Auto-merge (`auto_merge: true`, inherited project -> step -> task; set it with
   `factory backlog auto-merge ID --on|--off|--inherit` or on a task with
   `factory task edit ID --auto-merge on|off|inherit`, e.g. off for a task that changes
   the guard or permissions): after the run its PR is approved and merged like
   `factory task approve` (`merged_by: auto-merge` in `task_prs`) only when the run
   `succeeded`, the workflow has a review phase whose last review approved with no blocking
   or unmet item, the PR merges cleanly and its checks finished with none red. Running
   checks are waited for (up to 30 minutes, else `checks_pending`); on GitHub a fresh PR
   gets a short wait for its checks to appear before "no checks" counts. A workflow without
   review is never merged automatically. Otherwise `data.auto_merge` (`merged`, `code`,
   `reason`) says why and the PR waits for `factory task approve`. With auto-continue the
   next task starts only after the merge; a PR that was not merged stops the chain
   (`not_merged`).
7. Parallel runs (`max_parallel_runs` in `.factory/config.yaml`, whole number ≥ 1,
   default 1): with 1 the chain runs one task after another. Above 1 the chain keeps up to
   that many slots taken (each run a separate `factory task run` process) and refills a
   slot whenever one frees. A slot is taken by a task with a live run and by a task whose
   PR is open and unmerged (waiting for review); a task with both counts once. The slot
   frees when the PR is merged or closed, or when the run ends without a PR (failed,
   blocked, stopped). The dashboard's "Spustit továrnu" and its `free_slots` count the
   same way. When PRs waiting for review hold the free slots, the next ready task is
   skipped as `no_slot` (`waits_on` lists those PRs' tasks). A task starts only when its effective `writes` do not overlap (path
   prefix) the `writes` of a running run nor the files of an open, unmerged PR; otherwise it
   is skipped as `writes_overlap` (which run or PR, which paths). A task whose `writes` cover
   the whole repo or a whole package (e.g. `src/`) runs alone (`exclusive`,
   `data.chain.exclusive`). The dashboard's Runs screen shows the chain, its free slots and
   the skipped tasks.

### Pause, resume and stop

Run states in `data.run.state` and `task_runs`: `running`, `succeeded`, `failed`,
`aborted` (its process is gone), `stopped`. A running run also has `pause`: `null`,
`pausing` or `paused`; a run with `pause: paused` is shown in state `paused`
(`factory task show`, the dashboard).

1. `factory task pause ID [--run RUN_ID] --json` asks the running run to pause. The phase in
   progress (an agent, a test) runs to its end; meanwhile `pause` is `pausing`. Before the
   next phase the run turns `paused` and waits: its process stays alive, no further phase
   starts. Nothing is frozen. A pause asked for in the last phase is dropped when the run
   finishes. Errors: `run_not_running` (the task has no running run, or it is finished),
   `run_already_paused` (`pausing` or `paused` already); exit 2, nothing changed.
2. `factory task resume ID [--run RUN_ID] --json` lets a paused run go on with its next
   phase in the same worktree and branch (a pending `pausing` is dropped). Errors:
   `run_not_running`, `run_not_paused` (the run has no pause); exit 2, nothing changed.
3. A paused run keeps its place among the parallel runs (`max_parallel_runs`) and blocks
   another run of its task; an auto-continue chain waits until it ends.
4. `factory task stop ID --json` stops a running or paused run (state `stopped`); its
   worktree and branch stay. A pause does not survive a restart of the machine: the run's
   process is gone and the run becomes `aborted`.

### Retry

`factory task retry ID [--run RUN_ID] [--note TEXT] [--from PHASE] [--harness NAME [--model
NAME] [--thinking LEVEL]] --json` continues the task's newest `failed`, `stopped` or
`aborted` run (or `--run`) as its next attempt.

1. Same `run_id`, worktree, branch and trace session: tokens and costs add up,
   `data.run.attempt` counts the attempts and `data.run.retries` says per retry where it
   continued (`from_phase`), how many done phases it took over (`replayed`), the note and
   `rebased_onto`.
2. Done phases are not repeated: their results are replayed from the run's phase journal
   (`<session>/workflow_journal.json`). The run continues from the phase that failed or
   was stopped; after `accept_not_met` from the top-level step whose result failed
   `accept` (a review loop gets a new round); with `--from` from that phase (a phase name
   like `review_2` or a step like `review`, `unknown_phase` otherwise).
3. Every agent rejoins its coding-agent session when its harness and model are unchanged.
   The prompt gets a `## Continuation` section (the done phases, `context_handoff/`) and
   `--note` as `## Retry note`.
4. Base is pulled from the remote first, like `factory config pull`; a base that cannot
   move stops the retry with that error (`base_diverged`, `dirty_base`, `run_in_progress`)
   and nothing changes. When base moved past the branch, the branch is rebased onto it
   (uncommitted work is committed first); a conflict is resolved like `factory task
   resolve`. When that fails the branch stays as it was and the run is `failed`. After a
   rebase the tests run again before any review.
5. Configuration, backlog and workflow come from the current base. A phase whose step the
   workflow no longer has starts the workflow from its first phase, with a warning.
6. Refused, exit 2, nothing changed: `task_done` (the task's PR is merged or the task is
   `done`), `already_running`, `run_not_retryable` (a succeeded or running run, or a
   resolve run), `unknown_branch` (the run's branch is gone).
7. After a `writes_violation` (`data.run.failure_detail.paths` are the worktree paths
   written outside `writes`) or a `writes_needed` (an agent reported `fail_kind:
   needs_writes`: `failure_detail.paths` and `needs_writes` hold the paths this task must
   change, with the agent's reasons), `factory task retry ID --allow-writes --json` first
   adds them to the task's `writes` and commits the backlog to base (by `base_changes`),
   then retries from the failed phase with the allowed paths in the `## Retry note`;
   `data.allow_writes` lists `allow` and `refused` (with `reason`). A path in
   `protected_files`, in the backlog or in the main checkout is never added. Refused:
   `not_writes_violation`, `nothing_to_allow`, `writes_in_pr` (base_changes: pr and the
   backlog PR is not merged yet: retry after the merge), `writes_not_committed` (the
   backlog in base lacks the new effective writes: commit the backlog with
   `factory backlog commit --json`, then `factory task retry ID --json`). The retry
   starts only after all allowed paths are verified in the base commit, including
   when the commit reports no change or a backlog PR has merged. A `ci-fix` run is not
   retried (`run_not_retryable`): see Approve.
8. `--harness` (with `--model`, `--thinking`) continues with every agent on another
   harness, e.g. after `failure_kind: quota`; it replaces the run's roster override. The
   switch is in `data.run.retries[].harness_switch` (`from`, `to`, `reason`, `auto`), in
   the run's events (the `request` phase) and in the PR body (`Přepnutí harnessu`).
9. Quota fallback: a run or retry that fails with `quota` retries itself once on the
   fallback harness, when one is set: `fallback: {harness, model?, thinking?}` in
   `.factory/local.yaml` (this computer, wins), on the agent in `.factory/agents.yaml`, or
   in its `defaults`. When the fallback fails too, the run stays failed (`quota`) and no
   further switch happens. Without a fallback, retry with `--harness` yourself.

### Review

1. `factory task show ID --json` gives the task, its body, its runs and its PRs.
2. Review the PR diff through the hosting provider (`gh pr diff`, `az repos pr`, or
   `git diff` of the branch in local mode).
3. Then approve, or return with a note.

### Approve

1. `factory task approve ID --json` adds a commit with `status: done` and a
   `## Běhy` line to the PR, merges it, updates base from the remote and removes
   the branch's worktrees.
2. On `conflict`, resolve first (below), then approve again.
3. On GitHub an approve review is sent as `github_approver` (`config.yaml`, another
   account logged in to `gh`) when it is set; otherwise none is sent. Azure DevOps gets
   an approval vote. `data.reviewed` says whether a host approval was sent.
4. Red CI checks start up to 2 `ci-fix` runs on the PR branch. One that ends
   `writes_needed` (its agent needs paths outside the task's `writes`) leaves the PR open
   with `writes_needed` and the paths (`task_prs.auto_merge_error`). `factory task approve
   ID --allow-writes --json` adds them to the task's `writes` in base (same refusals as
   `task retry --allow-writes`, `data.allow_writes`) and approves again, so the new
   `ci-fix` runs may write them.

### Publish

`factory task publish ID --json` repeats the end of `task run` for the task's last
succeeded run that has no pull request (its push or PR failed, `pr_failed`): it pushes the
branch and opens the PR with the body kept from the run, recorded in `task_prs` as after a
run. When the hosting already has an open PR for the branch that factory does not know, it
is adopted (recorded, its body updated) instead of opening a second one. A push failing on
the network (RPC failed, HTTP2 framing, remote end hung up, timeout) is retried a few times
with a pause, including hosting server errors (HTTP 5xx, Internal Server Error); a rejected
push (non-fast-forward, rights, protected branch or hook) is not.

After publication, effective `auto_merge` merges the PR under the same review and CI
conditions as `task run`, including conflict resolution and `auto_merge_error` on failure.
The same auto-continue chain then selects ready tasks in kanban order and respects slots
and `max_parallel_runs`. No `--auto` flag is needed: inherited `auto_continue` applies.
Without `auto_merge` the PR stays open for review.
`data.auto_merge` and `data.chain` report the merge outcome, runs and stop reason.
The dashboard starts publish with auto-merge in a separate process so CI never blocks it.

- `data.run`, `data.pr`, `data.pr_error`: as for a run.
- `no_succeeded_run` (exit 2): no succeeded run; nothing changed.
- `pr_exists` (exit 2): the task or the branch already has a PR; nothing changed.
- `already_running` (exit 2): a run of the task is in progress; nothing changed.
- `pr_failed` (exit 1): push or PR failed again; `data.pr_error` says why, retry later.

The dashboard shows `pr_error` of a succeeded run without a PR and a button that publishes it.

### Return

`factory task return ID --note TEXT --json` starts a new run on the same branch with the note
in the prompt; the pull request is updated. Read `data.run` and `data.pr` as for a run.

### Resolve

`factory task resolve ID --json` rebases the PR branch onto the current base. On a conflict
an agent changes only the conflicted files. Conflicted files inside a generated output
(`generated:` in `.factory/config.yaml`, entries `{path, command, timeout}`) are not the
agent's: the code step `rebuild` runs the output's `command` and the rebuilt output is
committed with the resolution. Then the suite runs and the branch is
force-pushed with a lease and the PR updated. On failure the branch stays as it was before
the rebase. Then approve.

### Sync and cleanup

- `factory backlog sync --json` finds task PRs merged outside factory and opens one sync PR
  that marks them done (never commits to base, never merges it).
- `factory task clean --json` removes worktrees of merged, closed or abandoned runs.

### Upgrade

`factory upgrade BUNDLE [--dry-run] --json` installs a newer HAIFA from a distribution
bundle `haifa-<version>.zip`. `BUNDLE` is a path or a URL; only `https` and `file` URLs are
downloaded (into a temporary directory), any other scheme is `invalid_url`, a failed
download `download_failed`.

1. An editable install (development from the HAIFA repo) is refused with
   `editable_install`; update it with `git pull` instead.
2. The bundle is unpacked into a temporary directory and every file listed in `SHA256SUMS`
   is verified. A broken zip, a missing `SHA256SUMS`, a listed file that is missing, or no
   `aifactory` wheel or `constraints.txt` is `bundle_invalid`; a wrong sum is
   `checksum_mismatch` (`data.file`, `data.expected`, `data.actual`).
3. The target version is read from the wheel's metadata. Not newer than the installed
   `__version__` is `up_to_date` (exit 1, `data.current_version`, `data.target_version`);
   nothing is installed.
4. `data` has `current_version`, `target_version`, `wheel`, `min_factory_version` of the
   library (`null` without a library or minimum), `command` (argv), `dry_run`,
   `installed`, `stdout`, `stderr`, `restart_dashboard` and `library_refreshed`. A target
   below the library's `min_factory_version` adds a warning.
5. `--dry-run` stops here and runs nothing. Without it the command
   `uv tool install --force <wheel> --constraints constraints.txt` runs in the unpacked
   bundle: `uv_missing` when uv is not on PATH, `upgrade_failed` (exit 1, uv's output in
   `data`) when uv fails; the old version stays. On success `installed` is `true` and
   `data.next` reminds to restart a running dashboard (`factory obs`).
6. After the install the new `factory` refreshes the library: `library status --json`,
   `library clone --seed-fallback` when the library is missing, then
   `library seed --all` (`library_refreshed: true`). A failed refresh is a warning (then
   run `factory library seed --all --json`); the upgrade still succeeds. The
   Windows handoff script and the bundle installers do the same.

In short: `factory upgrade BUNDLE --dry-run --json`, `factory upgrade BUNDLE --json`,
restart a running dashboard, then `factory check --json`. Repos need no step: they inherit the new item versions at
their next run, and `factory --skill` must be regenerated for the new version.

### Library

The library is a git repository in the home directory, `$HAIFA_HOME/library`
(`HAIFA_LIBRARY` overrides the path). `library.yaml` holds `format: 1`, `id` (uuid4),
`name`, `min_factory_version` and `seed` (`type/name` to the seed version the item came
from). Items: `agents/<name>/{agent.yaml,system.md,user.md}`, `workflows/<name>.yaml`,
`skills/<name>/` (with `SKILL.md`) and `extensions/<name>/` (with `<name>.ts`).

1. `factory library init [--name N] --json` creates it from the seed in one commit
   (`library_exists`, `git_identity_missing`).
2. `factory library list [--type T] --json` lists the items of HEAD with `version`,
   `short_version` (8 characters) and the date and author of the last change.
   `factory library show TYP JMENO [--version V] --json` returns `files` and `history`
   (`n`, `version`, `commit`, `date`, `author`). Both read HEAD, never the working tree;
   the history is cached in `$HAIFA_HOME/cache/` and recomputed when the cache is gone.
3. `factory library import CESTA --type agent|workflow|skill|extension [--name N]
   --dry-run --json` validates a folder or file under the user's home directory
   (`outside_home`, also for a symlink leading out of it; `invalid_item` with
   `error.issues`) and returns the plan: `items[].action` (`create`, `update`,
   `unchanged`), `files` with diffs and `digest`. Review it, then run the same command
   without `--dry-run`. Equal content is a no-op (`committed: false`); new content under an
   existing name becomes the next version of that item.
4. A write holds `$HAIFA_HOME/library.lock`, so concurrent writes run one after another.
   It refuses a library with uncommitted changes (`library_dirty`; commit or discard them
   in the library yourself), commits on HEAD without a checkout and fast-forwards the
   branch and working tree. Nothing is forced and history is never rewritten.

#### Sharing the library

A team shares the library through a git remote, stored as `origin` without user and
password (`https://user:secret@host/x` is kept and shown as `https://host/x`; git asks its
credential helper).

1. `factory library init --remote URL --json` pushes the new library to an empty remote
   before it exists locally (`remote_not_empty`, `push_failed`; nothing is created).
   Others join with `factory library clone [URL] [--branch B] [--seed-fallback] --json`
   (`library_exists`, `clone_failed`, `invalid_library` when `library.yaml` lacks
   `format`, `id` or `name`); without URL the default library is cloned.
2. `factory library status [--fetch] --json` returns `library`, `id`, `name`, `remote`,
   `branch`, `ahead` and `behind` (from the last fetched refs unless `--fetch`),
   `last_fetch`, `dirty` with `uncommitted`, `min_factory_version`, `factory_version` and
   `compatible` (a warning when the installed factory is older), `unsent` (local commits
   in no open pull request), `sync_state` (`unshared`, `local`, `pending`, `behind`,
   `synced`) and `pull_requests` (the open pull requests sent from this library; `--fetch`
   asks GitHub which are still open).
3. Changes never go to the library branch of the remote directly; they go as a pull
   request and the library maintainer merges it. `factory library push --json` sends the
   unsent local commits as one pull request (`data.pull_request`: `branch`, `url`,
   `number`, `fork`; `pushed: 0` and `pull_request: null` when nothing is unsent;
   `library_behind`, `library_diverged`, `push_failed`, `pr_failed`); `no_remote` without
   a remote. On GitHub the branch `haifa-library/...` goes to the library repo when the
   user may write to it, else to the user's fork (`gh repo fork`); `gh pr create` opens the
   pull request. Other remotes get only the branch. Never merge, approve or force it.
4. `factory library pull --json` fast-forwards; local commits the remote lacks are rebased
   onto it and the merged ones drop out (`library_dirty`; a conflict is
   `library_diverged` and changes nothing).
5. A write to a shared library fetches first. Behind the remote it is `library_behind`
   (`data.fix` is `factory library pull`; run it, then repeat the write); diverged it is
   `library_diverged` (resolve it in the library with git, never by force). The commit is
   built on HEAD without a checkout and sent as a pull request (`data.pull_request`)
   before the local branch moves, so `push_failed` or `pr_failed` leaves local refs, index
   and files exactly as they were. The local library uses the change at once.

#### After a HAIFA upgrade

A new HAIFA can ship new versions of the seed agents, workflows and the `factory` skill
(`skill/factory`, generated from `factory --skill`). `seed` in `library.yaml` records the
seed version each item last took. Installing or upgrading HAIFA runs
`factory library seed --all`: every seed item is replaced with the installed seed version,
also items the team changed in the library (action `take`); items the seed lacks stay.
Customizations belong in the repos' `.factory/`, which the refresh never touches.
`--all` with `--take` is `usage_error`. Without `--all`:

1. `factory library status --json` has `seed_update_available` (with `seed_updates`, the
   `type/name` list) and a `seed_update_available` warning when the installed seed has
   versions the library's `seed` lacks.
2. `factory library seed --dry-run --json` returns the plan. `items[].action`: `create`
   (missing in the library, added), `update` (the team did not change it since the last
   seed, replaced), `unchanged` (the library already has the seed version) and `kept` (the
   team changed it; `items[].diff` shows what the seed would change). Review it, then run
   it without `--dry-run`. `seed` in `library.yaml` is set to the installed seed.
3. A `kept` item stays as the team made it. To take the seed version anyway, run
   `factory library seed --take TYP/JMENO --json` (repeatable; action `take`; `not_in_seed`
   when the seed lacks it). Items the seed no longer has are never deleted.
4. The write goes like `import` (lock, `library_dirty`, fetch, pull request). A second run without a
   new seed commits nothing (`committed: false`). Repos inherit the new versions at their
   next run; a repo's own copy stays as it is.

### Manage items

Agents, workflows, skills and extensions are inherited: a run looks in the repo's
`.factory/` (in base) first, then in the library (see Agents and prompts). Use the
narrowest change that does the job:

- Share an item with every repo: `factory library import PATH --type T --dry-run --json`,
  review, then without `--dry-run` (see Library). Repos inherit it at their next run.
- Change an agent's bindings for one repo: `factory config set agent NAME ... --json`.
- Drop the repo's own copy so the library item applies again:
  `factory config remove TYPE NAME --json`.
- Move the repo's own copy to the library: `factory config move TYPE NAME --json`.
- A repo-only item: write it into `.factory/` (e.g. `.factory/prompts/<name>/`,
  `.factory/workflows/<name>.yaml`); there is no command that adds one.
- Every change in the working tree reaches runs only after `factory config commit`
  (see Commit configuration). Details of set, remove and move are below.

### Items of the repo: config set / remove / move

```bash
factory config set agent builder --thinking high --tools read,bash --color '#112233' --json
factory config remove workflow solo --json
factory config move agent builder --commit --dry-run --json   # library files + repo deletions
factory config move agent builder --commit --expect <digest> --json
```

1. Modes, as `factory init`: `--dry-run` returns the plan (`data.files`, `removed`,
   `bindings`, `blockers`, `validation`, `digest`) and writes nothing. Without an option
   the plan is written to the working tree only (nothing committed; commit it later with
   `factory config commit`). `--commit [--pr] [-m TEXT]` makes it one commit on base
   (pushed before base moves) or a PR from `factory-config/<n>`. Review first:
   `--dry-run` (with `--commit` for the commit plan), check it, then repeat the command
   with `--expect <digest>`; a changed plan is `plan_changed`.
2. `set agent NAME` changes only the bindings in `agents.yaml`; comments and key order
   stay. An agent the repo inherits gets an entry that overrides only these keys.
   `--tools`/`--writes` take comma lists (`-` removes the key). An unknown `--thinking`
   is `invalid_value`.
3. `remove` deletes the repo's own copy; the library item of the same name applies again.
   When the library lacks it, an item in use is refused (`in_use`, `data.used_by`): an
   agent named by a workflow or `roles.yaml`, a workflow named by a backlog task in base, a
   skill or extension bound by a roster agent.
4. `move` ("Move to library" in the dashboard) writes the repo copy into the library under
   its name and deletes it from the repo. An agent keeps the `agent.yaml` defaults of its
   library item with the repo purpose (a new one takes its defaults from the roster entry);
   its roster entry keeps only its bindings, or goes when nothing else is left. Order:
   the library first (fetch, push without force), then the repo. A rejected library push
   (`push_failed`, `library_behind`, `library_diverged`, `library_dirty`) leaves the repo
   untouched. When the repo write fails after the library push, `data.library_commit`
   names the library commit; run the same move again. Without a library it is
   `library_missing`.
5. Blockers: `run_in_progress`, `invalid_plan` (working tree); `dirty_paths`,
   `invalid_plan`, `not_on_base`, `base_behind`, `base_diverged` (`--commit`).

### Repo instructions and skills

Agents run as if their harness had been started from the CLI in the repository. The
agent's role stays in its system prompt (`--system-prompt`, codex `developer_instructions`);
the repo's instructions and skills ride along through each harness's native mechanism:

- claude reads `CLAUDE.md`, `.claude/settings*.json` and `.claude/skills/`
  (`--setting-sources project,local`); nothing from `~/.claude` (hooks, plugins, skills,
  agents, CLAUDE.md). MCP stays off (`--strict-mcp-config`).
- codex reads `AGENTS.md` and `.agents/skills/`; `--ignore-user-config` leaves out
  `~/.codex/config.toml`.
- pi reads `AGENTS.md`, `CLAUDE.md` and `.agents/skills/` with project trust granted
  (`--approve`); personal skills, extensions and prompt templates are left out
  (`--no-skills`, `--no-extensions`, `--no-prompt-templates`).

Library skills and extensions the repo does not have are copied into the run's worktree
at its start (`.claude/skills/<name>/` and `.agents/skills/<name>/`, extensions to
`.factory/extensions/<name>/`), each with a `.gitignore` that keeps it out of git; they
are never committed. A skill of the repo wins over the library's.

A skill of the repo has its source in `.claude/skills/<name>/`; edit it there. `.agents/skills/` is
its exact copy for codex and pi, with no symlinks. After a change run
`factory skills sync [--repo PATH] --json` (`data.added`, `data.updated`, `data.removed`;
`skills_sync_failed` when it cannot write) and commit both trees. `factory check` compares
them in base and reports `skill_mirror_missing`, `skill_mirror_differs` and
`skill_mirror_extra` (scope repo).

Global files of the operator a harness cannot leave out are reported by `factory check` as
`codex_not_isolated` (`~/.codex/AGENTS.md`, `AGENTS.override.md`, `~/.codex/skills`,
`~/.agents/skills`) and `pi_not_isolated` (`AGENTS.md`, `AGENTS.override.md`, `CLAUDE.md`,
`APPEND_SYSTEM.md` in `~/.pi/agent`), scope machine.

`CLAUDE_SAFE_MODE=1`, `CODEX_SAFE_MODE=1` and `PI_SAFE_MODE=1` restore full isolation
(claude `--safe-mode`, codex `project_doc_max_bytes=0`, pi without context files, skills,
extensions and project trust). The default is off: native repo loading.


