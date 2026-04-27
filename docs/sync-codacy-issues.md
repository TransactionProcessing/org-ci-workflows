# Sync Codacy Issues → GitHub Issues

The reusable workflow `sync-codacy-issues.yml` fetches code-quality findings from [Codacy](https://app.codacy.com) and mirrors them as GitHub Issues in the calling repository.  
It handles full lifecycle management: **create**, **update**, **reopen**, and **close** GitHub issues as Codacy findings appear or disappear.

---

## Table of Contents

- [How it works](#how-it-works)
- [Quick start – per-repo caller workflow](#quick-start--per-repo-caller-workflow)
- [Required secrets](#required-secrets)
- [Inputs reference](#inputs-reference)
- [Permissions](#permissions)
- [Duplicate protection (dedupe)](#duplicate-protection-dedupe)
- [Labels used](#labels-used)
- [Sync behaviour in detail](#sync-behaviour-in-detail)
- [Troubleshooting](#troubleshooting)

---

## How it works

```
 ┌─────────────────────────────────────────────────────┐
 │ Calling repository (e.g. MyOrg/my-service)          │
 │  .github/workflows/codacy-sync.yml  (tiny caller)  │
 └──────────────────┬──────────────────────────────────┘
                    │  workflow_call  (secrets: inherit)
                    ▼
 ┌─────────────────────────────────────────────────────────────────┐
 │ TransactionProcessing/org-ci-workflows                          │
 │  .github/workflows/sync-codacy-issues.yml  (reusable workflow) │
 │                                                                 │
 │  1. POST /api/v3/.../issues/search  → Codacy API (paginated)   │
 │  2. Ensure labels exist in calling repo                         │
 │  3. Load existing issues (open + closed) tagged `codacy`        │
 │  4. For each Codacy finding:                                    │
 │       • NEW  → create GitHub issue                              │
 │       • CHANGED / CLOSED → update / reopen GitHub issue         │
 │       • NO LONGER IN CODACY → close GitHub issue               │
 └─────────────────────────────────────────────────────────────────┘
```

---

## Quick start – per-repo caller workflow

Copy the file [`docs/examples/caller-codacy-sync.yml`](examples/caller-codacy-sync.yml) to `.github/workflows/codacy-sync.yml` in your repository and commit it.  
No other changes are required in the target repository.

```yaml
# .github/workflows/codacy-sync.yml  (place this in YOUR repository)
name: Codacy Issue Sync

on:
  schedule:
    - cron: "0 6 * * 1-5"   # 06:00 UTC, Monday – Friday
  workflow_dispatch:          # allow manual trigger

jobs:
  sync:
    uses: TransactionProcessing/org-ci-workflows/.github/workflows/sync-codacy-issues.yml@main
    secrets: inherit          # forwards CODACY_API_TOKEN org secret automatically
```

To customise further, pass optional inputs:

```yaml
jobs:
  sync:
    uses: TransactionProcessing/org-ci-workflows/.github/workflows/sync-codacy-issues.yml@main
    secrets: inherit
    with:
      codacy_org:      "MyOrg"           # Codacy org slug (default: GitHub owner)
      codacy_repo:     "my-service"      # Codacy repo slug (default: GitHub repo name)
      severity_filter: "Error,Warning"   # omit Info-level issues
      extra_labels:    "needs-triage"    # add a custom label to every synced issue
      assignees:       "alice,bob"       # auto-assign to team members
      dry_run:         false             # set to true to preview without writing
```

---

## Required secrets

| Secret | Where to set | Notes |
| --- | --- | --- |
| `CODACY_API_TOKEN` | GitHub **Organisation** secret | Read-only Codacy API token. Created under *Codacy → Organisation → Integrations → API tokens*. Needs at minimum **read** access to repositories. |

Because the caller uses `secrets: inherit`, the org-level secret is automatically forwarded to the reusable workflow.  
No repository-level secret configuration is needed in the calling repo.

> **Note:** `GITHUB_TOKEN` is used automatically for all GitHub Issues API calls.  
> No additional GitHub token secret is required.

---

## Inputs reference

| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `codacy_org` | `string` | No | GitHub repo owner | Codacy organisation slug exactly as shown in the Codacy URL: `https://app.codacy.com/gh/<codacy_org>/...` |
| `codacy_repo` | `string` | No | GitHub repo name | Codacy repository slug exactly as shown in the Codacy URL: `https://app.codacy.com/gh/<org>/<codacy_repo>/...` |
| `severity_filter` | `string` | No | `"Error,Warning,Info"` | Comma-separated list of Codacy severity levels to include. Valid values: `Error`, `Warning`, `Info`. |
| `extra_labels` | `string` | No | `""` | Comma-separated list of additional labels to apply to every synced issue (in addition to the always-present `codacy` label and the auto-detected category label). |
| `assignees` | `string` | No | `""` | Comma-separated GitHub usernames to assign to every newly created issue. |
| `dry_run` | `boolean` | No | `false` | When `true`, log all planned actions without making any changes to GitHub issues. Useful for testing. |

---

## Permissions

The reusable workflow sets the following minimal permissions on `GITHUB_TOKEN`:

```yaml
permissions:
  issues: write    # create / update / close GitHub issues and labels
  contents: read   # required by the runner environment
```

These are inherited from the **calling** repository's `GITHUB_TOKEN`.  
Make sure the repository's *Settings → Actions → General → Workflow permissions* is set to **"Read and write permissions"** or that the calling workflow explicitly grants `issues: write`.

---

## Duplicate protection (dedupe)

Every GitHub issue created by this workflow contains a hidden HTML comment at the very top of the issue body:

```html
<!-- codacy-dedupe:<codacy-issue-id> -->
```

where `<codacy-issue-id>` is the stable, unique ID assigned by Codacy to each finding.

**Before creating any new issue**, the workflow:
1. Lists **all** existing issues in the repository (both `open` and `closed`) that carry the `codacy` label.
2. Parses the `<!-- codacy-dedupe:… -->` marker from each issue body.
3. Builds an in-memory map of `dedupeKey → existing GitHub issue`.
4. Only creates a new GitHub issue if the dedupe key is **not** already present in that map.

This means:
- Re-running the workflow never creates duplicates.
- Manually closing a synced issue will cause it to be **reopened** on the next run if Codacy still reports the finding (see [sync behaviour](#sync-behaviour-in-detail)).
- Manually deleting a synced issue **without** removing the label/marker will allow a fresh issue to be created on the next run.

---

## Labels used

The workflow creates and manages the following labels in the target repository (idempotent – existing labels are left unchanged):

| Label | Colour | When applied |
| --- | --- | --- |
| `codacy` | Blue `#0075ca` | **All** issues created by this workflow (used for ownership tracking) |
| `bug` | Red `#b60205` | Codacy category matches `bug`, `error`, `errorprone`, `error-prone` |
| `security` | Dark-red `#d93f0b` | Codacy category matches `security` |
| `performance` | Orange `#e57504` | Codacy category matches `performance` |
| `refactor` | Pink `#eb69a2` | Codacy category matches `refactor`, `best-practice`, `code-style`, `complexity`, `unused-code` |
| `task` | Green `#0e8a16` | Issue message contains the word `todo` |

You can add further labels per run via the `extra_labels` input.

---

## Sync behaviour in detail

On each run the workflow reconciles the full set of Codacy findings against GitHub issues:

| Situation | Action |
| --- | --- |
| Codacy finding exists, no matching GitHub issue | **Create** new GitHub issue |
| Codacy finding exists, matching GitHub issue is **open** and unchanged | Skip (no change) |
| Codacy finding exists, matching GitHub issue is **open** but title/body changed | **Update** the existing issue |
| Codacy finding exists, matching GitHub issue is **closed** | **Reopen** and update the issue |
| Codacy finding no longer reported, matching GitHub issue is **open** | **Close** the issue with an explanatory comment |
| Codacy finding no longer reported, matching GitHub issue is **closed** | No action (already closed) |

The workflow **only manages issues it owns** (identified by the `codacy` label **and** the `<!-- codacy-dedupe:… -->` marker).  
Hand-crafted issues or issues from other workflows are never touched.

---

## Troubleshooting

**`CODACY_API_TOKEN secret is not set`**  
→ Ensure the `CODACY_API_TOKEN` secret exists at the **organisation** level in GitHub and that the calling repository has access to it (*Settings → Secrets and variables → Actions → Organisation secrets*).

**`Codacy API responded with 404`**  
→ The `codacy_org` / `codacy_repo` slugs do not match what Codacy expects.  
Open `https://app.codacy.com/gh/<org>/<repo>/dashboard` and copy the exact slugs from the URL.

**`Codacy API responded with 401`**  
→ The `CODACY_API_TOKEN` is invalid or has expired. Regenerate it in Codacy and update the secret.

**No issues created even though Codacy reports findings**  
→ Run with `dry_run: true` first to inspect the planned actions in the workflow logs.  
Also check `severity_filter` – the default includes `Error,Warning,Info` but the finding level in Codacy must exactly match one of the listed values (case-sensitive).

**GitHub issues are not being closed for resolved findings**  
→ Confirm the old issues have the `codacy` label and contain the `<!-- codacy-dedupe:… -->` marker. Issues without the marker are ignored by the close step.
