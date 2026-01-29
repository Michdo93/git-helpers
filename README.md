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

#### Repair-Git-Folder-Structure

Problem: Sometimes it happens that you push a subfolder instead of the contents of that folder. This creates an incorrect folder hierarchy in the Git repository.
Solution: The solution is to clone the repository, move all files up one level, delete the subfolder, commit all changes, and push again.

##### Installation

```
cd "C:\WINDOWS\System32"
code repair-git-folder-structure.ps1
```

Then enter this code and save as admin:

```
param (
    [Parameter(Mandatory=$true)]
    [string]$RepoUrl
)

# 1. Repo-Namen aus der URL extrahieren (z.B. "MeinProjekt" aus "https://github.com/user/MeinProjekt.git")
$repoName = ($RepoUrl -split "/" | Select-Object -Last 1).Replace(".git", "")
$tempPath = Join-Path $env:TEMP "git-repair-$repoName"

Write-Host "Klone $repoName nach $tempPath..." -ForegroundColor Cyan

# Falls der Temp-Ordner noch von einem alten Versuch existiert, löschen
if (Test-Path $tempPath) { Remove-Item $tempPath -Recurse -Force }

# 2. Repository klonen
git clone $RepoUrl $tempPath
Set-Location $tempPath

# 3. Prüfen, ob der "Ordner-im-Ordner" Fehler vorliegt
if (Test-Path ".\$repoName") {
    Write-Host "Strukturfehler gefunden. Korrigiere..." -ForegroundColor Yellow

    # Alles aus dem Unterordner eine Ebene hoch (inkl. versteckter Dateien)
    # Wir nutzen eine temporäre Liste, um Konflikte beim Verschieben zu vermeiden
    Get-ChildItem -Path ".\$repoName\*" | ForEach-Object {
        Move-Item -Path $_.FullName -Destination "." -Force
    }

    # Den leeren Unterordner löschen
    Remove-Item ".\$repoName" -Force

    # 4. Änderungen in Git registrieren und hochladen
    git add .
    git commit -m "Fix: Removed redundant subfolder and flattened structure"
    git push origin main # Oder 'master', falls dein Default-Branch so heißt

    Write-Host "Reparatur abgeschlossen und auf GitHub gepusht!" -ForegroundColor Green
} else {
    Write-Host "Keine doppelte Ordnerstruktur '$repoName/$repoName' gefunden." -ForegroundColor Red
}

# Zurück zum Ursprung und Temp aufräumen
Set-Location ..
# Optional: Remove-Item $tempPath -Recurse -Force
```

##### Usage

```
cd "C:\WINDOWS\System32"
.\repair-git-folder-structure.ps1 -RepoUrl "https://github.com/YourUserName/YourProject.git"
```

## Linux

### Bash

### Shell
