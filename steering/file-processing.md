# File Processing Rules

This steering file provides detailed instructions for processing project files during CDK upgrades. It defines which files should be replaced completely, which should be merged intelligently, and how to handle each file type.

## Table of Contents

1. [File Categorization Rules](#file-categorization-rules)
2. [cdk.json Processing Rules](#cdkjson-processing-rules)
3. [.gitignore Processing Rules](#gitignore-processing-rules)
4. [package.json Processing Rules](#packagejson-processing-rules)
5. [Template Incompatibility Handling](#template-incompatibility-handling)

---

## File Categorization Rules

### Overview

When upgrading a CDK project, files from the generated template must be applied to the existing project. Different files require different processing strategies to preserve user customizations while ensuring compatibility with the latest CDK version.

### File Processing Categories

Files are categorized into two main processing strategies:

| Category | Strategy | Files | Details |
|----------|----------|-------|---------|
| **Complete Replacement** | Overwrite entirely with template version | `tsconfig.json`, `jest.config.js` | No merging, no preservation of existing settings |
| **Targeted Replacement** | Preserve specific fields only, replace all others | `cdk.json` | Only "app" field preserved, everything else from template |
| **Additive Merge** | Add new entries, preserve existing | `.gitignore` | Template entries added, user entries kept |
| **Selective Merge** | Update specific packages, preserve others | `package.json` | CDK packages updated, non-CDK packages preserved |

### Files to Replace Completely

#### tsconfig.json

**Strategy**: Complete replacement

**Rationale**:
- TypeScript compiler options must match CDK's expected configuration
- Incorrect tsconfig settings can cause compilation failures
- CDK templates include optimized settings for CDK development
- User customizations to tsconfig are rare and usually unnecessary for CDK projects

**Process**:
```
1. Copy template tsconfig.json to project directory
2. Overwrite existing file completely (no backup needed)
```

**Template tsconfig.json example**:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["es2022"],
    "declaration": true,
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": false,
    "inlineSourceMap": true,
    "inlineSources": true,
    "experimentalDecorators": true,
    "strictPropertyInitialization": false,
    "skipLibCheck": true,
    "typeRoots": ["./node_modules/@types"]
  },
  "exclude": ["node_modules", "cdk.out"]
}
```

**Output**:
```
Processing tsconfig.json...
  Action: REPLACE
  Reason: TypeScript configuration must match CDK requirements
  Result: ✓ Replaced with template version
```

### Files to Merge Intelligently

#### cdk.json

**Strategy**: Targeted replacement - preserve ONLY "app" field, replace ALL other fields with template values

**CRITICAL**: This is NOT a merge operation. Only the "app" field is extracted from user's file; everything else comes from the template.

**Details**: See [cdk.json Processing Rules](#cdkjson-processing-rules)

#### .gitignore

**Strategy**: Additive merge - add new entries, preserve existing

**Details**: See [.gitignore Processing Rules](#gitignore-processing-rules)

#### package.json

**Strategy**: Selective merge - update CDK packages, preserve user packages

**Details**: See [package.json Processing Rules](#packagejson-processing-rules)

#### jest.config.js

**Strategy**: Complete replacement - NO merging, NO preservation of existing settings

**Rationale**:
- Jest configuration must match CDK's expected test setup
- CDK templates include optimized Jest settings for CDK development
- Template includes CDK-specific test helpers (e.g., aws-cdk-lib/testhelpers/jest-autoclean)
- User customizations to Jest config can be re-applied after upgrade if needed
- Ensures compatibility with updated CDK testing patterns

**CRITICAL RULES**:
- DO NOT attempt to merge user's existing jest.config.js with template
- DO NOT preserve any existing settings from user's jest.config.js
- DO NOT create backup files - rely on git for recovery
- The result MUST be byte-for-byte identical to the template jest.config.js

**Process**:
```
1. Read template jest.config.js content
2. Write template content to project's jest.config.js (complete overwrite)
3. Verify the file content matches template exactly
4. Inform user that custom settings need to be re-applied manually
```

**Template jest.config.js example**:
```javascript
module.exports = {
  testEnvironment: 'node',
  roots: ['<rootDir>/test'],
  testMatch: ['**/*.test.ts'],
  transform: {
    '^.+\\.tsx?$': 'ts-jest'
  },
  setupFilesAfterEnv: ['aws-cdk-lib/testhelpers/jest-autoclean'],
};
```

**Output**:
```
Processing jest.config.js...
  Action: REPLACE (complete overwrite)
  Reason: Jest configuration must match CDK test requirements
  ⚠ User customizations will be lost - re-apply manually after upgrade
  Result: ✓ Replaced with template version (verified identical)
```$': 'ts-jest'
  },
  setupFilesAfterEnv: ['aws-cdk-lib/testhelpers/jest-autoclean'],
};
```

**Output**:
```
Processing jest.config.js...
  Action: REPLACE
  Reason: Jest configuration must match CDK test requirements
  Result: ✓ Replaced with template version
```

### Files to Skip

Some files should NOT be processed during upgrade:

| File/Directory | Reason |
|----------------|--------|
| `bin/*.ts` | Contains user application entry point |
| `lib/*.ts` | Contains user stack definitions and business logic |
| `test/*.ts` | Contains user test files |
| `node_modules/` | Will be regenerated by npm install |
| `cdk.out/` | Will be regenerated by cdk synth |
| `*.js` | Generated from TypeScript compilation |
| `*.d.ts` | Generated TypeScript declarations |

**Important**: Never modify user source code files (`bin/`, `lib/`, `test/`) during file processing. These files may require separate deprecated construct handling.

### Source-Map-Support Cleanup

**CRITICAL**: All CDK projects must be thoroughly cleaned of `source-map-support` references, which are deprecated in CDK v2.

#### Files to Check and Clean

1. **package.json**: Remove `source-map-support` from dependencies and devDependencies
2. **All TypeScript source files** (`bin/*.ts`, `lib/*.ts`, `test/*.ts`): Remove import statements
3. **All JavaScript files** (if any): Remove require statements
4. **jest.config.js**: Remove from setupFiles array (handled by replacement)
5. **Any custom setup files**: Remove references

#### Source Code Cleanup Patterns

**TypeScript files** - Remove these import patterns:
```typescript
// Remove these lines
import 'source-map-support/register';
import * as sourceMapSupport from 'source-map-support';
sourceMapSupport.install();

// Also remove require patterns
require('source-map-support/register');
const sourceMapSupport = require('source-map-support');
```

**JavaScript files** - Remove these require patterns:
```javascript
// Remove these lines
require('source-map-support/register');
const sourceMapSupport = require('source-map-support');
sourceMapSupport.install();
```

#### Cleanup Algorithm

```bash
# Scan all TypeScript and JavaScript files for source-map-support references
find . -name "*.ts" -o -name "*.js" | grep -v node_modules | grep -v cdk.out | while read file; do
  # Remove import statements (no backup files - rely on git)
  sed -i "/import.*source-map-support/d" "$file"
  
  # Remove require statements (no backup files - rely on git)
  sed -i "/require.*source-map-support/d" "$file"
  
  # Remove sourceMapSupport.install() calls (no backup files - rely on git)
  sed -i "/sourceMapSupport\.install/d" "$file"
done

# Check package.json for source-map-support dependency
if grep -q "source-map-support" package.json; then
  echo "⚠ Found source-map-support in package.json - will be removed during package.json processing"
fi
```

#### Cleanup Verification

After cleanup, verify no references remain:

```bash
# Verify no source-map-support references remain
echo "=== Source-Map-Support Cleanup Verification ==="

# Check source files
SOURCE_REFS=$(find . -name "*.ts" -o -name "*.js" | grep -v node_modules | grep -v cdk.out | xargs grep -l "source-map-support" || true)

if [ -n "$SOURCE_REFS" ]; then
  echo "✗ Source-map-support references still found in:"
  echo "$SOURCE_REFS"
  exit 1
else
  echo "✓ No source-map-support references found in source files"
fi

# Check package.json (will be cleaned during package.json processing)
if grep -q "source-map-support" package.json; then
  echo "ℹ source-map-support found in package.json (will be removed during dependency update)"
else
  echo "✓ No source-map-support found in package.json"
fi

echo "Source-map-support cleanup verification complete"
```

### File Processing Order

Process files in this specific order to ensure consistency:

```
1. source-map-support cleanup (scan and clean all source files)
2. tsconfig.json (REPLACE - affects TypeScript compilation)
3. jest.config.js (REPLACE - complete overwrite with template version)
4. package.json (update dependencies)
5. cdk.json (TARGETED REPLACE - preserve only "app" field)
6. .gitignore (ADDITIVE MERGE - add new ignore patterns)
```

**Rationale**:
- source-map-support cleanup must happen first to remove all deprecated references
- tsconfig.json must be correct before any TypeScript operations
- jest.config.js must be completely replaced before package.json to ensure test dependencies align
- package.json dependencies affect subsequent npm install
- cdk.json configuration affects cdk synth behavior
- .gitignore is least critical and processed last

### File Processing Summary Output

After processing all files, display a summary:

```
=== File Processing Summary ===

Source Code Cleanup:
  ✓ source-map-support - Removed from 3 source files (bin/app.ts, lib/my-stack.ts, test/my-stack.test.ts)

Files Completely Replaced (template version used):
  ✓ tsconfig.json - Replaced with template version
  ✓ jest.config.js - Replaced with template version (verified identical to template)

Files with Targeted Replacement:
  ✓ cdk.json - Preserved "app" field only, ALL other fields replaced with template values
    - app: "npx ts-node --prefer-ts-exts bin/my-service.ts" (preserved)
    - context, watch, all others: replaced with template (user customizations lost)

Files Merged:
  ✓ package.json - Updated CDK dependencies, preserved user packages
  ✓ .gitignore - Added 3 new CDK-related entries

Files Skipped:
  - bin/app.ts (user source code - cleaned of source-map-support)
  - lib/my-stack.ts (user source code - cleaned of source-map-support)
  - test/my-stack.test.ts (user test code - cleaned of source-map-support)

⚠ IMPORTANT: Custom settings in jest.config.js and cdk.json (except app) have been replaced.
   Re-apply custom settings manually if needed.

Total: 1 cleanup step, 2 files replaced, 1 file targeted-replaced, 2 files merged, 3 files skipped
```



---

## cdk.json Processing Rules

### Overview

The `cdk.json` file is the CDK configuration file that tells the CDK Toolkit how to execute the application. It contains critical settings that must be carefully processed to preserve the user's application entry point while ensuring all other configuration matches the latest CDK version requirements.

### CRITICAL: Processing Strategy

**Strategy**: Preserve ONLY the "app" field, REPLACE ALL other fields with template values

**This is NOT a merge operation** - it is a targeted extraction and complete replacement:
1. Extract the "app" field value from user's cdk.json
2. Take the entire template cdk.json
3. Replace only the "app" field in template with user's value
4. Write the result (template structure with user's app value)

### Critical Field: "app"

**The "app" field is the ONLY field that MUST be preserved from the user's project.**

The `app` field specifies the command to execute the CDK application:

```json
{
  "app": "npx ts-node --prefer-ts-exts bin/my-service.ts"
}
```

**Why preserve**:
- Contains the path to the user's application entry point
- Entry point filename is project-specific (e.g., `bin/my-service.ts`)
- Template uses generic name (`bin/temp.ts`) which doesn't exist in user project

### Fields to REPLACE (NOT Merge)

**ALL fields except "app" are completely replaced with template values:**

| Field | Action | Reason |
|-------|--------|--------|
| `context` | **REPLACE** | CDK feature flags must match target version exactly |
| `watch` | **REPLACE** | Watch patterns must match latest CDK expectations |
| `output` | **REPLACE** | CDK may have changed default output handling |
| `build` | **REPLACE** | CDK may have updated build process |
| `requireApproval` | **REPLACE** | CDK may have changed approval defaults |
| `toolkitStackName` | **REPLACE** | CDK may have updated bootstrap stack handling |
| `profile` | **REPLACE** | CDK may have changed profile handling |
| **Any other fields** | **REPLACE** | Must match latest CDK expectations |

**⚠ IMPORTANT**: User customizations in context, watch, and all other fields will be LOST. Users must manually re-add custom settings after upgrade if needed.

### cdk.json Processing Algorithm

```
function processCdkJson(userCdkJson, templateCdkJson):
  # Step 1: Extract ONLY the app field from user's cdk.json
  userAppValue = userCdkJson.app
  
  # Step 2: Start with a COMPLETE COPY of template cdk.json
  result = deepCopy(templateCdkJson)
  
  # Step 3: Replace ONLY the app field with user's value
  result.app = userAppValue
  
  # Step 4: Verify the result
  # - app field should equal user's original app value
  # - ALL other fields should be identical to template
  
  return result
```

**CRITICAL VERIFICATION**:
After processing, verify:
1. `result.app === userCdkJson.app` (user's app preserved)
2. For all other keys: `result[key] === templateCdkJson[key]` (template values used)

### Example Processing

**User's cdk.json** (before):
```json
{
  "app": "npx ts-node --prefer-ts-exts bin/my-service.ts",
  "watch": {
    "include": ["**"],
    "exclude": ["custom-exclude/**"]
  },
  "context": {
    "@aws-cdk/core:stackRelativeExports": true,
    "my-custom-context": "user-value"
  }
}
```

**Template cdk.json**:
```json
{
  "app": "npx ts-node --prefer-ts-exts bin/temp.ts",
  "watch": {
    "include": ["**"],
    "exclude": [
      "README.md",
      "cdk*.json",
      "**/*.d.ts",
      "**/*.js",
      "tsconfig.json",
      "package*.json",
      "yarn.lock",
      "node_modules",
      "test"
    ]
  },
  "context": {
    "@aws-cdk/aws-lambda:recognizeLayerVersion": true,
    "@aws-cdk/core:checkSecretUsage": true,
    "@aws-cdk/core:target-partitions": ["aws", "aws-cn"]
  }
}
```

**Result** (after processing):
```json
{
  "app": "npx ts-node --prefer-ts-exts bin/my-service.ts",
  "watch": {
    "include": ["**"],
    "exclude": [
      "README.md",
      "cdk*.json",
      "**/*.d.ts",
      "**/*.js",
      "tsconfig.json",
      "package*.json",
      "yarn.lock",
      "node_modules",
      "test"
    ]
  },
  "context": {
    "@aws-cdk/aws-lambda:recognizeLayerVersion": true,
    "@aws-cdk/core:checkSecretUsage": true,
    "@aws-cdk/core:target-partitions": ["aws", "aws-cn"]
  }
}
```

**Key changes**:
- ✓ `app`: Preserved user's value (`bin/my-service.ts`)
- ✗ `watch`: User's custom exclude (`custom-exclude/**`) is LOST - replaced with template
- ✗ `context`: User's custom context (`my-custom-context`) is LOST - replaced with template
- ✗ All other fields: Replaced with template values

### cdk.json Processing Output

```
Processing cdk.json...
  Action: REPLACE ALL (except app field)
  
  Preserved from user:
    ✓ app: "npx ts-node --prefer-ts-exts bin/my-service.ts"
  
  Replaced with template values:
    ✗ context: All CDK feature flags replaced with template version
    ✗ watch: Replaced with template watch configuration
    ✗ All other fields: Replaced with template values
  
  ⚠ WARNING: User customizations in the following fields have been LOST:
    - context.my-custom-context
    - watch.exclude (custom patterns)
  
  ⚠ Re-apply custom settings manually if needed after upgrade
  
  Verification:
    ✓ app field matches user's original value
    ✓ All other fields match template exactly
  
  Result: ✓ Processed successfully
```

### Handling Missing cdk.json

If the user's project is missing cdk.json (unusual but possible):

```
=== cdk.json Processing ===

⚠ Warning: No cdk.json found in project!

This is unusual for a CDK project. Options:
1. Create cdk.json from template (will need manual app path configuration)
2. Skip cdk.json processing (may cause issues)

Select option (1/2): _
```

If option 1 is selected:
```
Creating cdk.json from template...

⚠ IMPORTANT: You must update the "app" field in cdk.json!

Current value: "npx ts-node --prefer-ts-exts bin/temp.ts"
Expected format: "npx ts-node --prefer-ts-exts bin/<your-app-name>.ts"

Please update this value to match your application entry point.
```



---

## .gitignore Processing Rules

### Overview

The `.gitignore` file specifies which files and directories should be excluded from git version control. During CDK upgrades, new CDK-related entries may need to be added while preserving all existing project-specific entries.

### Processing Strategy: Additive Merge

**Strategy**: Add new CDK-related entries from template, preserve all existing entries.

**Rationale**:
- User's .gitignore may contain project-specific patterns
- Removing existing patterns could cause unwanted files to be tracked
- New CDK versions may introduce new generated files that should be ignored
- Additive approach is safe and non-destructive

### CDK-Related .gitignore Entries

The CDK template typically includes these entries:

```gitignore
# CDK asset staging directory
.cdk.staging

# CDK output directory
cdk.out

# Dependency directories
node_modules/

# TypeScript compiled output
*.js
*.d.ts

# CDK context file (optional, some projects track this)
cdk.context.json

# npm debug logs
npm-debug.log*

# Test coverage
coverage/

# IDE settings (optional)
.idea/
*.swp
*.swo
```

### Entries to Add (If Missing)

Check for and add these essential CDK entries if not present:

| Entry | Purpose | Priority |
|-------|---------|----------|
| `cdk.out` | CDK synthesis output directory | Required |
| `.cdk.staging` | CDK asset staging directory | Required |
| `node_modules/` | npm dependencies | Required |
| `*.js` | Compiled JavaScript files | Recommended |
| `*.d.ts` | TypeScript declaration files | Recommended |
| `cdk.context.json` | CDK context cache | Optional |

### Entries to Preserve

**All existing entries must be preserved**, including:

- Project-specific ignore patterns
- IDE configuration ignores
- Build output directories
- Environment files
- Custom patterns added by the user

### .gitignore Merge Algorithm

```
function mergeGitignore(userGitignore, templateGitignore):
  # Parse both files into sets of entries
  userEntries = parseGitignore(userGitignore)
  templateEntries = parseGitignore(templateGitignore)
  
  # Find new entries from template not in user's file
  newEntries = []
  for entry in templateEntries:
    if entry not in userEntries:
      newEntries.append(entry)
  
  # If no new entries, no changes needed
  if newEntries.isEmpty():
    return userGitignore  # No changes
  
  # Append new entries to user's gitignore
  result = userGitignore
  result += "\n\n# CDK Upgrade - Added entries\n"
  for entry in newEntries:
    result += entry + "\n"
  
  return result
```

### Parsing .gitignore Files

When parsing .gitignore files, handle:

1. **Comments**: Lines starting with `#` are comments
2. **Empty lines**: Preserve for readability
3. **Negation patterns**: Lines starting with `!` negate a pattern
4. **Directory patterns**: Patterns ending with `/` match directories only
5. **Wildcards**: `*`, `**`, `?` patterns

**Normalization for comparison**:
```
1. Trim whitespace from each line
2. Ignore empty lines and comments for comparison
3. Normalize directory separators (/ vs \)
4. Compare patterns case-sensitively
```

### Example Merge

**User's .gitignore**:
```gitignore
# Dependencies
node_modules/

# Build output
dist/
*.js

# Environment
.env
.env.local

# IDE
.vscode/
.idea/
```

**Template .gitignore**:
```gitignore
# Dependency directories
node_modules/

# CDK output
cdk.out
.cdk.staging

# TypeScript
*.js
*.d.ts

# Logs
npm-debug.log*
```

**Merged result**:
```gitignore
# Dependencies
node_modules/

# Build output
dist/
*.js

# Environment
.env
.env.local

# IDE
.vscode/
.idea/

# CDK Upgrade - Added entries
cdk.out
.cdk.staging
*.d.ts
npm-debug.log*
```

### Duplicate Detection

Before adding entries, check for duplicates considering:

1. **Exact matches**: `node_modules/` == `node_modules/`
2. **Trailing slash variations**: `node_modules` ≈ `node_modules/`
3. **Pattern equivalents**: `*.js` covers individual `.js` files

**Duplicate handling**:
```
# These are considered duplicates (don't add again):
node_modules/  ≈  node_modules
cdk.out/       ≈  cdk.out
*.js           ≈  *.js

# These are NOT duplicates (add if missing):
*.js           ≠  lib/*.js  (different scope)
cdk.out        ≠  cdk.out/  (file vs directory - keep both for safety)
```

### .gitignore Processing Output

```
Processing .gitignore...
  Action: MERGE (additive)
  
  Existing entries preserved: 8
  
  New entries added:
    + cdk.out
    + .cdk.staging
    + *.d.ts
    + npm-debug.log*
  
  Entries already present (skipped):
    - node_modules/ (already exists)
    - *.js (already exists)
  
  Result: ✓ Added 4 new CDK-related entries
```

### Handling Missing .gitignore

If the project has no .gitignore file:

```
=== .gitignore Processing ===

⚠ Warning: No .gitignore found in project!

Creating .gitignore from CDK template...

Created .gitignore with standard CDK entries:
  - node_modules/
  - cdk.out
  - .cdk.staging
  - *.js
  - *.d.ts
  - npm-debug.log*

Result: ✓ Created new .gitignore file
```

### Special Considerations

#### cdk.context.json

The `cdk.context.json` file is a special case:

- **Some teams track it**: Contains cached context values for reproducible deployments
- **Some teams ignore it**: Prefer fresh context lookups

**Recommendation**: Do NOT automatically add `cdk.context.json` to .gitignore. Instead, inform the user:

```
Note: cdk.context.json is not in your .gitignore.

This file caches CDK context values (like VPC IDs, AMI IDs).
- Track it: Ensures reproducible deployments across team members
- Ignore it: Always uses fresh lookups (may cause drift)

Your current choice is preserved. To change, manually edit .gitignore.
```

#### Negation Patterns

If the user has negation patterns (e.g., `!important.js`), ensure new patterns don't override them:

```
# User's pattern
*.js
!important.js

# Don't add *.js again - it would override the negation
```



---

## package.json Processing Rules

### Overview

The `package.json` file is the most complex file to process during CDK upgrades. It contains dependencies, scripts, and project metadata that must be carefully merged to update CDK packages while preserving user customizations and non-CDK dependencies.

### Processing Strategy: Template-First Merge

**Strategy**: Use template versions for all packages that appear in the CDK template, preserve only non-template packages.

**Key principles**:
1. **Template packages take precedence**: Any package present in the CDK template uses the template version
2. **Handle duplicate packages**: If a package appears in both dependencies and devDependencies, follow CDK template placement
3. **Remove deprecated packages**: Remove packages that CDK no longer uses (e.g., source-map-support)
4. **Preserve non-template packages**: Keep user packages that don't appear in the template
5. **Update aws-cdk-lib version**: Use **TARGET_CDK_LIB_VERSION** instead of template version
6. **Verify aws-cdk version**: Ensure it matches **TARGET_CDK_CLI_VERSION**

### Important: Template-First Package Processing

**Critical**: Package versions and placement are determined by the CDK template, with specific overrides:

| Package | Version Source | Placement | Reason |
|---------|----------------|-----------|--------|
| `aws-cdk-lib` | **TARGET_CDK_LIB_VERSION** | As per template | Override template version with target |
| `aws-cdk` | **TARGET_CDK_CLI_VERSION** (verify) | As per template | Must match CLI used for generation |
| **All other template packages** | **Template version** | **Template placement** | Ensure exact CDK compatibility |
| Non-template packages | User version | User placement | Preserve user customizations |

### Deprecated Package Removal

**source-map-support**: This package was removed in CDK v2 and should be deleted from user projects unless the new CDK template includes it again.

```
Packages to remove (if not in template):
- source-map-support (removed in CDK v2)
- Any @aws-cdk/* packages (CDK v1 packages, replaced by aws-cdk-lib)
```

### Scripts Processing

#### CDK-Related Scripts

These scripts should be added or updated from the template:

| Script | Command | Purpose |
|--------|---------|---------|
| `build` | `tsc` | Compile TypeScript |
| `watch` | `tsc -w` | Watch mode compilation |
| `test` | `jest` | Run tests |
| `cdk` | `cdk` | CDK CLI shortcut |

**Template scripts example**:
```json
{
  "scripts": {
    "build": "tsc",
    "watch": "tsc -w",
    "test": "jest",
    "cdk": "cdk"
  }
}
```

#### Script Merge Rules

```
For each script in template:
  If script exists in user's package.json:
    If script is CDK-related (build, watch, test, cdk):
      Update to template value
    Else:
      Preserve user's value
  Else:
    Add script from template

For each script in user's package.json:
  If script not in template:
    Preserve user's script (user-added)
```

#### Example Script Merge

**User's scripts**:
```json
{
  "scripts": {
    "build": "tsc && npm run lint",
    "test": "jest --coverage",
    "lint": "eslint .",
    "deploy": "cdk deploy --all",
    "start": "node dist/index.js"
  }
}
```

**Template scripts**:
```json
{
  "scripts": {
    "build": "tsc",
    "watch": "tsc -w",
    "test": "jest",
    "cdk": "cdk"
  }
}
```

**Merged result**:
```json
{
  "scripts": {
    "build": "tsc && npm run lint",
    "test": "jest --coverage",
    "lint": "eslint .",
    "deploy": "cdk deploy --all",
    "start": "node dist/index.js",
    "watch": "tsc -w",
    "cdk": "cdk"
  }
}
```

**Note**: User's customized `build` and `test` scripts are preserved because they contain additional commands beyond the basic template.

### Dependencies Processing

#### Version Override Rules

**CRITICAL**: When updating dependencies, use the target versions determined earlier:

```
dependencies:
  aws-cdk-lib: Use TARGET_CDK_LIB_VERSION (e.g., "^2.175.3")
               NOT the version from generated template
  
devDependencies:
  aws-cdk: Verify matches TARGET_CDK_CLI_VERSION
           If different, report error
```

#### Template Package Processing Rules

**Core principle**: If a package appears in the CDK template, use the template's version and placement.

```
For each package in template (dependencies + devDependencies):
  If package is "aws-cdk-lib":
    Use TARGET_CDK_LIB_VERSION with template placement
  Else if package is "aws-cdk":
    Verify version matches TARGET_CDK_CLI_VERSION
    Use template placement
  Else:
    Use template version and placement
    
For each package in user's project:
  If package exists in template:
    Skip (already processed above)
  Else if package is deprecated (source-map-support, @aws-cdk/*):
    Remove from user's project
  Else:
    # Non-template package - MUST be preserved
    If package appears in both user dependencies and devDependencies:
      Keep in devDependencies only (remove duplicate from dependencies)
    Else:
      Preserve user's version and placement exactly as-is
```

#### Handling Duplicate Packages

When a package appears in both dependencies and devDependencies:

```
Priority order:
1. Template placement takes precedence (if package exists in template)
2. If template has package in devDependencies, remove from dependencies
3. If template has package in dependencies, remove from devDependencies
4. If user has duplicates but template doesn't have the package, keep in devDependencies (PRESERVE THE PACKAGE)
```

**CRITICAL**: Rule 4 means non-template packages should NEVER be completely removed due to duplication.

**Example duplicate resolution for non-template package**:

User's package.json:
```json
{
  "dependencies": {
    "env-var": "^7.1.1"
  },
  "devDependencies": {
    "env-var": "^7.1.1"
  }
}
```

Template package.json:
```json
{
  // env-var not present in template
}
```

**Correct result** (keep in devDependencies):
```json
{
  "dependencies": {},
  "devDependencies": {
    "env-var": "^7.1.1"
  }
}
```

**❌ WRONG result** (completely removed):
```json
{
  "dependencies": {},
  "devDependencies": {}
}
```

**Example duplicate resolution**:

User's package.json:
```json
{
  "dependencies": {
    "typescript": "^5.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0"
  }
}
```

Template package.json:
```json
{
  "devDependencies": {
    "typescript": "~5.4.5"
  }
}
```

Result (template placement wins):
```json
{
  "dependencies": {},
  "devDependencies": {
    "typescript": "~5.4.5"
  }
}
```

#### aws-cdk Version Verification

**The template's `aws-cdk` version MUST match TARGET_CDK_CLI_VERSION.**

```
function verifyCliVersion(templatePackageJson, TARGET_CDK_CLI_VERSION):
  templateCliVersion = templatePackageJson.devDependencies["aws-cdk"]
  
  if templateCliVersion != TARGET_CDK_CLI_VERSION:
    reportError(
      "Template aws-cdk version mismatch!",
      "Template version: " + templateCliVersion,
      "Expected (TARGET_CDK_CLI_VERSION): " + TARGET_CDK_CLI_VERSION,
      "This indicates a version determination issue."
    )
    return ERROR
  
  return OK
```

**Error output if mismatch**:
```
=== Version Mismatch Error ===

✗ Template aws-cdk version does not match TARGET_CDK_CLI_VERSION!

Template aws-cdk version: 2.1006.0
TARGET_CDK_CLI_VERSION: 2.1005.0

This should not happen. Possible causes:
1. Template was generated with wrong CLI version
2. Version determination had an error
3. npm cache returned unexpected version

Please report this issue and try again.
```

#### Deprecated Package Detection

Identify and remove packages that are no longer used in CDK v2:

```
Deprecated packages to remove:
- source-map-support (removed in CDK v2)
- @aws-cdk/* (CDK v1 packages, replaced by aws-cdk-lib in v2)

Exception: If the new CDK template includes these packages again, keep them with template version.
```

**Removal process**:
```
function removeDeprecatedPackages(userPackageJson, templatePackageJson):
  deprecatedPackages = ["source-map-support"]
  
  for package in deprecatedPackages:
    # Only remove if not present in template (CDK might have re-added it)
    if package not in templatePackageJson.dependencies and 
       package not in templatePackageJson.devDependencies:
      
      if package in userPackageJson.dependencies:
        remove userPackageJson.dependencies[package]
        log("Removed deprecated package from dependencies: " + package)
      
      if package in userPackageJson.devDependencies:
        remove userPackageJson.devDependencies[package]
        log("Removed deprecated package from devDependencies: " + package)
```

#### Complete Dependency Processing Algorithm

```
function processPackageJsonDependencies(userPackageJson, templatePackageJson, TARGET_CDK_LIB_VERSION, TARGET_CDK_CLI_VERSION):
  result = {
    dependencies: {},
    devDependencies: {}
  }
  
  # Step 1: Process all template packages first
  for section in ["dependencies", "devDependencies"]:
    for package, version in templatePackageJson[section]:
      if package == "aws-cdk-lib":
        # Special case: use target version instead of template version
        result[section][package] = applyVersionPrefix(userPackageJson, package, TARGET_CDK_LIB_VERSION)
      elif package == "aws-cdk":
        # Special case: verify version matches target
        if version != TARGET_CDK_CLI_VERSION:
          reportError("Template aws-cdk version mismatch")
        result[section][package] = applyVersionPrefix(userPackageJson, package, TARGET_CDK_CLI_VERSION)
      else:
        # Use template version and placement
        result[section][package] = version
  
  # Step 2: Add non-template packages from user (preserve user customizations)
  for section in ["dependencies", "devDependencies"]:
    for package, version in userPackageJson[section]:
      # Skip if already processed from template
      if package in result.dependencies or package in result.devDependencies:
        continue
      
      # Skip deprecated packages (unless template includes them)
      if isDeprecatedPackage(package):
        log("Removed deprecated package: " + package)
        continue
      
      # Preserve user's non-template package
      result[section][package] = version
  
  return result

function isDeprecatedPackage(package):
  deprecatedPackages = [
    "source-map-support",
    # Add other deprecated packages as needed
  ]
  
  # CDK v1 packages (all @aws-cdk/* except @aws-cdk/cli-lib-alpha)
  if package.startsWith("@aws-cdk/") and package != "@aws-cdk/cli-lib-alpha":
    return true
  
  return package in deprecatedPackages
```

### Example Dependency Processing

**Target versions determined earlier**:
```
TARGET_CDK_CLI_VERSION: 2.1005.0
TARGET_CDK_LIB_VERSION: 2.175.3
```

**User's package.json**:
```json
{
  "dependencies": {
    "aws-cdk-lib": "^2.150.0",
    "constructs": "^10.0.0",
    "typescript": "^5.0.0",
    "source-map-support": "^0.5.21",
    "axios": "^1.6.0",
    "lodash": "^4.17.21"
  },
  "devDependencies": {
    "aws-cdk": "^2.150.0",
    "typescript": "~5.3.3",
    "ts-node": "^10.9.1",
    "@types/node": "20.10.0",
    "jest": "^29.7.0",
    "ts-jest": "^29.1.1",
    "@types/jest": "^29.5.11",
    "eslint": "^8.56.0",
    "@typescript-eslint/parser": "^6.18.0"
  }
}
```

**Template package.json** (generated with TARGET_CDK_CLI_VERSION):
```json
{
  "dependencies": {
    "aws-cdk-lib": "2.1005.0",
    "constructs": "^10.3.0"
  },
  "devDependencies": {
    "aws-cdk": "2.1005.0",
    "typescript": "~5.4.5",
    "ts-node": "^10.9.2",
    "@types/node": "22.7.4",
    "jest": "^29.7.0",
    "ts-jest": "^29.2.5",
    "@types/jest": "^29.5.14"
  }
}
```

**After processing**:
```json
{
  "dependencies": {
    "aws-cdk-lib": "^2.175.3",
    "constructs": "^10.3.0",
    "axios": "^1.6.0",
    "lodash": "^4.17.21"
  },
  "devDependencies": {
    "aws-cdk": "^2.1005.0",
    "typescript": "~5.4.5",
    "ts-node": "^10.9.2",
    "@types/node": "22.7.4",
    "jest": "^29.7.0",
    "ts-jest": "^29.2.5",
    "@types/jest": "^29.5.14",
    "eslint": "^8.56.0",
    "@typescript-eslint/parser": "^6.18.0",
    "env-var": "^7.1.1"
  }
}
```

**Key changes**:
- `aws-cdk-lib`: `^2.150.0` → `^2.175.3` (TARGET_CDK_LIB_VERSION, preserving ^ prefix)
- `constructs`: `^10.0.0` → `^10.3.0` (template version)
- `typescript`: Moved from dependencies to devDependencies (template placement), version updated to `~5.4.5`
- `source-map-support`: **REMOVED** (deprecated in CDK v2)
- `aws-cdk`: `^2.150.0` → `^2.1005.0` (TARGET_CDK_CLI_VERSION, preserving ^ prefix)
- Template packages: All updated to template versions
- Non-template packages (`axios`, `lodash`, `eslint`, etc.): **Preserved unchanged**
- **`env-var`: Moved from both sections to devDependencies only (duplicate resolution for non-template package)**

### Third-Party CDK Libraries

Third-party CDK construct libraries are treated the same as any other package: **if they appear in the template, use template version; if not, preserve user version**.

#### Identifying Third-Party CDK Libraries

Common patterns:
```
@aws-solutions-constructs/*
cdk-*
@cdklabs/*
```

#### Processing Strategy for Third-Party Libraries

```
For each third-party CDK library in user's project:
  If library appears in CDK template:
    Use template version and placement (same as any template package)
  Else:
    Preserve user's version and placement (non-template package)
    Optionally: Check compatibility with target aws-cdk-lib version
```

#### Compatibility Check (Optional)

For third-party libraries not in the template, optionally check compatibility:

```
=== Third-Party Library Check ===

Found non-template CDK libraries:
  - @aws-solutions-constructs/aws-lambda-dynamodb: ^2.50.0

Checking compatibility with aws-cdk-lib@2.175.3...

✓ @aws-solutions-constructs/aws-lambda-dynamodb
  Current: ^2.50.0
  Latest: ^2.55.0
  Compatible: Yes
  Action: Keep current version (not in template)
  
Note: You may manually update to ^2.55.0 if desired.
```

#### Handling Incompatible Libraries

For third-party libraries that are not in the template but may have compatibility issues:

```
=== Third-Party Library Compatibility Warning ===

⚠ Potential compatibility issue detected!

Package: some-old-cdk-construct
Current version: ^1.0.0
Required aws-cdk-lib: >=2.100.0, <2.150.0
Target aws-cdk-lib: 2.175.3

This library may not be compatible with the target CDK version.

Action: Version preserved (not in template)
Recommendation: Check for a newer version or consider alternatives

The library version will NOT be changed automatically.
```

### Preserving Non-Template Dependencies

**Critical rule**: Only packages that don't appear in the CDK template are preserved from the user's project.

Non-template dependencies include:
- Application dependencies not used by CDK (axios, lodash, express, etc.)
- User-specific testing utilities (not covered by template)
- User-specific linting tools (eslint configurations, prettier)
- User-specific build tools (webpack, esbuild, if not in template)
- Any package not present in the CDK template

**Important**: Even common packages like `typescript` or `jest` will be updated to template versions if they appear in the template.

### package.json Processing Output

```
Processing package.json...
  Action: TEMPLATE-FIRST MERGE
  
  Template packages processed:
    aws-cdk-lib: ^2.150.0 → ^2.175.3 (TARGET_CDK_LIB_VERSION)
    constructs: ^10.0.0 → ^10.3.0 (template version)
    aws-cdk: ^2.150.0 → ^2.1005.0 (TARGET_CDK_CLI_VERSION)
    typescript: ^5.0.0 → ~5.4.5 (template version, moved to devDependencies)
    ts-node: ^10.9.1 → ^10.9.2 (template version)
    @types/node: 20.10.0 → 22.7.4 (template version)
    jest: ^29.7.0 → ^29.7.0 (template version, unchanged)
    ts-jest: ^29.1.1 → ^29.2.5 (template version)
    @types/jest: ^29.5.11 → ^29.5.14 (template version)
  
  Deprecated packages removed:
    - source-map-support: ^0.5.21 (removed from dependencies)
  
  Non-template packages preserved:
    axios: ^1.6.0
    lodash: ^4.17.21
    eslint: ^8.56.0
    @typescript-eslint/parser: ^6.18.0
    env-var: ^7.1.1 (moved to devDependencies, duplicate resolved)
  
  Scripts:
    Added: watch, cdk
    Preserved: build, test, lint, deploy, start
  
  Result: ✓ Updated 9 template packages, removed 1 deprecated package, preserved 5 non-template packages
```

### Version Prefix Handling

Preserve the user's version prefix style when updating to target versions:

| User's Style | Template | Result |
|--------------|----------|--------|
| `^2.150.0` | `2.175.3` | `^2.175.3` |
| `~2.150.0` | `2.175.3` | `~2.175.3` |
| `2.150.0` | `2.175.3` | `2.175.3` |
| `>=2.150.0` | `2.175.3` | `>=2.175.3` |

**Algorithm**:
```
function applyVersionPrefix(userPackageJson, packageName, targetVersion):
  # Find the package in user's dependencies or devDependencies
  userVersion = findUserPackageVersion(userPackageJson, packageName)
  
  if userVersion:
    prefix = extractPrefix(userVersion)  # ^, ~, >=, etc.
    return prefix + targetVersion
  else:
    # Package not in user's project, use template version as-is
    return targetVersion

function extractPrefix(version):
  if version.startsWith("^"):
    return "^"
  elif version.startsWith("~"):
    return "~"
  elif version.startsWith(">="):
    return ">="
  elif version.startsWith(">"):
    return ">"
  else:
    return ""  # Exact version
```



---

## Template Incompatibility Handling

### Overview

Occasionally, the CDK template structure may change significantly between versions, or the user's project may have configurations that conflict with the expected template format. This section provides guidance for detecting and handling these incompatibility scenarios.

### Types of Template Incompatibilities

#### 1. Structural Changes

The template file structure has changed significantly:

| Change Type | Example | Impact |
|-------------|---------|--------|
| New required files | Template adds `.npmignore` | May need new configuration |
| Removed files | Template removes deprecated config files | User's config may be orphaned |
| Renamed files | `cdk.json` → `cdk.config.json` | File processing rules need update |
| Directory restructure | `lib/` → `src/` | Path references may break |

#### 2. Configuration Schema Changes

The format of configuration files has changed:

| Change Type | Example | Impact |
|-------------|---------|--------|
| New required fields | `cdk.json` requires `version` field | Merge may produce invalid config |
| Removed fields | `context` field deprecated | User's context values may be lost |
| Field type changes | `app` changes from string to object | Merge logic may fail |
| Nested structure changes | `watch.exclude` becomes `watch.ignored` | Field mapping needed |

#### 3. Dependency Conflicts

Package dependencies have incompatible requirements:

| Change Type | Example | Impact |
|-------------|---------|--------|
| Peer dependency conflicts | `constructs` requires different version | npm install may fail |
| Removed packages | `@types/node` no longer needed | May leave orphaned dependencies |
| New required packages | Template adds `esbuild` | User may need additional setup |

### Detection of Unexpected Changes

#### Pre-Processing Validation

Before processing files, validate the template structure:

```
function validateTemplateStructure(template):
  expectedFiles = [
    "cdk.json",
    "package.json", 
    "tsconfig.json",
    ".gitignore",
    "jest.config.js"  // Optional but commonly expected
  ]
  
  for file in expectedFiles:
    if file not in template.files:
      reportMissingFile(file)
      return INCOMPATIBLE
  
  # Validate cdk.json structure
  if not template.cdkJson.has("app"):
    reportMissingField("cdk.json", "app")
    return INCOMPATIBLE
  
  # Validate package.json structure
  if not template.packageJson.has("dependencies"):
    reportMissingField("package.json", "dependencies")
    return INCOMPATIBLE
  
  return COMPATIBLE
```

#### Schema Validation

Validate that configuration files match expected schemas:

```
=== Template Validation ===

Validating template structure...

cdk.json:
  ✓ Has "app" field
  ✓ Has "context" field
  ✓ Has "watch" field

package.json:
  ✓ Has "dependencies" section
  ✓ Has "devDependencies" section
  ✓ Has "scripts" section
  ✓ Contains aws-cdk-lib dependency
  ✓ Contains aws-cdk devDependency

tsconfig.json:
  ✓ Has "compilerOptions" section
  ✓ Has valid target setting

jest.config.js:
  ✓ Has valid Jest configuration
  ✓ Compatible with TypeScript setup
  ⚠ Uses ts-jest (check version compatibility)

Template validation: ✓ PASSED
```

### Incompatibility Detection Output

When incompatibilities are detected:

```
=== Template Incompatibility Detected ===

⚠ The CDK template structure has changed unexpectedly!

Detected issues:

1. Missing expected file: README.md
   The template now includes a README file.
   Impact: Project documentation may be missing.

2. New required field in cdk.json: "requireApproval"
   The template now includes deployment approval settings.
   Impact: Merge may need to add this field.

3. Changed dependency versions in package.json
   Template uses newer versions of testing dependencies.
   Impact: May require dependency updates.

These changes require manual review before proceeding.
```

### Manual Review Request Format

When manual review is required, present clear information:

```
╔══════════════════════════════════════════════════════════════════╗
║                    MANUAL REVIEW REQUIRED                        ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  The CDK template has changes that cannot be automatically       ║
║  merged. Please review the following before proceeding:          ║
║                                                                  ║
║  Files requiring review:                                         ║
║    • cdk.json - New deployment approval settings                ║
║    • package.json - Updated testing dependencies                ║
║                                                                  ║
║  Template location: /path/to/project/temp                        ║
║                                                                  ║
║  Options:                                                        ║
║    1. Review changes and proceed with caution                    ║
║    2. Abort upgrade and investigate manually                     ║
║    3. Skip incompatible files (partial upgrade)                  ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝

Select option (1/2/3): _
```

### Handling Specific Incompatibilities

#### Missing Template Files

```
=== Missing Template File ===

Expected file not found in template: jest.config.js

This may indicate:
1. CDK has changed its default test configuration
2. Template generation failed partially
3. CDK version uses different testing setup

Recommended actions:
1. Check CDK release notes for testing changes
2. Verify template generation completed successfully
3. Consider keeping existing test configuration

Proceed without updating test configuration? (y/n): _
```

#### New Required Files

```
=== New Template File Detected ===

Template includes new file: .npmignore

This file is not present in your project.

Options:
1. Add file from template (recommended)
2. Skip this file (may cause issues)
3. Review file contents first

File contents preview:
─────────────────────────────────────
*.ts
!*.d.ts
*.js.map
*.d.ts.map
node_modules
cdk.out
.cdk.staging
─────────────────────────────────────

Select option (1/2/3): _
```

#### Jest Configuration Updates

```
=== Jest Configuration Update Detected ===

Template has updated Jest configuration with new settings.

Your current jest.config.js:
{
  testEnvironment: 'node',
  testMatch: ['**/*.test.ts'],
  transform: { '^.+\\.tsx?$': 'ts-jest' }
}

Template jest.config.js:
{
  testEnvironment: 'node',
  roots: ['<rootDir>/test'],
  testMatch: ['**/*.test.ts'],
  transform: { '^.+\\.tsx?$': 'ts-jest' },
  setupFilesAfterEnv: ['aws-cdk-lib/testhelpers/jest-autoclean']
}

Changes detected:
+ Added roots configuration
+ Added setupFilesAfterEnv with CDK auto-cleanup helper

Options:
1. Merge template changes with your configuration (recommended)
2. Keep your current configuration unchanged
3. Replace with template configuration completely

Select option (1/2/3): _

If option 1 selected:
─────────────────────────────────────
Merging Jest configuration...

✓ Preserved your custom settings
✓ Added roots configuration for test discovery
✓ Added CDK auto-cleanup helper (aws-cdk-lib/testhelpers/jest-autoclean)
⚠ Review merged configuration for any conflicts
```

```
=== Configuration Schema Mismatch ===

File: cdk.json
Issue: Field structure has changed

Your current structure:
{
  "app": "npx ts-node bin/app.ts",
  "context": { ... }
}

Template structure:
{
  "app": "npx ts-node bin/app.ts",
  "requireApproval": "never",
  "context": { ... }
}

The template has added a new "requireApproval" field.

Options:
1. Add the new field from template (recommended)
2. Keep your current format (may miss new CDK features)
3. Abort and review CDK migration guide

Select option (1/2/3): _
```

### Partial Upgrade Strategy

When full upgrade is not possible, offer partial upgrade:

```
=== Partial Upgrade Available ===

Some files cannot be automatically upgraded due to incompatibilities.

Files that CAN be upgraded:
  ✓ tsconfig.json
  ✓ .gitignore
  ✓ package.json (dependencies only)
  ✓ jest.config.js (replace with template version)

Files that CANNOT be upgraded:
  ✗ cdk.json (schema mismatch)
  ✗ package.json (scripts section)

Proceed with partial upgrade?
- Upgradeable files will be processed
- Incompatible files will be skipped
- Manual intervention required for skipped files

Proceed with partial upgrade? (y/n): _
```

### Logging Incompatibilities

All incompatibilities should be logged for later review:

```
=== Incompatibility Log ===

Project: my-cdk-project
Timestamp: 2025-01-02T10:30:00Z
CDK Version: 2.175.3

Incompatibilities detected:

1. [WARN] cdk.json new field
   - Field: requireApproval
   - Expected: missing
   - Found: "never"
   - Action: Added to user configuration

2. [INFO] New template file
   - File: README.md
   - Action: Added to project

3. [ERROR] Dependency version conflict
   - Package: @types/jest
   - Template version: ^29.5.14
   - User version: ^29.0.0
   - Action: Updated to template version

Log saved to: .cdk-upgrade-log.json
```

### Recovery from Failed Upgrades

When incompatibilities cause upgrade failure:

```
=== Upgrade Failed - Recovery Options ===

The upgrade could not be completed due to incompatibilities.

Current state:
  - Some files may have been modified
  - No git commit was created
  - Original files are unchanged in git

Recovery options:

1. Revert all changes:
   git checkout -- .
   
2. Review changes and fix manually:
   git diff
   
3. Stash changes for later:
   git stash push -m "Partial CDK upgrade"

4. Get help:
   - Check CDK migration guide: https://docs.aws.amazon.com/cdk/v2/guide/migrating-v2.html
   - Review CDK release notes for breaking changes
   - Open an issue if this seems like a bug

Select recovery option (1/2/3/4): _
```

### Best Practices for Handling Incompatibilities

1. **Always validate template before processing**
   - Check for expected files and structure
   - Detect schema changes early

2. **Provide clear, actionable error messages**
   - Explain what changed
   - Suggest specific remediation steps
   - Link to relevant documentation

3. **Offer multiple resolution paths**
   - Automatic fix when safe
   - Manual review when uncertain
   - Abort when risky

4. **Log all incompatibilities**
   - Enable post-upgrade review
   - Help diagnose issues
   - Improve future upgrades

5. **Never leave project in broken state**
   - Either complete upgrade successfully
   - Or revert to pre-upgrade state
   - Partial upgrades only with explicit user consent

### Jest Configuration Integration Notes

When processing `jest.config.js`, special attention should be paid to:

1. **Dependency Coordination**: Ensure Jest-related packages in `package.json` match the configuration requirements
2. **TypeScript Integration**: Verify that Jest configuration works with the updated `tsconfig.json`
3. **Complete Replacement**: Replace entire configuration with template version for CDK compatibility
4. **Source-map-support cleanup**: Ensure all `source-map-support` references are removed from source files during cleanup step
5. **User Customization Notice**: Inform users that custom Jest settings will need to be re-applied after upgrade

The Jest configuration processing should be robust enough to handle:
- Missing Jest configuration (create from template)
- Conflicting Jest versions (update to template version)
- **Complete replacement** with CDK-optimized template configuration
- **Source-map-support cleanup** (handled in separate cleanup step before file processing)
- **User notification** about custom settings that need to be re-applied

