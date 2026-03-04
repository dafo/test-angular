# Testing the Fixed Workflow Locally

Before deploying to your private repo, test the transformations locally to ensure they work.

## Quick Test Script

Save this as `test-transformations.ps1` and run it:

```powershell
# Create a test copy
$testDir = "test-licensed-build"
if (Test-Path $testDir) {
    Remove-Item $testDir -Recurse -Force
}
Copy-Item -Path "projects\igniteui-angular" -Destination $testDir -Recurse

Write-Host "🧪 Testing transformations..." -ForegroundColor Cyan

# Test 1: Multi-entry-points replacement
Write-Host "`n1️⃣ Testing import replacements..."
Get-ChildItem -Path $testDir -Recurse -Filter "*.ts" | Select-Object -First 5 | ForEach-Object {
    $content = Get-Content $_.FullName -Raw
    $before = ($content -split "`n" | Select-String "from [`"']igniteui-angular" | Select-Object -First 3)
    
    # Apply fix
    $newContent = $content -replace "from ([`"'])igniteui-angular/", 'from $1@infragistics/igniteui-angular/'
    $newContent = $newContent -replace "from ([`"'])igniteui-angular([`"'])", 'from $1@infragistics/igniteui-angular$2'
    
    $after = ($newContent -split "`n" | Select-String "from [`"']@infragistics" | Select-Object -First 3)
    
    if ($before -and $after) {
        Write-Host "  ✓ File: $($_.Name)" -ForegroundColor Green
        Write-Host "    Before: $($before -join ', ')" -ForegroundColor Yellow
        Write-Host "    After:  $($after -join ', ')" -ForegroundColor Green
    }
}

# Test 2: Check migration imports
Write-Host "`n2️⃣ Testing migration imports..."
$migrationFiles = Get-ChildItem -Path "$testDir\migrations" -Recurse -Filter "*.ts" -ErrorAction SilentlyContinue
if ($migrationFiles) {
    $migrationFiles | Select-Object -First 2 | ForEach-Object {
        $content = Get-Content $_.FullName -Raw
        $hasBadImport = $content -match "from ''igniteui-angular"
        $hasGoodImport = $content -match "from '@infragistics/igniteui-angular"
        
        if ($hasBadImport) {
            Write-Host "  ❌ Found double quote issue in: $($_.Name)" -ForegroundColor Red
        } elseif ($hasGoodImport) {
            Write-Host "  ✓ Correct import in: $($_.Name)" -ForegroundColor Green
        }
    }
}

# Test 3: Verify no double-prefixing
Write-Host "`n3️⃣ Checking for double-prefixing..."
Get-ChildItem -Path $testDir -Recurse -Filter "*.ts" | ForEach-Object {
    $content = Get-Content $_.FullName -Raw
    if ($content -match "@infragistics/@infragistics") {
        Write-Host "  ❌ DOUBLE PREFIX FOUND: $($_.FullName)" -ForegroundColor Red
    }
}

Write-Host "`n✅ Test complete. Check results above." -ForegroundColor Cyan
Write-Host "To clean up: Remove-Item '$testDir' -Recurse -Force" -ForegroundColor Gray

# Cleanup
Read-Host "`nPress Enter to remove test directory"
Remove-Item $testDir -Recurse -Force
Write-Host "✓ Cleaned up" -ForegroundColor Green
```

## Run the Test

```powershell
# From repository root
.\test-transformations.ps1
```

## What to Look For

### ✅ Good Results

```
Before: from 'igniteui-angular/core'
After:  from '@infragistics/igniteui-angular/core'

Before: from "igniteui-angular"
After:  from "@infragistics/igniteui-angular"
```

### ❌ Bad Results (Double Prefixing)

```
from '@infragistics/@infragistics/igniteui-angular/core'
```

If you see double prefixing, the regex needs adjustment.

### ❌ Bad Results (Wrong Quotes)

```
from ''igniteui-angular/migrations/common
```

Should be single quote `'`, not double single quote `''`.

## Full Build Test

After transformations pass, test the full build:

```powershell
# Apply all transformations
# ... run all your replacement commands ...

# Clear caches
Remove-Item -Path ".angular", "node_modules/.cache" -Recurse -Force -ErrorAction SilentlyContinue

# Try build
npm run build:lib

# Check result
if ($LASTEXITCODE -eq 0) {
    Write-Host "✅ BUILD SUCCEEDED!" -ForegroundColor Green
} else {
    Write-Host "❌ BUILD FAILED" -ForegroundColor Red
}
```

## Key Differences in Fixed Version

| Issue | Before (Broken) | After (Fixed) |
|-------|----------------|---------------|
| Migration imports | `from ''igniteui-angular` | `from '@infragistics/igniteui-angular` |
| Multi-entry regex | Could double-prefix | Checks if not already prefixed |
| Import order | Animations before multi | Multi before animations |
| Memory | No NODE_OPTIONS | `NODE_OPTIONS: --max-old-space-size=8192` |
| Error handling | No try-catch | Added error handling in critical steps |

## Deploy to Private Repo

Once local tests pass:

1. Copy [.github/PRIVATE_REPO_WORKFLOW_CORRECTED.yml](.github/PRIVATE_REPO_WORKFLOW_CORRECTED.yml) to your private repo
2. Rename to `.github/workflows/licensed-release.yml`
3. Commit and push to default branch
4. Test with a new release from the public repo
