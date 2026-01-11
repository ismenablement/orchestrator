# Orchestrator Workflow Using Trigger and Wait 

## Problem Statement
A simple orchestrator workflow to manage and coordinate builds across multiple repositories.

The goal of this project is to test the ability to run an orchestrator flow that triggers `workflow_dispatch` workflows in repoA/B/C/D/E in a specific order, with dependencies and coordination.

The setup also includes an invalidation mechanism: when a PR is created or updated in any repo, it invalidates all related PRs across repos (e.g., on DPTA-**** branches), treating them as a coordinated 'transactional/monolithic' change. This ensures they must be merged together or not at all, preventing inconsistent or partial merges."

### Repo Structure:
- **Orchestrator** - Contains the orchestration logic and workflows.
- **RepoA-E** - Contains application code (simulated with text files for testing).

In a real-world scenario, repoA-E would hold Java code requiring builds. The build processes must be executed in a specific order (e.g., repoA first, then repoB/C/D in parallel, then repoE). 

The orchestrator triggers `workflow_dispatch` in each repo and monitors build status via GitHub API polling. A reusable **TriggerAndWait** workflow handles this logic.

The main orchestrator workflow lives in the **Orchestrator** repo, with simple logic defining sequential and/or parallel execution order using `needs:` dependencies.

NOTE: Adding "exit 1" status in any of the repos build.yml can "simulate" failure of the build process and show that merging will remain blocked!

### Notes About Alternatives:
- **PoC for `workflow_call`**: A separate PoC will explore calling build workflows from repoA-E directly via `workflow_call` (instead of `workflow_dispatch`).  
- **Event-driven pipelines**: Using `workflow_run` to detect job completion and `repository_dispatch` to notify the orchestrator will be evaluated last, as it adds complexity.

---

## Workflow Logic 

### Repos A-E Specifics

#### Configuration at Repo Level:
- **Branch Protection Ruleset**: Requires the `orchestrator-build` status check to pass before merging to `main`. This blocks merges until the orchestrator posts a successful status.
- **PAT Token (`PAT_WORKFLOWS`)**: A Personal Access Token with full repository permissions (including `repo`, `workflow`, `statuses:write`) is used for cross-repo API calls. In production, scope this token more narrowly.

#### Workflows in Each Repo:
1. **`build.yml`**: 
   - **Trigger**: `workflow_dispatch` (manually or by the orchestrator).
   - **Purpose**: Simulates the build process for the application. Runs only when explicitly triggered (not on PR events).
   
2. **`invalidate-pr.yml`**: 
   - **Trigger**: `pull_request` (types: `opened`, `synchronize`).
   - **Purpose**: When a PR is created or updated, this workflow dispatches an `invalidate` event to the orchestrator. The orchestrator then invalidates all open PRs on `DPTA-****` branches (e.g., by removing or marking the `orchestrator-build` status as pending/failed), ensuring they cannot be merged until the orchestrator runs successfully again.
   
3. **`post-orchestrator.yml`**: 
   - **Trigger**: `repository_dispatch` (type: `orchestrate-success`).
   - **Purpose**: Triggered by the orchestrator after a successful build. Posts the `orchestrator-build` commit status to the PR's head SHA, satisfying the branch protection rule and enabling the merge button.

---

### Workflows in the Orchestrator Repo

1. **`orchestrate.yml`** (Main Workflow):
   - **Trigger**: `workflow_dispatch` (manual, requires `branch` input like `DPTA-1122`).
   - **Purpose**: Defines the order of execution for builds across repoA-E using `needs:` dependencies (e.g., repoA → repoB/C/D in parallel → repoE).
   - **Final Step**: `post-statuses` job runs after all builds succeed. It dispatches `orchestrate-success` events to each repo with an open PR on the specified branch, triggering their `post-orchestrator.yml` workflows to post the success status.

2. **`trigger-and-wait.yml`** (Reusable Workflow):
   - **Trigger**: `workflow_call` (invoked by `orchestrate.yml`).
   - **Purpose**: 
     - Checks if the specified branch exists in the target repo.
     - If the branch exists, triggers the repo's `build.yml` workflow via `workflow_dispatch`.
     - Polls the GitHub API to wait for the build to complete (success/failure).
     - If the branch doesn't exist, skips the build (shows as "skipped" in logs).
   - **Inputs**: `repo`, `branch`, `workflow`, `run_id`, `run_timeout_seconds`.

3. **`orchestrate-invalidate.yml`**:
   - **Trigger**: `repository_dispatch` (type: `invalidate`).
   - **Purpose**: Central invalidation logic. When a PR is opened/updated in any repoA-E, their `invalidate-pr.yml` dispatches here. This workflow then invalidates all open PRs on `DPTA-****` branches across all repos (e.g., by removing or resetting the `orchestrator-build` status), preventing merges until the orchestrator runs successfully.

4. **`test-post-statuses.yml`** (Optional, for Testing):
   - **Trigger**: `workflow_dispatch` (manual, takes `branch` and `repo` inputs).
   - **Purpose**: Simulates the `post-statuses` step by dispatching `orchestrate-success` to a single repo. Useful for quick testing without running the full orchestrator.
   
   **Example Command**:
   ```bash
   cd /Users/e1083457/Repos/orchestrator && gh workflow run test-post-statuses.yml --ref main -f branch=DPTA-1122 -f repo=ismenablement/repoB
   ```

---

## Workflow Flow (End-to-End)

### Scenario: Creating a PR and Merging via Orchestrator

1. **Developer Creates PR**:
   - A developer creates a PR from branch `DPTA-1122` to `main` in repoB and repoD.
   - The PR triggers `invalidate-pr.yml` in repoB (and also repoD), which dispatches `invalidate` to the orchestrator.
   - The orchestrator's `orchestrate-invalidate.yml` runs, invalidating all `DPTA-****` PRs (e.g., by clearing the `orchestrator-build` status or marking it as pending).
   - The PR cannot be merged yet (blocked by branch protection).
   - The "invalidate" logic ensures that when a PR is created or updated in any repo (e.g., repoB), it triggers invalidation across all repos (A-E) for related branches (e.g., DPTA-****).
   - This treats the PRs as a "transactional/monolithic change"—meaning they must be coordinated and merged together (or not at all), rather than independently. If one PR changes, all related ones are invalidated to prevent inconsistent or partial merges.

1. **Running the Orchestrator**:
   - A user manually triggers `orchestrate.yml` from the orchestrator repo's Actions tab, entering `DPTA-1122` as the branch.
   - The orchestrator runs builds in the defined order:
     - `repoA-build` → triggers repoA's `build.yml` on `DPTA-1122` (or skips if no such branch exists).
     - `repoB-build`, `repoC-build`, `repoD-build` → run in parallel (needs `repoA-build`).
     - `repoE-build` → runs after B/C/D complete.
   - If any build fails, the orchestrator stops.

2. **Post-Statuses Step in the Orchestrator**:
   - After all builds succeed, the `post-statuses` job runs.
   - It loops over repoA-E, finds PRs with head branch `DPTA-1122`, and dispatches `orchestrate-success` to each repo with the PR's head SHA.

3. **Posting Success Status**:
   - Each repo's `post-orchestrator.yml` receives the dispatch and posts a commit status (`state=success`, `context=orchestrator-build`) to the PR's head SHA.
   - The branch protection rule is satisfied, and the merge button is enabled.

4. **Merging the PR**:
   - The developer merges the PR to `main`.
   - After merging, they run:
     ```bash
     git checkout main && git pull
     git branch -d DPTA-1122  # Delete local branch
     git push origin --delete DPTA-1122  # Delete remote branch
     ```

---

## Key Features

- **Selective Builds**: Only repos with the specified branch (e.g., `DPTA-1122`) are built; others are skipped.
- **Branch Protection**: PRs cannot be merged without a successful `orchestrator-build` status, ensuring coordinated releases.
- **Invalidation**: Any PR change invalidates all related PRs, preventing stale merges.
- **Modular Workflows**: Separate workflows for building, invalidating, and posting statuses keep logic clean and maintainable.

---

## Prerequisites

- **GitHub CLI (`gh`)**: Used in workflows for API calls and PR creation. Install from [cli.github.com](https://cli.github.com/).
- **PAT Token (`PAT_WORKFLOWS`)**: A Personal Access Token with `repo`, `workflow`, and `statuses:write` scopes, stored as a secret in each repo and the orchestrator repo.
- **Branch Protection Rulesets**: Configured in each repoA-E to require `orchestrator-build` before merging to `main`.