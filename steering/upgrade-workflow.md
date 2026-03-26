# CDK Upgrade Workflow

This steering file provides the complete step-by-step workflow for upgrading TypeScript CDK v2 projects to the latest stable version.

**CRITICAL EXECUTION REQUIREMENTS:**
- This workflow must be followed exactly as specified
- No steps can be skipped or modified
- If any step fails, the process must exit with error reporting
- No fallback modes or alternative approaches are supported
- All bash commands and npm operations must execute as documented

## Workflow Validation Protocol

**BEFORE STARTING**: The agent must explicitly confirm:

1. **Power Identification**: "Using CDK Upgrader Power workflow"
2. **Steering File Reference**: "Following upgrade-workflow.md process"
3. **Algorithm Confirmation**: "Applying smart version selection (latest stable if > 7 days old)"
4. **Template Generation Commitment**: "Will generate CDK template using selected CLI version"

**If these confirmations are missing, STOP and restart with proper Power invocation.**

## Execution Sequence Validation

Each major step must be announced before execution:

```bash
echo "=== CDK Upgrader Power: Step X ===
echo "Following: upgrade-workflow.md"
echo "Action: [specific action being performed]"
```

**Example**:
```bash
echo "=== CDK Upgrader Power: Step 1 ==="
echo "Following: upgrade-workflow.md - Project Discovery"
echo "Action: Scanning for CDK projects with cdk.json files"
```

## Table of Contents

1. [Project Discovery](#project-discovery)
2. [Multi-Project Management](#multi-project-management)
3. [Target Version Determination](#target-version-determination)
4. [Environment Check](#environment-check)
5. [Template Generation](#template-generation)
6. [Deprecated Construct Updates](#deprecated-construct-updates)
7. [Git Integration](#git-integration)
8. [Post-Upgrade Validation](#post-upgrade-validation)
9. [Summary Report](#summary-report)

---

## Project Discovery

### Overview

The first step in the upgrade workflow is discovering all CDK projects in the workspace. This section provides instructions for scanning directories, identifying valid TypeScript CDK v2 projects, and validating git tracking.

### Scanning for CDK Projects

Scan the current directory and all subdirectories to find CDK projects:

1. **Search for CDK project indicators**:
   - Look for `cdk.json` files in the directory tree
   - Each `cdk.json` file indicates a potential CDK project root

2. **Directory traversal command**:
   ```bash
   find . -name "cdk.json" -type f
   ```

3. **For each found `cdk.json`**, record the parent directory as a potential CDK project location

### Identifying TypeScript CDK v2 Projects

For each potential CDK project, validate it meets the upgrade criteria:

#### Step 1: Check for TypeScript Configuration

Verify the project has a `tsconfig.json` file in the same directory as `cdk.json`:

```bash
# Check if tsconfig.json exists
test -f "<project_path>/tsconfig.json" && echo "TypeScript project"
```

If `tsconfig.json` is missing, **skip this project** and report:
> "Skipping <project_path>: Not a TypeScript project (no tsconfig.json found)"

#### Step 2: Verify package.json Exists

Check for `package.json` in the project directory:

```bash
test -f "<project_path>/package.json" && echo "Has package.json"
```

If missing, **skip this project** and report:
> "Skipping <project_path>: No package.json found"

#### Step 3: Confirm CDK v2 Project

Read the `package.json` and check for CDK v2 dependencies:

**Required indicators for CDK v2**:
- `aws-cdk-lib` in dependencies or devDependencies (CDK v2 uses this unified package)
- Version should start with `2.` (e.g., `"aws-cdk-lib": "^2.150.0"`)

**CDK v1 indicators (skip these projects)**:
- `@aws-cdk/core` or individual `@aws-cdk/*` packages indicate CDK v1
- Report: "Skipping <project_path>: CDK v1 project detected. This upgrader only supports CDK v2."

#### Step 4: Extract Current Version Information

From `package.json`, extract:

1. **CDK Construct Library version** (`aws-cdk-lib`):
   ```json
   {
     "dependencies": {
       "aws-cdk-lib": "^2.150.0"  // Extract this version
     }
   }
   ```

2. **CDK CLI version** (if present in devDependencies):
   ```json
   {
     "devDependencies": {
       "aws-cdk": "^2.150.0"  // May be different from aws-cdk-lib
     }
   }
   ```

3. **TypeScript version**:
   ```json
   {
     "devDependencies": {
       "typescript": "~5.4.5"
     }
   }
   ```

### Git Tracking Validation

Before proceeding with any project, validate git tracking:

#### Step 1: Check for Git Repository

```bash
cd "<project_path>"
git rev-parse --git-dir
```

If this command fails, the project is not in a git repository.

**Action**: Report warning and require user confirmation:
> "Warning: <project_path> is not tracked by git. Upgrades cannot be committed automatically. Continue anyway? (y/n)"

#### Step 2: Verify Git is Functional

```bash
git status --porcelain
```

If this fails, git may not be properly configured.

**Action**: Report error:
> "Error: Git is not functional in <project_path>. Please ensure git is properly configured."

### Project Discovery Output Format

After scanning, present discovered projects in this format:

```
=== CDK Project Discovery Results ===

Found X CDK project(s):

1. <project_name_1>
   Path: <full_path>
   CDK Lib Version: 2.X.Y
   TypeScript: Yes
   Git Tracked: Yes/No

2. <project_name_2>
   Path: <full_path>
   CDK Lib Version: 2.X.Y
   TypeScript: Yes
   Git Tracked: Yes/No

Skipped Y project(s):
- <path>: <reason>
- <path>: <reason>
```

### Validation Checklist

Before proceeding to the next phase, ensure each project passes:

- [ ] Has `cdk.json` file
- [ ] Has `tsconfig.json` file (TypeScript project)
- [ ] Has `package.json` file
- [ ] Contains `aws-cdk-lib` dependency (CDK v2)
- [ ] Version starts with `2.` (not CDK v1)
- [ ] Git repository exists (warning if not)



---

## Multi-Project Management

### Overview

When multiple CDK projects are discovered in a workspace, the upgrader processes each project independently while providing unified progress reporting and error isolation.

### Project Selection Workflow

#### Automatic Mode (Default)

By default, all discovered eligible projects are queued for upgrade:

```
=== Project Selection ===

The following projects will be upgraded:
1. [x] my-service (deployment/cdk/my-service)
2. [x] api-gateway (deployment/cdk/api-gateway)
3. [x] lambda-functions (deployment/cdk/lambda-functions)

Proceed with upgrading all 3 projects? (y/n)
```

#### Selective Mode

When the user requests selective upgrade, present an interactive selection:

```
=== Project Selection ===

Select projects to upgrade (enter numbers separated by commas, or 'all'):

1. [ ] my-service (deployment/cdk/my-service) - v2.150.0
2. [ ] api-gateway (deployment/cdk/api-gateway) - v2.145.0
3. [ ] lambda-functions (deployment/cdk/lambda-functions) - v2.160.0

Enter selection (e.g., "1,3" or "all"): _
```

After selection:
```
Selected projects for upgrade:
- my-service
- lambda-functions

Proceed? (y/n)
```

### Processing Order

Process projects in the order they were discovered or selected. For each project:

1. **Announce start**: "Starting upgrade for project: <project_name>"
2. **Execute upgrade steps** (environment check, template generation, file processing, validation)
3. **Report completion**: Success or failure with details
4. **Move to next project**: Regardless of previous project's outcome

### Progress Reporting Format

#### Per-Project Progress

Display progress for each project being processed:

```
=== Upgrading Project 1 of 3: my-service ===

[1/6] Checking environment... ✓
[2/6] Generating CDK template... ✓
[3/6] Processing files...
      - cdk.json: app preserved, all other fields replaced ✓
      - package.json: updated ✓
      - tsconfig.json: replaced ✓
      - .gitignore: merged ✓
[4/6] Updating deprecated constructs... ✓ (2 constructs updated)
[5/6] Running validation...
      - npm install: ✓
      - cdk synth: ✓
[6/6] Creating git commit... ✓

✓ Project my-service upgraded successfully!
  CDK Lib: 2.150.0 → 2.175.3
  CDK CLI: 2.150.0 → 2.1005.0
```

#### Overall Progress

Show overall progress across all projects:

```
=== Overall Progress ===
[====================] 2/3 projects complete

Completed:
  ✓ my-service - Success
  ✓ api-gateway - Success

In Progress:
  ⟳ lambda-functions - Processing files...

Remaining:
  (none)
```

### Error Isolation Between Projects

#### Principle: Fail-Safe Isolation

When one project fails, it MUST NOT affect other projects:

1. **Capture all errors** for the failing project
2. **Log the failure** with detailed error information
3. **Continue to the next project** without interruption
4. **Include failure in final summary**

#### Error Handling Flow

```
=== Upgrading Project 2 of 3: api-gateway ===

[1/6] Checking environment... ✓
[2/6] Generating CDK template... ✓
[3/6] Processing files... ✓
[4/6] Updating deprecated constructs... ✓
[5/6] Running validation...
      - npm install: ✓
      - cdk synth: ✗ FAILED

✗ Project api-gateway upgrade failed!
  Error: TypeScript compilation error in lib/api-gateway-stack.ts
  Details: Property 'runtime' does not exist on type 'FunctionProps'
  
  Suggested fix: Check deprecated construct mappings for Lambda runtime changes.
  
  Changes have been reverted for this project.

Continuing with remaining projects...

=== Upgrading Project 3 of 3: lambda-functions ===
...
```

#### Rollback on Failure

When a project upgrade fails during validation:

1. **Do NOT commit changes** to git
2. **Clean up temp directory** - If temp folder was created, delete it immediately
3. **Report the failure** with specific error details
4. **Provide suggested fixes** when possible
5. **Leave files in modified state** for user inspection (or offer to revert)

```bash
# Cleanup temp directory on failure
if [ -d "$TEMP_DIR" ]; then
  rm -rf "$TEMP_DIR"
  echo "Cleaned up temporary directory: $TEMP_DIR"
fi
```

```
Project api-gateway failed validation. Options:
1. Keep modified files for manual inspection
2. Revert all changes (git checkout)

Select option (1/2): _
```

### Multi-Project Summary

After all projects are processed, provide a comprehensive summary:

```
=== CDK Upgrade Summary ===

Total Projects: 3
  ✓ Successful: 2
  ✗ Failed: 1

Successful Upgrades:
┌─────────────────┬──────────────┬──────────────┬─────────────┐
│ Project         │ CDK Lib      │ CDK CLI      │ Commit      │
├─────────────────┼──────────────┼──────────────┼─────────────┤
│ my-service      │ 2.150→2.175.3│ 2.150→2.1005 │ abc1234     │
│ lambda-functions│ 2.160→2.175.3│ 2.160→2.1005 │ def5678     │
└─────────────────┴──────────────┴──────────────┴─────────────┘

Failed Upgrades:
┌─────────────────┬─────────────────────────────────────────────┐
│ Project         │ Error                                       │
├─────────────────┼─────────────────────────────────────────────┤
│ api-gateway     │ cdk synth failed: TypeScript compilation    │
│                 │ error in lib/api-gateway-stack.ts           │
└─────────────────┴─────────────────────────────────────────────┘

Next Steps:
- Review failed projects and apply suggested fixes
- Run 'git push' to push successful upgrades to remote
- For api-gateway: Check lib/api-gateway-stack.ts for deprecated constructs
```

### Batch Processing Rules

1. **Sequential Processing**: Process one project at a time to avoid resource conflicts
2. **Shared Template**: Generate the CDK template once and reuse for all projects
3. **Independent Validation**: Each project runs its own npm install and cdk synth
4. **Separate Commits**: Each successful project gets its own git commit
5. **Unified Reporting**: Aggregate all results into a single summary report



---

## Target Version Determination

### Overview

Before proceeding with upgrades, the system must determine the appropriate target versions for both CDK CLI and CDK Construct Library. The algorithm follows a **stability-first strategy**: **use latest minor with highest patch if > 7 days old, otherwise fall back to previous minor**.

### Version Selection Strategy

The CDK Upgrader uses a **stability-first strategy**:
- **CDK CLI**: Use latest stable version if > 7 days old, otherwise use previous stable version
- **CDK Construct Library**: Use latest stable version if > 7 days old, otherwise use previous stable version, then apply CLI compatibility check
- **Compatibility Rule**: CLI release date >= Lib release date (MANDATORY)
- **Template Generation**: MANDATORY - Always generate fresh CDK template for comparison

**⚠️ CRITICAL EXECUTION WARNING**:

Before executing version selection, the agent MUST announce:

```bash
echo "=== CDK Upgrader Power: Version Selection ==="
echo "Following: version-strategy.md 5-step algorithm"
echo "Step 1: Select CLI version (use latest stable if > 7 days old, otherwise previous stable)"  
echo "Step 2: Record CLI release date as compatibility constraint"
echo "Step 3: Select Lib version (use latest stable if > 7 days old, otherwise previous stable)"
echo "Step 4: MANDATORY compatibility check (CLI release date >= Lib release date)"
echo "Step 5: Final validation of both versions"
echo ""
echo "⚠️  WARNING: Skipping Step 4 will result in incompatible versions!"
echo "⚠️  All 5 steps MUST be executed in sequence!"
```

**RED FLAGS - STOP EXECUTION IF YOU SEE**:
- Direct usage of `npm view aws-cdk version` without the full 5-step algorithm
- Direct usage of `npm view aws-cdk-lib version` without the full 5-step algorithm  
- **Compatibility check (Step 4) being skipped or omitted**
- Missing cross-platform date calculation functions
- Hardcoded version fallbacks without proper validation

### Step 1: Query CDK CLI Versions from npm Registry

```bash
# Cross-platform date calculation function (REQUIRED)
calculate_days_since() {
    local release_date="$1"
    local current_timestamp=$(date +%s)
    
    # Handle different date formats and OS compatibility
    if [[ "$OSTYPE" == "darwin"* ]]; then
        # macOS - try multiple date formats
        local release_timestamp
        
        # Try ISO format with milliseconds (2025-12-17T14:36:35.919Z)
        release_timestamp=$(date -j -f "%Y-%m-%dT%H:%M:%S" "${release_date%.*}" +%s)
        
        # Try date only format (2025-12-17)
        if [ -z "$release_timestamp" ] || [ "$release_timestamp" = "0" ]; then
            release_timestamp=$(date -j -f "%Y-%m-%d" "${release_date%%T*}" +%s)
        fi
        
        # Fallback: try without format specifier
        if [ -z "$release_timestamp" ] || [ "$release_timestamp" = "0" ]; then
            release_timestamp=$(date -j -f "%Y-%m-%d" "${release_date}" +%s || echo "0")
        fi
    else
        # Linux - GNU date handles most formats automatically
        local release_timestamp=$(date -d "$release_date" +%s || echo "0")
    fi
    
    # Validate timestamp
    if [ -z "$release_timestamp" ] || [ "$release_timestamp" = "0" ]; then
        echo "ERROR: Unable to parse date: $release_date" >&2
        echo "OS: $OSTYPE" >&2
        return 1
    fi
    
    # Calculate and return days difference
    local days_diff=$(( (current_timestamp - release_timestamp) / 86400 ))
    echo "$days_diff"
}

# Cross-platform timestamp comparison function
compare_timestamps() {
    local date1="$1"
    local date2="$2"
    
    if [[ "$OSTYPE" == "darwin"* ]]; then
        # macOS
        local ts1=$(date -j -f "%Y-%m-%dT%H:%M:%S" "${date1%.*}" +%s || \
                    date -j -f "%Y-%m-%d" "${date1%%T*}" +%s || echo "0")
        local ts2=$(date -j -f "%Y-%m-%dT%H:%M:%S" "${date2%.*}" +%s || \
                    date -j -f "%Y-%m-%d" "${date2%%T*}" +%s || echo "0")
    else
        # Linux
        local ts1=$(date -d "$date1" +%s || echo "0")
        local ts2=$(date -d "$date2" +%s || echo "0")
    fi
    
    if [ "$ts1" = "0" ] || [ "$ts2" = "0" ]; then
        echo "ERROR: Unable to parse dates for comparison" >&2
        return 1
    fi
    
    # Return comparison result: 0 if ts1 <= ts2, 1 if ts1 > ts2
    [ "$ts1" -le "$ts2" ]
}

# Get CDK CLI version information
npm view aws-cdk versions --json
npm view aws-cdk time --json
```

**Parse the response to extract**:
- Latest version number
- Release date of latest version
- Previous version number (if needed)

### Step 2: Determine TARGET_CDK_CLI_VERSION

**Rule**: Find the highest patch in the latest minor version that is > 7 days old. If not available, move to previous minor.

```bash
# Get all CLI versions and release dates
ALL_CLI_VERSIONS=$(npm view aws-cdk versions --json)
ALL_CLI_TIMES=$(npm view aws-cdk time --json)

# Group by minor version and find latest minor
LATEST_MINOR=$(echo "$ALL_CLI_VERSIONS" | jq -r 'map(split(".")[1] | tonumber) | max')

# Find highest patch in latest minor
SELECTED_VERSION=$(echo "$ALL_CLI_VERSIONS" | \
  jq -r "[.[] | select(startswith(\"2.$LATEST_MINOR.\"))] | sort | last")

# Check 7-day stability rule
SELECTED_RELEASE_DATE=$(echo "$ALL_CLI_TIMES" | jq -r ".[\"$SELECTED_VERSION\"]")
DAYS_SINCE_RELEASE=$(calculate_days_since "$SELECTED_RELEASE_DATE")

while [ "$DAYS_SINCE_RELEASE" -lt 7 ]; do
  echo "Selected CLI version is less than 7 days old, checking previous minor..."
  LATEST_MINOR=$((LATEST_MINOR - 1))
  SELECTED_VERSION=$(echo "$ALL_CLI_VERSIONS" | \
    jq -r "[.[] | select(startswith(\"2.$LATEST_MINOR.\"))] | sort | last")
  SELECTED_RELEASE_DATE=$(echo "$ALL_CLI_TIMES" | jq -r ".[\"$SELECTED_VERSION\"]")
  DAYS_SINCE_RELEASE=$(calculate_days_since "$SELECTED_RELEASE_DATE")
done

TARGET_CDK_CLI_VERSION=$SELECTED_VERSION
CLI_RELEASE_DATE=$SELECTED_RELEASE_DATE

echo "TARGET_CDK_CLI_VERSION: $TARGET_CDK_CLI_VERSION"
echo "CLI_RELEASE_DATE: $CLI_RELEASE_DATE"
```

**Output format**:
```
=== Step 1: CDK CLI Version Selection ===

Latest CLI version: 2.1006.0 (released 2025-01-01, 1 day ago)
⚠ Latest version is less than 7 days old

Selected TARGET_CDK_CLI_VERSION: 2.1005.0 (released 2024-12-20, 13 days ago)
CLI_RELEASE_DATE: 2024-12-20 (used as constraint for Lib selection)
```

Or if latest is stable:
```
=== Step 1: CDK CLI Version Selection ===

Latest CLI version: 2.1005.0 (released 2024-12-20, 13 days ago)
✓ Latest version is more than 7 days old

Selected TARGET_CDK_CLI_VERSION: 2.1005.0
CLI_RELEASE_DATE: 2024-12-20 (used for compatibility validation)
```

### Step 3: CDK Construct Library Version Selection

**Rule**: Find the highest patch in the latest minor version that is > 7 days old. If not available, move to previous minor.

```bash
# Get all Lib versions and release dates
ALL_LIB_VERSIONS=$(npm view aws-cdk-lib versions --json)
ALL_LIB_TIMES=$(npm view aws-cdk-lib time --json)

# Group by minor version and find latest minor
LATEST_LIB_MINOR=$(echo "$ALL_LIB_VERSIONS" | jq -r 'map(split(".")[1] | tonumber) | max')

# Find highest patch in latest minor
SELECTED_LIB_VERSION=$(echo "$ALL_LIB_VERSIONS" | \
  jq -r "[.[] | select(startswith(\"2.$LATEST_LIB_MINOR.\"))] | sort | last")

# Check 7-day stability rule
SELECTED_LIB_RELEASE_DATE=$(echo "$ALL_LIB_TIMES" | jq -r ".[\"$SELECTED_LIB_VERSION\"]")
DAYS_SINCE_LIB_RELEASE=$(calculate_days_since "$SELECTED_LIB_RELEASE_DATE")

while [ "$DAYS_SINCE_LIB_RELEASE" -lt 7 ]; do
  echo "Selected Lib version is less than 7 days old, checking previous minor..."
  LATEST_LIB_MINOR=$((LATEST_LIB_MINOR - 1))
  SELECTED_LIB_VERSION=$(echo "$ALL_LIB_VERSIONS" | \
    jq -r "[.[] | select(startswith(\"2.$LATEST_LIB_MINOR.\"))] | sort | last")
  SELECTED_LIB_RELEASE_DATE=$(echo "$ALL_LIB_TIMES" | jq -r ".[\"$SELECTED_LIB_VERSION\"]")
  DAYS_SINCE_LIB_RELEASE=$(calculate_days_since "$SELECTED_LIB_RELEASE_DATE")
done

TARGET_CDK_LIB_VERSION=$SELECTED_LIB_VERSION

echo "TARGET_CDK_LIB_VERSION: $TARGET_CDK_LIB_VERSION"
```

**Output format**:
```
=== Step 3: CDK Construct Library Version Selection ===

All Lib versions: 2.230.0, 2.231.0, 2.231.1, 2.232.0, 2.232.1, 2.232.2, 2.233.0
Latest: 2.233.0 (checking 7-day stability rule)

Latest minor in remaining: 2.232.x
Previous minor: 2.231.x
Highest patch in 2.232.x: 2.232.2 (released 2024-12-10, 14 days ago)
✓ Version is more than 7 days old

Selected TARGET_CDK_LIB_VERSION: 2.232.2
```

### Step 4: MANDATORY Compatibility Validation

**⚠️ CRITICAL**: This step MUST NOT be skipped. Skipping this check will result in incompatible versions.

**Rule**: Verify that TARGET_CDK_LIB_VERSION release date <= CLI_RELEASE_DATE. If not, continue searching backwards until a compatible version is found.

```bash
echo "=== Step 4: MANDATORY Compatibility Validation ==="
echo "Following: version-strategy.md Step 4 (CLI release date >= Lib release date)"
echo ""

# Get release dates for both versions
CLI_RELEASE_DATE=$(npm view aws-cdk time --json | jq -r ".[\"$TARGET_CDK_CLI_VERSION\"]")
LIB_RELEASE_DATE=$(npm view aws-cdk-lib time --json | jq -r ".[\"$TARGET_CDK_LIB_VERSION\"]")

# Validate we have both dates
if [ -z "$CLI_RELEASE_DATE" ] || [ "$CLI_RELEASE_DATE" = "null" ]; then
    echo "ERROR: Unable to get CLI release date for $TARGET_CDK_CLI_VERSION" >&2
    exit 1
fi

if [ -z "$LIB_RELEASE_DATE" ] || [ "$LIB_RELEASE_DATE" = "null" ]; then
    echo "ERROR: Unable to get Lib release date for $TARGET_CDK_LIB_VERSION" >&2
    exit 1
fi

echo "CLI Version: $TARGET_CDK_CLI_VERSION (released: $CLI_RELEASE_DATE)"
echo "Lib Version: $TARGET_CDK_LIB_VERSION (released: $LIB_RELEASE_DATE)"
echo ""

# Perform compatibility check using cross-platform timestamp comparison
if ! compare_timestamps "$LIB_RELEASE_DATE" "$CLI_RELEASE_DATE"; then
    echo "✗ COMPATIBILITY CHECK FAILED!"
    echo "Lib version $TARGET_CDK_LIB_VERSION released AFTER CLI version $TARGET_CDK_CLI_VERSION"
    echo "Lib: $LIB_RELEASE_DATE > CLI: $CLI_RELEASE_DATE"
    echo ""
    echo "Searching for earlier compatible Lib version..."
    
    # Get all lib versions for backward search
    ALL_LIB_VERSIONS=$(npm view aws-cdk-lib versions --json)
    ALL_LIB_TIMES=$(npm view aws-cdk-lib time --json)
    
    # Find compatible version by searching backwards
    FOUND_COMPATIBLE=false
    
    # Extract current minor version number
    CURRENT_LIB_MINOR=$(echo "$TARGET_CDK_LIB_VERSION" | cut -d. -f2)
    
    # Search backwards through minor versions
    for ((SEARCH_MINOR=CURRENT_LIB_MINOR-1; SEARCH_MINOR>=150; SEARCH_MINOR--)); do
        echo "Checking 2.$SEARCH_MINOR.x series..."
        
        # Find highest patch in this minor version
        CANDIDATE_VERSION=$(echo "$ALL_LIB_VERSIONS" | \
            jq -r "[.[] | select(startswith(\"2.$SEARCH_MINOR.\"))] | sort | last")
        
        if [ "$CANDIDATE_VERSION" = "null" ] || [ -z "$CANDIDATE_VERSION" ]; then
            echo "  No versions found in 2.$SEARCH_MINOR.x series"
            continue
        fi
        
        # Get release date for candidate
        CANDIDATE_RELEASE_DATE=$(echo "$ALL_LIB_TIMES" | jq -r ".[\"$CANDIDATE_VERSION\"]")
        
        if [ -z "$CANDIDATE_RELEASE_DATE" ] || [ "$CANDIDATE_RELEASE_DATE" = "null" ]; then
            echo "  Unable to get release date for $CANDIDATE_VERSION"
            continue
        fi
        
        # Check 7-day stability rule
        CANDIDATE_DAYS=$(calculate_days_since "$CANDIDATE_RELEASE_DATE")
        if [ "$CANDIDATE_DAYS" -lt 7 ]; then
            echo "  $CANDIDATE_VERSION is too new ($CANDIDATE_DAYS days old)"
            continue
        fi
        
        # Check compatibility with CLI
        if compare_timestamps "$CANDIDATE_RELEASE_DATE" "$CLI_RELEASE_DATE"; then
            echo "  ✓ Found compatible version: $CANDIDATE_VERSION"
            echo "    Release date: $CANDIDATE_RELEASE_DATE"
            echo "    Days old: $CANDIDATE_DAYS"
            TARGET_CDK_LIB_VERSION="$CANDIDATE_VERSION"
            LIB_RELEASE_DATE="$CANDIDATE_RELEASE_DATE"
            FOUND_COMPATIBLE=true
            break
        else
            echo "  $CANDIDATE_VERSION still too new (released after CLI)"
        fi
    done
    
    if [ "$FOUND_COMPATIBLE" = false ]; then
        echo ""
        echo "ERROR: No compatible aws-cdk-lib version found!"
        echo "All available versions were released after CLI version $TARGET_CDK_CLI_VERSION"
        echo "This may indicate:"
        echo "1. The selected CLI version is very old"
        echo "2. npm registry data is incomplete"
        echo ""
        echo "Please try again later or select a newer CLI version."
        exit 1
    fi
fi

echo "✓ COMPATIBILITY CHECK PASSED"
echo "Final versions:"
echo "  CLI: $TARGET_CDK_CLI_VERSION (released: $CLI_RELEASE_DATE)"
echo "  Lib: $TARGET_CDK_LIB_VERSION (released: $LIB_RELEASE_DATE)"
echo ""
```

**Output format (successful compatibility)**:
```
=== Step 4: MANDATORY Compatibility Validation ===
Following: version-strategy.md Step 4 (CLI release date >= Lib release date)

CLI Version: 2.1100.1 (released: 2025-12-17T14:36:35.919Z)
Lib Version: 2.233.0 (released: 2025-12-18T20:43:49.076Z)

✗ COMPATIBILITY CHECK FAILED!
Lib version 2.233.0 released AFTER CLI version 2.1100.1
Lib: 2025-12-18T20:43:49.076Z > CLI: 2025-12-17T14:36:35.919Z

Searching for earlier compatible Lib version...
Checking 2.232.x series...
  Highest patch: 2.232.2
  ✓ Found compatible version: 2.232.2
    Release date: 2025-12-12T10:30:00.000Z
    Days old: 22

✓ COMPATIBILITY CHECK PASSED
Final versions:
  CLI: 2.1100.1 (released: 2025-12-17T14:36:35.919Z)
  Lib: 2.232.2 (released: 2025-12-12T10:30:00.000Z)
```
    2.175.3 (2024-12-25)

Candidate versions: 2.175.2, 2.175.1, 2.175.0, 2.174.1, 2.174.0, ...
```

### Step 5: Final Validation and Output

**Rule**: Verify all constraints are satisfied before proceeding.

```bash
echo "=== Step 5: Final Validation ==="
echo ""

# Final validation checklist
CLI_RELEASE_DATE=$(npm view aws-cdk time --json | jq -r ".[\"$TARGET_CDK_CLI_VERSION\"]")
LIB_RELEASE_DATE=$(npm view aws-cdk-lib time --json | jq -r ".[\"$TARGET_CDK_LIB_VERSION\"]")

# Check 1: CLI version stability (> 7 days old)
CLI_DAYS=$(calculate_days_since "$CLI_RELEASE_DATE")
if [ "$CLI_DAYS" -lt 7 ]; then
    echo "ERROR: CLI version is less than 7 days old (should not happen)" >&2
    exit 1
fi

# Check 2: Lib version stability (> 7 days old)  
LIB_DAYS=$(calculate_days_since "$LIB_RELEASE_DATE")
if [ "$LIB_DAYS" -lt 7 ]; then
    echo "ERROR: Lib version is less than 7 days old (should not happen)" >&2
    exit 1
fi

# Check 3: MANDATORY compatibility check
if ! compare_timestamps "$LIB_RELEASE_DATE" "$CLI_RELEASE_DATE"; then
    echo "ERROR: Lib release date > CLI release date (should not happen after Step 4)" >&2
    echo "CLI: $CLI_RELEASE_DATE, Lib: $LIB_RELEASE_DATE" >&2
    exit 1
fi

echo "Final validation results:"
echo "✓ CLI version stability: $TARGET_CDK_CLI_VERSION ($CLI_DAYS days old)"
echo "✓ Lib version stability: $TARGET_CDK_LIB_VERSION ($LIB_DAYS days old)"
echo "✓ Compatibility constraint: Lib release date <= CLI release date"
echo ""
echo "All validations passed!"
```

**Output format**:
```
=== Step 5: Final Validation ===

Final validation results:
✓ CLI version stability: 2.1100.1 (17 days old)
✓ Lib version stability: 2.232.2 (22 days old)
✓ Compatibility constraint: Lib release date <= CLI release date

All validations passed!
```

After all checks pass, display the final target versions with full validation summary:

```bash
echo "=== Target Version Selection Complete ==="
echo ""
echo "Selected versions (all validation steps passed):"
echo "TARGET_CDK_CLI_VERSION: $TARGET_CDK_CLI_VERSION"
echo "TARGET_CDK_LIB_VERSION: $TARGET_CDK_LIB_VERSION"
echo ""
echo "Validation summary:"
echo "✓ Step 1: CLI version selected (latest stable > 7 days old)"
echo "✓ Step 2: CLI release date recorded as constraint"  
echo "✓ Step 3: Lib version selected (latest stable > 7 days old)"
echo "✓ Step 4: Compatibility check passed (CLI release date >= Lib release date)"
echo "✓ Step 5: Final validation complete"
echo ""
echo "These versions will be used for all project upgrades."
```

**Expected output**:
```
=== Target Version Selection Complete ===

Selected versions (all validation steps passed):
TARGET_CDK_CLI_VERSION: 2.1100.1
TARGET_CDK_LIB_VERSION: 2.232.2

Validation summary:
✓ Step 1: CLI version selected (latest stable > 7 days old)
✓ Step 2: CLI release date recorded as constraint  
✓ Step 3: Lib version selected (latest stable > 7 days old)
✓ Step 4: Compatibility check passed (CLI release date >= Lib release date)
✓ Step 5: Final validation complete

These versions will be used for all project upgrades.
```

### Store Target Versions

Store the determined versions for use in subsequent upgrade steps:

```bash
export TARGET_CDK_CLI_VERSION="2.1005.0"
export TARGET_CDK_LIB_VERSION="2.174.1"
export CLI_RELEASE_DATE="2024-12-20"
```

These values will be used in:
- Template generation: `npx -y cdk@$TARGET_CDK_CLI_VERSION init ...`
- package.json updates: Set aws-cdk-lib to TARGET_CDK_LIB_VERSION
- package.json verification: Verify aws-cdk matches TARGET_CDK_CLI_VERSION

### Error Handling

#### npm Registry Unavailable

```
=== Target Version Determination Failed ===

✗ Unable to query npm registry

Error: npm ERR! network request failed

Solutions:
1. Check internet connection
2. If behind proxy, configure npm:
   npm config set proxy http://proxy:8080
3. Try again later

Cannot proceed without determining target versions.
```

#### No Candidate Versions Available

```
=== Target Version Determination ===

CLI_RELEASE_DATE: 2024-12-01

⚠ No aws-cdk-lib versions found released on or before 2024-12-01

This may indicate:
1. The selected CLI version is very old
2. npm registry data is incomplete

Fallback: Using latest compatible version with warning
```

#### No Previous Minor Version in Candidates

```
=== Target Version Determination ===

Candidate versions: 2.176.0, 2.176.1

⚠ Only one minor version (2.176.x) available in candidates
   Cannot select "previous minor" version

Fallback: Using highest patch of available minor: 2.176.1
  Note: This version may be less stable than a previous minor release

TARGET_CDK_LIB_VERSION: 2.176.1 (fallback)
```



---

## Environment Check

### Overview

Before proceeding with upgrades, validate that the development environment meets the requirements for the latest CDK version. This check occurs after template generation to ensure compatibility with the specific CDK version being installed.

### Pre-Flight System Validation

Before any upgrade operations, verify basic system requirements:

#### Git Availability Check

```bash
git --version
```

**Expected output**: `git version X.Y.Z`

**If git is not found**:
> "Error: Git is not installed or not in PATH. Please install git before proceeding."
> 
> Installation instructions:
> - macOS: `brew install git`
> - Ubuntu/Debian: `sudo apt-get install git`
> - Windows: Download from https://git-scm.com/

### Node.js Version Check

#### Step 1: Detect Current Node.js Version

```bash
node --version
```

**Expected output**: `vX.Y.Z` (e.g., `v20.11.0`)

**If Node.js is not found**:
> "Error: Node.js is not installed. Please install Node.js 18.x or later."

#### Step 2: Parse and Compare Version

Extract the major version number and compare against requirements:

```bash
# Get major version
node -e "console.log(process.version.split('.')[0].replace('v', ''))"
```

**Minimum required**: Node.js 18.x (as of CDK 2.175.x)

#### Step 3: Version Compatibility Matrix

| CDK Version | Minimum Node.js | Recommended Node.js |
|-------------|-----------------|---------------------|
| 2.170.x+    | 18.x            | 20.x LTS            |
| 2.150.x+    | 16.x            | 18.x LTS            |
| 2.100.x+    | 14.x            | 18.x LTS            |

### npm Version Check

#### Step 1: Detect Current npm Version

```bash
npm --version
```

**Expected output**: `X.Y.Z` (e.g., `10.2.4`)

**If npm is not found**:
> "Error: npm is not installed. npm is typically installed with Node.js."

#### Step 2: Version Requirements

| Node.js Version | Bundled npm | Minimum npm Required |
|-----------------|-------------|----------------------|
| 20.x            | 10.x        | 9.x                  |
| 18.x            | 9.x         | 8.x                  |

### Upgrade Command Generation

When environment checks fail, generate appropriate upgrade commands based on the detected system.

#### Detecting the System Package Manager

**Check for nvm (Node Version Manager)**:
```bash
command -v nvm || [ -d "$HOME/.nvm" ]
```

**Check for Homebrew (macOS)**:
```bash
command -v brew
```

**Check for apt (Debian/Ubuntu)**:
```bash
command -v apt-get
```

#### Node.js Upgrade Commands

**For nvm users (recommended)**:
```bash
# Install and use Node.js 20 LTS
nvm install 20
nvm use 20
nvm alias default 20
```

**For Homebrew users (macOS)**:
```bash
# Install Node.js 20
brew install node@20

# Or upgrade existing installation
brew upgrade node
```

**For apt users (Debian/Ubuntu)**:
```bash
# Using NodeSource repository
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
```

**For Windows users**:
```
Download and install from: https://nodejs.org/
Recommended: Node.js 20.x LTS
```

#### npm Upgrade Commands

**Universal npm upgrade**:
```bash
npm install -g npm@latest
```

**If permission errors occur**:
```bash
# macOS/Linux with sudo
sudo npm install -g npm@latest

# Or fix npm permissions (recommended)
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
export PATH=~/.npm-global/bin:$PATH
npm install -g npm@latest
```

### Environment Check Output Format

#### All Checks Pass

```
=== Environment Check ===

Node.js: v20.11.0 ✓ (required: ≥18.x)
npm:     10.2.4   ✓ (required: ≥9.x)
git:     2.42.0   ✓

All environment checks passed. Proceeding with upgrade...
```

#### Checks Fail - Node.js Outdated

```
=== Environment Check ===

Node.js: v16.20.0 ✗ (required: ≥18.x)
npm:     8.19.4   ✓ (required: ≥8.x)
git:     2.42.0   ✓

✗ Environment check failed!

Node.js version 16.20.0 is below the minimum required version 18.x.

Detected system: nvm

To upgrade Node.js, run:
  nvm install 20
  nvm use 20
  nvm alias default 20

After upgrading, restart your terminal and run the upgrade again.
```

#### Checks Fail - npm Outdated

```
=== Environment Check ===

Node.js: v18.19.0 ✓ (required: ≥18.x)
npm:     7.24.0   ✗ (required: ≥8.x)
git:     2.42.0   ✓

✗ Environment check failed!

npm version 7.24.0 is below the minimum required version 8.x.

To upgrade npm, run:
  npm install -g npm@latest

After upgrading, run the upgrade again.
```

#### Checks Fail - Multiple Issues

```
=== Environment Check ===

Node.js: v14.21.0 ✗ (required: ≥18.x)
npm:     6.14.0   ✗ (required: ≥8.x)
git:     2.42.0   ✓

✗ Environment check failed!

Multiple issues detected:

1. Node.js version 14.21.0 is below the minimum required version 18.x.
2. npm version 6.14.0 is below the minimum required version 8.x.

Detected system: Homebrew (macOS)

To fix all issues, run:
  brew install node@20

This will install Node.js 20.x with a compatible npm version.

After upgrading, restart your terminal and run the upgrade again.
```

### Halting vs Proceeding

#### When to Halt

The upgrade process MUST halt when:
- Node.js version is below minimum required
- npm version is below minimum required
- git is not available
- Required tools cannot be detected

#### When to Proceed

The upgrade process may proceed when:
- All version checks pass
- User explicitly acknowledges warnings and chooses to continue

#### User Override Option

For edge cases where the user wants to proceed despite warnings:

```
Warning: Node.js version 17.9.0 is not officially supported but may work.
Minimum required: 18.x

Do you want to proceed anyway? This may cause unexpected issues. (y/n): _
```

**Note**: Never allow override for critical failures (e.g., git not available).



---

## Template Generation

### Overview

The CDK Upgrader uses a template-based approach for upgrades. A fresh CDK template is **MANDATORY** and generated using the TARGET_CDK_CLI_VERSION, which serves as the reference for updating project files, dependencies, and configurations.

**CRITICAL**: Template generation cannot be skipped. It is essential for:
- Comparing current project structure with latest CDK standards
- Identifying configuration changes needed
- Ensuring all project files are properly updated
- Validating that the selected CDK versions work together

### Temporary Directory Creation

#### Step 1: Create Temporary Directory in Project Folder

Create a temporary directory in the current project folder, preferring the name "temp":

```bash
cd "<project_path>"

# Always try to use "temp" as the directory name first
if [ -d "temp" ]; then
  # temp already exists, create with random suffix
  RANDOM_SUFFIX=$(cat /dev/urandom | LC_ALL=C tr -dc 'a-z0-9' | fold -w 6 | head -n 1)
  TEMP_DIR="temp_$RANDOM_SUFFIX"
  mkdir "$TEMP_DIR"
  echo "Template directory: $TEMP_DIR (temp was already in use)"
else
  # temp doesn't exist, use it
  TEMP_DIR="temp"
  mkdir "$TEMP_DIR"
  echo "Template directory: $TEMP_DIR"
fi
```

**Directory naming priority**:
1. **First choice**: `temp` (if directory doesn't exist)
2. **Fallback**: `temp_<random>` (only if `temp` already exists)

**Expected output**: 
- If "temp" doesn't exist: `Template directory: temp`
- If "temp" exists: `Template directory: temp_abc123 (temp was already in use)`

#### Step 2: Verify Directory Creation

```bash
test -d "$TEMP_DIR" && echo "Temporary directory created successfully"
```

**If creation fails**:
> "Error: Failed to create temporary directory. Check disk space and permissions."

#### Step 3: Record for Cleanup

Store the temporary directory path for later cleanup and set up automatic cleanup on any failure:

```bash
# Ensure cleanup on exit (success, failure, or interruption)
trap "rm -rf $TEMP_DIR" EXIT

# Also set up cleanup function for explicit calls
cleanup_temp() {
  if [ -d "$TEMP_DIR" ]; then
    rm -rf "$TEMP_DIR"
    echo "Temporary directory cleaned up: $TEMP_DIR"
  fi
}
```

**Important**: This trap ensures that the temp directory is always cleaned up, regardless of how the script exits (success, failure, or user interruption).

### Template Generation Command

#### Execute CDK Init with TARGET_CDK_CLI_VERSION

Navigate to the temporary directory and generate a fresh TypeScript CDK template using the determined TARGET_CDK_CLI_VERSION:

```bash
cd "$TEMP_DIR"
npx -y cdk@$TARGET_CDK_CLI_VERSION init app --language typescript --generate-only
```

**Command breakdown**:
- `npx -y`: Uses npx with auto-confirm (-y flag) to avoid prompts
- `cdk@$TARGET_CDK_CLI_VERSION`: Uses the specific CDK CLI version determined earlier (e.g., `cdk@2.1005.0`)
- `init app`: Initializes a new CDK application
- `--language typescript`: Specifies TypeScript as the language
- `--generate-only`: Generates files without initializing git or installing dependencies

**Example with actual version**:
```bash
npx -y cdk@2.1005.0 init app --language typescript --generate-only
```

#### Expected Output

```
Applying project template app for typescript
# Welcome to your CDK TypeScript project

This is a blank project for CDK development with TypeScript.

The `cdk.json` file tells the CDK Toolkit how to execute your app.

## Useful commands

* `npm run build`   compile typescript to js
* `npm run watch`   watch for changes and compile
* `npm run test`    perform the jest unit tests
* `npx cdk deploy`  deploy this stack to your default AWS account/region
* `npx cdk diff`    compare deployed stack with current state
* `npx cdk synth`   emits the synthesized CloudFormation template

✅ All done!
```

#### Generated Template Structure

After successful generation, the template directory contains:

```
$TEMP_DIR/
├── bin/
│   └── temp.ts                    # App entry point
├── lib/
│   └── temp-stack.ts              # Stack definition
├── test/
│   └── temp.test.ts               # Test file
├── .gitignore
├── .npmignore
├── cdk.json
├── jest.config.js
├── package.json
├── README.md
└── tsconfig.json
```

### Handling Template Generation Failures

#### Network Errors

If `npx -y cdk@$TARGET_CDK_CLI_VERSION` fails due to network issues:

```
=== Template Generation ===

Executing: npx -y cdk@2.1005.0 init app --language typescript --generate-only

✗ Template generation failed!

Error: npm ERR! network request failed
       npm ERR! network This is a problem related to network connectivity.

Cleaning up temporary directory...
Template directory cleaned up: temp

Retry options:
1. Check your internet connection
2. If behind a proxy, configure npm proxy settings:
   npm config set proxy http://proxy.example.com:8080
   npm config set https-proxy http://proxy.example.com:8080
3. Retry the operation

Retrying in 5 seconds... (attempt 2 of 3)
```

**Retry strategy**:
- Attempt 1: Immediate
- Attempt 2: Wait 5 seconds
- Attempt 3: Wait 15 seconds
- **After each failure**: Clean up temp directory
- After 3 failures: Halt and report error

#### Permission Errors

```
✗ Template generation failed!

Error: EACCES: permission denied, mkdir 'temp'

Cleaning up any partial temp directory...
Template directory cleaned up: temp

To fix permission issues:
1. Check write permissions for project directory
2. Try running with appropriate permissions
3. Check if the directory is read-only
```

#### CDK CLI Errors

```
✗ Template generation failed!

Error: CDK CLI error: Unknown option: --generate-only

This may indicate an incompatible CDK CLI version.
TARGET_CDK_CLI_VERSION: 2.1005.0

Cleaning up temporary directory...
Template directory cleaned up: temp

To fix:
1. Clear npx cache: npx clear-npx-cache
2. Verify the version exists: npm view aws-cdk@2.1005.0 version
3. Try a different version
```

### Extracting Template Information

After successful generation, extract key information from the template:

#### Step 1: Read Template package.json

```bash
cat "$TEMP_DIR/package.json"
```

Extract:
- `aws-cdk-lib` version (from template - will be overridden with TARGET_CDK_LIB_VERSION)
- `aws-cdk` CLI version (should match TARGET_CDK_CLI_VERSION)
- `typescript` version
- Required Node.js version (from `engines` field if present)
- Script definitions

**Important**: The template's `aws-cdk-lib` version may differ from TARGET_CDK_LIB_VERSION. During file processing, we will use TARGET_CDK_LIB_VERSION instead.

#### Step 2: Verify aws-cdk Version Matches TARGET_CDK_CLI_VERSION

```bash
TEMPLATE_CLI_VERSION=$(cat "$TEMP_DIR/package.json" | jq -r '.devDependencies["aws-cdk"]')

if [ "$TEMPLATE_CLI_VERSION" != "$TARGET_CDK_CLI_VERSION" ]; then
  echo "Warning: Template aws-cdk version ($TEMPLATE_CLI_VERSION) differs from TARGET_CDK_CLI_VERSION ($TARGET_CDK_CLI_VERSION)"
  echo "This may indicate a version mismatch issue."
fi
```

#### Step 3: Read Template cdk.json

```bash
cat "$TEMP_DIR/cdk.json"
```

Extract:
- Default CDK configuration options
- Context values
- Feature flags

#### Step 4: Read Template tsconfig.json

```bash
cat "$TEMP_DIR/tsconfig.json"
```

Extract:
- TypeScript compiler options
- Target ES version
- Module settings

#### Step 5: Read Template jest.config.js

```bash
cat "$TEMP_DIR/jest.config.js"
```

Extract:
- Jest test environment settings
- Test file patterns
- Transform configuration
- Setup files (e.g., aws-cdk-lib/testhelpers/jest-autoclean)

**CRITICAL**: This file MUST be read and used for complete replacement of the project's jest.config.js. Do NOT skip this step.

### Template Information Output

```
=== Template Generation Complete ===

Template generated successfully in: <project_path>/temp

Using CDK CLI version: 2.1005.0 (TARGET_CDK_CLI_VERSION)

Template Versions (from generated package.json):
  aws-cdk-lib:   2.175.0 (will be overridden with TARGET_CDK_LIB_VERSION: 2.175.3)
  aws-cdk (CLI): 2.1005.0 ✓ (matches TARGET_CDK_CLI_VERSION)
  typescript:    ~5.4.5
  constructs:    ^10.0.0

Template Files:
  - cdk.json (CDK configuration)
  - package.json (dependencies and scripts)
  - tsconfig.json (TypeScript configuration)
  - .gitignore (git ignore patterns)
  - jest.config.js (test configuration)

This template will be used as reference for upgrading your projects.
Note: aws-cdk-lib will be set to TARGET_CDK_LIB_VERSION (2.175.3), not the template version.
```

### Template Cleanup Instructions

#### Automatic Cleanup

After all projects are processed (success or failure), clean up the temporary directory:

```bash
# Remove template directory
rm -rf "$TEMP_DIR"
echo "Template directory cleaned up"
```

#### Cleanup Verification

```bash
# Verify cleanup
test ! -d "$TEMP_DIR" && echo "Cleanup successful"
```

#### Cleanup on Error

**CRITICAL**: If the upgrade process fails at any point, ensure temp directory cleanup occurs immediately:

```bash
# Trap ensures cleanup on any exit (success, failure, or interruption)
trap cleanup_template EXIT

cleanup_template() {
  if [ -d "$TEMP_DIR" ]; then
    rm -rf "$TEMP_DIR"
    echo "Template directory cleaned up after error: $TEMP_DIR"
  fi
}
```

**Cleanup triggers**:
- Template generation failure
- File processing errors
- Validation failures (npm install, cdk synth)
- Git operation failures
- User interruption (Ctrl+C)
- Any unexpected script termination

**Important**: Cleanup must occur even if the upgrade process is interrupted or fails unexpectedly. The trap ensures this happens automatically.

### Template Reuse for Multiple Projects

When upgrading multiple projects:

1. **Generate template once** at the start of the batch process (in a shared location)
2. **Reuse the same template** for all projects
3. **Clean up only after all projects** are processed

This approach:
- Reduces network calls (single `npx -y cdk@$TARGET_CDK_CLI_VERSION` execution)
- Ensures consistency across all project upgrades
- Speeds up batch processing

```
=== Multi-Project Template Usage ===

Template generated once with TARGET_CDK_CLI_VERSION: 2.1005.0
Reusing for all 3 projects:
  ✓ my-service - using template
  ✓ api-gateway - using template
  ✓ lambda-functions - using template

Template cleanup after all projects complete.
```



---

## File Processing

### Overview

After template generation, the upgrader processes project files by comparing the generated template with the existing project files. This involves intelligent merging and selective replacement to update CDK-related configurations while preserving user customizations.

**CRITICAL**: File processing follows the detailed rules defined in `file-processing.md`. This section provides the workflow integration, while the steering file contains the complete processing algorithms.

### File Processing Strategy

The upgrader categorizes files into different processing strategies:

| Strategy | Files | Approach |
|----------|-------|----------|
| **Complete Replacement** | `tsconfig.json`, `jest.config.js` | Complete replacement with template version - no merging, no preservation |
| **Targeted Replacement** | `cdk.json` | Preserve ONLY "app" field, replace ALL other fields with template values |
| **Additive Merge** | `.gitignore` | Add new entries, preserve existing |
| **Selective Merge** | `package.json` | Update CDK packages, preserve non-CDK packages |

### Step 1: Pre-Processing Validation

Before processing any files, validate the template structure:

```bash
echo "=== CDK Upgrader Power: Step 4 ==="
echo "Following: upgrade-workflow.md - File Processing"
echo "Action: Validating template structure and processing project files"
```

**Template validation**:
```bash
cd "$TEMP_DIR"

# Verify expected template files exist
# CRITICAL: jest.config.js MUST be included - do NOT skip it
REQUIRED_FILES=("cdk.json" "package.json" "tsconfig.json" "jest.config.js" ".gitignore")
for file in "${REQUIRED_FILES[@]}"; do
  if [ ! -f "$file" ]; then
    echo "✗ Template validation failed: Missing $file"
    exit 1
  fi
done

echo "✓ Template structure validated (including jest.config.js)"
```

### Step 2: File Processing Execution

**CRITICAL**: ALL files listed below MUST be processed. Do NOT skip any file.

**Files to read from BOTH project and template directories**:
1. `package.json` - from project AND template
2. `cdk.json` - from project AND template
3. `tsconfig.json` - from project AND template
4. `jest.config.js` - from project AND template (MUST NOT SKIP)
5. `.gitignore` - from project AND template

Process files in the specific order defined in `file-processing.md`:

```bash
cd "<project_path>"

echo "=== File Processing: <project_name> ==="

# Process files in required order - DO NOT SKIP ANY FILE
process_source_map_cleanup
process_file "tsconfig.json" "REPLACE"
process_file "jest.config.js" "REPLACE"    # CRITICAL: Do NOT skip this
process_file "package.json" "SELECTIVE_MERGE"
process_file "cdk.json" "TARGETED_REPLACE"
process_file ".gitignore" "ADDITIVE_MERGE"
```

#### Processing source-map-support Cleanup

```bash
process_source_map_cleanup() {
  echo "Processing source-map-support cleanup..."
  echo "  Action: CLEANUP"
  echo "  Reason: source-map-support is deprecated in CDK v2"
  
  # Find and clean all TypeScript and JavaScript files
  local files_cleaned=0
  
  find . -name "*.ts" -o -name "*.js" | grep -v node_modules | grep -v cdk.out | while read file; do
    if grep -q "source-map-support" "$file"; then
      # Remove import statements
      sed -i "/import.*source-map-support/d" "$file"
      
      # Remove require statements  
      sed -i "/require.*source-map-support/d" "$file"
      
      # Remove sourceMapSupport.install() calls
      sed -i "/sourceMapSupport\.install/d" "$file"
      
      files_cleaned=$((files_cleaned + 1))
      echo "    Cleaned: $file"
    fi
  done
  
  echo "  Result: ✓ Cleaned source-map-support from $files_cleaned files"
}
```

#### Processing tsconfig.json (Replace)

```bash
process_tsconfig() {
  echo "Processing tsconfig.json..."
  echo "  Action: REPLACE"
  echo "  Reason: TypeScript configuration must match CDK requirements"
  
  # Replace with template version
  cp "$TEMP_DIR/tsconfig.json" "tsconfig.json"
  
  echo "  Result: ✓ Replaced with template version"
}
```

#### Processing jest.config.js (Complete Replacement)

```bash
process_jest_config() {
  echo "Processing jest.config.js..."
  echo "  Action: COMPLETE REPLACEMENT"
  echo "  Reason: Jest configuration must match CDK test requirements"
  echo "  ⚠ WARNING: User customizations will be lost - re-apply manually after upgrade"
  
  # CRITICAL: Complete replacement - DO NOT merge, DO NOT preserve any settings
  cp "$TEMP_DIR/jest.config.js" "jest.config.js"
  
  # Verify the file matches template exactly
  if diff -q "$TEMP_DIR/jest.config.js" "jest.config.js" > /dev/null 2>&1; then
    echo "  Verification: ✓ File matches template exactly"
  else
    echo "  Verification: ✗ File does not match template - ERROR"
    exit 1
  fi
  
  echo "  Result: ✓ Replaced with template version (verified identical)"
}
```

#### Processing package.json (Merge)

```bash
process_package_json() {
  echo "Processing package.json..."
  echo "  Action: SELECTIVE MERGE"
  
  # CRITICAL: Use TARGET versions, not template versions
  echo "  Using TARGET_CDK_LIB_VERSION: $TARGET_CDK_LIB_VERSION"
  echo "  Using TARGET_CDK_CLI_VERSION: $TARGET_CDK_CLI_VERSION"
  
  # Execute merge algorithm from file-processing.md
  merge_package_json "$TEMP_DIR/package.json" "package.json" "$TARGET_CDK_LIB_VERSION" "$TARGET_CDK_CLI_VERSION"
  
  echo "  Result: ✓ Updated CDK dependencies, preserved user packages"
}
```

**Key merge operations**:
- Update `aws-cdk-lib` to **TARGET_CDK_LIB_VERSION** (not template version)
- Verify `aws-cdk` matches **TARGET_CDK_CLI_VERSION**
- Update TypeScript and testing dependencies to template versions
- Preserve all non-CDK dependencies unchanged
- Merge scripts (add CDK scripts, preserve user scripts)

#### Processing cdk.json (Targeted Replacement)

```bash
process_cdk_json() {
  echo "Processing cdk.json..."
  echo "  Action: TARGETED REPLACEMENT (NOT a merge)"
  echo "  ⚠ WARNING: Only 'app' field preserved, ALL other fields replaced with template"
  
  # Step 1: Extract ONLY the app field from user's cdk.json
  USER_APP_VALUE=$(jq -r '.app' "cdk.json")
  echo "  Extracted user's app value: $USER_APP_VALUE"
  
  # Step 2: Start with a COMPLETE COPY of template cdk.json
  cp "$TEMP_DIR/cdk.json" "cdk.json.new"
  
  # Step 3: Replace ONLY the app field with user's value
  jq --arg app "$USER_APP_VALUE" '.app = $app' "cdk.json.new" > "cdk.json"
  rm "cdk.json.new"
  
  # Step 4: Verify the result
  RESULT_APP=$(jq -r '.app' "cdk.json")
  if [ "$RESULT_APP" = "$USER_APP_VALUE" ]; then
    echo "  Verification: ✓ app field matches user's original value"
  else
    echo "  Verification: ✗ app field mismatch - ERROR"
    exit 1
  fi
  
  echo "  Preserved from user:"
  echo "    ✓ app: $USER_APP_VALUE"
  echo "  Replaced with template values:"
  echo "    ✗ context: All CDK feature flags replaced"
  echo "    ✗ watch: Replaced with template configuration"
  echo "    ✗ All other fields: Replaced with template values"
  echo "  ⚠ User customizations in context, watch, and other fields have been LOST"
  echo "  ⚠ Re-apply custom settings manually if needed after upgrade"
  
  echo "  Result: ✓ Processed successfully - app preserved, all other fields replaced"
}
```

**CRITICAL Processing Rules**:
- This is a **TARGETED REPLACEMENT**, NOT a merge operation
- **ONLY** the `app` field is preserved from user's cdk.json
- **ALL** other fields (context, watch, output, build, requireApproval, toolkitStackName, profile, etc.) are replaced with template values
- User customizations in context, watch, and other fields will be **LOST**
- Users must manually re-add custom settings after upgrade if needed

#### Processing .gitignore (Merge)

```bash
process_gitignore() {
  echo "Processing .gitignore..."
  echo "  Action: ADDITIVE MERGE"
  
  # Execute merge algorithm from file-processing.md
  merge_gitignore "$TEMP_DIR/.gitignore" ".gitignore"
  
  local new_entries=$(count_new_gitignore_entries)
  echo "  Added $new_entries new CDK-related entries"
  echo "  Preserved all existing entries"
  
  echo "  Result: ✓ Added CDK entries, preserved user patterns"
}
```

**Key merge operations**:
- Add new CDK-related ignore patterns (`cdk.out`, `.cdk.staging`, etc.)
- Preserve all existing user ignore patterns
- Detect and skip duplicate entries
- Maintain file formatting and comments

### Step 3: File Processing Validation

After processing all files, validate the results:

```bash
echo "=== File Processing Validation ==="

# Validate JSON files are well-formed
validate_json_file "package.json"
validate_json_file "cdk.json"

# Validate TypeScript configuration
validate_tsconfig "tsconfig.json"

# Check for required dependencies
validate_cdk_dependencies "package.json" "$TARGET_CDK_LIB_VERSION" "$TARGET_CDK_CLI_VERSION"

echo "✓ All processed files validated successfully"
```

### Step 4: File Processing Summary

Display a comprehensive summary of all file processing operations:

```bash
echo "=== File Processing Summary: <project_name> ==="
echo ""
echo "Source Code Cleanup:"
echo "  ✓ source-map-support - Removed from 3 source files"
echo ""
echo "Files Replaced:"
echo "  ✓ tsconfig.json - Replaced with template version"
echo "  ✓ jest.config.js - Replaced with template version"
echo ""
echo "Files Merged:"
echo "  ✓ package.json - Updated CDK dependencies, preserved user packages"
echo "    - aws-cdk-lib: ^2.150.0 → ^$TARGET_CDK_LIB_VERSION"
echo "    - aws-cdk: ^2.150.0 → ^$TARGET_CDK_CLI_VERSION"
echo "    - typescript: ~5.3.3 → ~5.4.5"
echo "    - Preserved: axios, lodash, eslint (3 non-CDK packages)"
echo ""
echo "  ✓ cdk.json - Targeted replacement (app preserved, all other fields replaced)"
echo "    - Preserved app: npx ts-node --prefer-ts-exts bin/my-service.ts"
echo "    - Replaced: context, watch, all other fields with template values"
echo "    - ⚠ User customizations in context/watch have been replaced"
echo "    - Updated watch configuration"
echo ""
echo "  ✓ .gitignore - Added CDK entries, preserved user patterns"
echo "    - Added: cdk.out, .cdk.staging, *.d.ts"
echo "    - Preserved: 8 existing patterns"
echo ""
echo "Files Skipped:"
echo "  - bin/my-service.ts (user source code - cleaned of source-map-support)"
echo "  - lib/my-stack.ts (user source code - cleaned of source-map-support)"
echo "  - test/my-stack.test.ts (user test code - cleaned of source-map-support)"
echo ""
echo "Total: 1 cleanup step, 2 files replaced, 3 files merged, 3 files skipped"
```

### Error Handling During File Processing

#### Template Incompatibility

If template structure is incompatible with current project:

```bash
echo "=== File Processing Error: Template Incompatibility ==="
echo ""
echo "✗ Template structure has changed unexpectedly!"
echo ""
echo "Detected issues:"
echo "1. cdk.json schema change: 'app' field changed from string to object"
echo "2. Missing expected file: jest.config.js"
echo ""
echo "This requires manual review. See file-processing.md for guidance."
echo ""
echo "Options:"
echo "1. Proceed with partial upgrade (skip incompatible files)"
echo "2. Abort upgrade and review manually"
echo "3. Force upgrade (may cause issues)"
echo ""
echo "Select option (1/2/3): _"
```

#### File Processing Failure

If individual file processing fails:

```bash
echo "=== File Processing Error: <project_name> ==="
echo ""
echo "✗ Failed to process package.json!"
echo ""
echo "Error: Invalid JSON syntax in template package.json"
echo "Details: Unexpected token '}' at line 15"
echo ""
echo "This may indicate:"
echo "1. Template generation produced invalid JSON"
echo "2. File corruption during template creation"
echo "3. Incompatible CDK CLI version"
echo ""
echo "Recommended actions:"
echo "1. Regenerate template with: npx -y cdk@$TARGET_CDK_CLI_VERSION init"
echo "2. Verify template files manually"
echo "3. Report issue if problem persists"
echo ""
echo "File processing aborted. No changes committed."
```

### Integration with Deprecated Constructs

File processing may reveal deprecated construct usage that requires updates:

```bash
echo "=== Deprecated Constructs Detected During File Processing ==="
echo ""
echo "While processing files, deprecated construct usage was detected:"
echo ""
echo "Files requiring construct updates:"
echo "  - lib/my-stack.ts: Lambda runtime string → Runtime enum"
echo "  - lib/api-stack.ts: BucketEncryption.KMS_MANAGED → S3_MANAGED"
echo ""
echo "These will be addressed in the next step: Deprecated Construct Updates"
echo "See deprecated-constructs.md for detailed migration rules."
```

### File Processing Best Practices

1. **Always validate template before processing**
   - Ensure all expected files are present
   - Verify JSON files are well-formed
   - Check for schema compatibility

2. **Use TARGET versions, not template versions**
   - `aws-cdk-lib` must use TARGET_CDK_LIB_VERSION
   - `aws-cdk` must match TARGET_CDK_CLI_VERSION
   - Other dependencies can use template versions

3. **Preserve user customizations**
   - Never overwrite user source code files
   - Maintain user's dependency preferences for non-CDK packages
   - Keep user's custom configuration values

4. **Provide detailed feedback**
   - Show exactly what was changed and why
   - Report preserved vs updated values
   - Explain any skipped operations

5. **Ensure atomic operations**
   - Either complete all file processing successfully
   - Or revert all changes on any failure
   - Never leave project in partially processed state

### File Processing Output Format

```
=== CDK Upgrader Power: Step 4 Complete ===

File Processing Results:
  ✓ source-map-support: Cleaned from 3 source files
  ✓ tsconfig.json: Replaced with template version
  ✓ jest.config.js: Replaced with template version (verified identical)
  ✓ package.json: Merged (8 CDK packages updated, 4 user packages preserved)
  ✓ cdk.json: Targeted replacement (app preserved, all other fields replaced with template)
  ✓ .gitignore: Additive merge (added 3 CDK entries, preserved 8 user patterns)

Ready for next step: Deprecated Construct Updates
```



---

## Deprecated Construct Updates

### Overview

After file processing, the upgrader scans TypeScript source files for deprecated CDK constructs and updates them to their modern equivalents. This step follows the detailed guidance in `deprecated-constructs.md`.

**CRITICAL**: This step modifies user source code files (`lib/`, `bin/`, `test/`) which were intentionally preserved during file processing. All changes must be carefully validated.

### Step-by-Step Execution

#### Step 1: Announce Deprecated Construct Scan

```bash
echo "=== CDK Upgrader Power: Step 5 ==="
echo "Following: deprecated-constructs.md"
echo "Action: Scanning for deprecated constructs in TypeScript files"
```

#### Step 2: Identify Source Files to Scan

Scan all TypeScript files in the project, excluding generated files:

```bash
# Find all TypeScript source files
find . -name "*.ts" -type f \
  -not -path "./node_modules/*" \
  -not -path "./cdk.out/*" \
  -not -path "./.git/*" \
  -not -name "*.d.ts"
```

**Priority scan order**:
1. `lib/**/*.ts` - Stack definitions and custom constructs (highest priority)
2. `bin/**/*.ts` - Application entry points
3. `test/**/*.ts` - Test files

#### Step 3: Scan for Deprecated Patterns

For each TypeScript file, scan for deprecated patterns defined in `deprecated-constructs.md`:

```bash
echo "=== Scanning for Deprecated Constructs ==="

# Deprecated patterns to search (from deprecated-constructs.md)
DEPRECATED_PATTERNS=(
  # Lambda Runtime Deprecations
  "Runtime\.NODEJS_14_X"
  "Runtime\.NODEJS_16_X"
  "Runtime\.NODEJS_18_X"
  "Runtime\.PYTHON_3_8"
  "Runtime\.PYTHON_3_9"
  "Runtime\.DOTNET_CORE_3_1"
  "Runtime\.JAVA_8[^_]"
  "Runtime\.JAVA_11"
  
  # S3 Bucket Encryption Changes
  "BucketEncryption\.Kms[^_]"
  "BucketEncryption\.S3Managed"
  "BucketEncryption\.Unencrypted[^_]"
  
  # VPC and Networking Changes
  "SubnetType\.ISOLATED[^_]"
  "SubnetType\.PRIVATE[^_W]"
  "securityGroup[^s].*:"
  
  # Construct Base Class Changes
  "extends cdk\.Construct"
  "import.*Construct.*from.*@aws-cdk/core"
  "import.*Construct.*from.*aws-cdk-lib['\"]"
  
  # CDK v1 Import Patterns (should not exist in v2)
  "@aws-cdk/core"
  "@aws-cdk/aws-"
)

# Scan each file
for file in $(find . -name "*.ts" -type f -not -path "./node_modules/*" -not -path "./cdk.out/*"); do
  for pattern in "${DEPRECATED_PATTERNS[@]}"; do
    if grep -qE "$pattern" "$file"; then
      echo "Found deprecated pattern in: $file"
      grep -nE "$pattern" "$file"
    fi
  done
done
```

#### Step 4: Apply Auto-Migratable Updates

For patterns that can be automatically migrated, apply the updates:

```bash
echo "=== Applying Deprecated Construct Updates ==="

# Function to update deprecated constructs
update_deprecated_constructs() {
  local file="$1"
  local changes_made=0
  
  # Lambda Runtime Updates (from deprecated-constructs.md mapping)
  if grep -qE "Runtime\.NODEJS_14_X|Runtime\.NODEJS_16_X" "$file"; then
    sed -i '' 's/Runtime\.NODEJS_14_X/Runtime.NODEJS_20_X/g' "$file"
    sed -i '' 's/Runtime\.NODEJS_16_X/Runtime.NODEJS_20_X/g' "$file"
    echo "  ✓ Updated Lambda runtime to NODEJS_20_X"
    changes_made=$((changes_made + 1))
  fi
  
  if grep -qE "Runtime\.NODEJS_18_X" "$file"; then
    sed -i '' 's/Runtime\.NODEJS_18_X/Runtime.NODEJS_20_X/g' "$file"
    echo "  ✓ Updated Lambda runtime NODEJS_18_X to NODEJS_20_X"
    changes_made=$((changes_made + 1))
  fi
  
  if grep -qE "Runtime\.PYTHON_3_8|Runtime\.PYTHON_3_9" "$file"; then
    sed -i '' 's/Runtime\.PYTHON_3_8/Runtime.PYTHON_3_12/g' "$file"
    sed -i '' 's/Runtime\.PYTHON_3_9/Runtime.PYTHON_3_12/g' "$file"
    echo "  ✓ Updated Python runtime to PYTHON_3_12"
    changes_made=$((changes_made + 1))
  fi
  
  if grep -qE "Runtime\.DOTNET_CORE_3_1" "$file"; then
    sed -i '' 's/Runtime\.DOTNET_CORE_3_1/Runtime.DOTNET_8/g' "$file"
    echo "  ✓ Updated .NET runtime to DOTNET_8"
    changes_made=$((changes_made + 1))
  fi
  
  if grep -qE "Runtime\.JAVA_8[^_]|Runtime\.JAVA_11" "$file"; then
    sed -i '' 's/Runtime\.JAVA_8\([^_]\)/Runtime.JAVA_21\1/g' "$file"
    sed -i '' 's/Runtime\.JAVA_11/Runtime.JAVA_21/g' "$file"
    echo "  ✓ Updated Java runtime to JAVA_21"
    changes_made=$((changes_made + 1))
  fi
  
  # S3 Bucket Encryption Updates
  if grep -qE "BucketEncryption\.Kms[^_]" "$file"; then
    sed -i '' 's/BucketEncryption\.Kms\([^_]\)/BucketEncryption.KMS_MANAGED\1/g' "$file"
    echo "  ✓ Updated BucketEncryption.Kms to KMS_MANAGED"
    changes_made=$((changes_made + 1))
  fi
  
  if grep -qE "BucketEncryption\.S3Managed" "$file"; then
    sed -i '' 's/BucketEncryption\.S3Managed/BucketEncryption.S3_MANAGED/g' "$file"
    echo "  ✓ Updated BucketEncryption.S3Managed to S3_MANAGED"
    changes_made=$((changes_made + 1))
  fi
  
  if grep -qE "BucketEncryption\.Unencrypted[^_]" "$file"; then
    sed -i '' 's/BucketEncryption\.Unencrypted\([^_]\)/BucketEncryption.UNENCRYPTED\1/g' "$file"
    echo "  ✓ Updated BucketEncryption.Unencrypted to UNENCRYPTED"
    changes_made=$((changes_made + 1))
  fi
  
  # VPC SubnetType Updates
  if grep -qE "SubnetType\.ISOLATED[^_]" "$file"; then
    sed -i '' 's/SubnetType\.ISOLATED\([^_]\)/SubnetType.PRIVATE_ISOLATED\1/g' "$file"
    echo "  ✓ Updated SubnetType.ISOLATED to PRIVATE_ISOLATED"
    changes_made=$((changes_made + 1))
  fi
  
  if grep -qE "SubnetType\.PRIVATE[^_W]" "$file"; then
    sed -i '' 's/SubnetType\.PRIVATE\([^_W]\)/SubnetType.PRIVATE_WITH_EGRESS\1/g' "$file"
    echo "  ✓ Updated SubnetType.PRIVATE to PRIVATE_WITH_EGRESS"
    changes_made=$((changes_made + 1))
  fi
  
  # Construct Import Updates (CDK v2 requires 'constructs' package)
  if grep -qE "import.*\{.*Construct.*\}.*from.*['\"]aws-cdk-lib['\"]" "$file"; then
    # This is a complex update - may need manual review
    echo "  ⚠ Found Construct import from aws-cdk-lib (should be from 'constructs')"
    echo "    Manual review recommended"
  fi
  
  if grep -qE "extends cdk\.Construct" "$file"; then
    # Check if 'constructs' import exists
    if ! grep -qE "import.*\{.*Construct.*\}.*from.*['\"]constructs['\"]" "$file"; then
      # Add constructs import at the top
      sed -i '' "1i\\
import { Construct } from 'constructs';
" "$file"
      echo "  ✓ Added Construct import from 'constructs' package"
    fi
    sed -i '' 's/extends cdk\.Construct/extends Construct/g' "$file"
    echo "  ✓ Updated base class from cdk.Construct to Construct"
    changes_made=$((changes_made + 1))
  fi
  
  return $changes_made
}

# Process each source file
TOTAL_CHANGES=0
for file in $(find . -name "*.ts" -type f -not -path "./node_modules/*" -not -path "./cdk.out/*" -not -name "*.d.ts"); do
  echo "Processing: $file"
  update_deprecated_constructs "$file"
  TOTAL_CHANGES=$((TOTAL_CHANGES + $?))
done

echo ""
echo "Total deprecated constructs updated: $TOTAL_CHANGES"
```

#### Step 5: Identify Manual Migration Required

Some deprecated patterns cannot be automatically migrated and require manual review:

```bash
echo "=== Checking for Manual Migration Required ==="

MANUAL_MIGRATION_REQUIRED=()

# Check for complex patterns that need manual review
for file in $(find . -name "*.ts" -type f -not -path "./node_modules/*" -not -path "./cdk.out/*"); do
  
  # Complex mixin patterns
  if grep -qE "function.*<.*extends.*Construct" "$file"; then
    MANUAL_MIGRATION_REQUIRED+=("$file: Complex mixin pattern with Construct")
  fi
  
  # Dynamic construct generation
  if grep -qE "new.*\[.*\].*\(" "$file"; then
    MANUAL_MIGRATION_REQUIRED+=("$file: Dynamic construct instantiation")
  fi
  
  # CDK v1 imports (should have been caught earlier, but double-check)
  if grep -qE "@aws-cdk/" "$file"; then
    MANUAL_MIGRATION_REQUIRED+=("$file: CDK v1 import pattern detected")
  fi
  
  # securityGroup singular property (needs array conversion)
  if grep -qE "securityGroup\s*:" "$file" && ! grep -qE "securityGroups\s*:" "$file"; then
    MANUAL_MIGRATION_REQUIRED+=("$file: securityGroup (singular) should be securityGroups (array)")
  fi
done

if [ ${#MANUAL_MIGRATION_REQUIRED[@]} -gt 0 ]; then
  echo ""
  echo "⚠ The following files require manual migration:"
  for item in "${MANUAL_MIGRATION_REQUIRED[@]}"; do
    echo "  - $item"
  done
  echo ""
  echo "See deprecated-constructs.md for detailed migration guidance."
fi
```

#### Step 6: Generate Migration Guidance Report

For constructs requiring manual migration, generate detailed guidance:

```bash
echo "=== Migration Guidance Report ==="

if [ ${#MANUAL_MIGRATION_REQUIRED[@]} -gt 0 ]; then
  cat << 'EOF'
╔══════════════════════════════════════════════════════════════════╗
║                    MANUAL MIGRATION REQUIRED                     ║
╠══════════════════════════════════════════════════════════════════╣
║ Some deprecated constructs could not be automatically migrated.  ║
║ Please review the following items and apply fixes manually.      ║
╚══════════════════════════════════════════════════════════════════╝

For detailed migration instructions, see:
  steering/deprecated-constructs.md

Common manual migrations:

1. securityGroup → securityGroups (array)
   Before: securityGroup: mySecurityGroup
   After:  securityGroups: [mySecurityGroup]

2. Complex mixin patterns with Construct
   See "Custom Construct Handling" section in deprecated-constructs.md

3. CDK v1 imports
   Replace @aws-cdk/* imports with aws-cdk-lib equivalents

EOF
fi
```

### Deprecated Construct Update Output Format

```
=== CDK Upgrader Power: Step 5 Complete ===

Deprecated Construct Updates:
  Scanned files: 12
  Files with deprecated constructs: 3
  
  Auto-migrated:
    ✓ lib/my-stack.ts: Runtime.NODEJS_16_X → Runtime.NODEJS_20_X
    ✓ lib/my-stack.ts: BucketEncryption.Kms → BucketEncryption.KMS_MANAGED
    ✓ lib/api-stack.ts: SubnetType.PRIVATE → SubnetType.PRIVATE_WITH_EGRESS
    ✓ lib/constructs/custom.ts: cdk.Construct → Construct (with import)
  
  Manual migration required:
    ⚠ lib/api-stack.ts:45 - securityGroup needs conversion to securityGroups array
  
  Total auto-migrated: 4
  Total manual required: 1

Ready for next step: Post-Upgrade Validation
```

### Integration with Lambda Runtimes

For Lambda runtime updates, also consult `lambda-runtimes.md` for:
- Compatibility warnings between runtime versions
- Inline Lambda code considerations
- External Lambda handler updates

```bash
echo "=== Lambda Runtime Upgrade Summary ==="
echo "See lambda-runtimes.md for detailed runtime compatibility information."
echo ""
echo "Upgraded runtimes:"
echo "  - NODEJS_16_X → NODEJS_20_X (2 occurrences)"
echo "  - PYTHON_3_9 → PYTHON_3_12 (1 occurrence)"
echo ""
echo "⚠ Note: Verify Lambda code compatibility with new runtime versions."
echo "  Node.js 20 has breaking changes from Node.js 16."
echo "  Python 3.12 has breaking changes from Python 3.9."
```

### Error Handling

If deprecated construct updates fail:

```bash
echo "=== Deprecated Construct Update Failed ==="
echo ""
echo "Error: Unable to update deprecated constructs in $file"
echo ""
echo "Possible causes:"
echo "1. File is read-only"
echo "2. Complex code pattern not supported by auto-migration"
echo "3. Syntax error in source file"
echo ""
echo "Options:"
echo "1. Fix manually and continue"
echo "2. Skip this file and continue with others"
echo "3. Abort upgrade for this project"
echo ""
echo "The file has NOT been modified. Original content preserved."
```

### Cross-Platform Compatibility

**IMPORTANT**: The sed commands above use macOS syntax (`sed -i ''`). For Linux compatibility:

```bash
# Detect OS and use appropriate sed syntax
if [[ "$OSTYPE" == "darwin"* ]]; then
  SED_INPLACE="sed -i ''"
else
  SED_INPLACE="sed -i"
fi

# Use the variable in sed commands
$SED_INPLACE 's/old/new/g' "$file"
```



---

## Git Integration

### Overview

The CDK Upgrader integrates with git to ensure clean tracking of upgrade changes. This includes verifying clean working directories, ensuring you're on a feature branch (not main/master), and creating semantic commits after successful upgrades.

### Working Directory Clean Check

#### Step 1: Navigate to Project Directory

```bash
cd "<project_path>"
```

#### Step 2: Check Git Status

```bash
git status --porcelain
```

**Expected output for clean directory**: (empty output)

**Output with uncommitted changes**:
```
 M lib/my-stack.ts
 M package.json
?? new-file.ts
```

#### Step 3: Interpret Status Codes

| Code | Meaning |
|------|---------|
| `M`  | Modified |
| `A`  | Added |
| `D`  | Deleted |
| `R`  | Renamed |
| `??` | Untracked |
| `UU` | Unmerged (conflict) |

#### Step 4: Handle Uncommitted Changes

If uncommitted changes are detected, **halt the upgrade** and prompt the user:

```
=== Git Status Check: <project_name> ===

✗ Working directory is not clean!

Uncommitted changes detected:
  M  lib/my-stack.ts
  M  package.json
  ?? new-file.ts

The upgrade process requires a clean working directory to ensure
changes can be properly tracked and reverted if needed.

Options:
1. Commit your changes:
   git add .
   git commit -m "WIP: save current work"

2. Stash your changes:
   git stash push -m "Before CDK upgrade"

3. Discard changes (CAUTION: this will lose uncommitted work):
   git checkout -- .
   git clean -fd

After resolving, run the upgrade again.
```

**Important**: Never automatically commit or stash user changes. Always require explicit user action.

### Feature Branch Validation

#### Step 1: Check Current Branch

```bash
git branch --show-current
```

**Expected output**: Branch name (e.g., `feature/cdk-upgrade`, `develop`, `my-feature`)

#### Step 2: Validate Not on Main/Master

The upgrade process requires you to be on a feature branch, NOT on `main` or `master`:

```bash
CURRENT_BRANCH=$(git branch --show-current)
if [ "$CURRENT_BRANCH" = "main" ] || [ "$CURRENT_BRANCH" = "master" ]; then
  echo "Error: Cannot upgrade on $CURRENT_BRANCH branch"
  exit 1
fi
```

#### Step 3: Handle Main/Master Branch

If the current branch is `main` or `master`, **halt the upgrade** and exit with an error:

```
=== Git Branch Check: <project_name> ===

✗ Cannot proceed with upgrade on 'main' branch!

For safety reasons, CDK upgrades should be performed on a feature branch,
not directly on main or master. This allows you to:
- Review changes before merging
- Easily revert if issues arise
- Follow proper code review workflows

Error: Upgrade aborted. Please create a feature branch first and run the upgrade again.
```

**Important**: The upgrader will NOT create branches automatically. The user must create a feature branch manually before running the upgrade.

### Pre-Upgrade Git Validation

Before starting any file modifications:

```
=== Pre-Upgrade Git Validation: <project_name> ===

Checking git status... ✓ Clean working directory
Checking git repository... ✓ Valid git repository
Checking current branch... ✓ On branch: feature/cdk-upgrade (not main/master)

Ready to proceed with upgrade.
```

**Validation fails if on main/master**:

```
=== Pre-Upgrade Git Validation: <project_name> ===

Checking git status... ✓ Clean working directory
Checking git repository... ✓ Valid git repository
Checking current branch... ✗ On branch: main

✗ Upgrade aborted: Cannot upgrade on main/master branch.

Error: Please create a feature branch first and run the upgrade again.
```

### Commit Message Format

#### Semantic Commit Convention

All upgrade commits follow the semantic commit format:

```
feat(cdk): upgrade CDK to version X.Y.Z
```

**Format breakdown**:
- `feat`: Type - indicates a new feature/enhancement
- `(cdk)`: Scope - indicates CDK-related changes
- `upgrade CDK to version X.Y.Z`: Description - specific version upgraded to

#### Version in Commit Message

The version in the commit message refers to the **CDK Construct Library version** (`aws-cdk-lib`), not the CLI version:

```
feat(cdk): upgrade CDK to version 2.175.3
```

#### Extended Commit Message (Optional)

For detailed tracking, include additional information in the commit body:

```
feat(cdk): upgrade CDK to version 2.175.3

CDK Construct Library: 2.150.0 → 2.175.3
CDK CLI: 2.150.0 → 2.1005.0
TypeScript: 5.3.3 → 5.4.5

Changes:
- Updated package.json dependencies
- Replaced tsconfig.json with latest template
- Replaced jest.config.js with template version
- cdk.json: preserved app field, replaced all other fields with template
- Updated 2 deprecated constructs
- Upgraded Lambda runtime: nodejs18.x → nodejs20.x
```

### Creating Git Commits

#### Step 1: Stage All Changes

After successful upgrade and validation, stage all modified files:

```bash
cd "<project_path>"
git add .
```

#### Step 2: Verify Staged Changes

```bash
git diff --cached --stat
```

**Expected output**:
```
 cdk.json      |  5 +++--
 package.json  | 12 ++++++------
 tsconfig.json | 20 ++++++++++----------
 .gitignore    |  3 +++
 4 files changed, 22 insertions(+), 18 deletions(-)
```

#### Step 3: Create Commit

```bash
git commit -m "feat(cdk): upgrade CDK to version 2.175.3"
```

**Expected output**:
```
[main abc1234] feat(cdk): upgrade CDK to version 2.175.3
 4 files changed, 22 insertions(+), 18 deletions(-)
```

#### Step 4: Record Commit Hash

Capture the commit hash for reporting:

```bash
git rev-parse --short HEAD
```

**Output**: `abc1234`

### Separate Commits Per Project

When upgrading multiple projects, each project receives its own commit:

```
=== Git Commits Created ===

Project: my-service
  Commit: abc1234
  Message: feat(cdk): upgrade CDK to version 2.175.3
  Branch: main

Project: api-gateway
  Commit: def5678
  Message: feat(cdk): upgrade CDK to version 2.175.3
  Branch: main

Project: lambda-functions
  Commit: ghi9012
  Message: feat(cdk): upgrade CDK to version 2.175.3
  Branch: main

Total commits created: 3
```

### No Automatic Push

**Important**: The CDK Upgrader does NOT automatically push commits to remote repositories.

```
=== Upgrade Complete ===

Git commits have been created locally but NOT pushed to remote.

To push your changes:
  git push origin main

Or to push all branches:
  git push --all origin

Review the commits before pushing:
  git log --oneline -n 3
```

**Rationale**:
- Allows user to review changes before sharing
- Prevents accidental pushes to protected branches
- Respects CI/CD workflows that may require PR-based changes
- Avoids authentication issues with remote repositories

### Handling Git Operation Failures

#### Commit Failure

```
=== Git Commit Failed: <project_name> ===

✗ Failed to create git commit!

Error: error: gpg failed to sign the data
       fatal: failed to write commit object

Possible causes:
1. GPG signing is configured but GPG agent is not running
2. Git user.email or user.name is not configured

Solutions:
1. Disable GPG signing temporarily:
   git config --local commit.gpgsign false

2. Configure git user:
   git config --local user.email "you@example.com"
   git config --local user.name "Your Name"

3. Start GPG agent:
   gpgconf --launch gpg-agent

After fixing, the upgrade changes are still staged. Create commit manually:
  git commit -m "feat(cdk): upgrade CDK to version 2.175.3"
```

#### Repository Not Found

```
=== Git Error: <project_name> ===

✗ Git repository not found!

Error: fatal: not a git repository (or any of the parent directories): .git

This project is not tracked by git. Options:
1. Initialize git repository:
   cd <project_path>
   git init
   git add .
   git commit -m "Initial commit"

2. Skip git integration for this project (changes will not be committed)

Select option (1/2): _
```

#### Permission Denied

```
=== Git Error: <project_name> ===

✗ Git operation failed!

Error: error: could not lock config file .git/config: Permission denied

The git repository may have permission issues.

Solutions:
1. Check file ownership:
   ls -la <project_path>/.git/

2. Fix permissions:
   chmod -R u+rw <project_path>/.git/

3. Check if another process is using the repository
```

### Git Integration Summary

After all git operations complete:

```
=== Git Integration Summary ===

Projects with commits:
  ✓ my-service (abc1234)
  ✓ lambda-functions (ghi9012)

Projects without commits:
  ✗ api-gateway - Upgrade failed, no commit created

Reminder: Commits are local only. Push when ready:
  git push origin main
```



---

## Post-Upgrade Validation

### Overview

After applying upgrade changes to a project, validation ensures the upgrade was successful. This involves running `npm install` to update dependencies and `cdk synth` to verify the CDK application compiles and synthesizes correctly.

### npm Install Execution

#### Step 1: Navigate to Project Directory

```bash
cd "<project_path>"
```

#### Step 2: Remove Existing node_modules (Optional but Recommended)

For a clean installation, remove existing node_modules:

```bash
rm -rf node_modules package-lock.json
```

**Note**: This ensures no stale dependencies interfere with the upgrade.

#### Step 3: Execute npm Install

```bash
npm install
```

**Expected output**:
```
added 250 packages, and audited 251 packages in 15s

50 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
```

#### Step 4: Verify Installation Success

Check exit code:
```bash
echo $?  # Should be 0
```

Verify node_modules exists:
```bash
test -d node_modules && echo "node_modules created successfully"
```

Verify package-lock.json updated:
```bash
test -f package-lock.json && echo "package-lock.json updated"
```

### npm Install Error Handling

#### Dependency Resolution Errors

```
=== npm Install Failed: <project_name> ===

✗ npm install failed!

Error: npm ERR! ERESOLVE unable to resolve dependency tree
       npm ERR! peer aws-cdk-lib@"^2.100.0" from some-cdk-construct@1.0.0

This indicates a peer dependency conflict.

Possible solutions:
1. Update the conflicting package:
   npm update some-cdk-construct

2. Use legacy peer deps (temporary workaround):
   npm install --legacy-peer-deps

3. Check if the package has a newer version compatible with CDK 2.175.x

Conflicting package: some-cdk-construct@1.0.0
Required peer: aws-cdk-lib@"^2.100.0"
Installed: aws-cdk-lib@2.175.3
```

### npm Run Command Error Handling

When executing npm run commands (build, test, lint, etc.) during validation, errors may occur due to code issues introduced by the upgrade. The upgrader should attempt to resolve common issues automatically.

#### Step 1: Execute npm run Commands with Error Capture

```bash
cd "<project_path>"

# Execute common npm scripts and capture errors
for script in "build" "compile" "test" "lint"; do
  if npm run "$script" --if-present 2>&1; then
    echo "✓ npm run $script: Success"
  else
    echo "✗ npm run $script: Failed"
    handle_npm_run_error "$script" "$?"
  fi
done
```

#### Step 2: TypeScript Compilation Errors

```
=== npm run build Failed: <project_name> ===

✗ npm run build failed!

Error: lib/my-stack.ts(15,5): error TS2339: Property 'runtime' does not exist on type 'FunctionProps'
       lib/api-stack.ts(23,10): error TS2304: Cannot find name 'BucketEncryption'

TypeScript compilation errors detected.

Attempting automatic fixes...

[1/2] Fixing deprecated Lambda runtime property...
      ✓ Updated lib/my-stack.ts: runtime property → Runtime.NODEJS_20_X

[2/2] Fixing missing BucketEncryption import...
      ✓ Updated lib/api-stack.ts: Added import for BucketEncryption

Retrying npm run build...
✓ npm run build: Success after fixes
```

#### Step 3: Common Error Resolution Patterns

**Lambda Runtime Errors**:
```bash
# Detect and fix Lambda runtime issues
fix_lambda_runtime_errors() {
  local file="$1"
  
  # Fix string-based runtime to enum (no backup files - rely on git)
  sed -i 's/runtime: ['\''"]nodejs18\.x['\''"]*/runtime: lambda.Runtime.NODEJS_20_X/g' "$file"
  sed -i 's/runtime: ['\''"]nodejs16\.x['\''"]*/runtime: lambda.Runtime.NODEJS_18_X/g' "$file"
  
  # Ensure lambda import exists (no backup files - rely on git)
  if ! grep -q "import.*lambda.*from.*aws-cdk-lib" "$file"; then
    sed -i '1i import * as lambda from '\''aws-cdk-lib/aws-lambda'\'';' "$file"
  fi
  
  echo "✓ Fixed Lambda runtime in $file"
}
```

**Missing Import Errors**:
```bash
# Detect and fix missing imports
fix_missing_imports() {
  local file="$1"
  local error_output="$2"
  
  # Extract missing identifiers from TypeScript errors
  local missing_imports=$(echo "$error_output" | grep "Cannot find name" | sed -n "s/.*Cannot find name '\([^']*\)'.*/\1/p" | sort -u)
  
  for import_name in $missing_imports; do
    case "$import_name" in
      "BucketEncryption"|"Bucket")
        add_import "$file" "aws-s3" "s3"
        ;;
      "Function"|"Runtime")
        add_import "$file" "aws-lambda" "lambda"
        ;;
      "RestApi"|"LambdaIntegration")
        add_import "$file" "aws-apigateway" "apigateway"
        ;;
      *)
        echo "⚠ Unknown missing import: $import_name (manual fix required)"
        ;;
    esac
  done
}

add_import() {
  local file="$1"
  local service="$2"
  local alias="$3"
  
  if ! grep -q "import.*$alias.*from.*aws-cdk-lib/$service" "$file"; then
    sed -i "1i import * as $alias from 'aws-cdk-lib/$service';" "$file"
    echo "✓ Added import for $service as $alias in $file"
  fi
}
```

**Deprecated API Errors**:
```bash
# Fix deprecated construct APIs
fix_deprecated_apis() {
  local file="$1"
  
  # S3 Bucket encryption changes (no backup files - rely on git)
  sed -i 's/BucketEncryption\.KMS_MANAGED/BucketEncryption.S3_MANAGED/g' "$file"
  
  # API Gateway changes (no backup files - rely on git)
  sed -i 's/defaultIntegration:/defaultMethodOptions: { integration:/g' "$file"
  
  # Lambda function changes (no backup files - rely on git)
  sed -i 's/\.addEventSource(/.addEventSourceMapping(/g' "$file"
  
  echo "✓ Fixed deprecated APIs in $file"
}
```

#### Step 4: Test Errors and Fixes

```
=== npm run test Failed: <project_name> ===

✗ npm run test failed!

Error: Test suite failed to run
       Cannot find module '@aws-cdk/assertions'

Missing test dependencies detected.

Attempting automatic fixes...

[1/1] Updating test imports to CDK v2...
      ✓ Updated test/my-stack.test.ts: @aws-cdk/assertions → aws-cdk-lib/assertions

Retrying npm run test...
✓ npm run test: Success after fixes
```

**Test Import Fixes**:
```bash
# Fix test file imports
fix_test_imports() {
  local test_file="$1"
  
  # Update CDK v1 test imports to v2 (no backup files - rely on git)
  sed -i 's/@aws-cdk\/assertions/aws-cdk-lib\/assertions/g' "$test_file"
  sed -i 's/@aws-cdk\/core/aws-cdk-lib/g' "$test_file"
  
  # Update test patterns (no backup files - rely on git)
  sed -i 's/expect(stack)\.to(haveResource(/Template.fromStack(stack).hasResourceProperties(/g' "$test_file"
  
  echo "✓ Updated test imports in $test_file"
}
```

#### Step 5: Lint Errors and Fixes

```
=== npm run lint Failed: <project_name> ===

✗ npm run lint failed!

Error: /lib/my-stack.ts
       15:1  error  'lambda' is defined but never used  @typescript-eslint/no-unused-vars
       23:5  error  Missing semicolon                   @typescript-eslint/semi

Linting errors detected.

Attempting automatic fixes...

[1/2] Running eslint --fix for auto-fixable issues...
      ✓ Fixed 1 auto-fixable lint error

[2/2] Removing unused imports...
      ✓ Removed unused 'lambda' import from lib/my-stack.ts

Retrying npm run lint...
✓ npm run lint: Success after fixes
```

**Lint Auto-Fix**:
```bash
# Attempt to auto-fix lint errors
fix_lint_errors() {
  local project_path="$1"
  
  cd "$project_path"
  
  # Try eslint auto-fix first
  if command -v eslint >/dev/null 2>&1; then
    npx eslint . --fix --ext .ts,.js || true
    echo "✓ Applied eslint auto-fixes"
  fi
  
  # Remove unused imports (common after CDK upgrades)
  find . -name "*.ts" -not -path "./node_modules/*" | while read -r file; do
    remove_unused_imports "$file"
  done
}

remove_unused_imports() {
  local file="$1"
  local temp_file=$(mktemp)
  
  # Simple unused import removal (basic pattern)
  grep -v "^import.*from.*aws-cdk.*//.*unused" "$file" > "$temp_file" || cp "$file" "$temp_file"
  mv "$temp_file" "$file"
}
```

#### Step 6: Error Resolution Summary

After attempting fixes, provide a summary:

```
=== npm Run Error Resolution Summary: <project_name> ===

Errors Encountered: 3
  ✓ TypeScript compilation: Fixed (2 errors)
  ✓ Lint issues: Fixed (1 error)
  ✗ Test failures: Manual fix required (1 error)

Automatic Fixes Applied:
  • lib/my-stack.ts: Updated Lambda runtime property
  • lib/api-stack.ts: Added missing S3 import
  • lib/my-stack.ts: Removed unused import

Remaining Issues:
  • test/my-stack.test.ts: Test assertion pattern needs manual update
    
    Error: expect(...).to.have.property is not a function
    
    Fix: Update test assertion from CDK v1 to v2 pattern:
    Before: expect(template).to.have.property('Resources')
    After:  Template.fromStack(stack).hasResourceProperties('AWS::S3::Bucket', {...})

Manual intervention required for remaining test issues.
```

#### Step 7: Fallback Strategy

If automatic fixes fail, provide clear guidance:

```
=== Automatic Fix Failed: <project_name> ===

✗ Could not automatically resolve all npm run errors.

Remaining errors require manual intervention:

1. TypeScript Compilation Error:
   File: lib/custom-construct.ts:45
   Error: Type 'CustomProps' is not assignable to type 'StackProps'
   
   Suggested fix: Review the CustomProps interface and ensure compatibility
   with the latest CDK version.

2. Test Failure:
   File: test/integration.test.ts:12
   Error: Timeout exceeded in async test
   
   Suggested fix: Increase test timeout or review async test patterns.

Options:
1. Fix manually and re-run validation:
   - Edit the files above
   - Run: npm run build && npm run test
   - Run: npx cdk synth

2. Skip this project and continue with others

3. Revert changes and try again later:
   git checkout -- .

Choose option (1/2/3): _
```

#### Network Errors

```
=== npm Install Failed: <project_name> ===

✗ npm install failed!

Error: npm ERR! network request to https://registry.npmjs.org/aws-cdk-lib failed

Network connectivity issue detected.

Solutions:
1. Check internet connection
2. If behind proxy, configure npm:
   npm config set proxy http://proxy:8080
3. Try again with verbose logging:
   npm install --verbose

Retrying in 5 seconds... (attempt 2 of 3)
```

#### Permission Errors

```
=== npm Install Failed: <project_name> ===

✗ npm install failed!

Error: npm ERR! EACCES: permission denied, mkdir '/path/to/node_modules'

Permission issue detected.

Solutions:
1. Check directory ownership:
   ls -la <project_path>

2. Fix permissions:
   sudo chown -R $(whoami) <project_path>

3. If using nvm, ensure it's properly configured
```

### cdk synth Validation

#### Step 1: Execute cdk synth

After successful npm install, run cdk synth to validate the CDK application:

```bash
cd "<project_path>"
npx cdk synth
```

**Expected output**:
```
Resources:
  CDKMetadata:
    Type: AWS::CDK::Metadata
    Properties:
      Analytics: v2:deflate64:...
    Metadata:
      aws:cdk:path: MyStack/CDKMetadata/Default
    Condition: CDKMetadataAvailable
...
```

The command should output CloudFormation YAML/JSON without errors.

#### Step 2: Verify Synthesis Success

Check exit code:
```bash
echo $?  # Should be 0
```

Verify cdk.out directory created:
```bash
test -d cdk.out && echo "CDK synthesis successful"
```

### cdk synth Error Handling

#### TypeScript Compilation Errors

```
=== cdk synth Failed: <project_name> ===

✗ cdk synth failed!

Error: lib/my-stack.ts(15,5): error TS2339: Property 'runtime' does not exist on type 'FunctionProps'.

TypeScript compilation error detected.

Location: lib/my-stack.ts, line 15, column 5
Error: Property 'runtime' does not exist on type 'FunctionProps'

This may be caused by:
1. Deprecated construct API changes
2. Missing type definitions
3. Incompatible construct versions

Suggested fixes:
1. Check deprecated-constructs.md for API changes
2. Verify import statements are correct
3. Review the CDK migration guide for breaking changes

The upgrade changes have NOT been committed.
```

#### Missing Dependencies

```
=== cdk synth Failed: <project_name> ===

✗ cdk synth failed!

Error: Cannot find module '@aws-cdk/aws-lambda'

Missing module detected: @aws-cdk/aws-lambda

This is a CDK v1 import pattern. In CDK v2, use:
  import { aws_lambda as lambda } from 'aws-cdk-lib';

Or:
  import * as lambda from 'aws-cdk-lib/aws-lambda';

Files to update:
  - lib/my-stack.ts (line 3)
  - lib/lambda-construct.ts (line 1)
```

#### Context/Environment Errors

```
=== cdk synth Failed: <project_name> ===

✗ cdk synth failed!

Error: Cannot determine account/region. Please configure your environment.

CDK requires AWS account/region context for synthesis.

Solutions:
1. Set environment variables:
   export CDK_DEFAULT_ACCOUNT=123456789012
   export CDK_DEFAULT_REGION=us-east-1

2. Configure AWS CLI:
   aws configure

3. Use explicit environment in stack:
   new MyStack(app, 'MyStack', {
     env: { account: '123456789012', region: 'us-east-1' }
   });

For validation purposes, you can also use:
   npx cdk synth --no-lookups
```

#### Stack Synthesis Errors

```
=== cdk synth Failed: <project_name> ===

✗ cdk synth failed!

Error: Resolution error: Unable to resolve object tree...

CDK synthesis encountered a resolution error.

This may be caused by:
1. Circular dependencies between constructs
2. Invalid token references
3. Missing required properties

Stack trace:
  at MyStack.resolve (lib/my-stack.ts:45)
  at ...

Review the stack definition at lib/my-stack.ts:45
```

### Error Reporting Format

#### Validation Success

```
=== Post-Upgrade Validation: <project_name> ===

[1/2] Running npm install...
      ✓ Dependencies installed successfully
      ✓ 250 packages installed
      ✓ 0 vulnerabilities found

[2/2] Running cdk synth...
      ✓ TypeScript compilation successful
      ✓ CDK synthesis successful
      ✓ CloudFormation template generated

✓ Validation passed! Project is ready for commit.
```

#### Validation Failure

```
=== Post-Upgrade Validation: <project_name> ===

[1/2] Running npm install...
      ✓ Dependencies installed successfully

[2/2] Running cdk synth...
      ✗ FAILED

Error Summary:
┌────────────────────────────────────────────────────────────────┐
│ TypeScript Compilation Error                                   │
├────────────────────────────────────────────────────────────────┤
│ File: lib/my-stack.ts                                          │
│ Line: 15, Column: 5                                            │
│ Error: Property 'runtime' does not exist on type 'FunctionProps'│
├────────────────────────────────────────────────────────────────┤
│ Suggested Fix:                                                 │
│ The Lambda construct API has changed. Update the runtime       │
│ property to use the new Runtime class:                         │
│                                                                │
│ Before: runtime: 'nodejs18.x'                                  │
│ After:  runtime: lambda.Runtime.NODEJS_20_X                    │
└────────────────────────────────────────────────────────────────┘

✗ Validation failed! Changes will NOT be committed.

Options:
1. Review and fix the errors manually
2. Revert changes: git checkout -- .
3. Skip this project and continue with others
```

### Changes Summary

After successful validation, provide a summary of all changes made:

```
=== Changes Summary: <project_name> ===

Files Modified:
  ✓ package.json
    - aws-cdk-lib: ^2.150.0 → ^2.175.3
    - aws-cdk: ^2.150.0 → ^2.1005.0
    - typescript: ~5.3.3 → ~5.4.5
    - Added script: "cdk": "cdk"

  ✓ cdk.json
    - Merged new CDK context flags
    - Preserved app entry point

  ✓ tsconfig.json
    - Replaced with latest template version
    - Updated target: ES2020 → ES2022

  ✓ .gitignore
    - Added: *.js.map
    - Added: .cdk.staging/

Constructs Updated:
  ✓ lib/my-stack.ts
    - Updated Lambda runtime: nodejs18.x → nodejs20.x

Dependencies:
  ✓ 250 packages installed
  ✓ package-lock.json updated
  ✓ 0 vulnerabilities

Validation:
  ✓ npm install: Success
  ✓ cdk synth: Success
```



---

## Summary Report

### Overview

After all upgrade operations complete, generate a comprehensive summary report that provides clear visibility into what was accomplished, what failed, and what actions the user should take next.

### Success/Failure Status Format

#### Overall Status Header

```
╔══════════════════════════════════════════════════════════════════╗
║                    CDK UPGRADE SUMMARY REPORT                    ║
╠══════════════════════════════════════════════════════════════════╣
║  Status: COMPLETED WITH WARNINGS                                 ║
║  Date: 2025-01-02 14:30:45                                       ║
║  Duration: 2m 34s                                                ║
╚══════════════════════════════════════════════════════════════════╝
```

**Status values**:
- `SUCCESS` - All projects upgraded successfully
- `COMPLETED WITH WARNINGS` - Some projects succeeded, some failed
- `FAILED` - All projects failed to upgrade

#### Project Status Table

```
┌─────────────────────┬──────────┬─────────────────┬─────────────────┐
│ Project             │ Status   │ CDK Lib Version │ Commit          │
├─────────────────────┼──────────┼─────────────────┼─────────────────┤
│ my-service          │ ✓ Success│ 2.150.0→2.175.3 │ abc1234         │
│ api-gateway         │ ✗ Failed │ 2.145.0→(none)  │ (not committed) │
│ lambda-functions    │ ✓ Success│ 2.160.0→2.175.3 │ def5678         │
└─────────────────────┴──────────┴─────────────────┴─────────────────┘

Summary: 2 succeeded, 1 failed, 0 skipped
```

### Changes Summary Format

#### Successful Upgrades Detail

For each successfully upgraded project, provide detailed changes:

```
═══════════════════════════════════════════════════════════════════
PROJECT: my-service
═══════════════════════════════════════════════════════════════════

Version Changes:
  aws-cdk-lib:    ^2.150.0  →  ^2.175.3
  aws-cdk (CLI):  ^2.150.0  →  ^2.1005.0
  typescript:     ~5.3.3    →  ~5.4.5
  constructs:     ^10.0.0   →  ^10.3.0

Files Modified (4):
  • package.json      - Dependencies and scripts updated
  • cdk.json          - App preserved, all other fields replaced with template
  • tsconfig.json     - Replaced with latest template
  • jest.config.js    - Replaced with template version
  • .gitignore        - New CDK entries added

Constructs Updated (2):
  • lib/invite-stack.ts:23
    - Lambda runtime: nodejs18.x → nodejs20.x
  • lib/invite-stack.ts:45
    - Deprecated BucketEncryption.KMS_MANAGED → BucketEncryption.S3_MANAGED

Validation Results:
  • npm install:  ✓ 250 packages, 0 vulnerabilities
  • cdk synth:    ✓ CloudFormation template generated

Git Commit:
  • Hash: abc1234
  • Message: feat(cdk): upgrade CDK to version 2.175.3
  • Branch: main
```

#### Failed Upgrades Detail

For each failed project, provide error details and remediation steps:

```
═══════════════════════════════════════════════════════════════════
PROJECT: api-gateway (FAILED)
═══════════════════════════════════════════════════════════════════

Upgrade attempted but failed during: cdk synth

Error Details:
┌────────────────────────────────────────────────────────────────┐
│ TypeScript Compilation Error                                   │
├────────────────────────────────────────────────────────────────┤
│ File: lib/api-gateway-stack.ts                                 │
│ Line: 67                                                       │
│ Error: Property 'defaultIntegration' does not exist on type    │
│        'RestApiProps'                                          │
└────────────────────────────────────────────────────────────────┘

Root Cause:
  The 'defaultIntegration' property was deprecated in CDK 2.160.0
  and removed in CDK 2.170.0.

Suggested Fix:
  Replace 'defaultIntegration' with 'defaultMethodOptions':

  Before:
    const api = new RestApi(this, 'Api', {
      defaultIntegration: new LambdaIntegration(handler)
    });

  After:
    const api = new RestApi(this, 'Api', {
      defaultMethodOptions: {
        integration: new LambdaIntegration(handler)
      }
    });

Current State:
  • Files modified but NOT committed
  • To revert: git checkout -- .
  • To keep changes for manual fix: leave as-is
```

### Aggregate Statistics

```
═══════════════════════════════════════════════════════════════════
AGGREGATE STATISTICS
═══════════════════════════════════════════════════════════════════

Projects Processed:     3
  Successful:           2 (67%)
  Failed:               1 (33%)
  Skipped:              0 (0%)

Total Changes:
  Files Modified:       8
  Files Replaced:       2
  Constructs Updated:   4
  Runtimes Upgraded:    2

Dependencies Updated:
  aws-cdk-lib:          2 projects
  aws-cdk (CLI):        2 projects
  typescript:           2 projects

Git Commits Created:    2
  Total lines changed:  +156 / -98
```

### Next Steps Section

```
═══════════════════════════════════════════════════════════════════
NEXT STEPS
═══════════════════════════════════════════════════════════════════

✓ Successful Projects:
  1. Review the changes:
     git log --oneline -n 2
     git show abc1234
     git show def5678

  2. Push to remote when ready:
     git push origin main

  3. Run your test suite to verify functionality:
     npm test

✗ Failed Projects:
  1. api-gateway requires manual intervention:
     • Review error details above
     • Apply suggested fixes
     • Run upgrade again or fix manually

  2. After fixing, validate manually:
     cd deployment/cdk/api-gateway
     npm install
     npx cdk synth

  3. Commit manually after successful validation:
     git add .
     git commit -m "feat(cdk): upgrade CDK to version 2.175.3"

📚 Resources:
  • CDK Migration Guide: https://docs.aws.amazon.com/cdk/v2/guide/migrating-v2.html
  • CDK API Reference: https://docs.aws.amazon.com/cdk/api/v2/
  • Troubleshooting: See steering/troubleshooting.md
```

### Report Output Options

#### Console Output (Default)

The summary report is displayed directly in the console/chat interface.

#### Markdown File Export (Optional)

If requested, export the report to a markdown file:

```
Report exported to: ./cdk-upgrade-report-2025-01-02.md
```

#### JSON Export (Optional)

For programmatic processing, export as JSON:

```json
{
  "status": "COMPLETED_WITH_WARNINGS",
  "timestamp": "2025-01-02T14:30:45Z",
  "duration": "2m 34s",
  "projects": [
    {
      "name": "my-service",
      "path": "deployment/cdk/my-service",
      "status": "success",
      "previousVersion": "2.150.0",
      "newVersion": "2.175.3",
      "commitHash": "abc1234",
      "filesModified": 4,
      "constructsUpdated": 2
    },
    {
      "name": "api-gateway",
      "path": "deployment/cdk/api-gateway",
      "status": "failed",
      "error": "TypeScript compilation error",
      "errorFile": "lib/api-gateway-stack.ts",
      "errorLine": 67
    }
  ],
  "summary": {
    "total": 3,
    "successful": 2,
    "failed": 1,
    "skipped": 0
  }
}
```

### Validation and Git Repository Check

Before generating the final report, validate:

```
═══════════════════════════════════════════════════════════════════
PRE-REPORT VALIDATION
═══════════════════════════════════════════════════════════════════

Checking git repository status...
  ✓ All successful projects have clean working directories
  ✓ All commits created successfully
  ⚠ api-gateway has uncommitted changes (upgrade failed)

Checking project integrity...
  ✓ my-service: node_modules present, cdk.out generated
  ✓ lambda-functions: node_modules present, cdk.out generated
  ⚠ api-gateway: validation incomplete

Report generation complete.
```

### Error Scenarios in Reporting

#### All Projects Failed

```
╔══════════════════════════════════════════════════════════════════╗
║                    CDK UPGRADE SUMMARY REPORT                    ║
╠══════════════════════════════════════════════════════════════════╣
║  Status: FAILED                                                  ║
║  Date: 2025-01-02 14:30:45                                       ║
║  Duration: 45s                                                   ║
╚══════════════════════════════════════════════════════════════════╝

All 3 projects failed to upgrade.

Common Issues Detected:
  • Node.js version incompatibility (affects all projects)
  • Network connectivity issues during npm install

Recommended Actions:
  1. Upgrade Node.js to version 18.x or later
  2. Check network connectivity
  3. Run the upgrade again after resolving issues

No changes have been committed.
```

#### No Projects Found

```
╔══════════════════════════════════════════════════════════════════╗
║                    CDK UPGRADE SUMMARY REPORT                    ║
╠══════════════════════════════════════════════════════════════════╣
║  Status: NO PROJECTS FOUND                                       ║
║  Date: 2025-01-02 14:30:45                                       ║
╚══════════════════════════════════════════════════════════════════╝

No eligible CDK projects were found in the workspace.

Searched locations:
  • Current directory and all subdirectories

Requirements for eligible projects:
  • Must have cdk.json file
  • Must have package.json with aws-cdk-lib dependency
  • Must be TypeScript (tsconfig.json present)
  • Must be CDK v2 (not v1)

If you expected to find projects, check:
  1. You are in the correct workspace directory
  2. Projects have the required CDK configuration files
  3. Projects are using CDK v2 (aws-cdk-lib), not v1 (@aws-cdk/*)
```

