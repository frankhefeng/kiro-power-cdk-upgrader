# CDK Upgrade Troubleshooting Guide

This steering file provides solutions for common issues encountered during CDK upgrades, including network problems, system issues, TypeScript compilation errors, git-related problems, and cross-platform compatibility issues.

## Table of Contents

1. [Network Issues](#network-issues)
2. [System Issues](#system-issues)
3. [TypeScript Compilation Errors](#typescript-compilation-errors)
4. [Git Issues](#git-issues)
5. [Cross-Platform Compatibility](#cross-platform-compatibility)

---

## Network Issues

### Overview

Network connectivity problems can cause failures during CDK template generation and npm package installation. This section provides retry strategies and solutions for common network-related issues.

### Retry Strategy with Exponential Backoff

When network operations fail, implement exponential backoff retry:

**Retry Configuration**:
| Attempt | Wait Time | Total Elapsed |
|---------|-----------|---------------|
| 1       | 0s        | 0s            |
| 2       | 5s        | 5s            |
| 3       | 15s       | 20s           |
| 4       | 30s       | 50s           |
| Max     | 60s       | -             |

**Retry Algorithm**:
```
function executeWithRetry(operation, maxAttempts = 3):
  for attempt in 1 to maxAttempts:
    try:
      result = operation()
      return result
    catch NetworkError as error:
      if attempt == maxAttempts:
        throw error
      
      waitTime = min(5 * (2 ^ (attempt - 1)), 60)  # 5s, 10s, 20s, 40s, 60s max
      
      print("Network error: " + error.message)
      print("Retrying in " + waitTime + " seconds... (attempt " + (attempt + 1) + " of " + maxAttempts + ")")
      
      sleep(waitTime)
```

### npx cdk@latest Failure Handling

#### Symptom: Network Timeout

```
npm ERR! network request to https://registry.npmjs.org/aws-cdk failed
npm ERR! network This is a problem related to network connectivity.
npm ERR! network In most cases you are behind a proxy or have bad network settings.
```

**Solutions**:

1. **Check Internet Connection**:
   ```bash
   # Test basic connectivity
   ping -c 3 registry.npmjs.org
   
   # Test HTTPS connectivity
   curl -I https://registry.npmjs.org/aws-cdk
   ```

2. **Configure Proxy Settings** (if behind corporate proxy):
   ```bash
   # Set npm proxy
   npm config set proxy http://proxy.company.com:8080
   npm config set https-proxy http://proxy.company.com:8080
   
   # Or use environment variables
   export HTTP_PROXY=http://proxy.company.com:8080
   export HTTPS_PROXY=http://proxy.company.com:8080
   ```

3. **Clear npm Cache**:
   ```bash
   npm cache clean --force
   npx clear-npx-cache
   ```

4. **Use Alternative Registry** (temporary):
   ```bash
   npm config set registry https://registry.npmmirror.com
   # Remember to reset after: npm config set registry https://registry.npmjs.org
   ```

#### Symptom: SSL Certificate Error

```
npm ERR! code UNABLE_TO_GET_ISSUER_CERT_LOCALLY
npm ERR! unable to get local issuer certificate
```

**Solutions**:

1. **Update CA Certificates**:
   ```bash
   # macOS
   brew install ca-certificates
   
   # Ubuntu/Debian
   sudo apt-get update && sudo apt-get install ca-certificates
   ```

2. **Configure npm to Use System CA** (not recommended for production):
   ```bash
   npm config set strict-ssl false  # Temporary workaround only
   ```

3. **Set Custom CA Certificate**:
   ```bash
   npm config set cafile /path/to/corporate-ca.crt
   ```

#### Symptom: DNS Resolution Failure

```
npm ERR! code EAI_AGAIN
npm ERR! errno EAI_AGAIN
npm ERR! request to https://registry.npmjs.org/aws-cdk failed
```

**Solutions**:

1. **Check DNS Configuration**:
   ```bash
   # Test DNS resolution
   nslookup registry.npmjs.org
   
   # Try alternative DNS
   nslookup registry.npmjs.org 8.8.8.8
   ```

2. **Flush DNS Cache**:
   ```bash
   # macOS
   sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
   
   # Linux
   sudo systemd-resolve --flush-caches
   ```

3. **Use IP Address Directly** (temporary):
   ```bash
   # Add to /etc/hosts
   104.16.16.35 registry.npmjs.org
   ```

### Network Issue Output Format

```
=== Network Error Detected ===

✗ Failed to execute: npx cdk@latest init app --language typescript --generate-only

Error: npm ERR! network request failed
       npm ERR! network timeout at: https://registry.npmjs.org/aws-cdk

Diagnosis:
  - Internet connectivity: ✓ OK
  - DNS resolution: ✓ OK
  - npm registry access: ✗ TIMEOUT

Suggested actions:
  1. Wait a moment and retry (registry may be temporarily slow)
  2. Check if you're behind a proxy and configure npm accordingly
  3. Try clearing npm cache: npm cache clean --force

Retrying in 5 seconds... (attempt 2 of 3)
```

### npm Install Network Failures

#### Symptom: Package Download Timeout

```
npm ERR! network timeout at: https://registry.npmjs.org/aws-cdk-lib/-/aws-cdk-lib-2.175.3.tgz
```

**Solutions**:

1. **Increase Timeout**:
   ```bash
   npm config set fetch-timeout 120000  # 2 minutes
   npm config set fetch-retries 5
   ```

2. **Use Offline Mode** (if packages cached):
   ```bash
   npm install --prefer-offline
   ```

3. **Install Packages Individually**:
   ```bash
   # If bulk install fails, try installing CDK packages separately
   npm install aws-cdk-lib@2.175.3
   npm install aws-cdk@2.1005.0 --save-dev
   ```

### Network Recovery Output

```
=== Network Recovery ===

Previous attempt failed due to network timeout.

Retry attempt 2 of 3...
  Waiting 5 seconds before retry...
  Executing: npx cdk@latest init app --language typescript --generate-only
  
✓ Command succeeded on retry!

Continuing with upgrade process...
```

---

## System Issues

### Overview

System-level issues such as insufficient disk space and file permission errors can prevent successful CDK upgrades. This section provides detection and resolution guidance.

### Disk Space Warnings

#### Pre-Flight Disk Space Check

Before starting upgrade operations, check available disk space:

```bash
# Check available space in current directory
df -h .

# Check space needed (approximate)
# - Template generation: ~50MB
# - npm install: ~200-500MB per project
# - cdk synth output: ~10-50MB per project
```

**Minimum Requirements**:
| Operation | Minimum Space | Recommended |
|-----------|---------------|-------------|
| Template generation | 100MB | 500MB |
| npm install (per project) | 500MB | 1GB |
| cdk synth (per project) | 100MB | 500MB |
| Total (single project) | 700MB | 2GB |

#### Symptom: Disk Space Exhausted

```
npm ERR! code ENOSPC
npm ERR! syscall write
npm ERR! errno -28
npm ERR! nospc ENOSPC: no space left on device
```

**Solutions**:

1. **Check Current Disk Usage**:
   ```bash
   # Overall disk usage
   df -h
   
   # Find large directories
   du -sh */ | sort -hr | head -20
   
   # Find large files
   find . -type f -size +100M -exec ls -lh {} \;
   ```

2. **Clean npm Cache**:
   ```bash
   # Show cache size
   npm cache ls
   
   # Clean cache
   npm cache clean --force
   
   # Typical savings: 500MB - 2GB
   ```

3. **Remove node_modules from Old Projects**:
   ```bash
   # Find all node_modules directories
   find . -name "node_modules" -type d -prune
   
   # Remove node_modules from specific project
   rm -rf /path/to/old-project/node_modules
   ```

4. **Clean CDK Output Directories**:
   ```bash
   # Remove cdk.out directories
   find . -name "cdk.out" -type d -exec rm -rf {} +
   
   # Remove .cdk.staging directories
   find . -name ".cdk.staging" -type d -exec rm -rf {} +
   ```

5. **Clean Temporary Files**:
   ```bash
   # macOS
   rm -rf ~/Library/Caches/npm
   rm -rf /tmp/cdk-*
   
   # Linux
   rm -rf ~/.npm/_cacache
   rm -rf /tmp/cdk-*
   ```

#### Disk Space Warning Output

```
=== Disk Space Check ===

Checking available disk space...

Current usage:
  Filesystem: /dev/disk1s1
  Total: 500GB
  Used: 480GB (96%)
  Available: 20GB

⚠ Warning: Low disk space detected!

Estimated space needed for upgrade:
  - Template generation: ~50MB
  - npm install (3 projects): ~1.5GB
  - cdk synth (3 projects): ~150MB
  - Total estimated: ~1.7GB

Available space (20GB) is sufficient, but consider cleaning up:
  - npm cache: ~1.2GB (npm cache clean --force)
  - Old cdk.out directories: ~500MB

Proceed with upgrade? (y/n): _
```

### File Permission Errors

#### Symptom: Permission Denied on npm Operations

```
npm ERR! code EACCES
npm ERR! syscall mkdir
npm ERR! path /usr/local/lib/node_modules
npm ERR! errno -13
npm ERR! Error: EACCES: permission denied
```

**Solutions**:

1. **Fix npm Global Directory Permissions** (Recommended):
   ```bash
   # Create npm global directory in home
   mkdir -p ~/.npm-global
   
   # Configure npm to use it
   npm config set prefix '~/.npm-global'
   
   # Add to PATH (add to ~/.bashrc or ~/.zshrc)
   export PATH=~/.npm-global/bin:$PATH
   
   # Reload shell
   source ~/.bashrc  # or source ~/.zshrc
   ```

2. **Fix Ownership of npm Directories**:
   ```bash
   # Check current ownership
   ls -la ~/.npm
   ls -la /usr/local/lib/node_modules
   
   # Fix ownership (replace YOUR_USER with your username)
   sudo chown -R $(whoami) ~/.npm
   sudo chown -R $(whoami) /usr/local/lib/node_modules
   ```

3. **Use nvm** (Recommended for Development):
   ```bash
   # Install nvm
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
   
   # Install Node.js via nvm (no sudo needed)
   nvm install 20
   nvm use 20
   ```

#### Symptom: Permission Denied on Project Files

```
Error: EACCES: permission denied, open '/path/to/project/package.json'
```

**Solutions**:

1. **Check File Ownership**:
   ```bash
   ls -la /path/to/project/
   ```

2. **Fix Project Directory Permissions**:
   ```bash
   # Fix ownership
   sudo chown -R $(whoami) /path/to/project/
   
   # Fix permissions
   chmod -R u+rw /path/to/project/
   ```

3. **Check for Locked Files**:
   ```bash
   # macOS - check for locked files
   ls -lO /path/to/project/
   
   # Remove lock flag
   chflags -R nouchg /path/to/project/
   ```

#### Symptom: Permission Denied on Temporary Directory

```
Error: EACCES: permission denied, mkdir '/path/to/project/temp'
```

**Solutions**:

1. **Use Alternative Temp Directory**:
   ```bash
   # Set custom temp directory
   export TMPDIR=~/tmp
   mkdir -p ~/tmp
   
   # Or for npm specifically
   npm config set tmp ~/tmp
   ```

2. **Fix System Temp Directory Permissions**:
   ```bash
   # Check temp directory
   ls -la /tmp
   
   # Fix permissions (if needed)
   sudo chmod 1777 /tmp
   ```

### Permission Error Output Format

```
=== Permission Error Detected ===

✗ Failed to write file: /path/to/project/package.json

Error: EACCES: permission denied

File details:
  Path: /path/to/project/package.json
  Owner: root
  Permissions: -rw-r--r--
  Current user: developer

The file is owned by 'root' but you are running as 'developer'.

Solutions:
  1. Fix file ownership:
     sudo chown $(whoami) /path/to/project/package.json
  
  2. Fix directory ownership (recommended):
     sudo chown -R $(whoami) /path/to/project/
  
  3. Run with elevated permissions (not recommended):
     sudo npm install

After fixing permissions, run the upgrade again.
```

---

## TypeScript Compilation Errors

### Overview

After upgrading CDK packages, TypeScript compilation errors may occur due to API changes, deprecated constructs, or type definition updates. This section covers common errors and their fixes.

### Common Errors After Upgrade

#### Error: Property Does Not Exist

```typescript
error TS2339: Property 'runtime' does not exist on type 'FunctionProps'.
```

**Cause**: API changed in newer CDK version.

**Fix**:
```typescript
// Before (CDK 2.150.x)
new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.NODEJS_18_X,  // Property may have moved
  ...
});

// After (CDK 2.175.x) - Check current API
new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.NODEJS_20_X,  // Use updated runtime
  ...
});
```

**Resolution Steps**:
1. Check the CDK API documentation for the new version
2. Look for migration guides in CDK release notes
3. Update property names/values as needed

#### Error: Type Mismatch

```typescript
error TS2322: Type 'string' is not assignable to type 'Duration'.
```

**Cause**: Type definitions became stricter.

**Fix**:
```typescript
// Before
timeout: '30',  // String was accepted

// After
import { Duration } from 'aws-cdk-lib';
timeout: Duration.seconds(30),  // Must use Duration type
```

#### Error: Module Not Found

```typescript
error TS2307: Cannot find module '@aws-cdk/aws-lambda' or its corresponding type declarations.
```

**Cause**: Using CDK v1 import style in CDK v2 project.

**Fix**:
```typescript
// Before (CDK v1 style - WRONG)
import * as lambda from '@aws-cdk/aws-lambda';

// After (CDK v2 style - CORRECT)
import * as lambda from 'aws-cdk-lib/aws-lambda';
// Or
import { aws_lambda as lambda } from 'aws-cdk-lib';
```

#### Error: Deprecated API Usage

```typescript
error TS2345: Argument of type '{ vpc: Vpc; }' is not assignable to parameter of type 'DatabaseInstanceProps'.
  Property 'engine' is missing in type '{ vpc: Vpc; }' but required in type 'DatabaseInstanceProps'.
```

**Cause**: Required properties added in newer version.

**Fix**:
```typescript
// Before (property was optional)
new rds.DatabaseInstance(this, 'Database', {
  vpc: myVpc,
});

// After (property is now required)
new rds.DatabaseInstance(this, 'Database', {
  vpc: myVpc,
  engine: rds.DatabaseInstanceEngine.postgres({ version: rds.PostgresEngineVersion.VER_15 }),
});
```

### Suggested Fixes for Common Patterns

#### Pattern 1: Lambda Runtime Updates

```typescript
// Old runtimes (deprecated)
lambda.Runtime.NODEJS_14_X  // EOL
lambda.Runtime.NODEJS_16_X  // EOL
lambda.Runtime.PYTHON_3_8   // EOL
lambda.Runtime.PYTHON_3_9   // Approaching EOL

// New runtimes (recommended)
lambda.Runtime.NODEJS_20_X  // Current LTS
lambda.Runtime.NODEJS_22_X  // Latest
lambda.Runtime.PYTHON_3_12  // Current
lambda.Runtime.PYTHON_3_13  // Latest
```

#### Pattern 2: Construct ID Changes

```typescript
// Some constructs changed their ID requirements
// Before
new s3.Bucket(this, 'my-bucket');  // Lowercase with hyphens

// After (if error occurs)
new s3.Bucket(this, 'MyBucket');  // PascalCase may be required
```

#### Pattern 3: Import Path Changes

```typescript
// Consolidated imports in CDK v2
// Before (multiple imports)
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

// Alternative (single import)
import { aws_lambda as lambda, aws_s3 as s3, aws_dynamodb as dynamodb } from 'aws-cdk-lib';
```

### TypeScript Error Resolution Workflow

```
=== TypeScript Compilation Error ===

✗ cdk synth failed with TypeScript errors!

Errors found: 3

Error 1 of 3:
  File: lib/my-stack.ts
  Line: 45
  Error: TS2339: Property 'runtime' does not exist on type 'FunctionProps'.
  
  Suggested fix:
    The 'runtime' property may have been renamed or moved.
    Check CDK 2.175.x API documentation for lambda.Function.
    
    Common fix:
    - Ensure you're using lambda.Runtime.NODEJS_20_X (not deprecated runtime)
    - Check if property moved to a different configuration object

Error 2 of 3:
  File: lib/my-stack.ts
  Line: 67
  Error: TS2307: Cannot find module '@aws-cdk/aws-s3'.
  
  Suggested fix:
    You're using CDK v1 import style. Update to CDK v2:
    
    Before: import * as s3 from '@aws-cdk/aws-s3';
    After:  import * as s3 from 'aws-cdk-lib/aws-s3';

Error 3 of 3:
  File: lib/database-stack.ts
  Line: 23
  Error: TS2345: Property 'engine' is missing.
  
  Suggested fix:
    The 'engine' property is now required for DatabaseInstance.
    Add the engine configuration:
    
    engine: rds.DatabaseInstanceEngine.postgres({
      version: rds.PostgresEngineVersion.VER_15
    })

Please fix these errors and run the upgrade validation again.
```

### Preserving Type Annotations

When upgrading, ensure type annotations remain valid:

```typescript
// Type annotations may need updates
// Before
const bucket: s3.Bucket = new s3.Bucket(this, 'Bucket');

// After (if Bucket type changed)
const bucket: s3.IBucket = new s3.Bucket(this, 'Bucket');
// Or let TypeScript infer
const bucket = new s3.Bucket(this, 'Bucket');
```

**Best Practice**: Let TypeScript infer types where possible to reduce upgrade friction.

---

## Git Issues

### Overview

Git integration is essential for tracking upgrade changes and enabling rollback. This section covers common git issues including dirty working directory handling, branch requirements, and git operation failures.

### Dirty Working Directory Handling

#### Pre-Upgrade Git Status Check

Before any upgrade operations, verify the working directory is clean:

```bash
cd "<project_path>"
git status --porcelain
```

**Expected output for clean directory**: (empty - no output)

**Output with uncommitted changes**:
```
 M lib/my-stack.ts
 M package.json
?? new-file.ts
```

#### Dirty Directory Error Output

```
=== Git Status Check: my-service ===

✗ Working directory is not clean!

Uncommitted changes detected:
  Modified:   lib/my-stack.ts
  Modified:   package.json
  Untracked:  new-file.ts

The upgrade process requires a clean working directory to:
  - Ensure upgrade changes can be clearly identified
  - Enable easy rollback if issues occur
  - Create clean, atomic commits

Options to resolve:

1. Commit your current changes:
   git add .
   git commit -m "WIP: save current work before CDK upgrade"

2. Stash your changes (recommended for temporary work):
   git stash push -m "Before CDK upgrade"
   # After upgrade, restore with: git stash pop

3. Discard changes (⚠ WARNING: loses uncommitted work):
   git checkout -- .
   git clean -fd

After resolving, run the upgrade again.
```

#### Handling Specific Change Types

**Modified files**:
```bash
# View what changed
git diff lib/my-stack.ts

# Commit specific files
git add lib/my-stack.ts
git commit -m "Update stack configuration"
```

**Untracked files**:
```bash
# Add to .gitignore if not needed
echo "new-file.ts" >> .gitignore

# Or add and commit
git add new-file.ts
git commit -m "Add new file"
```

**Deleted files**:
```bash
# Confirm deletion
git add -u
git commit -m "Remove unused files"

# Or restore
git checkout -- deleted-file.ts
```

### Feature Branch Requirement

#### Branch Check Before Upgrade

**CRITICAL**: Upgrades should NOT be performed on `main` or `master` branches directly.

```bash
# Check current branch
git branch --show-current
```

**Protected branches** (do not upgrade directly):
- `main`
- `master`
- `develop`
- `production`
- `release/*`

#### Branch Check Output

```
=== Git Branch Check: my-service ===

Current branch: main

✗ Cannot upgrade on protected branch!

You are currently on the 'main' branch. CDK upgrades should be
performed on a feature branch to:
  - Enable code review before merging
  - Allow easy rollback by deleting the branch
  - Follow GitFlow/GitHub Flow best practices
  - Protect the main branch from potentially breaking changes

Please create a feature branch before proceeding:

  git checkout -b feature/cdk-upgrade
  # or
  git checkout -b chore/upgrade-cdk-2.175

After creating the feature branch, run the upgrade again.

Suggested branch names:
  - feature/cdk-upgrade
  - chore/upgrade-cdk-2.175.3
  - update/cdk-dependencies
```

#### Creating Feature Branch

```bash
# Create and switch to feature branch
git checkout -b feature/cdk-upgrade

# Or with more specific name
git checkout -b chore/upgrade-cdk-$(date +%Y%m%d)
```

#### Branch Validation Algorithm

```
function validateBranch(projectPath):
  currentBranch = exec("git branch --show-current", cwd=projectPath)
  
  protectedBranches = ["main", "master", "develop", "production"]
  protectedPatterns = ["release/*", "hotfix/*"]
  
  # Check exact matches
  if currentBranch in protectedBranches:
    return PROTECTED_BRANCH_ERROR
  
  # Check patterns
  for pattern in protectedPatterns:
    if matchesPattern(currentBranch, pattern):
      return PROTECTED_BRANCH_ERROR
  
  return OK
```

### Git Operation Failures

#### Symptom: Git Not Found

```
fatal: not a git repository (or any of the parent directories): .git
```

**Solutions**:

1. **Initialize Git Repository**:
   ```bash
   cd /path/to/project
   git init
   git add .
   git commit -m "Initial commit"
   ```

2. **Check if in Subdirectory**:
   ```bash
   # Find git root
   git rev-parse --show-toplevel
   
   # Navigate to correct directory
   cd $(git rev-parse --show-toplevel)
   ```

#### Symptom: Git Commit Failed

```
error: gpg failed to sign the data
fatal: failed to write commit object
```

**Solutions**:

1. **Disable GPG Signing Temporarily**:
   ```bash
   git config --local commit.gpgsign false
   ```

2. **Fix GPG Configuration**:
   ```bash
   # Check GPG key
   gpg --list-secret-keys --keyid-format LONG
   
   # Set correct key
   git config --global user.signingkey YOUR_KEY_ID
   
   # Start GPG agent
   gpgconf --launch gpg-agent
   ```

3. **Configure Git User**:
   ```bash
   git config --local user.email "you@example.com"
   git config --local user.name "Your Name"
   ```

#### Symptom: Permission Denied on Git Operations

```
error: could not lock config file .git/config: Permission denied
```

**Solutions**:

1. **Fix .git Directory Permissions**:
   ```bash
   chmod -R u+rw .git/
   ```

2. **Check for Locked Files**:
   ```bash
   # Remove stale lock files
   rm -f .git/index.lock
   rm -f .git/config.lock
   ```

3. **Check File Ownership**:
   ```bash
   ls -la .git/
   sudo chown -R $(whoami) .git/
   ```

#### Symptom: Merge Conflicts

```
error: Your local changes to the following files would be overwritten by merge
```

**Solutions**:

1. **Stash Changes First**:
   ```bash
   git stash
   # Perform operation
   git stash pop
   ```

2. **Commit Changes First**:
   ```bash
   git add .
   git commit -m "Save work before upgrade"
   ```

### Git Error Output Format

```
=== Git Operation Failed ===

✗ Failed to create commit!

Command: git commit -m "feat(cdk): upgrade CDK to version 2.175.3"

Error: error: gpg failed to sign the data
       fatal: failed to write commit object

Diagnosis:
  - Git repository: ✓ Valid
  - Working directory: ✓ Clean (staged changes ready)
  - GPG signing: ✗ Failed

The commit could not be created due to GPG signing failure.

Solutions:
  1. Disable GPG signing for this repository:
     git config --local commit.gpgsign false
     
  2. Fix GPG agent:
     gpgconf --launch gpg-agent
     
  3. Create commit manually after fixing:
     git commit -m "feat(cdk): upgrade CDK to version 2.175.3"

Your upgrade changes are staged and ready to commit.
After fixing the issue, create the commit manually.
```

### Git Recovery Procedures

#### Rollback Uncommitted Changes

```bash
# Discard all uncommitted changes
git checkout -- .
git clean -fd

# Or selectively
git checkout -- package.json
git checkout -- tsconfig.json
```

#### Rollback Committed Changes

```bash
# Undo last commit (keep changes staged)
git reset --soft HEAD~1

# Undo last commit (discard changes)
git reset --hard HEAD~1

# Revert specific commit (creates new commit)
git revert <commit-hash>
```

#### Recovery Output

```
=== Upgrade Rollback ===

The upgrade for 'my-service' failed during validation.

Changes made:
  - package.json (modified)
  - tsconfig.json (replaced)
  - cdk.json (merged)
  - .gitignore (updated)

Rollback options:

1. Keep changes for manual inspection:
   Changes remain in working directory for review.
   You can manually fix issues and commit.

2. Discard all changes:
   git checkout -- .
   This will restore all files to their pre-upgrade state.

3. Selective rollback:
   git checkout -- package.json  # Restore specific file

Select option (1/2/3): _
```

### Complete Git Workflow Check

```
=== Pre-Upgrade Git Validation ===

Project: my-service
Path: /workspace/deployment/cdk/my-service

Checks:
  ✓ Git repository exists
  ✓ Git is functional
  ✓ Working directory is clean
  ✓ On feature branch: feature/cdk-upgrade
  ✓ Branch is up to date with origin

Ready to proceed with upgrade.

After upgrade completes:
  - Changes will be committed automatically
  - Commit message: feat(cdk): upgrade CDK to version 2.175.3
  - Remember to push: git push origin feature/cdk-upgrade
  - Create pull request for review
```

---

## Cross-Platform Compatibility

### Overview

The CDK Upgrader must work reliably on both macOS and Linux systems. This section provides guidance for operating system detection, cross-platform command generation, shell compatibility requirements, and platform-specific troubleshooting.

### Shell Compatibility Requirements

**CRITICAL**: All shell scripts MUST be compatible with both bash and zsh (the default shells on macOS and Linux).

**Safe Shell Syntax**:
```bash
#!/bin/bash
# OR
#!/usr/bin/env bash

# Use POSIX-compatible syntax when possible
if [ "$condition" = "value" ]; then    # POSIX compatible
  echo "Safe syntax"
fi

# Variable validation before arithmetic
if [ -n "$VARIABLE" ] && [ "$VARIABLE" -gt 0 ]; then
  echo "Variable is valid and positive"
fi

# String comparisons
if [ "$OSTYPE" = "darwin" ]; then      # Exact match
  echo "macOS"
elif [ "${OSTYPE#darwin}" != "$OSTYPE" ]; then  # Prefix match (POSIX)
  echo "macOS variant"
fi
```

**Bash-specific syntax (use with caution)**:
```bash
# Only use when bash is guaranteed
if [[ "$OSTYPE" == "darwin"* ]]; then  # Bash pattern matching
  echo "macOS"
fi

# Regex matching (bash only)
if [[ "$version" =~ ^[0-9]+\.[0-9]+$ ]]; then
  echo "Valid version format"
fi
```

**Variable Validation Pattern**:
```bash
# ALWAYS validate variables before arithmetic operations
validate_numeric() {
    local var_name="$1"
    local var_value="$2"
    
    if [ -z "$var_value" ]; then
        echo "ERROR: $var_name is empty" >&2
        return 1
    fi
    
    # POSIX-compatible numeric check
    case "$var_value" in
        ''|*[!0-9]*) 
            echo "ERROR: $var_name must be numeric, got: '$var_value'" >&2
            return 1
            ;;
    esac
}

# Usage
DAYS_SINCE=$(calculate_days_since "$release_date")
validate_numeric "DAYS_SINCE" "$DAYS_SINCE"
if [ "$DAYS_SINCE" -gt 7 ]; then
    echo "Version is stable"
fi
```

### Cross-Platform Command Generation

#### Detecting Current Platform

Before generating any system-specific commands, detect the operating system:

**Detection Methods**:

1. **Using Node.js `process.platform`**:
   ```javascript
   const platform = process.platform;
   // Returns: 'darwin' (macOS), 'linux', 'win32', etc.
   
   const isMacOS = platform === 'darwin';
   const isLinux = platform === 'linux';
   ```

2. **Using Shell Commands**:
   ```bash
   # Method 1: uname command
   uname -s
   # Returns: Darwin (macOS), Linux
   
   # Method 2: Check for specific files
   if [[ -f /System/Library/CoreServices/SystemVersion.plist ]]; then
     echo "macOS"
   elif [[ -f /etc/os-release ]]; then
     echo "Linux"
   fi
   ```

3. **Architecture Detection**:
   ```bash
   # Get architecture
   uname -m
   # Returns: x86_64, arm64, aarch64
   
   # Combined platform and architecture
   uname -sm
   # Returns: Darwin arm64, Linux x86_64, etc.
   ```

#### Platform Information Structure

```typescript
interface PlatformInfo {
  platform: 'darwin' | 'linux' | 'win32';
  architecture: 'x64' | 'arm64' | 'x86';
  version?: string;
  packageManager?: 'brew' | 'apt' | 'yum' | 'dnf';
  nodeManager?: 'nvm' | 'n' | 'system';
}
```

#### Detection Output Format

```
=== Platform Detection ===

Operating System: macOS (Darwin)
Architecture: arm64 (Apple Silicon)
Version: macOS 14.2.1
Node.js Manager: nvm (detected from ~/.nvm)
Package Manager: Homebrew (detected at /opt/homebrew/bin/brew)

Platform-specific optimizations enabled:
  ✓ Using Homebrew for system packages
  ✓ Using nvm for Node.js management
  ✓ macOS-specific command syntax
```

### Cross-Platform Command Generation

#### Universal Command Patterns

Use commands that work identically on both macOS and Linux:

**Safe Cross-Platform Commands**:
```bash
# File operations
ls -la                    # List files (same on both)
mkdir -p directory        # Create directory (same on both)
rm -rf directory          # Remove directory (same on both)
cp -r source dest         # Copy recursively (same on both)

# Process management
ps aux                    # List processes (same on both)
kill -9 PID              # Kill process (same on both)

# Network
curl -I https://example.com  # HTTP request (same on both)
ping -c 3 hostname          # Ping (same on both)

# Git operations
git status               # All git commands are identical
git add .
git commit -m "message"
```

**Commands to Avoid** (platform-specific):
```bash
# These vary between platforms
open file.txt            # macOS only (use xdg-open on Linux)
pbcopy                   # macOS only (use xclip on Linux)
launchctl               # macOS only (use systemctl on Linux)
brew install            # macOS/Linux Homebrew (use apt on Ubuntu)
```

#### Cross-Platform Command Generation Strategy

```typescript
interface CommandGenerator {
  // Generate platform-neutral commands when possible
  generateUniversalCommand(operation: string, params: any): string;
  
  // Generate platform-specific commands when needed
  generatePlatformCommand(operation: string, platform: Platform, params: any): string;
  
  // Validate command compatibility
  isCommandCrossPlatform(command: string): boolean;
}

// Example implementation
function generateNodeUpgradeCommand(platform: Platform, targetVersion: string): string[] {
  const commands: string[] = [];
  
  if (platform === 'darwin') {
    // macOS options
    if (hasNVM()) {
      commands.push(`nvm install ${targetVersion}`);
      commands.push(`nvm use ${targetVersion}`);
    } else if (hasHomebrew()) {
      commands.push(`brew install node@${targetVersion}`);
      commands.push(`brew link node@${targetVersion} --force`);
    } else {
      commands.push(`# Please install Node.js ${targetVersion} from nodejs.org`);
    }
  } else if (platform === 'linux') {
    // Linux options
    if (hasNVM()) {
      commands.push(`nvm install ${targetVersion}`);
      commands.push(`nvm use ${targetVersion}`);
    } else if (hasApt()) {
      commands.push(`curl -fsSL https://deb.nodesource.com/setup_${targetVersion}.x | sudo -E bash -`);
      commands.push(`sudo apt-get install -y nodejs`);
    } else {
      commands.push(`# Please install Node.js ${targetVersion} using your package manager`);
    }
  }
  
  return commands;
}
```

#### Command Compatibility Guidelines

**DO**: Use these cross-platform patterns
```bash
# Environment variables
export VAR_NAME=value     # Works on both

# Path operations
cd /path/to/directory     # Absolute paths work on both
cd ./relative/path        # Relative paths work on both

# File checks
if [ -f "file.txt" ]; then    # POSIX shell syntax
  echo "File exists"
fi

# Command chaining
command1 && command2      # Sequential execution
command1 || command2      # Alternative execution
```

**DON'T**: Use these platform-specific patterns
```bash
# Platform-specific paths
/Applications/...         # macOS only
/usr/share/applications/  # Linux only

# Platform-specific commands
defaults write ...        # macOS only
gsettings set ...         # Linux only

# Shell-specific syntax
[[ condition ]]           # Bash-specific (use [ condition ] for POSIX compatibility)
```specific (use [ condition ] for POSIX)
```

### Command Simplification Strategies

#### Avoiding Command Hanging

**Problem**: Complex nested commands can hang or block execution, especially when run programmatically.

**Problematic Patterns**:
```bash
# Deeply nested commands (can hang)
cd project && npm install && (cdk synth || echo 'Failed') && cd ..

# Complex pipe chains (can block)
npm list | grep aws-cdk | awk '{print $2}' | sed 's/@//' | head -1

# Interactive commands without proper handling
npm install  # May prompt for user input

# Background processes without proper cleanup
npm run dev &  # Starts background process
```

#### Command Simplification Rules

**Rule 1**: Break complex commands into simple sequential steps
```bash
# Instead of:
cd project && npm install && cdk synth && cd ..

# Use separate commands:
cd project
npm install
cdk synth
cd ..
```

**Rule 2**: Avoid deeply nested conditional logic
```bash
# Instead of:
command1 && (command2 || (command3 && command4)) && command5

# Use simple sequential logic:
command1
if [ $? -eq 0 ]; then
  command2
  if [ $? -ne 0 ]; then
    command3
    command4
  fi
  command5
fi
```

**Rule 3**: Use explicit flags to prevent interactivity
```bash
# Instead of:
npm install

# Use non-interactive flags:
npm install --no-audit --no-fund --silent
```

**Rule 4**: Handle timeouts explicitly
```bash
# Instead of:
curl https://registry.npmjs.org/aws-cdk

# Use timeout controls:
timeout 30 curl --max-time 10 https://registry.npmjs.org/aws-cdk
```

#### Safe Command Patterns

**Pattern 1**: Simple sequential execution
```bash
# Execute one command at a time
executeCommand("cd /path/to/project")
executeCommand("npm install --silent")
executeCommand("cdk synth --quiet")
```

**Pattern 2**: Explicit error handling
```bash
# Check exit codes explicitly
result = executeCommand("npm install")
if (result.exitCode !== 0) {
  handleError(result.stderr)
  return
}
```

**Pattern 3**: Timeout protection
```bash
# Set timeouts for potentially long-running commands
executeCommandWithTimeout("npm install", 300000)  # 5 minutes
executeCommandWithTimeout("cdk synth", 120000)    # 2 minutes
```

#### Command Simplification Examples

**Example 1**: Template Generation
```bash
# Complex (can hang):
cd temp && npx -y cdk@latest init app --language typescript --generate-only && cd .. || (echo "Failed" && exit 1)

# Simplified:
cd temp
npx -y cdk@latest init app --language typescript --generate-only
cd ..
```

**Example 2**: Dependency Installation
```bash
# Complex (can hang):
npm install --silent && (npm audit fix || true) && npm list aws-cdk-lib

# Simplified:
npm install --silent --no-audit --no-fund
npm list aws-cdk-lib --depth=0
```

**Example 3**: Git Operations
```bash
# Complex (can hang):
git add . && git commit -m "upgrade" && (git push || echo "Push failed")

# Simplified:
git add .
git commit -m "feat(cdk): upgrade CDK to version 2.175.3"
# Note: Don't auto-push, let user decide
```

### Platform-Specific Command Generation

#### When to Use Platform-Specific Commands

Use platform-specific commands only when:
1. No cross-platform alternative exists
2. Platform-specific optimization is significant
3. Different package managers are required

#### macOS-Specific Commands

**Homebrew Package Management**:
```bash
# Check if Homebrew is available
if command -v brew >/dev/null 2>&1; then
  echo "Homebrew detected"
  
  # Install Node.js
  brew install node@20
  brew link node@20 --force
  
  # Update Homebrew
  brew update
  brew upgrade node
fi
```

**macOS System Information**:
```bash
# Get macOS version
sw_vers -productVersion

# Get system architecture
uname -m

# Check for Apple Silicon
if [[ $(uname -m) == "arm64" ]]; then
  echo "Apple Silicon (M1/M2) detected"
  # Use ARM-optimized paths
  export PATH="/opt/homebrew/bin:$PATH"
fi
```

**macOS-Specific Paths**:
```bash
# Homebrew paths
if [[ -d "/opt/homebrew" ]]; then
  # Apple Silicon Homebrew
  HOMEBREW_PREFIX="/opt/homebrew"
elif [[ -d "/usr/local/Homebrew" ]]; then
  # Intel Homebrew
  HOMEBREW_PREFIX="/usr/local"
fi
```

#### Linux-Specific Commands

**Package Manager Detection**:
```bash
# Detect Linux package manager
if command -v apt >/dev/null 2>&1; then
  PACKAGE_MANAGER="apt"
  INSTALL_CMD="sudo apt-get install -y"
elif command -v yum >/dev/null 2>&1; then
  PACKAGE_MANAGER="yum"
  INSTALL_CMD="sudo yum install -y"
elif command -v dnf >/dev/null 2>&1; then
  PACKAGE_MANAGER="dnf"
  INSTALL_CMD="sudo dnf install -y"
fi
```

**Node.js Installation on Linux**:
```bash
# Ubuntu/Debian
if [[ -f /etc/debian_version ]]; then
  curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
  sudo apt-get install -y nodejs
  
# RHEL/CentOS/Fedora
elif [[ -f /etc/redhat-release ]]; then
  curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -
  sudo yum install -y nodejs
fi
```

**Linux System Information**:
```bash
# Get Linux distribution
if [[ -f /etc/os-release ]]; then
  source /etc/os-release
  echo "Distribution: $NAME $VERSION"
fi

# Get architecture
uname -m
```

#### Platform-Specific Command Output

```
=== Platform-Specific Command Generation ===

Target: Node.js 20 upgrade
Platform: macOS (Darwin arm64)

Detected package managers:
  ✓ Homebrew: /opt/homebrew/bin/brew
  ✓ nvm: ~/.nvm/nvm.sh

Recommended upgrade path:
  1. Using nvm (preferred for development):
     nvm install 20
     nvm use 20
     nvm alias default 20
  
  2. Using Homebrew (system-wide):
     brew install node@20
     brew link node@20 --force

Selected: nvm (maintains multiple Node.js versions)

Generated commands:
  nvm install 20
  nvm use 20
  node --version  # Verify installation
```

### Platform-Specific Troubleshooting

#### macOS Troubleshooting

**Issue**: Homebrew Permission Errors
```bash
# Symptom
Error: Permission denied @ dir_s_mkdir - /usr/local/Cellar

# Solution
sudo chown -R $(whoami) /usr/local/Cellar
sudo chown -R $(whoami) /usr/local/Homebrew
```

**Issue**: Apple Silicon Compatibility
```bash
# Symptom
arch: can't execute binary file: Exec format error

# Solution - Use Rosetta for Intel binaries
arch -x86_64 npm install

# Or install ARM-native version
brew install --cask node
```

**Issue**: Xcode Command Line Tools Missing
```bash
# Symptom
xcrun: error: invalid active developer path

# Solution
xcode-select --install
```

**Issue**: macOS Gatekeeper Blocking
```bash
# Symptom
"npx" cannot be opened because the developer cannot be verified

# Solution
sudo spctl --master-disable  # Temporarily disable
# Or allow specific binary
sudo xattr -rd com.apple.quarantine /path/to/binary
```

#### Linux Troubleshooting

**Issue**: Package Manager Lock
```bash
# Symptom (Ubuntu/Debian)
E: Could not get lock /var/lib/dpkg/lock-frontend

# Solution
sudo killall apt apt-get
sudo rm /var/lib/apt/lists/lock
sudo rm /var/cache/apt/archives/lock
sudo rm /var/lib/dpkg/lock*
sudo dpkg --configure -a
```

**Issue**: Missing Development Tools
```bash
# Symptom
make: command not found
gcc: command not found

# Solution (Ubuntu/Debian)
sudo apt-get install build-essential

# Solution (RHEL/CentOS)
sudo yum groupinstall "Development Tools"
```

**Issue**: Node.js Version Conflicts
```bash
# Symptom
/usr/bin/node: No such file or directory

# Solution - Fix symlinks
sudo ln -sf /usr/bin/nodejs /usr/bin/node

# Or use alternatives system
sudo update-alternatives --install /usr/bin/node node /usr/bin/nodejs 10
```

**Issue**: Permission Denied on Global npm
```bash
# Symptom
npm ERR! Error: EACCES: permission denied

# Solution - Use nvm instead of system npm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install 20
```

#### Cross-Platform Troubleshooting Output

```
=== Cross-Platform Issue Detected ===

Issue: Node.js upgrade failed
Platform: Linux (Ubuntu 22.04)
Architecture: x86_64

Error: E: Package 'nodejs' has no installation candidate

Diagnosis:
  ✓ Internet connectivity: OK
  ✓ Package manager: apt (functional)
  ✗ Node.js repository: Not configured

Platform-specific solution for Ubuntu:
  1. Add NodeSource repository:
     curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
  
  2. Install Node.js:
     sudo apt-get install -y nodejs
  
  3. Verify installation:
     node --version
     npm --version

Alternative solution (recommended for development):
  Use nvm for version management:
     curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
     source ~/.bashrc
     nvm install 20
     nvm use 20

This approach works identically on both macOS and Linux.
```

#### Platform Detection and Command Selection

```typescript
// Example platform-aware command generation
function generateUpgradeCommands(platform: PlatformInfo): string[] {
  const commands: string[] = [];
  
  // Universal commands first
  commands.push('node --version');
  commands.push('npm --version');
  
  // Platform-specific upgrade commands
  if (platform.platform === 'darwin') {
    if (platform.nodeManager === 'nvm') {
      commands.push('nvm install 20');
      commands.push('nvm use 20');
    } else if (platform.packageManager === 'brew') {
      commands.push('brew install node@20');
      commands.push('brew link node@20 --force');
    }
  } else if (platform.platform === 'linux') {
    if (platform.nodeManager === 'nvm') {
      commands.push('nvm install 20');
      commands.push('nvm use 20');
    } else if (platform.packageManager === 'apt') {
      commands.push('curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -');
      commands.push('sudo apt-get install -y nodejs');
    }
  }
  
  // Universal verification
  commands.push('node --version');
  commands.push('npm --version');
  
  return commands;
}
```

This cross-platform compatibility section ensures the CDK Upgrader works reliably across different operating systems while providing clear troubleshooting guidance for platform-specific issues.
