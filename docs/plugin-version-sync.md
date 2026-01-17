# Plugin Version Sync Workflow

This guide explains how to set up automated version synchronization for Claude Code plugins. When you publish a GitHub release, the workflow automatically updates version numbers in your plugin files and notifies the marketplace.

## Architecture

```
Plugin Repo                          Marketplace Repo
┌──────────────────────┐             ┌───────────────────────────────┐
│ 1. Create release    │             │                               │
│    (v0.2.0)          │             │                               │
│         │            │             │                               │
│         v            │             │                               │
│ 2. release.yml runs  │             │                               │
│    - Updates version │             │                               │
│      in plugin.json, │             │                               │
│      SKILL.md        │             │                               │
│    - Commits changes │             │                               │
│         │            │             │                               │
│         v            │  dispatch   │                               │
│ 3. Sends             │────────────>│ 4. update-plugin.yml runs     │
│    repository_dispatch             │    - Updates marketplace.json │
│                      │             │      with new version         │
└──────────────────────┘             └───────────────────────────────┘
```

## Workflow Template

Add this workflow to your plugin repository at `.github/workflows/release.yml`:

```yaml
name: Release Plugin

on:
  release:
    types: [published]

permissions:
  contents: write

jobs:
  sync-version:
    name: Sync version files
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.release.target_commitish }}

      - name: Extract version from tag
        id: version
        run: |
          VERSION="${GITHUB_REF_NAME#v}"
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"

      - name: Update plugin.json
        run: |
          jq --arg v "${{ steps.version.outputs.version }}" \
             '.version = $v' \
             .claude-plugin/plugin.json > tmp && mv tmp .claude-plugin/plugin.json

      - name: Update SKILL.md frontmatter
        run: |
          # Adjust the path to match your skill location
          sed -i "s/^version: .*/version: ${{ steps.version.outputs.version }}/" \
            skills/<your-skill>/SKILL.md

      - name: Commit version updates
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git diff --quiet || (git add -A && git commit -m "chore: sync version to ${{ steps.version.outputs.version }}" && git push)

  notify-marketplace:
    name: Notify marketplace
    runs-on: ubuntu-latest
    needs: sync-version
    steps:
      - name: Trigger marketplace update
        run: |
          VERSION="${GITHUB_REF_NAME#v}"
          curl -X POST \
            -H "Authorization: Bearer ${{ secrets.MARKETPLACE_PAT }}" \
            -H "Accept: application/vnd.github+json" \
            https://api.github.com/repos/thirteen37/claude-code-plugins/dispatches \
            -d "{\"event_type\":\"plugin-release\",\"client_payload\":{\"plugin_name\":\"<your-plugin>\",\"version\":\"$VERSION\"}}"
```

## PAT Setup

The workflow requires a Personal Access Token (PAT) to trigger the marketplace update.

### Creating a Fine-Grained PAT

1. Go to GitHub Settings > Developer Settings > Personal Access Tokens > Fine-grained tokens
2. Click "Generate new token"
3. Configure:
   - **Token name**: `marketplace-dispatch` (or similar)
   - **Expiration**: Choose based on your needs (recommend 1 year)
   - **Repository access**: Select "Only select repositories" and choose `claude-code-plugins`
   - **Permissions**: Under "Repository permissions", set:
     - **Contents**: Read and write (required for repository_dispatch)
4. Generate and copy the token

### Adding the Secret

1. Go to your plugin repository's Settings > Secrets and variables > Actions
2. Click "New repository secret"
3. Name: `MARKETPLACE_PAT`
4. Value: Paste your PAT
5. Click "Add secret"

## Checklist for New Plugins

When adding version sync to a new plugin:

- [ ] Copy the `release.yml` template to `.github/workflows/release.yml`
- [ ] Update `plugin_name` in the dispatch payload to match your plugin's directory name in the marketplace
- [ ] Adjust version file paths:
  - `plugin.json` path (usually `.claude-plugin/plugin.json`)
  - `SKILL.md` path (e.g., `skills/your-skill/SKILL.md`)
- [ ] Add the `MARKETPLACE_PAT` secret to your repository
- [ ] Ensure your plugin is registered in the marketplace's `marketplace.json`

## Creating a Release

1. Go to your plugin repository on GitHub
2. Click "Releases" in the right sidebar
3. Click "Draft a new release"
4. Create a new tag with `v` prefix (e.g., `v0.2.0`)
5. Fill in release title and notes
6. Click "Publish release"

The workflow will automatically:
1. Update version in `plugin.json` and `SKILL.md`
2. Commit and push the changes
3. Notify the marketplace to update its version listing

## Troubleshooting

### Workflow not triggering
- Ensure the release is published, not just drafted
- Check that the workflow file is on the default branch

### Version not updating
- Verify the paths in the workflow match your actual file locations
- Check the workflow logs in GitHub Actions for errors

### Marketplace not updating
- Verify `MARKETPLACE_PAT` secret is set correctly
- Check that the PAT has not expired
- Ensure `plugin_name` matches the directory name in marketplace.json
- Check the marketplace repo's Actions tab for the `update-plugin.yml` run
