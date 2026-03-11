# dotfiles

Private config files for my development projects. Each project has its own folder with files symlinked into the repo.

## Setup

### macOS

```bash
git clone <this-repo> ~/Documents/dotfiles
```

**DiscordLogger**
```bash
ln -s ~/Documents/dotfiles/DiscordLogger/CLAUDE.md ~/Documents/DiscordLogger/CLAUDE.md
ln -s ~/Documents/dotfiles/DiscordLogger/.claude ~/Documents/DiscordLogger/.claude
```

### Windows (PowerShell as Admin, or with Developer Mode enabled)

```powershell
git clone <this-repo> "$HOME\Documents\dotfiles"
```

**DiscordLogger**
```powershell
New-Item -ItemType SymbolicLink -Path "$HOME\Documents\DiscordLogger\CLAUDE.md" -Target "$HOME\Documents\dotfiles\DiscordLogger\CLAUDE.md"
New-Item -ItemType SymbolicLink -Path "$HOME\Documents\DiscordLogger\.claude" -Target "$HOME\Documents\dotfiles\DiscordLogger\.claude"
```

## Hiding symlinks from each repo

Add to `.git/info/exclude` inside each project repo (never committed, local only):

```
CLAUDE.md
.claude
```