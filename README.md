# git-helpers
Bash, Shell, Powershell scripts to help working with Git

## Windows

### Powershell

#### Push-to-Github

Problem: You have many folders that you want to push to GitHub, but you want to save yourself the time of creating the external repository each time.
Solution: If the directory to be pushed already has the name of your repository, you can use this simple script.

##### Pre-Installation

1. At first you have to install `Git`. (I assume that most people already have Git pre-installed before they encounter their “problem.”)
2. Then you have to install `GitHub CLI (gh)`
3. Then you have to log in once with `gh`.

```
winget install --id GitHub.cli
gh auth login
```

##### Installation

```
cd "C:\WINDOWS\System32"
code push-to-github.ps1
```

Then enter this code and save as admin:

```
param (
    [Parameter(Mandatory=$true)]
    [string]$Path
)

# In das Verzeichnis wechseln
$targetDir = Get-Item $Path
Set-Location $targetDir.FullName

# Den Namen des Ordners als Repo-Namen speichern
$repoName = $targetDir.Name

Write-Host "Verarbeite: $repoName" -ForegroundColor Cyan

# 1. Git initialisieren, falls noch nicht geschehen
if (-not (Test-Path ".git")) {
    git init -b main
    Write-Host "Git Repository initialisiert." -ForegroundColor Green
}

# 2. Dateien hinzufügen und erster Commit
git add .
git commit -m "Initial commit"

# 3. GitHub Repository erstellen (Privat standardmäßig)
# --source=. sagt: Nutze den aktuellen Ordner
# --push sagt: Lade den Code sofort hoch
gh repo create $repoName --source=. --public --push

Write-Host "Erfolgreich! $repoName ist jetzt auf GitHub." -ForegroundColor Green
cd "C:\WINDOWS\System32"
```

##### Usage

```
cd "C:\WINDOWS\System32"
.\push-to-github.ps1 "path\to\directory"
```

## Linux

### Bash

### Shell
