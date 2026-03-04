# Quick Setup Checklist

Complete these steps to enable cross-repository release triggering from this public repo to your private repo.

## 📋 Setup Steps

### ✅ Step 1: Create Personal Access Token
- [ ] Go to GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens
- [ ] Click "Generate new token"
- [ ] Name it: `public-to-private-dispatch`
- [ ] Set expiration (recommend: 1 year)
- [ ] Repository access: Select only your **private repository**
- [ ] Permissions: Repository → Contents → Read and write
- [ ] Generate and **copy the token** (you won't see it again!)

### ✅ Step 2: Add Secret to This Public Repo
- [ ] Navigate to this repo on GitHub
- [ ] Go to Settings → Secrets and variables → Actions
- [ ] Click "New repository secret"
- [ ] Name: `PRIVATE_REPO_DISPATCH_TOKEN`
- [ ] Value: Paste the PAT from Step 1
- [ ] Click "Add secret"

### ✅ Step 3: Configure Public Repo Workflow
- [ ] Edit [.github/workflows/trigger-private-release.yml](.github/workflows/trigger-private-release.yml)
- [ ] Replace `YOUR_ORG/YOUR_PRIVATE_REPO` with your actual private repository
  - Format: `organization-name/repository-name`
  - Example: `my-company/licensed-builds`
- [ ] Commit and push to main branch

### ✅ Step 4: Create Workflow in Private Repo
- [ ] Copy [.github/PRIVATE_REPO_WORKFLOW_TEMPLATE.yml](.github/PRIVATE_REPO_WORKFLOW_TEMPLATE.yml) to your private repository
- [ ] Rename it to: `.github/workflows/licensed-release.yml`
- [ ] Customize the build/publish steps for your needs
- [ ] Commit to the default branch (main/master)

### ✅ Step 5: Add Secrets to Private Repo
Add these secrets in your **private repository** (Settings → Secrets and variables → Actions):

- [ ] `CODE_SIGNING_KEY` - Your signing certificate/key
- [ ] `SIGNING_PASSWORD` - Password for signing key (if needed)
- [ ] `NPM_TOKEN` - Private npm registry authentication token
- [ ] `NUGET_API_KEY` - NuGet API key (if publishing to NuGet)
- [ ] `SLACK_WEBHOOK_URL` - Slack webhook for notifications (optional)
- [ ] `PUBLIC_REPO_COMMENT_TOKEN` - PAT to comment on public releases (optional)

### ✅ Step 6: Test the Setup
- [ ] Create a test release in this public repo:
  - Tag: `v0.0.1-test`
  - Mark as pre-release
  - Publish
- [ ] Check Actions tab in public repo → Verify "Trigger Private Repo Release" runs successfully
- [ ] Check Actions tab in private repo → Verify "Licensed Release Pipeline" runs
- [ ] Review logs to ensure payload is received correctly
- [ ] Clean up test release after verification

### ✅ Step 7: Enable (if needed)
- [ ] Ensure GitHub Actions is enabled in both repositories
  - Settings → Actions → General → "Allow all actions and reusable workflows"
- [ ] If using environments in private repo, configure protection rules
  - Settings → Environments → production → Add reviewers

---

## 🎯 Quick Reference

### Public Repo Configuration
| Item | Value |
|------|-------|
| Workflow File | `.github/workflows/trigger-private-release.yml` |
| Secret Name | `PRIVATE_REPO_DISPATCH_TOKEN` |
| Event Type | `public-release-created` |
| Trigger | `release: [published]` |

### Private Repo Configuration
| Item | Value |
|------|-------|
| Workflow File | `.github/workflows/licensed-release.yml` |
| Trigger | `repository_dispatch: [public-release-created]` |
| Payload Access | `github.event.client_payload.release_tag` |

---

## 🔧 Customization Options

### Trigger different release types:
```yaml
# Public repo - adjust trigger
on:
  release:
    types: [published, prereleased, created]
```

### Skip draft releases:
```yaml
# Public repo - add condition
jobs:
  dispatch-to-private-repo:
    if: github.event.release.draft == false
```

### Filter by tag pattern:
```yaml
# Public repo - add condition
jobs:
  dispatch-to-private-repo:
    if: startsWith(github.event.release.tag_name, 'v')
```

### Require approval for production:
```yaml
# Private repo - add environment
jobs:
  publish:
    environment: production  # Requires approval
```

---

## 🐛 Troubleshooting

### Dispatch not working?
1. ✅ Verify secret `PRIVATE_REPO_DISPATCH_TOKEN` exists in public repo
2. ✅ Check repository path format: `org/repo` (no spaces, correct case)
3. ✅ Confirm event-type matches: `public-release-created` in both files
4. ✅ Ensure private repo workflow is on default branch
5. ✅ Check Actions are enabled in both repositories

### 401 Unauthorized?
- Token is invalid or expired → Regenerate PAT and update secret

### 404 Not Found?
- Token doesn't have access to private repo → Check PAT repository access
- Repository name is wrong → Verify org/repo path

### Private workflow not running?
- Workflow must be on default branch (main/master) to respond to dispatch events
- Check YAML syntax is valid
- Verify `repository_dispatch` trigger type matches

---

## 📚 Documentation

For detailed instructions, see [RELEASE_TRIGGER_SETUP.md](RELEASE_TRIGGER_SETUP.md)

---

## ✨ Ready to Go!

Once all checkboxes are complete:
1. Create a release in this public repo
2. Watch the magic happen in your private repo
3. Celebrate! 🎉

Need help? Review the full documentation or check the troubleshooting section.
