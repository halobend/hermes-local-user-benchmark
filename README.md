# Benchmark Design Write-Up

## Why This Benchmark Exists

This benchmark was built to measure the kind of agentic work the operator actually delegates, not generic coding puzzles. The design started from reviewed Codex session history on the local host (laptop) and on `desktop-pc`, then distilled the repeated work patterns into two suites:

- an offline fixture suite for repeatable, comparison-friendly testing
- a live SSH suite for real remote operations on a disposable host

The benchmark is meant to answer a practical question: can `hermes` local model profile do the same categories of work the operator already uses Codex for, and can it do that work with reasonable operator discipline?

## How The Session Review Drove The Design

The benchmark was grounded in top-level session history rather than imagined tasks. The session review focused on recurring asks such as:

- finding and resuming older sessions by topic
- downloading and organizing local GGUF models
- cloning or adapting Docker Compose files for new local models
- retargeting Hermes local profiles and helper skills
- debugging remote operational problems over SSH
- reading repo contracts and architecture docs before editing
- supervising Hermes as a worker without taking over the edits directly

That review showed a few strong patterns.

### 1. Local model infrastructure is core work

Local model setup was not a side issue. Repeated sessions involved:

- `hf download`
- organizing models under `~/models`
- building or cloning `docker-compose` files under `~/docker`
- checking VRAM before launch
- updating Hermes local config
- benchmarking local model endpoints

That directly produced the `qwen_rollout` task.

### 2. Session recovery is a real workflow

There were repeated asks to search old sessions, identify the right thread, and produce exact `codex resume <session_id>` commands. That became the `session_resume_lookup` task.

### 3. Ops work usually ends in a narrow code or config fix

Your remote debugging sessions were rarely just "look at logs". They usually required:

- identifying the true failure mode
- making a small fix
- preserving compatibility or existing downstream expectations
- avoiding destructive cleanup

That pattern is why the offline suite includes `opc_name_lookup`, and why the live suite includes repo repair and service recovery tasks.

### 4. Docs-first bootstrap matters

You repeatedly want the agent to read `AGENTS.md`, planning docs, and architecture docs before coding. That became `family_snapshot_bootstrap`.

### 5. Supervising Hermes is different from coding directly

You sometimes use Codex as a supervisor: monitor Hermes, judge whether it is stalled, decide whether to intervene, and write the next steering prompt. That is not the same skill as solving a code kata. Because your current local setup is effectively single-slot, this became an offline judgment fixture rather than a live nested-worker benchmark.

## Why There Are Two Suites

### Offline "pseudo" suite

The offline suite is deliberately fixture-based. It simulates the shape of your real work without touching live infrastructure. That makes runs repeatable and directly comparable across models.

It covers:

- session mining and recovery
- local model rollout and Hermes profile switching
- ops-style diagnosis that ends in a code fix
- docs-first bootstrap plus implementation
- worker supervision judgment

### Live SSH suite

The live suite exists because some of your real use cases cannot be honestly measured with offline fixtures alone. You explicitly use the agent to:

- SSH into another host
- inspect system state
- fix repos remotely
- push to Gitea
- recover systemd services
- verify the final running state over the network

That is what the `linux-host` suite measures.

## What The Benchmark Script Does

### Wrapper behavior

The benchmark wrapper first ensures `hermes` is pointed at the intended local model endpoint. It:

- reads the Hermes local config
- resolves the configured model and base URL
- uses the hardened local endpoint updater instead of trusting `hermes config set model.base_url`
- exports benchmark metadata such as model, provider, base URL, profile, and retry settings
- launches either the offline or live runner through `hermes chat -Q --yolo -q`

### Offline runner behavior

For each offline task, the runner:

- copies the fixture into a timestamped run directory
- starts `hermes` in that isolated working directory
- captures the full agent transcript and session id
- runs the task-local `grade.py`
- if the grader fails, resumes the same Hermes session and includes the last grader output in the retry prompt
- saves `agent_output.txt`, `grade_output.txt`, `summary.json`, and `report.md`

This makes the run reproducible while still preserving the way Hermes actually operates.

### Live runner behavior

For each live SSH task, the runner:

- resets the disposable remote task root on `linux-host`
- copies a `remote_seed` repo or task payload to the remote host when needed
- runs any `setup_remote.sh` required to create the intentionally broken starting state
- formats the task prompt with the real remote host, remote path, and when needed a unique Gitea repo name
- runs `hermes`
- runs a local grader that verifies the result by SSHing back into `linux-host`
- saves the same logs and summaries as the offline runner

So the live suite is not just checking local files. It is checking the actual remote host state after Hermes is done.

## What The Offline Tasks Measure

### `session_resume_lookup`

This task checks whether the agent can inspect a set of session files, identify the specific sessions related to:

- `hf download` for the local Qwen rollout
- docker-compose generation for missing Qwen folders

and then emit exact `codex resume <session_id>` commands.

This was included because searching old sessions is part of your actual workflow.

### `qwen_rollout`

This task checks whether the agent can reproduce your local model-infra workflow:

- inspect an existing Qwen 3.5 compose file
- derive a matching Qwen 3.6 compose file
- point Hermes local config at the new model and base URL
- preserve the existing older compose file

### `opc_name_lookup`

This simulates a real operational diagnosis flow where the external system drifted but downstream expectations must stay stable. The task expects a narrow patch, not a redesign.

### `family_snapshot_bootstrap`

This task checks for the docs-first pattern you use in Family Ancestry style work. The agent has to prove it read the right repo docs before it edits code.

### `hermes_supervisor`

This measures judgment rather than direct coding. The agent must inspect logs, background terminal state, and GPU metrics, then write:

- a monitor report
- the exact next prompt it would send to Hermes

without taking over the code edits itself.

## What The Live SSH Tasks Measure

### `linux-host_host_probe`

This is the simplest live task. It verifies that Hermes can:

- SSH to `linux-host`
- confirm basic host facts
- check passwordless sudo with `sudo -n`
- keep local artifacts local while still writing a remote verification marker

This intentionally catches sloppy local-vs-remote path handling.

### `linux-host_repo_fix`

This task seeds a broken disposable repo on `linux-host`, then expects Hermes to:

- read repo instructions
- read the local Gitea usage guide on the remote host
- fix the code with a small patch
- update the README
- commit with the exact required message
- add a unique Gitea SSH remote
- push `main`
- produce a local benchmark report

The grader verifies remote pytest, commit message, remote URL, pushed HEAD equality, README content, and report quality.

### `linux-host_service_recovery`

This task installs a deliberately broken systemd service on `linux-host`. Hermes must:

- diagnose why it fails
- make the smallest reasonable fix
- reload and restart the unit
- verify the service is active and writing heartbeat output
- write a local service report

This measures real SSH/systemd repair behavior.

### `linux-host_status_rollout`

This is the hardest live task. It combines code, Git, deployment, and verification. Hermes must:

- fix a broken remote repo
- run remote pytest
- commit the fix with the exact required message
- push to a unique Gitea repo
- restart a disposable systemd service
- verify the live HTTP JSON payload with `curl`
- write a local deployment report

This is much closer to the kind of remote "fix it and prove it" work you actually care about.

## Why Quality Scoring Was Added

Originally the benchmark only tracked pass/fail. After running the first live suite, that was clearly too generous. A run could pass while still taking awkward operator paths.

So the runners were upgraded to score quality separately from correctness. They now penalize things like:

- interactive sudo prompts
- force-pushes
- amend commits
- failed manual grader runs
- path confusion during recovery
- shell quoting recovery
- benchmark-level retries

That means a model can still solve the task, but the benchmark now distinguishes:

- functional pass
- clean pass
- quality-gate pass

This makes the benchmark much closer to how you actually judge agent performance in practice.

## Summary

The benchmark was designed from your real session history, not from abstract benchmark ideas. The offline suite captures repeatable versions of your common workflows. The live suite captures the real SSH-heavy operational work you actually care about. Together, they test:

- continuity across old sessions
- local model infrastructure work
- docs-first repo behavior
- safe ops diagnosis
- worker supervision judgment
- real remote repo and service operations over SSH

That is why the benchmark performed as a meaningful test of `hermes`, instead of just being another coding puzzle pack.
