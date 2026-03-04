# Cross-Repository Release Trigger Setup Guide

## Overview

This guide explains how to trigger a workflow in a private repository when a release is created in this public repository using GitHub's `repository_dispatch` event.

**Architecture:**
- **Public Repo (this repo)**: Announces releases via `repository_dispatch`
- **Private Repo**: Listens for dispatch events and handles licensed artifacts/publishing
- **Security**: Public repo has no access to private repo; private repo controls execution

---

## Prerequisites

- Admin access to both repositories
- Both repos must be in the same GitHub organization (or you need a PAT with cross-org permissions)
- GitHub account with permission to create Personal Access Tokens

---

## Step 1: Create a Personal Access Token (PAT)

The public repo needs a token to send dispatch events to the private repo.

### 1.1 Generate the PAT

1. Go to GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens** (recommended) or **Tokens (classic)**

2. Click **"Generate new token"**

3. Configure the token:
   - **Name**: `public-to-private-dispatch` (or similar)
   - **Expiration**: Choose appropriate duration (recommend: 1 year with calendar reminder)
   - **Repository access**: 
     - For fine-grained tokens: Select "Only select repositories" → Choose your **private repository**
   - **Permissions**:
     - Repository permissions → **Contents**: Read and write (for `repository_dispatch`)
     - OR for classic tokens: Select `repo` scope

4. Click **"Generate token"**

5. **IMPORTANT**: Copy the token immediately (you won't see it again)

### 1.2 Alternative: GitHub App (Enterprise Option)

For better security and no expiration issues, consider using a GitHub App instead:
- Create an app with `repository_dispatch` permissions
- Install only on the private repo
- Use app credentials in the public repo

---

## Step 2: Add Secret to Public Repository (This Repo)

1. Navigate to this repository on GitHub
2. Go to **Settings** → **Secrets and variables** → **Actions**
3. Click **"New repository secret"**
4. Create secret:
   - **Name**: `PRIVATE_REPO_DISPATCH_TOKEN`
   - **Value**: Paste the PAT from Step 1.1
5. Click **"Add secret"**

---

## Step 3: Create Workflow in Public Repository

Create a workflow file that triggers on release creation and sends a dispatch event.

### 3.1 Create the workflow file

Create `.github/workflows/trigger-private-release.yml`:

```yaml
name: Trigger Private Repo Release

on:
  release:
    types: [published, created]
    # Use 'published' for final releases, 'created' for all releases including drafts
    # You can also add 'prereleased' if you want to trigger on pre-releases

jobs:
  dispatch-to-private-repo:
    runs-on: ubuntu-latest
    
    steps:
      - name: Send repository dispatch to private repo
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.PRIVATE_REPO_DISPATCH_TOKEN }}
          repository: YOUR_ORG/YOUR_PRIVATE_REPO  # e.g., my-org/private-licensed-builds
          event-type: public-release-created
          client-payload: |
            {
              "release_tag": "${{ github.event.release.tag_name }}",
              "release_name": "${{ github.event.release.name }}",
              "release_body": ${{ toJSON(github.event.release.body) }},
              "release_url": "${{ github.event.release.html_url }}",
              "is_prerelease": ${{ github.event.release.prerelease }},
              "is_draft": ${{ github.event.release.draft }},
              "published_at": "${{ github.event.release.published_at }}",
              "author": "${{ github.event.release.author.login }}",
              "tarball_url": "${{ github.event.release.tarball_url }}",
              "zipball_url": "${{ github.event.release.zipball_url }}"
            }

      - name: Log dispatch success
        run: |
          echo "✅ Successfully dispatched release event to private repository"
          echo "Release: ${{ github.event.release.tag_name }}"
          echo "Event type: public-release-created"
```

### 3.2 Customize the workflow

**Replace the following values:**
- `YOUR_ORG/YOUR_PRIVATE_REPO` → Your actual organization and private repo name
- `event-type: public-release-created` → Custom event name (keep consistent with private repo)

**Optional customizations:**
- Add conditional logic to skip draft releases: `if: github.event.release.draft == false`
- Add filtering for specific tag patterns: `if: startsWith(github.event.release.tag_name, 'v')`
- Add notifications on failure (Slack, email, etc.)

---

## Step 4: Create Workflow in Private Repository

The private repo needs a workflow that listens for the `repository_dispatch` event.

### 4.1 Create the dispatch listener workflow

In your **private repository**, create `.github/workflows/licensed-release.yml`:

```yaml
name: Licensed Release Pipeline

on:
  repository_dispatch:
    types: [public-release-created]  # Must match event-type from public repo

jobs:
  process-release:
    runs-on: ubuntu-latest
    
    steps:
      - name: Log received event
        run: |
          echo "🎉 Received release event from public repository"
          echo "Release tag: ${{ github.event.client_payload.release_tag }}"
          echo "Release name: ${{ github.event.client_payload.release_name }}"
          echo "Is prerelease: ${{ github.event.client_payload.is_prerelease }}"
          echo "Published at: ${{ github.event.client_payload.published_at }}"
          echo "Author: ${{ github.event.client_payload.author }}"

      - name: Checkout private repository
        uses: actions/checkout@v4

      - name: Download public release assets (if needed)
        run: |
          # Example: Download the release tarball from public repo
          curl -L -o public-release.tar.gz "${{ github.event.client_payload.tarball_url }}"
          echo "Downloaded public release artifacts"

      - name: Build licensed artifacts
        run: |
          echo "Building licensed version..."
          # Add your build commands here
          # npm ci
          # npm run build:licensed
          # npm run sign-artifacts
          # etc.

      - name: Sign artifacts
        run: |
          echo "Signing artifacts with private key..."
          # Use secrets stored in THIS (private) repo
          # ${{ secrets.CODE_SIGNING_KEY }}

      - name: Publish to private registry
        run: |
          echo "Publishing to private npm/nuget/etc registry..."
          # Use secrets stored in THIS (private) repo
          # ${{ secrets.NPM_TOKEN }}

      - name: Create GitHub release in private repo
        uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ github.event.client_payload.release_tag }}
          name: "Licensed - ${{ github.event.client_payload.release_name }}"
          body: |
            # Licensed Release
            
            This is the licensed version corresponding to public release:
            - **Public Release**: ${{ github.event.client_payload.release_url }}
            - **Version**: ${{ github.event.client_payload.release_tag }}
            - **Published**: ${{ github.event.client_payload.published_at }}
            
            ## Changes
            ${{ github.event.client_payload.release_body }}
          files: |
            dist/**/*
            *.sig
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Notify team
        if: always()
        run: |
          # Add your notification logic
          # curl -X POST ${{ secrets.SLACK_WEBHOOK }} -d "..."
          echo "Pipeline completed"
```

### 4.2 Customize the private workflow

**Customize based on your needs:**
- Add actual build/sign/publish commands
- Configure secret access for signing keys, registry tokens, etc.
- Add approval gates for production releases
- Add conditional logic for prereleases vs stable releases
- Add error handling and rollback procedures

### 4.3 Add secrets to private repository

In your **private repository**:
1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Add all necessary secrets:
   - `CODE_SIGNING_KEY` - Your signing certificate
   - `NPM_TOKEN` - Private registry token
   - `SLACK_WEBHOOK` - Notification endpoint
   - Any other secrets needed for licensed builds

---

## Step 5: Testing the Setup

### 5.1 Test with a draft release

1. In this public repo, create a **draft release**:
   - Go to **Releases** → **Draft a new release**
   - Create a tag (e.g., `v1.0.0-test`)
   - Mark as **"This is a pre-release"** (optional)
   - **Publish release**

2. Verify in public repo:
   - Go to **Actions** tab
   - Look for "Trigger Private Repo Release" workflow
   - Check that it completes successfully

3. Verify in private repo:
   - Go to **Actions** tab
   - Look for "Licensed Release Pipeline" workflow
   - Verify it received the correct payload
   - Check the logs to see client_payload data

### 5.2 Verify the payload

In the private repo workflow logs, you should see:
```
Release tag: v1.0.0-test
Release name: Test Release
Is prerelease: true
Author: your-username
```

### 5.3 Delete test release

After successful testing, clean up:
- Delete the test release from both repositories
- Delete the test tag if needed: `git push --delete origin v1.0.0-test`

---

## Step 6: Security Considerations

### 6.1 Token security

✅ **DO:**
- Use fine-grained PATs with minimal permissions
- Set token expiration and calendar reminders
- Rotate tokens regularly
- Use GitHub Apps for production (no expiration)
- Store tokens only as repository secrets

❌ **DON'T:**
- Share tokens in code or documentation
- Use tokens with broader permissions than needed
- Store tokens in workflow files or commit history
- Use personal tokens (create a bot/service account instead)

### 6.2 Validation in private repo

Add validation to prevent malicious dispatch events:

```yaml
- name: Validate source
  run: |
    # Only accept events from expected public repo
    if [ "${{ github.event.sender.login }}" != "expected-bot-user" ]; then
      echo "❌ Unexpected sender"
      exit 1
    fi
    
    # Validate tag format
    if ! [[ "${{ github.event.client_payload.release_tag }}" =~ ^v[0-9]+\.[0-9]+\.[0-9]+ ]]; then
      echo "❌ Invalid version tag format"
      exit 1
    fi
```

### 6.3 Rate limiting

GitHub Actions has rate limits:
- 1,000 API requests per hour per repository
- Each `repository_dispatch` counts as one request
- For high-frequency releases, implement queuing/batching

---

## Step 7: Advanced Configurations

### 7.1 Handling multiple event types

Public repo can send different event types:

```yaml
# In public repo - different workflow for prereleases
- uses: peter-evans/repository-dispatch@v3
  with:
    event-type: public-prerelease-created  # Different event type
    # ... rest of config
```

Private repo handles both:

```yaml
on:
  repository_dispatch:
    types: 
      - public-release-created
      - public-prerelease-created

jobs:
  process-release:
    runs-on: ubuntu-latest
    steps:
      - name: Determine release type
        run: |
          if [ "${{ github.event.action }}" == "public-prerelease-created" ]; then
            echo "RELEASE_CHANNEL=beta" >> $GITHUB_ENV
          else
            echo "RELEASE_CHANNEL=stable" >> $GITHUB_ENV
          fi
```

### 7.2 Manual approval for production

Add environment protection to private repo:

```yaml
jobs:
  process-release:
    runs-on: ubuntu-latest
    environment: 
      name: production  # Requires manual approval
      url: https://your-private-registry.com/releases
```

Configure in private repo: **Settings** → **Environments** → **production** → Add required reviewers

### 7.3 Bi-directional communication

To report back to public repo (e.g., comment on release):

```yaml
# In private repo, after successful build
- name: Comment on public release
  uses: actions/github-script@v7
  with:
    github-token: ${{ secrets.PUBLIC_REPO_COMMENT_TOKEN }}
    script: |
      const owner = 'YOUR_ORG';
      const repo = 'YOUR_PUBLIC_REPO';
      const release_id = '${{ github.event.client_payload.release_id }}';
      
      await github.rest.repos.createReleaseComment({
        owner,
        repo,
        release_id,
        body: '✅ Licensed artifacts published successfully!'
      });
```

This requires an additional PAT with write access to the public repo.

---

## Troubleshooting

### Issue: Dispatch not triggering

**Symptoms**: Public workflow completes, but private workflow doesn't run

**Solutions**:
1. Verify `event-type` matches exactly in both repos
2. Check token has correct permissions (`repo` or `contents: write`)
3. Verify token is for correct private repository
4. Check private repo workflow file is on default branch (`main`/`master`)
5. Check Actions are enabled in private repo: **Settings** → **Actions** → **General**

### Issue: 401 Unauthorized

**Cause**: Token is invalid or lacks permissions

**Solution**:
1. Regenerate PAT with correct scopes
2. Update secret in public repository
3. Verify PAT wasn't revoked or expired

### Issue: 404 Not Found

**Cause**: Token doesn't have access to private repo or repo name is wrong

**Solution**:
1. Double-check repository path is `owner/repo` format
2. Verify token has access to that specific private repo
3. Check organization name spelling

### Issue: Payload data missing

**Cause**: JSON formatting errors in client-payload

**Solution**:
1. Use `toJSON()` for any string that might contain quotes/newlines
2. Test payload structure before production
3. Add error handling in private workflow

### Issue: Workflow not found in private repo

**Symptoms**: Dispatch sent but no workflow executes

**Solution**:
1. Ensure workflow file is in `.github/workflows/` directory
2. Push workflow to default branch (dispatch only triggers on default branch)
3. Check YAML syntax with `yamllint` or GitHub's validation
4. Verify repository_dispatch trigger is properly configured

---

## Maintenance

### Monthly tasks
- [ ] Verify workflows are still functioning
- [ ] Check token expiration dates
- [ ] Review workflow run history for errors
- [ ] Update dependencies (actions versions)

### When updating either repo
- [ ] Test dispatch flow after major changes
- [ ] Keep event-type names synchronized
- [ ] Document any payload structure changes
- [ ] Update version tags if payload changes

### Security audits
- [ ] Review token permissions quarterly
- [ ] Check who has access to secrets
- [ ] Rotate tokens annually
- [ ] Review workflow logs for anomalies

---

## Example: Complete Minimal Setup

### Public Repo: `.github/workflows/dispatch-release.yml`
```yaml
name: Dispatch Release
on:
  release:
    types: [published]

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.PRIVATE_REPO_DISPATCH_TOKEN }}
          repository: my-org/private-builds
          event-type: release
          client-payload: '{"tag": "${{ github.event.release.tag_name }}"}'
```

### Private Repo: `.github/workflows/handle-release.yml`
```yaml
name: Handle Release
on:
  repository_dispatch:
    types: [release]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building ${{ github.event.client_payload.tag }}"
      # Add your build steps
```

### Secret Setup
1. Generate PAT with `repo` scope
2. In public repo: Add secret `PRIVATE_REPO_DISPATCH_TOKEN`
3. Done!

---

## Resources

- [GitHub repository_dispatch documentation](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#repository_dispatch)
- [peter-evans/repository-dispatch action](https://github.com/peter-evans/repository-dispatch)
- [GitHub Actions secrets management](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [GitHub PAT documentation](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)

---

## Questions or Issues?

If you encounter problems:
1. Check the Troubleshooting section above
2. Review workflow run logs in both repositories
3. Verify all secrets are correctly configured
4. Test with a minimal payload first
5. Check GitHub Actions status page for platform issues

---

**Last Updated**: February 11, 2026
