# Sync Codacy Issues -> GitHub Issues

The reusable workflow `sync-codacy-issues.yml` fetches code-quality findings from [Codacy](https://app.codacy.com) and mirrors them as GitHub Issues in the calling repository.

It now uses a Neon Postgres database as the source of truth for sync state, so duplicate detection does not depend on issue labels or hidden body markers.

---

## How it works

```text
Calling repository
  .github/workflows/codacy-sync.yml
        |
        v
Reusable workflow
  .github/workflows/sync-codacy-issues.yml
        |
        +--> Codacy API: fetch current findings
        +--> Neon Postgres: read/write sync rows
        +--> GitHub Issues API: create, update, reopen, close
```

For each finding, the workflow:

1. Looks up the finding in Neon using the composite key:
   - `github_repository`
   - `codacy_org`
   - `codacy_repo`
   - `codacy_issue_id`
2. Creates the GitHub issue only if no row exists.
3. Updates or reopens the existing GitHub issue if the finding still exists.
4. Closes the GitHub issue and marks the row inactive when Codacy no longer reports it.

The first successful run bootstraps any previously created Codacy issues from the target repo into Neon, so the migration from the old label-based approach is automatic.

---

## Quick start

Copy [`docs/examples/caller-codacy-sync.yml`](examples/caller-codacy-sync.yml) to `.github/workflows/codacy-sync.yml` in your repository.

You must provide:

- `CODACY_API_TOKEN` as an organisation secret
- `NEON_DATABASE_URL` as a secret pointing at your Neon Postgres database

The caller can keep using `secrets: inherit` if both secrets are available to that repository.

---

## Required secrets

| Secret | Where to set | Notes |
| --- | --- | --- |
| `CODACY_API_TOKEN` | GitHub organisation secret | Read-only Codacy API token with access to the target Codacy repository. |
| `NEON_DATABASE_URL` | GitHub organisation secret or repository secret | Neon Postgres connection string, typically from the Neon console. |

---

## Inputs reference

| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `codacy_org` | `string` | No | GitHub repo owner | Codacy organisation slug. |
| `codacy_repo` | `string` | No | GitHub repo name | Codacy repository slug. |
| `severity_filter` | `string` | No | `"Error,High,Warning,Info"` | Comma-separated Codacy severity levels to include. |
| `extra_labels` | `string` | No | `""` | Optional labels to add to synced issues. These are metadata only and are not used for dedupe. |
| `assignees` | `string` | No | `""` | GitHub usernames to assign to newly created issues. |
| `dry_run` | `boolean` | No | `false` | When `true`, logs planned actions without mutating GitHub or Neon. |

---

## Neon schema

The workflow creates one table:

```sql
create table codacy_issue_sync (
  github_repository text not null,
  codacy_org text not null,
  codacy_repo text not null,
  codacy_issue_id text not null,
  github_issue_number bigint not null,
  active boolean not null default true,
  first_seen_at timestamptz not null default now(),
  last_seen_at timestamptz not null default now(),
  closed_at timestamptz,
  updated_at timestamptz not null default now(),
  primary key (github_repository, codacy_org, codacy_repo, codacy_issue_id)
);
```

This means one Neon database can safely track multiple GitHub repositories and multiple Codacy repository mappings at the same time.

---

## Issue body

Each synced GitHub issue includes:

- the Codacy issue link
- GitHub repository name
- Codacy org/repo
- file path
- line number
- rule ID
- severity
- code snippet when Codacy provides one
- a visible sync metadata block for humans

The workflow does not rely on hidden comments for identity.

---

## Sync behaviour

| Situation | Action |
| --- | --- |
| Codacy finding exists, no database row exists | Create GitHub issue and insert Neon row |
| Codacy finding exists, database row exists and GitHub issue is open | Update only if title/body/labels changed |
| Codacy finding exists, database row exists and GitHub issue is closed | Reopen and update |
| Codacy finding no longer reported, database row exists | Close GitHub issue and mark row inactive |
| Codacy finding no longer reported, row already inactive | No action |

---

## Labels

Labels are classification metadata only.

The workflow creates and maintains a small managed label set, but labels are no longer used for identity or dedupe.

---

## Troubleshooting

**`Missing NEON_DATABASE_URL`**
-> Add the Neon connection string as a secret and forward it to the reusable workflow.

**`Codacy API responded with 401`**  
-> The `CODACY_API_TOKEN` is invalid or expired.

**No issues are created even though Codacy reports findings**
-> Check `severity_filter`, `codacy_org`, and `codacy_repo`.

**Existing issues were duplicated after switching to Neon**
-> The bootstrap step could not parse the old issue body or labels. Confirm the previous issues still contain the Codacy ID in the body or `codacy:` label.
