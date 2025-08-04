# Complete MkDocs GitHub Pages Setup Guide

## AI Assistant Instructions

If you're an AI helping with this project, here are the key details:

**User Environment:**
- **Platform**: macOS
- **Project location**: `~/Projects/xengineering_docs/` (note: folder name is `xengineering_docs`, repo name is `regulatordocumentation`)
- **Repository**: `markliquid1/regulatordocumentation`
- **Live site**: https://docs.xengineering.net
- **GitHub Pages URL**: https://markliquid1.github.io/regulatordocumentation

**User Preferences:**
- **Terminal commands**: Always provide copy-paste ready commands with NO comments
- **Always assume new terminal**: Start with `ls` and `cd ~/Projects/xengineering_docs` 
- **Multiple commands**: Group them together so user can paste all at once
- **No placeholders**: Use actual repo names, not "yourrepo" or "username"
- **Direct answers**: No explanatory fluff, just working solutions

**Current Setup:**
- Uses GitHub Actions deployment (NOT `mkdocs gh-deploy`)
- MkDocs Material theme with custom CSS
- Teal accent color (`#00a19a`)
- Custom domain configured
- Right sidebar disabled

**File Structure:**
```
xengineering_docs/
├── .github/workflows/deploy.yml
├── docs/
│   ├── stylesheets/extra.css
│   ├── basic-use/
│   ├── hardware/ 
│   ├── software/
│   ├── CNAME
│   └── index.md
├── mkdocs.yml
└── site/ (ignore - auto-generated)
```

---

## Overview

This guide shows how to set up a professional MkDocs documentation site with GitHub Actions deployment, avoiding the problematic `mkdocs gh-deploy` command that causes source/deployment sync issues.

## Initial Setup

### 1. Create GitHub Repository

1. Go to GitHub and create a new repository
2. Clone it locally:
   ```bash
   git clone https://github.com/yourusername/yourrepo.git
   cd yourrepo
   ```

### 2. Install MkDocs Locally

```bash
pip install mkdocs-material
```

### 3. Initialize MkDocs Project

```bash
mkdocs new .
```

This creates:
- `mkdocs.yml` (configuration file)
- `docs/` folder with `index.md`

### 4. Configure MkDocs

Replace the contents of `mkdocs.yml`:

```yaml
site_name: Xengineering Documentation
site_url: https://docs.xengineering.net
site_description: Technical documentation for Xengineering marine electrical regulator system
site_author: Xengineering

repo_name: markliquid1/regulatordocumentation
repo_url: https://github.com/markliquid1/regulatordocumentation

theme:
  name: material
  features:
    - navigation.sections
    - navigation.expand
    - search.highlight

plugins:
  - search

extra_css:
  - stylesheets/extra.css

markdown_extensions:
  - admonition
  - pymdownx.highlight
  - pymdownx.superfences
  - toc:
      permalink: true

nav:
  - Home: index.md
  - Getting Started: getting-started.md
  - Advanced: advanced.md

copyright: Copyright &copy; 2025 Your Name
```

### 5. Create Custom Styling (Optional)

```bash
mkdir -p docs/stylesheets
```

Create `docs/stylesheets/extra.css`:

```css
:root {
  --md-primary-fg-color: #333333;
  --md-accent-fg-color: #00a19a;
  --md-default-bg-color: #f5f5f5;
}

.md-typeset a {
  color: #00a19a;
}

.md-nav__link--active {
  color: #00a19a;
}

.md-nav__link:hover {
  color: #00a19a;
}

.md-sidebar--secondary {
  display: none !important;
}
```

### 6. Test Locally

```bash
mkdocs serve
```

Visit `http://127.0.0.1:8000` to preview your site.

## GitHub Actions Deployment Setup

### 1. Create Workflow File

```bash
mkdir -p .github/workflows
```

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy MkDocs to GitHub Pages

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.x'

      - name: Cache pip dependencies
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - name: Install MkDocs and dependencies
        run: |
          pip install mkdocs-material

      - name: Build documentation
        run: mkdocs build --clean

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./site

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 2. Initial Commit and Push

```bash
git add .
git commit -m "Initial MkDocs setup with GitHub Actions deployment"
git push origin main
```

### 3. Enable GitHub Pages

**CRITICAL STEP**: Go to your GitHub repository:
1. Settings → Pages
2. Under "Source" select **"GitHub Actions"** (NOT "Deploy from a branch")
3. Save

### 4. Wait for Deployment

- Go to Actions tab in your GitHub repo
- Watch the workflow run (takes 2-3 minutes)
- Site will be live at `https://markliquid1.github.io/regulatordocumentation`

## Setting Up Custom Domain (Optional)

### 1. Create CNAME File

```bash
echo "docs.xengineering.net" > docs/CNAME
```

### 2. Configure DNS

Add a CNAME record in your DNS settings:
- Name: `docs` (or whatever subdomain you want)
- Value: `markliquid1.github.io`

### 3. Update mkdocs.yml

Change the `site_url` in `mkdocs.yml`:
```yaml
site_url: https://docs.xengineering.net
```

### 4. Commit and Deploy

```bash
git add docs/CNAME mkdocs.yml
git commit -m "Add custom domain"
git push origin main
```

## Editing Your Site

### Daily Workflow

```bash
# Navigate to project
cd ~/Projects/xengineering_docs

# Edit files (use any editor)
# Make your changes...

# Deploy changes
cd ~/Projects/xengineering_docs
git add .
git commit -m "Update documentation"
git push origin main
```

**That's it!** GitHub Actions automatically builds and deploys your changes in 2-3 minutes.

### Adding New Pages

#### 1. Create the Markdown File

```bash
# Create new page
echo "# New Page Title

Content goes here..." > docs/new-page.md
```

#### 2. Add to Navigation

Edit `mkdocs.yml` and add to the `nav` section:

```yaml
nav:
  - Home: index.md
  - Getting Started: getting-started.md
  - New Page: new-page.md  # <-- Add this line
  - Advanced: advanced.md
```

#### 3. Create Subsections

For nested navigation:

```yaml
nav:
  - Home: index.md
  - User Guide:
    - Getting Started: user-guide/getting-started.md
    - Installation: user-guide/installation.md
    - Troubleshooting: user-guide/troubleshooting.md
  - Developer Guide:
    - API Reference: dev-guide/api.md
    - Contributing: dev-guide/contributing.md
```

Create the folder structure:
```bash
mkdir -p docs/user-guide docs/dev-guide
echo "# Getting Started" > docs/user-guide/getting-started.md
echo "# API Reference" > docs/dev-guide/api.md
```

**For copy-paste images:** Unfortunately, there's no elegant solution. MkDocs/GitHub requires images to be files in the repository. Your options:

1. **Drag and drop** (if using VS Code/editor that supports it)
2. **Screenshot to file, then drag** (macOS: Cmd+Shift+4, save to Desktop, drag to `docs/images/`)
3. **Use a different platform** like Notion, GitBook, or Confluence that support direct paste

The filesystem requirement is fundamental to how static site generators work - they need actual image files to include in the build.

```bash
mkdir -p docs/images
```

#### 2. Add Image Files

Copy your images to `docs/images/`:
```bash
cp ~/Desktop/screenshot.png docs/images/
cp ~/Desktop/diagram.jpg docs/images/
```

#### 3. Reference Images in Markdown

In any `.md` file:

```markdown
# My Page

Here's a screenshot:

![Screenshot description](images/screenshot.png)

You can also add captions:

![System diagram](images/diagram.jpg)
*Figure 1: System architecture overview*

For clickable images:
[![Clickable image](images/screenshot.png)](images/screenshot.png)
```

#### 4. Image Best Practices

- **Supported formats**: PNG, JPG, GIF, SVG, WebP
- **File naming**: Use lowercase, dashes instead of spaces (`system-diagram.png`)
- **Size**: Optimize images for web (usually under 1MB)
- **Alt text**: Always include descriptive alt text for accessibility

### Advanced Image Usage

#### Sizing Images with HTML

```markdown
<img src="images/logo.png" alt="Company logo" width="200">

<img src="images/diagram.png" alt="Architecture" style="max-width: 100%; height: auto;">
```

#### Image Galleries

```markdown
<div style="display: flex; gap: 10px;">
  <img src="images/photo1.jpg" alt="Photo 1" style="width: 30%;">
  <img src="images/photo2.jpg" alt="Photo 2" style="width: 30%;">
  <img src="images/photo3.jpg" alt="Photo 3" style="width: 30%;">
</div>
```

## Project Structure

Your final project should look like:

```
regulatordocumentation/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── docs/
│   ├── images/
│   │   ├── screenshot.png
│   │   └── diagram.jpg
│   ├── stylesheets/
│   │   └── extra.css
│   ├── user-guide/
│   │   ├── getting-started.md
│   │   └── installation.md
│   ├── CNAME (if using custom domain)
│   └── index.md
├── site/ (auto-generated, ignore this)
├── mkdocs.yml
└── README.md
```

## Troubleshooting

### Common Issues

**"Site not updating"**
- Check Actions tab for build errors
- Ensure GitHub Pages source is set to "GitHub Actions"
- Hard refresh browser (Ctrl+F5)

**"Images not showing"**
- Check image paths (relative to the markdown file)
- Ensure images are in `docs/images/` folder
- Check image file names (case sensitive)

**"Navigation not working"**
- Verify `nav` section in `mkdocs.yml` syntax
- Ensure file paths in nav match actual file locations
- Check for YAML indentation errors

**"Custom domain not working"**
- Check that `docs/CNAME` file contains docs.xengineering.net
- Verify DNS CNAME record points to `markliquid1.github.io`
- Allow 24-48 hours for DNS propagation

### Testing Changes Locally

Before pushing changes:

```bash
mkdocs serve
```

Visit `http://127.0.0.1:8000` to preview your changes locally.

## Why This Setup is Better

**Avoid `mkdocs gh-deploy`** - it deploys from local files (including uncommitted changes) causing sync issues between your source code and live site.

**GitHub Actions Benefits**:
- Only deploys committed code
- Full traceability of what's deployed
- Works for teams
- Automatic deployment on every push
- No local environment dependencies

## Quick Reference Commands

```bash
# Navigate to project
cd ~/Projects/regulatordocumentation

# Create new page
echo "# Page Title" > docs/new-page.md

# Add images
cp ~/Desktop/image.png docs/images/

# Deploy changes
cd ~/Projects/regulatordocumentation
git add .
git commit -m "Update docs"
git push origin main

# Test locally
mkdocs serve
```

---

**Remember**: After initial setup, your workflow is simply edit → commit → push. GitHub handles the rest automatically.