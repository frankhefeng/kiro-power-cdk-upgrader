---
name: "cdk-upgrader"
displayName: "CDK Upgrader"
description: "Automatically upgrade TypeScript CDK v2 projects to the latest stable version with intelligent merging, deprecated construct handling, and git integration"
keywords:
  - "cdk"
  - "aws"
  - "upgrade"
  - "typescript"
  - "infrastructure"
  - "automation"
  - "cloud-development-kit"
  - "devops"
author: "Feng He"
version: "1.0.0"
repository: "https://github.com/frankhefeng/kiro-power-cdk-upgrader"
---

# CDK Upgrader

## Overview

The CDK Upgrader is a Kiro Power that automates the process of upgrading TypeScript AWS CDK v2 projects to the latest stable version. It uses a template-based approach, generating fresh CDK templates as references and intelligently merging changes while preserving your business logic and custom configurations.

**Important**: This Power does not require additional MCP servers. All functionality is implemented through the standard Kiro agent capabilities and bash commands. However, execution must strictly follow the defined workflow processes outlined in the steering files.

### Execution Requirements

- **Strict Workflow Adherence**: The upgrade process must follow the exact steps defined in the steering files
- **No Shortcuts**: All steps including version selection, template generation, and validation are mandatory
- **Error Handling**: If any step fails, the process must report the error and exit cleanly
- **No Fallback Modes**: There are no alternative execution paths - the defined workflow is the only supported approach

### Key Capabilities

- **Multi-Project Discovery**: Automatically scans directories to find all TypeScript CDK v2 projects
- **Smart Version Selection**: Uses latest stable version if > 7 days old, otherwise falls back to previous stable version
- **Intelligent File Merging**: Preserves your custom configurations while updating CDK boilerplate
- **Deprecated Construct Handling**: Identifies and updates deprecated CDK constructs to modern equivalents
- **Lambda Runtime Upgrades**: Updates Lambda function runtimes to latest supported versions
- **Git Integration**: Creates clean semantic commits for each upgraded project
- **Dry-Run Mode**: Preview all changes before applying them
- **Post-Upgrade Validation**: Runs npm install and cdk synth to verify successful upgrades

## Onboarding Steps

Before using the CDK Upgrader, ensure your environment meets the following requirements:

### 1. Node.js Version Check

```bash
node --version
```

The CDK Upgrader requires Node.js 18.x or later. If your version is outdated:

**Using nvm (recommended):**
```bash
nvm install 20
nvm use 20
```

**Using Homebrew (macOS):**
```bash
brew install node@20
```

### 2. npm Version Check

```bash
npm --version
```

npm 9.x or later is recommended. To upgrade:

```bash
npm install -g npm@latest
```

### 3. Git Availability

```bash
git --version
```

Git must be installed and available in your PATH. The upgrader uses git to:
- Verify clean working directories before upgrades
- Ensure you're on a feature branch (not main/master) - will exit with error if on main/master
- Create semantic commits after successful upgrades

**Important**: The upgrader will NOT create git branches for you. You must create a feature branch manually before running upgrades.

### 4. CDK CLI (Optional)

The upgrader uses `npx -y cdk@$TARGET_CDK_CLI_VERSION init app --language typescript --generate-only` for template generation, so a global CDK installation is not required. However, having CDK installed can speed up operations:

```bash
npm install -g aws-cdk
```

### 5. Cross-Platform Compatibility Requirements

**CRITICAL**: The CDK Upgrader must work on both macOS and Linux systems. All shell scripts MUST include:

**Required Dependencies:**
```bash
# Validate required tools are available
command -v jq >/dev/null 2>&1 || { echo "ERROR: jq is required but not installed" >&2; exit 1; }
```

**Install jq if missing:**

**macOS:**
```bash
brew install jq
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt-get install jq
```

**Linux (RHEL/CentOS):**
```bash
sudo yum install jq
```

**npm Command Syntax Requirements:**

**CRITICAL**: All version selection scripts MUST use the correct npm command syntax:

```bash
# ✅ CORRECT: Get all release dates
npm view aws-cdk time --json

# ✅ CORRECT: Extract specific version date  
VERSION="2.1100.1"
RELEASE_DATE=$(npm view aws-cdk time --json | jq -r ".[\"$VERSION\"]")

# ❌ WRONG: This syntax returns empty results
npm view aws-cdk time.$VERSION
```

**Common npm Command Errors:**
- Using `npm view aws-cdk time.$VERSION` (returns empty)
- Missing `jq` for JSON parsing
- Not handling `null` values from jq output
- Missing error handling for network failures

**Error Handling Requirements:**
- All scripts MUST use `set -euo pipefail` for strict error handling
- All variables MUST be validated before arithmetic operations
- All date calculations MUST use cross-platform compatible functions
- NO hardcoded fallback values are allowed

**Platform Detection:**
The upgrader automatically detects the operating system using `$OSTYPE` and adapts date handling accordingly:
- `darwin*`: macOS (BSD date command)
- `linux*`: Linux (GNU date command)

### Environment Validation

Run this command to validate your environment is ready:

```bash
# Basic environment check
node --version && npm --version && git --version && jq --version

# Platform detection
echo "Operating System: $OSTYPE"

# Verify all requirements
if [[ "$OSTYPE" == "darwin"* ]]; then
    echo "✓ macOS detected - using BSD date commands"
elif [[ "$OSTYPE" == "linux"* ]]; then
    echo "✓ Linux detected - using GNU date commands"  
else
    echo "⚠ Unknown OS: $OSTYPE - may have compatibility issues"
fi
```

All commands should succeed and show appropriate versions before proceeding with upgrades.

**Minimum Requirements:**
- Node.js: ≥18.x
- npm: ≥9.x  
- git: any recent version
- jq: any recent version
- OS: macOS or Linux (Windows not supported)

## Usage

### Invoking the Upgrade Workflow

**IMPORTANT**: To ensure the correct workflow is executed, use these specific phrases:

> "Use the CDK Upgrader Power to upgrade my projects"

or

> "Execute the CDK upgrade workflow from the cdk-upgrader Power"

or

> "Run CDK upgrade following the steering files in cdk-upgrader/"

**DO NOT use generic phrases like**:
- ❌ "Upgrade CDK version to latest" 
- ❌ "Update my CDK dependencies"
- ❌ "Upgrade my CDK projects"

These may trigger simplified upgrade logic instead of the Power's comprehensive workflow.

### Workflow Validation

**Before starting any upgrade, the agent MUST**:

1. **Confirm Power Usage**: Explicitly state that the cdk-upgrader Power workflow will be used
2. **Reference Steering Files**: Mention that upgrade-workflow.md will be followed
3. **Validate Environment**: Check Node.js, npm, git versions
4. **Check Git Branch**: Ensure not on main/master branch
5. **Verify Clean Working Directory**: No uncommitted changes

**If any of these confirmations are missing, the wrong workflow is being used.**

### Workflow Execution Checklist

**The agent MUST follow this exact sequence**:

```
✅ Step 1: Announce Power Usage
   "I'll use the CDK Upgrader Power to upgrade your projects following the defined workflow."

✅ Step 2: Reference Steering Files  
   "Following the upgrade-workflow.md and version-strategy.md guidelines..."

✅ Step 3: Environment Validation
   - Check Node.js version (≥18.x required)
   - Check npm version (≥9.x recommended) 
   - Verify git is available
   - Confirm not on main/master branch
   - Verify clean working directory

✅ Step 4: Project Discovery
   - Scan for cdk.json files
   - Validate TypeScript CDK v2 projects
   - Extract current versions

✅ Step 5: Version Selection (5-Step Algorithm)
   - Step 1: Select CLI version (use latest stable if > 7 days old, otherwise previous stable)
   - Step 2: Record CLI release date as compatibility constraint
   - Step 3: Select Lib version (use latest stable if > 7 days old, otherwise previous stable)
   - Step 4: **MANDATORY: Verify CLI-Lib compatibility (CLI release date >= Lib release date)**
   - Step 5: Final validation and output
   - **If incompatible: Automatically search backwards for compatible version**

✅ Step 6: Template Generation (MANDATORY)
   - Create temp directory
   - Execute: npx -y cdk@$TARGET_CDK_CLI_VERSION init app --language typescript --generate-only
   - Verify template creation

✅ Step 7: File Processing
   - Intelligent file merging based on file-processing.md
   - source-map-support cleanup
   - tsconfig.json COMPLETE replacement (no merge)
   - jest.config.js COMPLETE replacement (no merge) - DO NOT SKIP
   - package.json selective merge
   - cdk.json targeted replacement (app preserved)
   - .gitignore additive merge

✅ Step 8: Deprecated Construct Updates (MANDATORY)
   - Scan TypeScript files for deprecated patterns
   - Apply auto-migratable updates (runtimes, encryption, subnet types)
   - Update Construct imports from 'constructs' package
   - Generate migration guidance for manual fixes
   - See deprecated-constructs.md for detailed mappings

✅ Step 9: Post-Upgrade Validation
   - Run npm install
   - Run cdk synth
   - Verify successful synthesis

✅ Step 10: Git Commit & Cleanup
   - Create semantic commit
   - Remove temp directory
   - Report results
```

**RED FLAGS - If you see these, the wrong workflow is being used**:
- ❌ Direct use of `npm view aws-cdk version` without 5-step version selection algorithm
- ❌ **Skipping Step 4 compatibility check (CLI release date >= Lib release date)**
- ❌ **Using incompatible CLI/Lib version combinations**
- ❌ Immediate package.json updates without template generation
- ❌ Skipping environment checks
- ❌ No temp directory creation
- ❌ No mention of steering files
- ❌ **Skipping deprecated construct updates**
- ❌ **Skipping jest.config.js replacement** (must be completely replaced with template version)

## Steering Files Map

The CDK Upgrader includes specialized steering files for different aspects of the upgrade process:

| Steering File | Purpose | When to Use |
|--------------|---------|-------------|
| `upgrade-workflow.md` | Complete step-by-step upgrade process | Main reference for the full upgrade workflow |
| `version-strategy.md` | Version selection strategy and compatibility rules | Understanding how CDK versions are selected |
| `file-processing.md` | File merge and replace rules | When handling specific file types (cdk.json, package.json, etc.) |
| `deprecated-constructs.md` | Deprecated construct mappings and migrations | When updating deprecated CDK constructs |
| `lambda-runtimes.md` | Lambda runtime upgrade guide | When upgrading Lambda function runtimes |
| `troubleshooting.md` | Common issues and solutions | When encountering errors or unexpected behavior |

### Steering File Details

#### upgrade-workflow.md
The main workflow guide covering:
- Project discovery and analysis
- Multi-project management
- Environment validation
- Template generation
- Git integration
- Post-upgrade validation
- Summary reporting

#### version-strategy.md
Explains the version selection strategy:
- CDK CLI vs Construct Library version independence
- **5-step algorithm**: 
  - Step 1: Select CLI version (use latest stable if > 7 days old, otherwise previous stable)
  - Step 2: Record CLI release date as compatibility constraint
  - Step 3: Select Lib version (use latest stable if > 7 days old, otherwise previous stable)
  - Step 4: MANDATORY compatibility check (CLI release date >= Lib release date)
  - Step 5: Final validation
- **Backward search by minor series**: When incompatible, check highest patch of each minor (2.233.x → 2.232.x → 2.231.x)
- npm registry query instructions
- Cross-platform date calculation functions

#### file-processing.md
Detailed rules for processing each file type:
- Files to replace completely (tsconfig.json, jest.config.js) - no merging, no preservation
- cdk.json: Targeted replacement - preserve ONLY "app" field, replace ALL other fields with template
- Files to merge additively (.gitignore) - add new entries, preserve existing
- Files to merge selectively (package.json) - update CDK packages, preserve non-CDK packages

#### deprecated-constructs.md
Comprehensive guide for deprecated constructs:
- Detection patterns
- Common deprecated constructs and replacements
- Custom construct handling
- Third-party library management

#### lambda-runtimes.md
Lambda runtime upgrade guidance:
- Runtime detection instructions
- Upgrade mappings (Node.js, Python, Java, .NET)
- Compatibility warnings
- Inline vs external Lambda handling

#### troubleshooting.md
Solutions for common issues:
- Network connectivity problems
- System requirements issues
- TypeScript compilation errors
- Git operation failures

---

## Execution Requirements and Constraints

### No MCP Dependencies

This Power is implemented entirely through standard Kiro agent capabilities:
- File system operations (read, write, execute)
- Bash command execution
- npm registry queries
- Git operations
- No external MCP servers required

### Mandatory Workflow Compliance

**The upgrade process MUST follow the exact workflow defined in the steering files:**

1. **Version Selection**: 5-step algorithm (use latest stable if > 7 days old, otherwise previous stable version, with mandatory compatibility check)
2. **Template Generation**: Always create fresh CDK template using `npx -y cdk@$TARGET_CDK_CLI_VERSION init`
3. **File Processing**: Based on file-processing.md rules - complete replacement for tsconfig.json/jest.config.js, targeted replacement for cdk.json (app only preserved), selective merge for package.json
4. **Validation**: Both npm install and cdk synth must pass
5. **Git Integration**: Clean commits with semantic messages

### Error Handling Policy

**On Any Error**:
- Immediately clean up temp directories
- Report specific error details
- Provide actionable remediation steps
- **Exit the process** - no fallback modes
- Do not attempt alternative approaches

**No Shortcuts Allowed**:
- Cannot skip version selection algorithm
- Cannot skip template generation
- Cannot skip validation steps
- Cannot bypass git requirements

### Failure Scenarios

The process will exit with error in these cases:
- Environment requirements not met (Node.js, npm, git)
- Not on a feature branch (main/master branches rejected)
- Working directory not clean
- Network issues preventing npm queries
- Template generation failures
- File processing errors
- Validation failures (npm install or cdk synth)
- Git operation failures

**Important**: Each failure provides specific guidance for resolution, but the process does not continue automatically.

## Common Implementation Errors

### Error: Wrong Workflow Being Used

**Symptoms**:
- Direct use of `npm view aws-cdk version` without 5-step algorithm
- Immediate package.json updates
- Messages like "using outdated version" (should use latest stable when available)
- No temp directory creation
- No steering file references

**Solution**: 
```
STOP the current process immediately.
Restart with explicit Power invocation:
"Use the CDK Upgrader Power to upgrade my projects"
```

### Error: Simplified Logic Detected

**Symptoms**:
- Skipping 5-step version selection algorithm
- No template generation
- Direct dependency updates
- Missing environment checks

**Solution**:
```
The agent is not following the Power's steering files.
Explicitly request: "Follow the upgrade-workflow.md process from the cdk-upgrader Power"
```

### Error: Compatibility Check Skipped

**Symptoms**:
- CLI and Lib versions selected without Step 4 compatibility validation
- Lib version released after CLI version
- Missing "CLI release date >= Lib release date" check
- No backward search when incompatible versions detected

**Solution**:
```
CRITICAL ERROR: Step 4 compatibility check was skipped!
This will result in incompatible versions and synthesis failures.
Restart with explicit requirement: "Follow the 5-step version selection algorithm including Step 4 mandatory compatibility validation"
```
