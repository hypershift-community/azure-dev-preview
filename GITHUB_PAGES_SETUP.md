# GitHub Pages Setup

This document describes the one-time setup required to enable GitHub Pages deployment for the azure-dev-preview repository.

## Prerequisites

- Repository admin access to `hypershift-community/azure-dev-preview`
- GitHub Actions workflow file already committed (`.github/workflows/docs.yml`)

## Configuration Steps

### 1. Enable GitHub Pages

1. Navigate to the repository on GitHub
2. Go to **Settings** → **Pages**
3. Under "Build and deployment":
   - **Source**: Select **Deploy from a branch**
   - **Branch**: Select **gh-pages**
   - **Folder**: Select **/ (root)**
4. Click **Save**

### 2. Configure Workflow Permissions

The GitHub Actions workflow needs write permissions to create and push to the `gh-pages` branch.

1. Navigate to **Settings** → **Actions** → **General**
2. Scroll down to **Workflow permissions**
3. Select: **Read and write permissions**
4. Check: **Allow GitHub Actions to create and approve pull requests**
5. Click **Save**

### 3. Verify Deployment

After the first push to `main` that modifies files in `docs/`:

1. Go to **Actions** tab to see the workflow run
2. Wait for the "Deploy Documentation" workflow to complete (1-2 minutes)
3. Go to **Settings** → **Pages** to see the published URL
4. Your site will be available at: **https://hypershift-community.github.io/azure-dev-preview/**

### 4. Branch Protection (Optional but Recommended)

Protect the `main` branch from accidental direct pushes:

1. Navigate to **Settings** → **Branches**
2. Click **Add branch protection rule**
3. Branch name pattern: `main`
4. Enable:
   - **Require pull request reviews before merging**
   - **Require status checks to pass before merging**
5. Click **Create**

**Important**: Do NOT protect the `gh-pages` branch - it's automatically managed by the workflow.

## Workflow Behavior

- **Automatic Deployment**: Triggered on push to `main` branch when files in `docs/` directory change
- **Manual Deployment**: Can be triggered manually from the **Actions** tab using "Run workflow"
- **Build Time**: Approximately 1-2 minutes per deployment
- **Container Used**: `quay.io/hypershift/mkdocs-material:9.6.8` (matches local development environment)

## Troubleshooting

### Workflow fails with "permission denied"

- Verify workflow permissions are set to "Read and write" (Step 2)

### Site not updating after workflow succeeds

- Check that GitHub Pages is enabled and points to `gh-pages` branch (Step 1)
- Wait 1-2 minutes for CDN propagation
- Hard refresh your browser (Ctrl+Shift+R or Cmd+Shift+R)

### Build fails in workflow

- Check the Actions tab for detailed error logs
- Verify the workflow runs successfully: it uses the same container as `make build-containerized`
- Test locally first: `cd docs && make build-containerized`

## Custom Domain (Optional)

To use a custom domain instead of `hypershift-community.github.io/azure-dev-preview`:

1. Go to **Settings** → **Pages**
2. Under "Custom domain", enter your domain (e.g., `docs.example.com`)
3. Update `docs/mkdocs.yml`:
   ```yaml
   site_url: https://docs.example.com/
   ```
4. Configure DNS:
   - Add CNAME record pointing to `hypershift-community.github.io`
   - Wait for DNS propagation (5-60 minutes)
5. Enable "Enforce HTTPS" in Pages settings

## Files Modified

This setup required the following files:

- `.github/workflows/docs.yml` - GitHub Actions workflow for deployment
- `docs/mkdocs.yml` - Updated with `site_url`, `repo_url`, and `repo_name`
- `GITHUB_PAGES_SETUP.md` - This documentation file

## Support

For issues with:
- **Workflow execution**: Check GitHub Actions logs
- **MkDocs build errors**: Test locally with `make build-containerized` in `docs/`
- **GitHub Pages configuration**: Refer to [GitHub Pages documentation](https://docs.github.com/en/pages)
