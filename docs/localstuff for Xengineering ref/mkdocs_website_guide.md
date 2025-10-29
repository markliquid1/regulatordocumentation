# MkDocs Documentation Website Setup Guide

## Overview

This guide shows how to set up a professional documentation website using MkDocs with GitHub Actions deployment. Your documentation will be published at https://docs.xengineering.net.

## Daily Workflow - START HERE

**This is what you do every day after editing markdown files locally:**

```bash
cd ~/Projects/xengineering_docs
git add .
git commit -m "Update documentation"
git push origin main
```

**That's it!** GitHub Actions automatically builds and deploys your changes to https://docs.xengineering.net in 2-3 minutes.

---

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

Replace the contents of `mkdocs.yml`. Here's the actual current configuration for the Xengineering documentation site:

```yaml
site_name: Alternator Regulator Documentation
site_url: https://docs.xengineering.net
site_description: Technical docs for Xengineering marine open-source alternator regulator system
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
 - Intro:
   - Features: basic-use/key_features_list.md
   - Getting Started: basic-use/getting-started.md
   - Installation: basic-use/installation.md
   - Quick Start Guide: basic-use/quick-start.md
   - Troubleshooting: basic-use/troubleshooting.md
 - Hardware:
   - System Architecture: hardware/index.md
   - Digital Inputs: hardware/digital-inputs.md
   - Switching Power Supply: hardware/switching_power_supply_doc.md
   - Digital Outputs: hardware/digital-outputs.md
   - Alternator Field Drive: hardware/alternator_field_drive.md
   - Baro and Temp Sensor: hardware/bmp390_documentation.md
   - NMEA0183 and VeDirect: hardware/NMEA0183_and_VeDirect.md
   - BatteryV and Current Monitor: hardware/ina228.md
   - Warning Buzzer: hardware/buzzer_control_doc.md
   - Tachometer Output: hardware/tach_out.md
   - USB-C: hardware/usb_c.md
   - NMEA2K: hardware/nmea2k.md
   - Analog Inputs: hardware/analoginputsADS1115.md
   - Input Protection: hardware/input-protection.md
   - ESP32 Controller: hardware/esp32-brain.md
   - LM2907 Resistor Mod: hardware/lm2907 resistor mod.md
   - Connectors & Wiring: hardware/connectors.md
 - Software - ESP32:
   - System Overview: software/esp32/system_overview.md
   - Sensor Systems: software/esp32/sensor_systems.md
   - Communication Interfaces: software/esp32/communication_interfaces.md
   - Field Control: software/esp32/field_control_docs.md
   - Learning Mode: software/esp32/Learning_Mode_System.md
   - Battery Management: software/esp32/battery_management_docs.md
   - Network & Web System: software/esp32/network_web_system_docs.md
   - Safety & Monitoring: software/esp32/safety_monitoring_docs.md
   - Advanced Features: software/esp32/advanced_features_docs.md
   - Configuration: software/esp32/configuration_development_docs.md
 - Software - Client:
   - HTML Structure: software/client/html.md
   - JavaScript Logic: software/client/javascript.md
   - CSS Styling: software/client/css.md
 - Local Xengineering Ref Only:
   - Partitioning: localstuff for Xengineering ref/esp32_partition_guide.md
   - OTA: localstuff for Xengineering ref/updated_ota_docs V3 .md
   - Planning: localstuff for Xengineering ref/updated_planning_doc.md
   - Biz Stuff: localstuff for Xengineering ref/business_analysis_summary.md
   - AI Notes: localstuff for Xengineering ref/AI_notes_Aug2025.md
   - Protrak MX2e Guide: localstuff for Xengineering ref/protrak_mx2e_guide.md
   - Xregulator Backup Guide: localstuff for Xengineering ref/xregulator_backup_guide.md
   - MkDocs Website Guide: localstuff for Xengineering ref/mkdocs_website_guide.md

copyright: Copyright &copy; 2025 Xengineering
```

**Note:** The "Local Xengineering Ref Only" section contains internal documentation not intended for external users. You can remove this section if publishing docs publicly.

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
cd ~/Projects/xengineering_docs
git add .
git commit -m "Update documentation"
git push origin main
```

**That's it!** GitHub Actions automatically builds and deploys your changes in 2-3 minutes.

### Adding New Pages

#### 1. Create the Markdown File

```bash
echo "# New Page Title

Content goes here..." > docs/new-page.md
```

#### 2. Add to Navigation

Edit `mkdocs.yml` and add to the `nav` section. For example, to add a new troubleshooting page:

```yaml
nav:
 - Home: index.md
 - Intro:
   - Features: basic-use/key_features_list.md
   - Getting Started: basic-use/getting-started.md
   - Installation: basic-use/installation.md
   - Quick Start Guide: basic-use/quick-start.md
   - Troubleshooting: basic-use/troubleshooting.md
   - New Troubleshooting Page: basic-use/new-troubleshooting.md  # <-- Add this line
```

#### 3. Create Subsections

The actual navigation uses nested sections. Here's how the Hardware section is structured:

```yaml
nav:
 - Hardware:
   - System Architecture: hardware/index.md
   - Digital Inputs: hardware/digital-inputs.md
   - ESP32 Controller: hardware/esp32-brain.md
   - Connectors & Wiring: hardware/connectors.md
```

To add a new hardware page:

```bash
echo "# New Hardware Component

Documentation for new component..." > docs/hardware/new-component.md
```

Then add it to the Hardware section in `mkdocs.yml`:

```yaml
 - Hardware:
   - System Architecture: hardware/index.md
   - Digital Inputs: hardware/digital-inputs.md
   - New Component: hardware/new-component.md  # <-- Add here
   - ESP32 Controller: hardware/esp32-brain.md
```

### Adding Images

**For copy-paste images:** Unfortunately, there's no elegant solution. MkDocs/GitHub requires images to be files in the repository. Your options:

1. **Drag and drop** (if using VS Code/editor that supports it)
2. **Screenshot to file, then drag** (macOS: Cmd+Shift+4, save to Desktop, drag to `docs/images/`)
3. **Use a different platform** like Notion, GitBook, or Confluence that support direct paste

The filesystem requirement is fundamental to how static site generators work - they need actual image files to include in the build.

#### 1. Create Images Folder

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
xengineering_docs/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── docs/
│   ├── images/
│   │   └── opto_schematic.png
│   ├── stylesheets/
│   │   └── extra.css
│   ├── basic-use/
│   │   ├── key_features_list.md
│   │   ├── getting-started.md
│   │   ├── installation.md
│   │   ├── quick-start.md
│   │   └── troubleshooting.md
│   ├── hardware/
│   │   ├── index.md
│   │   ├── esp32-brain.md
│   │   └── ... (all hardware docs)
│   ├── software/
│   │   ├── esp32/
│   │   │   └── ... (ESP32 firmware docs)
│   │   └── client/
│   │       └── ... (web interface docs)
│   ├── localstuff for Xengineering ref/
│   │   └── ... (internal reference docs)
│   ├── CNAME (custom domain)
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
cd ~/Projects/xengineering_docs

# Create new page
echo "# Page Title" > docs/new-page.md

# Add images
cp ~/Desktop/image.png docs/images/

# Deploy changes
cd ~/Projects/xengineering_docs
git add .
git commit -m "Update docs"
git push origin main

# Test locally
mkdocs serve
```

---

**Remember**: After initial setup, your workflow is simply edit → commit → push. GitHub handles the rest automatically.
