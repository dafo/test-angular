# Private Repo Workflow Fixes

## Issues Found

### 1. ❌ Wrong quotes in "Change migrations imports"
```yaml
# BROKEN - has double single quotes ''
$content = $content -replace "from ''igniteui-angular\/migrations\/common"

# FIXED - single quote '
$content = $content -replace "from 'igniteui-angular\/migrations\/common"
```

### 2. ❌ Multi-entry-points replacement can double-prefix
```yaml
# BROKEN - will replace even if already @infragistics/
$content = $content -replace "from ([`"'])igniteui-angular", 'from $1@infragistics/igniteui-angular'

# FIXED - only replace if NOT already @infragistics/
$content = $content -replace "from ([`"'])igniteui-angular(?!.*@infragistics)", 'from $1@infragistics/igniteui-angular'
```

### 3. ⚠️ Missing NODE_OPTIONS for build
Add to the build step:
```yaml
- name: npm run build:lib
  run: npm run build:lib
  env:
    NODE_OPTIONS: --max-old-space-size=8192
```

## Complete Fixed Steps

Replace these steps in your private repo workflow:

### Fix 1: Change migrations imports (line ~143)
```yaml
- name: Change migrations imports
  shell: pwsh
  run: |
    Get-ChildItem -Path "projects\igniteui-angular\migrations" -Recurse -Filter "*.ts" | ForEach-Object {
      $content = Get-Content $_.FullName -Raw
      # Fixed: single quote instead of double single quote
      $content = $content -replace "from 'igniteui-angular/migrations/common", "from '@infragistics/igniteui-angular/migrations/common"
      Set-Content $_.FullName -Value $content -NoNewline
    }
```

### Fix 2: Multi-entry-points replace imports (line ~149)
```yaml
- name: Multi-entry-points replace imports (21 and above)
  if: startsWith(github.event.client_payload.release_tag, '21') || startsWith(github.event.client_payload.release_tag, 'v21')
  shell: pwsh
  run: |
    Get-ChildItem -Path "projects\igniteui-angular" -Recurse -Filter "*.ts" | ForEach-Object {
      $content = Get-Content $_.FullName -Raw
      # Fixed: negative lookahead to prevent double-prefixing
      # Only replace if NOT already @infragistics/
      $content = $content -replace "from ([`"'])igniteui-angular(?!.*@infragistics)", 'from $1@infragistics/igniteui-angular'
      Set-Content $_.FullName -Value $content -NoNewline
    }
```

### Fix 3: Add NODE_OPTIONS to build step (line ~165)
```yaml
- name: npm run build:lib
  run: npm run build:lib
  env:
    NODE_OPTIONS: --max-old-space-size=8192
```

### Fix 4: Better order - do imports replacement BEFORE animations
Move "Change animation imports" to happen AFTER "Multi-entry-points replace imports" to avoid conflicts.

## Better Alternative: Use a Script

Instead of doing inline regex replacements, create a PowerShell script that's easier to test:

```yaml
- name: Apply licensed transformations
  shell: pwsh
  run: |
    # Create transformation script
    $script = @'
    function Replace-InFile {
      param($Pattern, $Path, $Replacement = '', $Filter = "*.ts")
      Get-ChildItem -Path $Path -Recurse -Filter $Filter | ForEach-Object {
        try {
          $content = Get-Content $_.FullName -Raw -ErrorAction Stop
          $newContent = $content -replace $Pattern, $Replacement
          if ($content -ne $newContent) {
            Set-Content $_.FullName -Value $newContent -NoNewline
            Write-Host "✓ Updated: $($_.Name)"
          }
        } catch {
          Write-Warning "Failed to process: $($_.FullName) - $_"
        }
      }
    }
    
    # Apply transformations
    Write-Host "Removing watermark from package.json..."
    Replace-InFile -Pattern '(,\r?\n\s+"igniteui-trial-watermark":\s+"[^"]*")' -Path "." -Filter "package.json"
    
    Write-Host "Updating package names..."
    Replace-InFile -Pattern '"name":\s*"\w+-\w+"' -Path "." -Filter "package.json" -Replacement '"name": "@infragistics/igniteui-angular"'
    
    Write-Host "Updating imports..."
    Replace-InFile -Pattern "from ([`"'])igniteui-angular([`"'/])" -Path "projects\igniteui-angular" -Replacement 'from $1@infragistics/igniteui-angular$2'
    
    Write-Host "✅ Transformations complete"
'@
    
    Invoke-Expression $script
```

## Testing Locally

Test these replacements locally before pushing:

```powershell
# Backup first
Copy-Item -Path "projects\igniteui-angular" -Destination "projects\igniteui-angular.backup" -Recurse

# Run your replacements
# ... your replacement commands here ...

# Test build
npm run build:lib

# If it fails, restore backup:
# Remove-Item -Path "projects\igniteui-angular" -Recurse -Force
# Rename-Item -Path "projects\igniteui-angular.backup" -NewName "igniteui-angular"
```
