# Lambda Runtime Upgrades

This steering file provides detailed instructions for detecting, upgrading, and handling Lambda function runtimes during CDK upgrades. It covers runtime detection patterns, upgrade mappings, compatibility warnings, and handling of both inline and external Lambda code.

## Table of Contents

1. [Runtime Detection Instructions](#runtime-detection-instructions)
2. [Runtime Upgrade Mappings](#runtime-upgrade-mappings)
3. [Compatibility Warning Format](#compatibility-warning-format)
4. [Inline vs External Lambda Handling](#inline-vs-external-lambda-handling)

---

## Runtime Detection Instructions

### Overview

Lambda functions in CDK projects specify their runtime environment through the `runtime` property. Before upgrading runtimes, you must scan TypeScript files to identify all Lambda function definitions and their current runtime specifications.

### Identifying Lambda Function Definitions

#### Step 1: Locate CDK Source Files

Lambda functions are typically defined in stack files within the `lib/` directory:

```bash
# Find all TypeScript files that may contain Lambda definitions
find . -name "*.ts" -type f \
  -not -path "./node_modules/*" \
  -not -path "./cdk.out/*" \
  -not -path "./.git/*"
```

**Priority scan locations**:
1. `lib/**/*.ts` - Stack definitions (primary location for Lambda functions)
2. `bin/**/*.ts` - Application entry points (may contain inline Lambdas)
3. Custom construct directories

#### Step 2: Identify Lambda Import Statements

Look for Lambda-related imports that indicate Lambda function usage:

```typescript
// Standard Lambda imports
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { Function, Runtime, Code } from 'aws-cdk-lib/aws-lambda';

// Lambda with Node.js specific constructs
import * as lambdaNodejs from 'aws-cdk-lib/aws-lambda-nodejs';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';

// Lambda with Python specific constructs
import * as lambdaPython from 'aws-cdk-lib/aws-lambda-python-alpha';
import { PythonFunction } from '@aws-cdk/aws-lambda-python-alpha';

```

### Pattern Matching for Runtime Specifications

#### Lambda Function Constructor Patterns

Scan for Lambda function instantiation patterns:

**Pattern 1: Standard Lambda Function**
```typescript
new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.NODEJS_18_X,  // Runtime specification
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
});
```

**Pattern 2: Direct Runtime Import**
```typescript
import { Function, Runtime } from 'aws-cdk-lib/aws-lambda';

new Function(this, 'MyFunction', {
  runtime: Runtime.NODEJS_18_X,  // Runtime specification
  handler: 'index.handler',
  code: Code.fromAsset('lambda'),
});
```

**Pattern 3: NodejsFunction (Implicit Node.js Runtime)**
```typescript
new NodejsFunction(this, 'MyFunction', {
  runtime: Runtime.NODEJS_18_X,  // Optional - defaults to NODEJS_18_X or later
  entry: 'lambda/index.ts',
});
```

**Pattern 4: PythonFunction**
```typescript
new PythonFunction(this, 'MyFunction', {
  runtime: Runtime.PYTHON_3_9,  // Runtime specification
  entry: 'lambda',
  index: 'handler.py',
});
```

#### Runtime Detection Regex Patterns

Use these regex patterns to identify runtime specifications:

**Pattern: All Runtime Values**
```regex
Runtime\.(NODEJS_\d+_X|PYTHON_\d+_\d+|JAVA_\d+|DOTNET_\d+|DOTNET_CORE_\d+_\d+|GO_\d+_X|RUBY_\d+_\d+|PROVIDED|PROVIDED_AL2|PROVIDED_AL2023)
```

**Pattern: Node.js Runtimes**
```regex
Runtime\.NODEJS_(\d+)_X
```

**Pattern: Python Runtimes**
```regex
Runtime\.PYTHON_(\d+)_(\d+)
```

**Pattern: Java Runtimes**
```regex
Runtime\.JAVA_(\d+)
```

**Pattern: .NET Runtimes**
```regex
Runtime\.(DOTNET_\d+|DOTNET_CORE_\d+_\d+)
```

### Scanning Script Example

Here's a comprehensive approach to scan for Lambda runtimes:

```bash
#!/bin/bash
# scan-lambda-runtimes.sh

PROJECT_PATH="$1"

echo "=== Scanning for Lambda Runtimes ==="
echo "Project: $PROJECT_PATH"
echo ""

# Find all TypeScript files
TS_FILES=$(find "$PROJECT_PATH" -name "*.ts" -type f \
  -not -path "*/node_modules/*" \
  -not -path "*/cdk.out/*" \
  -not -path "*/.git/*")

# Runtime patterns to search
PATTERNS=(
  "Runtime\.NODEJS_14_X"
  "Runtime\.NODEJS_16_X"
  "Runtime\.NODEJS_18_X"
  "Runtime\.NODEJS_20_X"
  "Runtime\.PYTHON_3_8"
  "Runtime\.PYTHON_3_9"
  "Runtime\.PYTHON_3_10"
  "Runtime\.PYTHON_3_11"
  "Runtime\.PYTHON_3_12"
  "Runtime\.JAVA_8"
  "Runtime\.JAVA_11"
  "Runtime\.JAVA_17"
  "Runtime\.JAVA_21"
  "Runtime\.DOTNET_CORE_3_1"
  "Runtime\.DOTNET_6"
  "Runtime\.DOTNET_8"
)

for file in $TS_FILES; do
  for pattern in "${PATTERNS[@]}"; do
    grep -n -E "$pattern" "$file" 2>/dev/null
  done
done
```

### Runtime Detection Output Format

After scanning, report findings in this format:

```
=== Lambda Runtime Scan: <project_name> ===

Scanned files: 8
Files with Lambda functions: 3

Findings:

File: lib/api-stack.ts
  Line 25: Runtime.NODEJS_18_X
    → Function: ApiHandler
    → Status: Current (but upgradeable to NODEJS_20_X)
  
  Line 48: Runtime.NODEJS_16_X
    → Function: LegacyProcessor
    → Status: DEPRECATED - Must upgrade

File: lib/data-stack.ts
  Line 15: Runtime.PYTHON_3_9
    → Function: DataProcessor
    → Status: Approaching EOL - Recommend upgrade

File: lib/auth-stack.ts
  Line 32: Runtime.JAVA_11
    → Function: AuthValidator
    → Status: Approaching EOL - Recommend upgrade

Summary:
  Total Lambda functions: 4
  Deprecated runtimes: 1 (requires immediate upgrade)
  Approaching EOL: 2 (recommend upgrade)
  Current runtimes: 1 (optional upgrade available)
```

---

## Runtime Upgrade Mappings

### Overview

This section provides comprehensive mappings from deprecated or outdated Lambda runtimes to their recommended replacements. Each mapping includes the source runtime, target runtime, and important migration notes.

### Node.js Runtime Upgrades

Node.js runtimes should be upgraded to the latest LTS version for security and performance improvements.

| Deprecated Runtime | Replacement | Status | Notes |
|-------------------|-------------|--------|-------|
| `Runtime.NODEJS_14_X` | `Runtime.NODEJS_20_X` | **EOL** | Node.js 14 reached end-of-life |
| `Runtime.NODEJS_16_X` | `Runtime.NODEJS_20_X` | **EOL** | Node.js 16 reached end-of-life |
| `Runtime.NODEJS_18_X` | `Runtime.NODEJS_20_X` | Current | Still supported, but 20 is latest LTS |

**Migration example - Node.js**:
```typescript
// Before (deprecated)
new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.NODEJS_16_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
});

// After (updated to latest LTS)
new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
});
```

**NodejsFunction migration**:
```typescript
// Before
new NodejsFunction(this, 'MyFunction', {
  runtime: Runtime.NODEJS_16_X,
  entry: 'lambda/index.ts',
});

// After
new NodejsFunction(this, 'MyFunction', {
  runtime: Runtime.NODEJS_20_X,
  entry: 'lambda/index.ts',
});
```

### Python Runtime Upgrades

Python runtimes should be upgraded to the latest supported version.

| Deprecated Runtime | Replacement | Status | Notes |
|-------------------|-------------|--------|-------|
| `Runtime.PYTHON_3_7` | `Runtime.PYTHON_3_12` | **EOL** | Python 3.7 reached end-of-life |
| `Runtime.PYTHON_3_8` | `Runtime.PYTHON_3_12` | **EOL** | Python 3.8 reached end-of-life |
| `Runtime.PYTHON_3_9` | `Runtime.PYTHON_3_12` | Approaching EOL | Recommend upgrade |
| `Runtime.PYTHON_3_10` | `Runtime.PYTHON_3_12` | Current | Still supported |
| `Runtime.PYTHON_3_11` | `Runtime.PYTHON_3_12` | Current | Still supported |

**Migration example - Python**:
```typescript
// Before (deprecated)
new lambda.Function(this, 'DataProcessor', {
  runtime: lambda.Runtime.PYTHON_3_9,
  handler: 'handler.main',
  code: lambda.Code.fromAsset('lambda/data-processor'),
});

// After (updated)
new lambda.Function(this, 'DataProcessor', {
  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'handler.main',
  code: lambda.Code.fromAsset('lambda/data-processor'),
});
```

**PythonFunction migration**:
```typescript
// Before
new PythonFunction(this, 'MyFunction', {
  runtime: Runtime.PYTHON_3_9,
  entry: 'lambda',
  index: 'handler.py',
});

// After
new PythonFunction(this, 'MyFunction', {
  runtime: Runtime.PYTHON_3_12,
  entry: 'lambda',
  index: 'handler.py',
});
```

### Java Runtime Upgrades

Java runtimes should be upgraded to the latest LTS version.

| Deprecated Runtime | Replacement | Status | Notes |
|-------------------|-------------|--------|-------|
| `Runtime.JAVA_8` | `Runtime.JAVA_21` | **Deprecated** | Java 8 (non-AL2) deprecated |
| `Runtime.JAVA_8_CORRETTO` | `Runtime.JAVA_21` | Current | Still supported on AL2 |
| `Runtime.JAVA_11` | `Runtime.JAVA_21` | Approaching EOL | Recommend upgrade |
| `Runtime.JAVA_17` | `Runtime.JAVA_21` | Current | Still supported |

**Migration example - Java**:
```typescript
// Before (deprecated)
new lambda.Function(this, 'JavaProcessor', {
  runtime: lambda.Runtime.JAVA_11,
  handler: 'com.example.Handler::handleRequest',
  code: lambda.Code.fromAsset('lambda/java-processor/target/processor.jar'),
});

// After (updated to latest LTS)
new lambda.Function(this, 'JavaProcessor', {
  runtime: lambda.Runtime.JAVA_21,
  handler: 'com.example.Handler::handleRequest',
  code: lambda.Code.fromAsset('lambda/java-processor/target/processor.jar'),
});
```

### .NET Runtime Upgrades

.NET runtimes should be upgraded to the latest supported version.

| Deprecated Runtime | Replacement | Status | Notes |
|-------------------|-------------|--------|-------|
| `Runtime.DOTNET_CORE_2_1` | `Runtime.DOTNET_8` | **EOL** | .NET Core 2.1 end-of-life |
| `Runtime.DOTNET_CORE_3_1` | `Runtime.DOTNET_8` | **EOL** | .NET Core 3.1 end-of-life |
| `Runtime.DOTNET_6` | `Runtime.DOTNET_8` | Current | Still supported, but 8 is latest LTS |

**Migration example - .NET**:
```typescript
// Before (deprecated)
new lambda.Function(this, 'DotNetProcessor', {
  runtime: lambda.Runtime.DOTNET_CORE_3_1,
  handler: 'MyAssembly::MyNamespace.Handler::FunctionHandler',
  code: lambda.Code.fromAsset('lambda/dotnet-processor/publish'),
});

// After (updated)
new lambda.Function(this, 'DotNetProcessor', {
  runtime: lambda.Runtime.DOTNET_8,
  handler: 'MyAssembly::MyNamespace.Handler::FunctionHandler',
  code: lambda.Code.fromAsset('lambda/dotnet-processor/publish'),
});
```

### Other Runtime Upgrades

| Runtime | Replacement | Status | Notes |
|---------|-------------|--------|-------|
| `Runtime.GO_1_X` | `Runtime.PROVIDED_AL2023` | **Deprecated** | Use custom runtime |
| `Runtime.RUBY_2_7` | `Runtime.RUBY_3_2` | **EOL** | Ruby 2.7 end-of-life |
| `Runtime.PROVIDED` | `Runtime.PROVIDED_AL2023` | **Deprecated** | Use AL2023 custom runtime |
| `Runtime.PROVIDED_AL2` | `Runtime.PROVIDED_AL2023` | Current | AL2023 recommended |

### Complete Runtime Upgrade Matrix

```
┌─────────────────────────┬─────────────────────────┬────────────────┐
│ Deprecated Runtime      │ Recommended Replacement │ Priority       │
├─────────────────────────┼─────────────────────────┼────────────────┤
│ NODEJS_14_X             │ NODEJS_20_X             │ CRITICAL       │
│ NODEJS_16_X             │ NODEJS_20_X             │ CRITICAL       │
│ NODEJS_18_X             │ NODEJS_20_X             │ RECOMMENDED    │
├─────────────────────────┼─────────────────────────┼────────────────┤
│ PYTHON_3_7              │ PYTHON_3_12             │ CRITICAL       │
│ PYTHON_3_8              │ PYTHON_3_12             │ CRITICAL       │
│ PYTHON_3_9              │ PYTHON_3_12             │ HIGH           │
│ PYTHON_3_10             │ PYTHON_3_12             │ RECOMMENDED    │
│ PYTHON_3_11             │ PYTHON_3_12             │ OPTIONAL       │
├─────────────────────────┼─────────────────────────┼────────────────┤
│ JAVA_8                  │ JAVA_21                 │ HIGH           │
│ JAVA_11                 │ JAVA_21                 │ HIGH           │
│ JAVA_17                 │ JAVA_21                 │ RECOMMENDED    │
├─────────────────────────┼─────────────────────────┼────────────────┤
│ DOTNET_CORE_2_1         │ DOTNET_8                │ CRITICAL       │
│ DOTNET_CORE_3_1         │ DOTNET_8                │ CRITICAL       │
│ DOTNET_6                │ DOTNET_8                │ RECOMMENDED    │
├─────────────────────────┼─────────────────────────┼────────────────┤
│ GO_1_X                  │ PROVIDED_AL2023         │ HIGH           │
│ RUBY_2_7                │ RUBY_3_2                │ CRITICAL       │
│ PROVIDED                │ PROVIDED_AL2023         │ HIGH           │
│ PROVIDED_AL2            │ PROVIDED_AL2023         │ RECOMMENDED    │
└─────────────────────────┴─────────────────────────┴────────────────┘
```

---

## Compatibility Warning Format

### Overview

When upgrading Lambda runtimes, some changes may introduce breaking changes or require code modifications. This section defines when to warn users about potential compatibility issues and the format for migration guidance.

### When to Warn About Breaking Changes

#### Scenario 1: Major Version Jumps

Warn when upgrading across multiple major versions:

```
⚠ COMPATIBILITY WARNING

Runtime upgrade: NODEJS_14_X → NODEJS_20_X

This upgrade spans multiple Node.js major versions (14 → 20).
Breaking changes may include:

1. Removed APIs:
   - process.binding() removed
   - Some crypto methods deprecated

2. Changed behavior:
   - ESM module resolution changes
   - URL parsing stricter

3. New requirements:
   - May require package.json "type": "module" for ESM

Recommendation: Test Lambda functions thoroughly after upgrade.
```

#### Scenario 2: Python Version Upgrades

Warn about Python-specific breaking changes:

```
⚠ COMPATIBILITY WARNING

Runtime upgrade: PYTHON_3_9 → PYTHON_3_12

Potential breaking changes:

1. Removed features:
   - distutils module removed (use setuptools)
   - Some deprecated functions removed

2. Changed behavior:
   - Type hint enforcement stricter
   - asyncio changes

3. Dependency considerations:
   - Some packages may not support Python 3.12 yet
   - Check requirements.txt compatibility

Recommendation: Verify all dependencies support Python 3.12.
```

#### Scenario 3: Java Version Upgrades

Warn about Java-specific considerations:

```
⚠ COMPATIBILITY WARNING

Runtime upgrade: JAVA_11 → JAVA_21

Potential breaking changes:

1. Removed features:
   - Security Manager deprecated
   - Some internal APIs removed

2. Changed behavior:
   - Strong encapsulation of JDK internals
   - Pattern matching changes

3. Build considerations:
   - Update Maven/Gradle to support Java 21
   - Update dependencies for Java 21 compatibility

Recommendation: Rebuild JAR with Java 21 and test thoroughly.
```

#### Scenario 4: .NET Version Upgrades

Warn about .NET-specific considerations:

```
⚠ COMPATIBILITY WARNING

Runtime upgrade: DOTNET_CORE_3_1 → DOTNET_8

This is a significant upgrade from .NET Core to .NET 8.

Potential breaking changes:

1. Framework changes:
   - .NET Core → .NET (unified platform)
   - Some APIs moved or deprecated

2. Project file changes:
   - Update TargetFramework to net8.0
   - Update NuGet packages

3. Build requirements:
   - Install .NET 8 SDK
   - Update CI/CD pipelines

Recommendation: Update project files and rebuild before deploying.
```

### Migration Guidance Format

When runtime upgrades require additional action, provide structured guidance:

```
╔══════════════════════════════════════════════════════════════════╗
║                    RUNTIME MIGRATION GUIDANCE                    ║
╠══════════════════════════════════════════════════════════════════╣
║ Function: <function_name>                                        ║
║ File: <file_path>                                                ║
║ Line: <line_number>                                              ║
╠══════════════════════════════════════════════════════════════════╣
║ Current Runtime: <current_runtime>                               ║
║ Target Runtime:  <target_runtime>                                ║
║ Risk Level:      <LOW|MEDIUM|HIGH>                               ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║ CDK Code Change:                                                 ║
║   Before: runtime: lambda.Runtime.<OLD_RUNTIME>                  ║
║   After:  runtime: lambda.Runtime.<NEW_RUNTIME>                  ║
║                                                                  ║
║ Additional Steps Required:                                       ║
║   1. <step_1>                                                    ║
║   2. <step_2>                                                    ║
║   3. <step_3>                                                    ║
║                                                                  ║
║ Testing Recommendations:                                         ║
║   - <test_recommendation_1>                                      ║
║   - <test_recommendation_2>                                      ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Risk Level Definitions

| Risk Level | Description | Auto-Upgrade? |
|------------|-------------|---------------|
| **LOW** | Minor version upgrade, minimal breaking changes | Yes |
| **MEDIUM** | Major version upgrade, some breaking changes possible | Yes with warning |
| **HIGH** | Significant upgrade, breaking changes likely | Requires confirmation |

### Warning Output Examples

#### Low Risk Upgrade

```
=== Lambda Runtime Upgrade: ApiHandler ===

File: lib/api-stack.ts
Line: 25

Upgrade: NODEJS_18_X → NODEJS_20_X
Risk Level: LOW

This is a minor upgrade within the same LTS track.
Breaking changes are unlikely.

✓ Runtime updated automatically.
```

#### Medium Risk Upgrade

```
=== Lambda Runtime Upgrade: DataProcessor ===

File: lib/data-stack.ts
Line: 15

Upgrade: PYTHON_3_9 → PYTHON_3_12
Risk Level: MEDIUM

⚠ This upgrade spans multiple Python versions.

Potential issues:
- Some deprecated functions removed in Python 3.11+
- Type hint behavior changes
- Check dependency compatibility

✓ Runtime updated. Please verify Lambda function works correctly.
```

#### High Risk Upgrade

```
=== Lambda Runtime Upgrade: LegacyProcessor ===

File: lib/legacy-stack.ts
Line: 42

Upgrade: NODEJS_14_X → NODEJS_20_X
Risk Level: HIGH

⚠ SIGNIFICANT UPGRADE - Manual verification required!

This upgrade spans 3 major Node.js versions (14 → 20).

Known breaking changes:
1. process.binding() removed
2. Some crypto APIs changed
3. ESM module resolution different
4. URL parsing stricter

Required actions:
1. Review Lambda code for deprecated API usage
2. Update any native dependencies
3. Test thoroughly in non-production environment

Proceed with upgrade? (y/n): _
```

---

## Inline vs External Lambda Handling

### Overview

Lambda functions in CDK can reference code in two ways: inline code (embedded in the CDK stack) or external code (referencing files/directories). Both types require runtime upgrades, but they have different considerations.

### Identifying Inline Lambda Functions

Inline Lambda functions have their code defined directly in the CDK stack:

**Pattern 1: Inline Code String**
```typescript
new lambda.Function(this, 'InlineFunction', {
  runtime: lambda.Runtime.NODEJS_18_X,
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
    exports.handler = async (event) => {
      return { statusCode: 200, body: 'Hello World' };
    };
  `),
});
```

**Pattern 2: Inline Code with InlineCode Class**
```typescript
new lambda.Function(this, 'InlineFunction', {
  runtime: lambda.Runtime.PYTHON_3_9,
  handler: 'index.handler',
  code: new lambda.InlineCode(`
def handler(event, context):
    return {'statusCode': 200, 'body': 'Hello World'}
  `),
});
```

#### Inline Lambda Detection Regex

```regex
Code\.fromInline\(|new\s+lambda\.InlineCode\(|InlineCode\(
```

### Identifying External Lambda Functions

External Lambda functions reference code from files or directories:

**Pattern 1: Asset from Directory**
```typescript
new lambda.Function(this, 'AssetFunction', {
  runtime: lambda.Runtime.NODEJS_18_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda/my-function'),
});
```

**Pattern 2: Asset from File**
```typescript
new lambda.Function(this, 'AssetFunction', {
  runtime: lambda.Runtime.PYTHON_3_9,
  handler: 'handler.main',
  code: lambda.Code.fromAsset('lambda/handler.py'),
});
```

**Pattern 3: S3 Bucket Reference**
```typescript
new lambda.Function(this, 'S3Function', {
  runtime: lambda.Runtime.JAVA_11,
  handler: 'com.example.Handler',
  code: lambda.Code.fromBucket(bucket, 'lambda/function.jar'),
});
```

**Pattern 4: Docker Image**
```typescript
new lambda.DockerImageFunction(this, 'DockerFunction', {
  code: lambda.DockerImageCode.fromImageAsset('lambda/docker'),
});
```

**Pattern 5: NodejsFunction (Always External)**
```typescript
new NodejsFunction(this, 'NodejsFunction', {
  runtime: Runtime.NODEJS_18_X,
  entry: 'lambda/index.ts',  // External TypeScript file
});
```

#### External Lambda Detection Regex

```regex
Code\.fromAsset\(|Code\.fromBucket\(|DockerImageCode\.|entry:\s*['"]
```

### Handling Inline Lambda Runtime Upgrades

#### Upgrade Process for Inline Lambdas

1. **Identify the runtime specification** in the CDK code
2. **Update the runtime value** to the target version
3. **Review inline code** for compatibility with new runtime

**Example upgrade**:
```typescript
// Before
new lambda.Function(this, 'InlineFunction', {
  runtime: lambda.Runtime.NODEJS_16_X,  // Deprecated
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
    exports.handler = async (event) => {
      return { statusCode: 200, body: 'Hello' };
    };
  `),
});

// After
new lambda.Function(this, 'InlineFunction', {
  runtime: lambda.Runtime.NODEJS_20_X,  // Updated
  handler: 'index.handler',
  code: lambda.Code.fromInline(`
    exports.handler = async (event) => {
      return { statusCode: 200, body: 'Hello' };
    };
  `),
});
```

#### Inline Code Compatibility Considerations

When upgrading inline Lambda runtimes, check the inline code for:

**Node.js inline code**:
- Deprecated API usage (e.g., `Buffer()` constructor)
- Callback-style vs async/await patterns
- CommonJS vs ESM syntax

**Python inline code**:
- Deprecated function usage
- Print statement vs print function
- String formatting changes

### Handling External Lambda Runtime Upgrades

#### Upgrade Process for External Lambdas

1. **Update the runtime specification** in the CDK code
2. **Verify external code compatibility** with new runtime
3. **Update dependencies** in the Lambda code directory
4. **Rebuild/repackage** if necessary (Java, .NET)

**Example upgrade**:
```typescript
// Before
new lambda.Function(this, 'AssetFunction', {
  runtime: lambda.Runtime.PYTHON_3_9,  // Deprecated
  handler: 'handler.main',
  code: lambda.Code.fromAsset('lambda/data-processor'),
});

// After
new lambda.Function(this, 'AssetFunction', {
  runtime: lambda.Runtime.PYTHON_3_12,  // Updated
  handler: 'handler.main',
  code: lambda.Code.fromAsset('lambda/data-processor'),
});
```

#### External Code Verification Steps

After updating the runtime in CDK, verify the external code:

**For Node.js Lambda (lambda/my-function/)**:
```bash
# Check package.json for engine requirements
cat lambda/my-function/package.json | jq '.engines'

# Update dependencies if needed
cd lambda/my-function
npm update
npm audit fix
```

**For Python Lambda (lambda/data-processor/)**:
```bash
# Check requirements.txt for compatibility
cat lambda/data-processor/requirements.txt

# Test with new Python version locally
cd lambda/data-processor
python3.12 -c "import handler; print('OK')"
```

**For Java Lambda**:
```bash
# Rebuild with new Java version
cd lambda/java-processor
mvn clean package -Dmaven.compiler.source=21 -Dmaven.compiler.target=21
```

**For .NET Lambda**:
```bash
# Update target framework and rebuild
cd lambda/dotnet-processor
# Edit .csproj: <TargetFramework>net8.0</TargetFramework>
dotnet publish -c Release
```

### NodejsFunction and Bundled Lambdas

`NodejsFunction` from `aws-cdk-lib/aws-lambda-nodejs` uses esbuild to bundle TypeScript/JavaScript code. These require special handling:

**Upgrade process**:
```typescript
// Before
new NodejsFunction(this, 'BundledFunction', {
  runtime: Runtime.NODEJS_16_X,
  entry: 'lambda/api-handler/index.ts',
  bundling: {
    minify: true,
    sourceMap: true,
  },
});

// After
new NodejsFunction(this, 'BundledFunction', {
  runtime: Runtime.NODEJS_20_X,
  entry: 'lambda/api-handler/index.ts',
  bundling: {
    minify: true,
    sourceMap: true,
  },
});
```

**Additional considerations for NodejsFunction**:
- esbuild target may need updating for new Node.js features
- Check `tsconfig.json` in Lambda directory for compatibility
- Verify bundled dependencies work with new runtime

### Docker-based Lambda Functions

Docker-based Lambda functions (`DockerImageFunction`) don't use CDK runtime specifications:

```typescript
// Docker functions don't have runtime property
new lambda.DockerImageFunction(this, 'DockerFunction', {
  code: lambda.DockerImageCode.fromImageAsset('lambda/docker'),
});
```

**Upgrade process for Docker Lambdas**:
1. Update the base image in `Dockerfile`
2. Rebuild the Docker image

**Example Dockerfile update**:
```dockerfile
# Before
FROM public.ecr.aws/lambda/nodejs:16

# After
FROM public.ecr.aws/lambda/nodejs:20
```

### Lambda Runtime Upgrade Summary Output

After processing all Lambda functions:

```
=== Lambda Runtime Upgrade Summary ===

Total Lambda functions found: 8

Inline Lambdas: 2
  ✓ InlineHandler (NODEJS_16_X → NODEJS_20_X)
  ✓ SimpleProcessor (PYTHON_3_9 → PYTHON_3_12)

External Lambdas: 5
  ✓ ApiHandler (NODEJS_18_X → NODEJS_20_X)
    Code path: lambda/api-handler/
  ✓ DataProcessor (PYTHON_3_9 → PYTHON_3_12)
    Code path: lambda/data-processor/
  ✓ AuthValidator (JAVA_11 → JAVA_21)
    Code path: lambda/auth-validator/
    ⚠ Requires JAR rebuild with Java 21
  ✓ ReportGenerator (DOTNET_CORE_3_1 → DOTNET_8)
    Code path: lambda/report-generator/
    ⚠ Requires project update and rebuild
  ✓ BundledFunction (NODEJS_16_X → NODEJS_20_X)
    Entry: lambda/bundled/index.ts

Docker Lambdas: 1
  ⚠ DockerFunction - Manual Dockerfile update required
    Path: lambda/docker/Dockerfile
    Update base image to latest runtime version

Next Steps:
1. Review external Lambda code for compatibility
2. Rebuild Java/NET Lambdas with new SDK versions
3. Update Docker base images manually
4. Test all Lambda functions after deployment
```

### Best Practices for Lambda Runtime Upgrades

1. **Test in non-production first**: Always deploy runtime upgrades to a test environment before production

2. **Check CloudWatch Logs**: After deployment, monitor Lambda logs for runtime errors

3. **Use Lambda versions/aliases**: Deploy new runtime as a new version, then shift traffic gradually

4. **Update dependencies**: Ensure Lambda dependencies are compatible with the new runtime

5. **Review AWS documentation**: Check AWS Lambda runtime support documentation for deprecation timelines

6. **Plan for future upgrades**: Choose runtimes with longer support windows (LTS versions)
