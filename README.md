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
    [Parameter(Mandatory=$false)] 
    [string]$Path = "."          # Default: Current directory
)

# 1. Save current location to the stack
Push-Location

try {
    # Resolve target directory and switch to it
    $targetDir = Get-Item $Path
    Set-Location $targetDir.FullName
    $repoName = $targetDir.Name

    Write-Host "🚀 Starting process for: $repoName" -ForegroundColor Cyan

    # 2. Git & LFS Setup
    if (-not (Test-Path ".git")) {
        git init -b main
        git lfs install
        Write-Host "✅ Git & LFS initialized." -ForegroundColor Green
    }

    # 3. LFS Tracking for large files (> 45MB)
    Write-Host "🔍 Searching for large files..." -ForegroundColor Yellow
    $largeFiles = Get-ChildItem -Recurse | Where-Object { $_.Length -gt 45MB -and -not $_.PSIsContainer }
    
    if ($largeFiles) {
        foreach ($file in $largeFiles) {
            # Convert absolute path to relative path for Git
            $relativePath = Resolve-Path -Path $file.FullName -Relative
            git lfs track $relativePath
        }
        git add .gitattributes
        Write-Host "📦 Large files are now tracked via LFS." -ForegroundColor Magenta
    }

    # 4. Add files and Commit
    git add .
    git commit -m "Initial commit with LFS tracking"

    # 5. Create GitHub Repo and push
    # Note: If repo already exists, gh will prompt or throw an error depending on environment
    gh repo create $repoName --source=. --public --push

    Write-Host "🎉 Success! $repoName is now on GitHub." -ForegroundColor Green
}
catch {
    Write-Host "❌ An error occurred: $_" -ForegroundColor Red
}
finally {
    # 6. Always return to the original starting directory
    Pop-Location
    Write-Host "📍 Returned to directory: $((Get-Location).Path)" -ForegroundColor Gray
}
```

##### Usage

```
push-to-github.ps1 "path\to\directory"
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

# 1. Extract repo name from URL (e.g., "MyProject" from "https://github.com/user/MyProject.git")
$repoName = ($RepoUrl -split "/" | Select-Object -Last 1).Replace(".git", "")
$tempPath = Join-Path $env:TEMP "git-repair-$repoName"

Write-Host "Cloning $repoName to $tempPath..." -ForegroundColor Cyan

# Delete temp folder if it exists from a previous failed run
if (Test-Path $tempPath) { 
    Remove-Item $tempPath -Recurse -Force 
}

# 2. Clone the repository
git clone $RepoUrl $tempPath
$originalLocation = Get-Location
Set-Location $tempPath

# 3. Check for the "folder-inside-folder" error
if (Test-Path ".\$repoName") {
    Write-Host "Structural error detected. Correcting..." -ForegroundColor Yellow

    # Move all items from the subfolder one level up
    # Using a loop to handle potential conflicts and hidden files
    Get-ChildItem -Path ".\$repoName\*" | ForEach-Object {
        Move-Item -Path $_.FullName -Destination "." -Force
    }

    # Remove the now empty subfolder
    Remove-Item ".\$repoName" -Force

    # 4. Register changes in Git and upload
    git add .
    git commit -m "Fix: Removed redundant subfolder and flattened structure"
    
    # Detect default branch name (main vs master)
    $branch = git symbolic-ref --short HEAD
    Write-Host "Pushing changes to $branch..." -ForegroundColor Cyan
    git push origin $branch

    Write-Host "Repair complete and pushed to GitHub!" -ForegroundColor Green
} else {
    Write-Host "No redundant folder structure '$repoName/$repoName' found." -ForegroundColor Red
}

# Return to original location and clean up temp files
Set-Location $originalLocation
# Optional: Remove-Item $tempPath -Recurse -Force
Write-Host "📍 Back in: $((Get-Location).Path)" -ForegroundColor Gray
```

##### Usage

```
repair-git-folder-structure.ps1 -RepoUrl "https://github.com/YourUserName/YourProject.git"
```

#### Migrate-to-Github

Problem: Sometimes you have the problem that you would like to fork, but for example, a fork from Bitbucket to GitHub is not possible.
Solution: The solution is called mirroring. You essentially clone the repository and then upload it to GitHub.

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
code migrate-to-github.ps1
```

Then enter this code and save as admin:

```
param (
    [Parameter(Mandatory=$true)]
    [string]$SourceUrl,
    
    [Parameter(Mandatory=$false)]
    [string]$GhUser = "MyGhUser"
)

# 1. Save current location
Push-Location

try {
    # Extract Repo Name
    $repoName = ($SourceUrl -split "/" | Select-Object -Last 1).Replace(".git", "")
    $tempFolder = "$repoName.git"

    Write-Host "🚀 Starting migration for: $repoName" -ForegroundColor Cyan

    # 2. Mirror clone the repository (includes all branches, tags, and refs)
    Write-Host "📥 Mirroring from source..." -ForegroundColor Yellow
    git clone --mirror $SourceUrl $tempFolder
    
    if (-not (Test-Path $tempFolder)) {
        throw "Failed to clone source repository."
    }

    Set-Location $tempFolder

    # 3. Create the repository on GitHub
    Write-Host "📤 Creating and pushing to GitHub..." -ForegroundColor Yellow
    # Note: --source=. uses the current mirrored metadata to populate the new repo
    gh repo create "$GhUser/$repoName" --public --source=. --push

    Write-Host "✅ Success! Repository is now at: https://github.com/$GhUser/$repoName" -ForegroundColor Green
}
catch {
    Write-Host "❌ An error occurred: $_" -ForegroundColor Red
}
finally {
    # 4. Cleanup and return
    Set-Location ..
    if (Test-Path $tempFolder) {
        Remove-Item $tempFolder -Recurse -Force
        Write-Host "🧹 Temporary mirror folder removed." -ForegroundColor Gray
    }
    Pop-Location
    Write-Host "📍 Returned to: $((Get-Location).Path)" -ForegroundColor Gray
}
```

##### Usage

```
migrate-to-github.ps1 -SourceUrl "https://bitbucket.org/user/repo.git" -GhUser "MyGhUser"
```

## Linux

### Bash / Shell

#### Push-to-Github

Problem: You have many folders that you want to push to GitHub, but you want to save yourself the time of creating the external repository each time.
Solution: If the directory to be pushed already has the name of your repository, you can use this simple script.

##### Pre-Installation

1. At first you have to install `Git`. (I assume that most people already have Git pre-installed before they encounter their “problem.”)
2. Then you have to install `GitHub CLI (gh)`
3. Then you have to log in once with `gh`.

```
sudo apt update && sudo apt install gh git git-lfs -y
gh auth login
```

##### Installation

```
nano /usr/local/bin/push-to-github.bash
```

Then enter this code and save it:

```
#!/bin/bash

# Target directory from argument or current directory
TARGET_DIR="${1:-.}"
# Save the starting directory (like Push-Location)
START_DIR=$(pwd)

# Resolve absolute path
cd "$TARGET_DIR" || exit
REPO_NAME=$(basename "$(pwd)")

echo -e "\e[36m🚀 Starting process for: $REPO_NAME\e[0m"

# 1. Git & LFS Setup
if [ ! -d ".git" ]; then
    git init -b main
    git lfs install
    echo -e "\e[32m✅ Git & LFS initialized.\e[0m"
fi

# 2. LFS Tracking for files > 45MB
echo -e "\e[33m🔍 Searching for large files...\e[0m"
# Find files larger than 45MB and track them
find . -type f -size +45M -not -path '*/.*' | while read -r file; do
    git lfs track "$file"
done

if [ -f ".gitattributes" ]; then
    git add .gitattributes
    echo -e "\e[35m📦 Large files are now tracked via LFS.\e[0m"
fi

# 3. Add files and Commit
git add .
git commit -m "Initial commit with LFS tracking"

# 4. Create GitHub Repo and push
gh repo create "$REPO_NAME" --source=. --public --push

echo -e "\e[32m🎉 Success! $REPO_NAME is now on GitHub.\e[0m"

# 5. Return to original directory (like Pop-Location)
cd "$START_DIR"
echo -e "\e[90m📍 Returned to directory: $(pwd)\e[0m"
```

At least you have to make it executable:

```
chmod +x /usr/local/bin/push-to-github.bash
```

##### Usage

```
push-to-github.bash "path\to\directory"
```

#### Repair-Git-Folder-Structure

Problem: Sometimes it happens that you push a subfolder instead of the contents of that folder. This creates an incorrect folder hierarchy in the Git repository.
Solution: The solution is to clone the repository, move all files up one level, delete the subfolder, commit all changes, and push again.

##### Installation

```
nano /usr/local/bin/repair-git-folder-structure.bash
```

Then enter this code and save it:

```
#!/bin/bash

if [ -z "$1" ]; then
    echo "Usage: $0 <GitHub-Repo-URL>"
    exit 1
fi

REPO_URL="$1"
# Extract repo name from URL
REPO_NAME=$(basename "$REPO_URL" .git)
TEMP_PATH="/tmp/git-repair-$REPO_NAME"
START_DIR=$(pwd)

echo -e "\e[36mCloning $REPO_NAME to $TEMP_PATH...\e[0m"

# Clean up old temp runs
rm -rf "$TEMP_PATH"

# 1. Clone the repo
git clone "$REPO_URL" "$TEMP_PATH"
cd "$TEMP_PATH" || exit

# 2. Check for the "folder-inside-folder" error
if [ -d "$REPO_NAME" ]; then
    echo -e "\e[33mStructural error detected. Correcting...\e[0m"

    # Move all items (including hidden ones, excluding . and ..) up
    find "$REPO_NAME" -mindepth 1 -maxdepth 1 -exec mv -t . {} +

    # Remove the now empty subfolder
    rmdir "$REPO_NAME"

    # 3. Register changes and push
    git add .
    git commit -m "Fix: Removed redundant subfolder and flattened structure"
    
    # Detect default branch (main or master)
    BRANCH=$(git symbolic-ref --short HEAD)
    echo -e "\e[36mPushing changes to $BRANCH...\e[0m"
    git push origin "$BRANCH"

    echo -e "\e[32mRepair complete and pushed to GitHub!\e[0m"
else
    echo -e "\e[31mNo redundant folder structure '$REPO_NAME/$REPO_NAME' found.\e[0m"
fi

# 4. Clean up and return
cd "$START_DIR"
echo -e "\e[90m📍 Back in: $(pwd)\e[0m"
```

At least you have to make it executable:

```
chmod +x /usr/local/bin/repair-git-folder-structure.bash
```

##### Usage

```
repair-git-folder-structure.bash -RepoUrl "https://github.com/YourUserName/YourProject.git"
```

#### Migrate-to-Github

Problem: Sometimes you have the problem that you would like to fork, but for example, a fork from Bitbucket to GitHub is not possible.
Solution: The solution is called mirroring. You essentially clone the repository and then upload it to GitHub.

##### Pre-Installation

1. At first you have to install `Git`. (I assume that most people already have Git pre-installed before they encounter their “problem.”)
2. Then you have to install `GitHub CLI (gh)`
3. Then you have to log in once with `gh`.

```
sudo apt update && sudo apt install gh git git-lfs -y
gh auth login
```

##### Installation

```
nano /usr/local/bin/migrate-to-github.bash
```

Then enter this code and save as admin:

```
#!/bin/bash

# Default GitHub User
GH_USER=${GH_USER:-"MyGhUser"}

# 1. Check for argument
if [ -z "$1" ]; then
    echo -e "\e[31mUsage: $0 <source-git-url>\e[0m"
    exit 1
fi

SOURCE_URL=$1
REPO_NAME=$(basename "$SOURCE_URL" .git)
TEMP_FOLDER="$REPO_NAME.git"
START_DIR=$(pwd)

echo -e "\e[36m🚀 Starting migration for: $REPO_NAME\e[0m"

# 2. Mirror clone (Local temporary folder)
echo -e "\e[33m📥 Mirroring from source...\e[0m"
git clone --mirror "$SOURCE_URL" "$TEMP_FOLDER"

if [ ! -d "$TEMP_FOLDER" ]; then
    echo -e "\e[31m❌ Error: Failed to clone repository.\e[0m"
    exit 1
fi

cd "$TEMP_FOLDER" || exit

# 3. Create and Push to GitHub
echo -e "\e[33m📤 Creating and pushing to GitHub...\e[0m"
gh repo create "$GH_USER/$REPO_NAME" --public --source=. --push

echo -e "\e[32m✅ Success! Repository is now at: https://github.com/$GH_USER/$REPO_NAME\e[0m"

# 4. Cleanup and return
cd "$START_DIR"
if [ -d "$TEMP_FOLDER" ]; then
    rm -rf "$TEMP_FOLDER"
    echo -e "\e[90m🧹 Temporary mirror folder removed.\e[0m"
fi

echo -e "\e[90m📍 Returned to directory: $(pwd)\e[0m"
```

At least you have to make it executable:

```
chmod +x /usr/local/bin/migrate-to-github.bash
```

##### Usage

```
migrate-to-github.bash -SourceUrl "https://bitbucket.org/user/repo.git" -GhUser "MyGhUser"
```





