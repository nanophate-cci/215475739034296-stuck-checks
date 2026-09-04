# Repro: GitHub checks pending because WFC never completes

Intercom **215475739034296** (Rox / Nayan). CircleCI workflows stay `running` (`stopped_at: null`) after every job has failed, so GitHub App checks never leave pending.

## What Rox actually hit

- Org `cb8602b7-d49c-4a91-89af-ea314fa749ea`, GitHub App project `circleci/S8esR1eZKbXdej8RAT82JM/R3a9Vi5xKr6UwMUbrneZS`
- Example: tests_ci_pr `137b3107-1b7e-4711-9bd0-bfb290a5de4f` — jobs failed, workflow still running
- WFC Unblock Job for `post-test-results` / `post-eval-results`: permissions-service returned `:not-available` for actor `a78f29da-2ccc-4c1c-89ec-a86ceb16a654` on context `84ed812d-d131-40bb-821a-990f2c5e830e` (`run-context`)
- After durable-queue retries, `jobs-unblocked` is dropped (`retry-limit-exceeded`). Workflow status is `error` internally but API/UI stay `running`. No `workflowCompleted` to GitHub.
- Tracking: PIPE-9769, PIPE-9720

Merging main "fixes" it because a **new actor** starts a new pipeline; the stuck checks are on the old SHA.

## This repo

Mirrors the job graph (failing job, then a **context** job that still runs on failed/canceled/success) plus a context-only `lint` workflow.

Two trigger paths:

1. **Human PR** (`nanophate`) — control. If the domain-service profile is warm, context jobs should start and the workflow should reach a terminal status.
2. **GitHub Actions bot PR** (`.github/workflows/auto-pr.yml`) — attempts the `:not-available` path. `github-actions[bot]` often has no usable CircleCI user profile, which is how permissions-service returns `not_available`.

Success criteria: workflow `status=running` and `stopped_at=null` after every job is `failed`, and the GitHub check stays pending.


## Context

Uses existing org context `dev-context` (`72dd40a5-33a6-4d54-b172-13086549ebf6`). Creating a dedicated `repro-stuck-checks` context returned 403 from `/api/v2/context` with this CLI token.
