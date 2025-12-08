# SRR-template-editor

**Template management tool consuming shared templates from SRR-shared-resources.**

This repository receives automatic updates of Jinja2, TextFSM, TTP, JSNAPy, and table view templates from the central shared resources repository.

---

## What This Repo Does

Template editing and management tool that uses shared templates for:
- Jinja2 configuration templates
- TextFSM parsing templates
- TTP (Template Text Parser) templates
- JSNAPy test templates
- Table view configurations

**Shared folders synced from SRR-shared-resources:**
- `jinja2/` → `template/jinja2/`
- `textfsm/` → `template/textfsm/`
- `ttp/` → `template/ttp/`
- `jsnapy/` → `template/jsnapy/`
- `tableview/` → `template/tableview/`

---

## Repository Structure

```
SRR-template-editor/
├── .github/
│   └── workflows/
│       ├── auto-pr-shared-resources.yml         # [Auto] Creates PR for downstream updates
│       └── trigger-shared-resource-update.yml   # [Auto] Sends changes upstream
├── template/                                    # Synced folders
│   ├── jinja2/                                  # Jinja2 templates (synced)
│   ├── textfsm/                                 # TextFSM parsers (synced)
│   ├── ttp/                                     # TTP templates (synced)
│   ├── jsnapy/                                  # JSNAPy tests (synced)
│   └── tableview/                               # Table views (synced)
└── [other template-editor specific files]
```

---

## How Sync Works

### Receiving Updates (Downstream: Shared → This Repo)

1. Shared resources repo pushes update to any template folder
2. This repo's `sync-auto` branch automatically updated
3. Auto-PR created: `sync-auto` → `main`
4. **Your job:** Review PR, check tests pass, merge

**Time:** PR appears ~60 seconds after shared repo push

### Sending Changes Upstream (This Repo → Shared)

1. Edit files in `template/jinja2/`, `textfsm/`, `ttp/`, `jsnapy/`, or `tableview/`
2. Commit to `main` (NO `[auto-sync]` tag)
3. Workflow detects change, sends webhook to shared repo
4. Shared repo creates PR with your changes
5. Shared repo maintainer reviews and merges
6. Your changes propagate to other consumers

**Time:** PR in shared repo appears ~45 seconds after your push

---

## Making Changes

### Editing Synced Templates

```bash
# 1. Edit template
vim template/jinja2/interface-config.j2

# 2. Commit and push to main
git add template/jinja2/interface-config.j2
git commit -m "Add VLAN support to interface template"
git push origin main

# 3. Workflow sends change to shared repo automatically
# 4. Wait for PR in shared repo
# 5. After merge there, change propagates to other consumers
```

### What NOT To Do

❌ **Never commit to `sync-auto` branch**
- This branch is force-pushed by shared repo
- Your commits will be lost on next sync

❌ **Don't delete synced folders**
- They'll reappear on next sync
- Update `consumer-config.yml` in shared repo instead

❌ **Don't bypass workflows**
- Commits with `[auto-sync]` tag won't trigger upstream sync
- This prevents infinite loops

---

## Workflows

### auto-pr-shared-resources.yml
**Trigger:** Push to `sync-auto` branch
**Function:** Creates PR when shared repo syncs updates
**Output:** PR from `sync-auto` → `main`

### trigger-shared-resource-update.yml
**Trigger:** Push to `main` affecting template folders
**Function:** Sends webhook to shared repo
**Output:** PR created in shared repo with your changes

**Path filters configured:**
```yaml
paths:
  - 'template/jinja2/**'
  - 'template/textfsm/**'
  - 'template/ttp/**'
  - 'template/jsnapy/**'
  - 'template/tableview/**'
```

Changes outside these paths don't trigger upstream sync.

---

## Branch Strategy

### Permanent Branches
- `main` - Production

### Automation Branch
- `sync-auto` - Receives updates from shared repo, force-pushed on every sync
  - **DO NOT commit manually**
  - Source branch for auto-created PRs
  - Wiped and rebuilt on every sync

---

## Required Secrets

### GITHUB_TOKEN
**Type:** Built-in token
**Setup:** Settings → Actions → General
- ✅ "Read and write permissions"
- ✅ "Allow GitHub Actions to create and approve pull requests"

**Used by:** `auto-pr-shared-resources.yml` to create PRs

### SHARED_REPO_PAT
**Type:** Personal Access Token
**Access:** SRR-shared-resources repository
**Permissions:** Write access for `repository_dispatch`

**Used by:** `trigger-shared-resource-update.yml` to send webhooks

---

## Reviewing Downstream PRs

When PR appears from `sync-auto` → `main`:

**Check:**
1. Files changed match expected template folders
2. No unexpected deletions
3. CI/tests pass (template syntax validation)
4. Diff looks reasonable

**Don't worry about:**
- Commit history (snapshot-based, no history from shared repo)
- Merge conflicts (impossible with force-push strategy)
- Multiple commits (always single commit per sync)

**Merge if:** Tests pass and changes look correct

---

## Troubleshooting

### PR Shows No Changes
**Cause:** Folder content unchanged since last sync
**Fix:** Close PR, not a bug

### Changes Not Triggering Upstream Sync
**Cause:** Changed files outside `template/` subfolders
**Fix:** Only changes to jinja2, textfsm, ttp, jsnapy, tableview trigger upstream sync

### Workflow Permission Error (403)
**Cause:** GITHUB_TOKEN lacks permissions
**Fix:** Settings → Actions → General → Enable "Read and write" + "Allow PR creation"

### Upstream Sync Webhook Failed
**Cause:** SHARED_REPO_PAT invalid or expired
**Fix:** Regenerate token, update secret in Settings → Secrets

**Full troubleshooting:** See `ai_spec_v4/07_TROUBLESHOOTING.md` in demo folder

---

## Loop Prevention

Automated commits from shared repo are tagged:
```
[auto-sync][template-editor] update shared folders

Source: moophat/SRR-shared-resources@abc123
```

`trigger-shared-resource-update.yml` skips commits containing `[auto-sync]` - prevents infinite loops.

---

## File Deletions

**Deletions in shared repo propagate automatically** to this repo.

Example: Shared repo deletes `jinja2/old-template.j2` → deleted here on next sync.

This is intentional (keeps consumers in sync with shared repo state).

---

## Adding New Synced Folder

1. Update `.github/consumer-config.yml` in shared repo:
   ```yaml
   template-editor:
     folders:
       jinja2: template/jinja2
       textfsm: template/textfsm
       ttp: template/ttp
       jsnapy: template/jsnapy
       tableview: template/tableview
       new-folder: template/new-folder  # Add this
   ```

2. Update `trigger-shared-resource-update.yml` in this repo:
   ```yaml
   paths:
     - 'template/jinja2/**'
     - 'template/textfsm/**'
     - 'template/ttp/**'
     - 'template/jsnapy/**'
     - 'template/tableview/**'
     - 'template/new-folder/**'  # Add this
   ```

3. Push to shared repo `main` - new folder syncs automatically

---

## Template Validation

**Recommended:** Add CI checks to validate template syntax:
- Jinja2: Check for syntax errors
- TextFSM: Validate template structure
- TTP: Verify template parsing
- JSNAPy: YAML syntax validation

This catches errors before they propagate to other consumers.

---

## Technical Specs

**Full system documentation:** `ai_spec_v4/` folder in demo repository

---

**Questions? Check shared resources repo or ai_spec_v4/00_OVERVIEW_v4.md**
