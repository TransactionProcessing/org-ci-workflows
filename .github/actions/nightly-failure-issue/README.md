# Nightly failure issue action

Composite action for nightly workflows that creates one open issue per failure class and comments on the existing issue on later runs.

## What it does

- Searches open issues for a hidden dedupe marker.
- Comments on the existing issue if it already exists.
- Creates a new issue only when no matching open issue is found.

## Inputs

- `github_token`: token with `issues:write`
- `title`: issue title
- `labels`: comma-separated labels
- `body`: repo-specific failure summary
- `dedupe_key`: stable key for duplicate detection
- `comment_prefix`: prefix used when adding follow-up comments

## Usage

```yaml
- name: Capture failure context
  if: ${{ failure() }}
  run: |
    {
      echo "ISSUE_BODY<<EOF"
      echo "Repo: ${{ github.repository }}"
      echo "Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
      echo "Commit: ${{ github.sha }}"
      echo "Failure URL: ${{ env.action_url }}"
      echo "EOF"
    } >> "$GITHUB_ENV"

- name: Notify on nightly failure
  if: ${{ failure() }}
  uses: TransactionProcessing/org-ci-workflows/.github/actions/nightly-failure-issue@main
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    title: Investigate Nightly Build Failure
    labels: nightlybuild
    dedupe_key: nightly-build
    body: ${{ env.ISSUE_BODY }}
```

## Notes

- Keep repo-specific build logic in the calling workflow.
- Use a stable `dedupe_key` so duplicate detection survives title changes.