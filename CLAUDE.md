# CLAUDE.md

## Windows Setup

### 1. Clone dotfiles
```powershell
git clone https://github.com/LVCHLANN/dotfiles.git "$HOME\Documents\dotfiles"
```

### 2. Generate SSH key for GitHub
```powershell
ssh-keygen -t ed25519 -C "lachlan@windows" -f "$HOME\.ssh\id_ed25519_windows"
```
Then add the contents of `~\.ssh\id_ed25519_windows.pub` to GitHub → Settings → SSH Keys.

### 3. Configure SSH
Add to `~\.ssh\config`:
```
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_windows
  IdentitiesOnly yes
```

### 4. Switch dotfiles remote to SSH
```powershell
cd "$HOME\Documents\dotfiles"
git remote set-url origin git@github.com:LVCHLANN/dotfiles.git
```

### 5. Auto-commit on change (Task Scheduler)
Create `autocommit.ps1` somewhere stable (e.g. `~\scripts\autocommit.ps1`):
```powershell
Set-Location "$HOME\Documents\dotfiles"
git add -A
$diff = git diff --cached --quiet
if ($LASTEXITCODE -ne 0) {
    $date = Get-Date -Format "yyyy-MM-dd HH:mm"
    git commit -m "auto: sync $date"
    git push
}
```

Then register it as a scheduled task running every minute:
```powershell
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NonInteractive -File $HOME\scripts\autocommit.ps1"
$trigger = New-ScheduledTaskTrigger -RepetitionInterval (New-TimeSpan -Minutes 1) -Once -At (Get-Date)
Register-ScheduledTask -TaskName "DotfilesAutoCommit" -Action $action -Trigger $trigger -RunLevel Highest
```
