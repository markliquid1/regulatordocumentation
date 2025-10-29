# Xregulator Backup System Guide

## FAQ: Understanding the Backup System

**Q: How does this backup system work?**  
A: Simple: You run `xregbackup stable-version` and it saves your ENTIRE XRegulator Arduino project folder to GitHub with that nickname AND creates a local backup copy in `/Users/joeceo/Projects/Old Backups/`. Later, run `xregrestore stable-version` and it downloads that complete snapshot from GitHub into a new folder. Every file, exactly as it was. No hunting through file histories.

**Filename Rules**  

Allowed:
- Lowercase letters and numbers: `stable-version-2` (This is git convention, although capital letters are also allowed)
- Hyphens and underscores: `working-bluetooth_fix`
- Forward slashes for grouping: `2025/stable-oct27`

Not allowed:
- Spaces: `stable version` ❌ (use `stable-version` instead)
- Special characters: `@#$%^&*()` ❌
- Starts with dot or hyphen: `.backup` or `-version` ❌

**Q: Where do I edit the backup commands?**  
A: They're in your `~/.zshrc` file. The functions `xregbackup`, `xreglist`, and `xregrestore` are defined there. If you need to reinstall them, see the setup instructions below.

**Q: Can I still access the old numbered backups?**  
A: Yes, all your old tags like `v013_20250620_233747` still exist. Use `git tag` to see them all, or `xregrestore v013_20250620_233747` to restore one.

**Q: Why does GitHub show an old backup name on the main page even after I created newer backups?**  
A: If you run multiple backups WITHOUT changing any files between them, Git sees them as identical and creates multiple tags pointing to the same commit. GitHub's main page shows the first commit message, not the most recent tag. All your backups still exist (click "Tags" to see them all), but the display is confusing. To make each backup show up separately on GitHub, change at least one file between backups - even adding a single space to a file works.

---

## Quick Start

### Daily Backup (with meaningful nickname)
```bash
cd /Users/joeceo/Documents/Arduino/Xregulator
xregbackup stable-after-motor-fix
```
This creates:
- GitHub backup with tag `stable-after-motor-fix`
- Local copy at `/Users/joeceo/Projects/Old Backups/Xregulator_stable-after-motor-fix/`

### List All Your Backups
```bash
xreglist
```

### Restore a Backup to New Folder
```bash
xregrestore stable-after-motor-fix
```
This creates `/Users/joeceo/Documents/Arduino/Xregulator_stable-after-motor-fix/` with that complete snapshot.

### View What's in a Backup (without downloading)
```bash
cd /Users/joeceo/Documents/Arduino/Xregulator
git show stable-after-motor-fix:Xregulator.ino
```

### Compare Two Backups
```bash
cd /Users/joeceo/Documents/Arduino/Xregulator
git diff backup1..backup2 -- Xregulator.ino
```

---

## Setup Instructions

If you need to set up the backup system on a new machine or reinstall it:

### 1. Install the Commands

Add these functions to your `~/.zshrc` file:

```bash
cat >> ~/.zshrc << 'EOF'

# Xregulator Backup System with Nicknames
xregbackup() {
    if [ -z "$1" ]; then
        echo "Usage: xregbackup <nickname>"
        echo "Example: xregbackup working-bluetooth-fix"
        return 1
    fi
    
    cd /Users/joeceo/Documents/Arduino/Xregulator
    git add .
    git commit -m "$1"
    git tag -a "$1" -m "Backup: $1 ($(date '+%Y-%m-%d %H:%M'))"
    git push --force origin main
    git push --force origin "$1"
    
    # Also copy to local backup folder
    BACKUP_DIR="/Users/joeceo/Projects/Old Backups/Xregulator_$1"
    rm -rf "$BACKUP_DIR"
    cp -R /Users/joeceo/Documents/Arduino/Xregulator "$BACKUP_DIR"
    
    echo "✓ Backed up as: $1"
    echo "✓ Local copy: $BACKUP_DIR"
}

xreglist() {
    cd /Users/joeceo/Documents/Arduino/Xregulator
    git tag -n1
}

xregrestore() {
    if [ -z "$1" ]; then
        echo "Usage: xregrestore <nickname>"
        echo "Available backups:"
        cd /Users/joeceo/Documents/Arduino/Xregulator && git tag
        return 1
    fi
    
    cd /Users/joeceo/Documents/Arduino
    git clone https://github.com/markliquid1/Regulator2025.git "Xregulator_$1"
    cd "Xregulator_$1"
    git checkout "$1"
    echo "✓ Restored to: /Users/joeceo/Documents/Arduino/Xregulator_$1"
}
EOF
source ~/.zshrc
```

### 2. Repository Details

- **Local project location**: `/Users/joeceo/Documents/Arduino/Xregulator/`
- **GitHub repository**: `https://github.com/markliquid1/Regulator2025.git`
- **Local backup copies**: `/Users/joeceo/Projects/Old Backups/Xregulator_<nickname>/`
- **What gets backed up**: Everything in the Xregulator Arduino project folder (both to GitHub and local backup folder)

### 3. Why Force Push?

The system uses `git push --force` to avoid sync issues. Since you're the only one using this repo and it's just a backup destination (not a collaboration space), force push ensures your local version always wins. This prevents the annoying "rejected push" errors that Git throws when it detects any differences.

---

## Advanced Usage

### Access Local Backups

Your local backups are stored at:
```bash
ls "/Users/joeceo/Projects/Old Backups/"
```

Each backup is a complete copy of your project folder with the nickname you gave it.

### View Old Versions

All your old timestamped backups are preserved as tags. They're permanent snapshots even after force pushing.

```bash
cd /Users/joeceo/Documents/Arduino/Xregulator
git tag
```

Shows all tags including old ones like `v013_20250620_233747`.

### Download Complete Old Version

```bash
cd /Users/joeceo/Documents/Arduino
git clone https://github.com/markliquid1/Regulator2025.git Xregulator_June20
cd Xregulator_June20
git checkout v013_20250620_233747
```

Now `/Users/joeceo/Documents/Arduino/Xregulator_June20/` contains the complete June 20th snapshot.

---

## Troubleshooting

**"Command not found: xregbackup"**
- Check if it's in `~/.zshrc`: `cat ~/.zshrc | grep xregbackup`
- Reload your shell: `source ~/.zshrc`

**"Not a git repository"**
- Make sure you're in the right directory: `cd /Users/joeceo/Documents/Arduino/Xregulator`
- Check if it's initialized: `git status`

**Local backup folder doesn't exist**
- The system will create `/Users/joeceo/Projects/Old Backups/` automatically
- Make sure you have write permissions to `/Users/joeceo/Projects/`
