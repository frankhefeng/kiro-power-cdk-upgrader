# Deprecated Construct Handling

This steering file provides detailed instructions for detecting, mapping, and migrating deprecated CDK constructs during upgrades. It covers both AWS CDK constructs and custom constructs, as well as third-party library handling.

## Table of Contents

1. [Construct Detection Instructions](#construct-detection-instructions)
2. [Common Deprecated Constructs Mapping](#common-deprecated-constructs-mapping)
3. [Custom Construct Handling](#custom-construct-handling)
4. [Third-Party Library Handling](#third-party-library-handling)
5. [Migration Guidance Format](#migration-guidance-format)

---

## Construct Detection Instructions

### Overview

Before upgrading CDK constructs, you must scan TypeScript files to identify deprecated constructs that need migration. This section provides instructions for scanning code files and pattern matching guidance.

### Scanning TypeScript Files for Deprecated Constructs

#### Step 1: Identify CDK Source Files

Locate all TypeScript files that may contain CDK constructs:

```bash
# Find all TypeScript files in the project (excluding node_modules and cdk.out)
find . -name "*.ts" -type f \
  -not -path "./node_modules/*" \
  -not -path "./cdk.out/*" \
  -not -path "./.git/*"
```

**Typical CDK project structure**:
```
project/
├── bin/
│   └── app.ts              # Application entry point
├── lib/
│   ├── my-stack.ts         # Stack definitions (primary scan target)
│   └── constructs/         # Custom constructs
│       └── my-construct.ts
└── test/
    └── my-stack.test.ts    # Test files (may contain constructs)
```

**Priority scan order**:
1. `lib/**/*.ts` - Stack definitions and custom constructs (highest priority)
2. `bin/**/*.ts` - Application entry points
3. `test/**/*.ts` - Test files (lower priority, but may need updates)

#### Step 2: Read File Contents

For each TypeScript file, read the contents for analysis:

```bash
cat "<file_path>"
```

### Pattern Matching Guidance

#### Identifying Import Statements

Look for CDK import statements that may reference deprecated constructs:

**CDK v2 import patterns**:
```typescript
// Main CDK library imports
import { Stack, App, Duration } from 'aws-cdk-lib';
import * as cdk from 'aws-cdk-lib';

// Service-specific imports
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import { Function, Runtime } from 'aws-cdk-lib/aws-lambda';

// Constructs library
import { Construct } from 'constructs';
```

**Deprecated import patterns to detect**:
```typescript
// DEPRECATED: CDK v1 style imports (should not exist in v2 projects)
import * as cdk from '@aws-cdk/core';           // ❌ CDK v1
import * as lambda from '@aws-cdk/aws-lambda';  // ❌ CDK v1

// DEPRECATED: Old construct import
import { Construct } from '@aws-cdk/core';      // ❌ Should be from 'constructs'
```

#### Identifying Construct Usage

Scan for construct instantiation patterns:

```typescript
// Pattern: new ConstructName(scope, id, props)
new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.NODEJS_18_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
});

// Pattern: ConstructName.fromXxx() static methods
const bucket = s3.Bucket.fromBucketName(this, 'ImportedBucket', 'my-bucket');
```

#### Deprecated Construct Detection Patterns

Use these regex patterns to identify potentially deprecated constructs:

**Pattern 1: Deprecated Runtime Values**
```regex
Runtime\.(NODEJS_14_X|NODEJS_16_X|PYTHON_3_8|PYTHON_3_9|DOTNET_CORE_3_1|JAVA_8|JAVA_11)
```

**Pattern 2: Deprecated Property Names**
```regex
\bsecurityGroup\b(?!s)  # Old singular form, should be securityGroups
\bvpcSubnets\b          # Check for deprecated subnet selection
```

**Pattern 3: Deprecated Class Names**
```regex
\bBucketEncryption\.Kms\b     # May need KMS_MANAGED or S3_MANAGED
\bInstanceType\.of\b          # Check for deprecated instance types
```

**Pattern 4: Deprecated Method Calls**
```regex
\.addDependency\(              # Check if using deprecated dependency methods
\.addPropertyOverride\(        # May need migration to newer patterns
```

### Construct Detection Output Format

After scanning, report findings in this format:

```
=== Deprecated Construct Scan: <project_name> ===

Scanned files: 12
Files with deprecated constructs: 3

Findings:

File: lib/my-stack.ts
  Line 15: Runtime.NODEJS_16_X
    → Deprecated: Node.js 16 runtime is deprecated
    → Replacement: Runtime.NODEJS_20_X
  
  Line 42: BucketEncryption.KMS
    → Deprecated: Use BucketEncryption.KMS_MANAGED
    → Replacement: BucketEncryption.KMS_MANAGED

File: lib/api-stack.ts
  Line 28: securityGroup (singular)
    → Deprecated: Use securityGroups (plural) array
    → Replacement: securityGroups: [securityGroup]

File: lib/constructs/custom-lambda.ts
  Line 8: extends cdk.Construct
    → Deprecated: Import Construct from 'constructs' package
    → Replacement: import { Construct } from 'constructs'

Total deprecated constructs found: 4
  - Auto-migratable: 3
  - Manual review required: 1
```

### Automated Scanning Script

Here's a comprehensive scanning approach:

```bash
#!/bin/bash
# scan-deprecated-constructs.sh

PROJECT_PATH="$1"

echo "=== Scanning for Deprecated Constructs ==="
echo "Project: $PROJECT_PATH"
echo ""

# Find all TypeScript files
TS_FILES=$(find "$PROJECT_PATH" -name "*.ts" -type f \
  -not -path "*/node_modules/*" \
  -not -path "*/cdk.out/*" \
  -not -path "*/.git/*")

# Deprecated patterns to search
PATTERNS=(
  "Runtime\.NODEJS_14_X"
  "Runtime\.NODEJS_16_X"
  "Runtime\.PYTHON_3_8"
  "Runtime\.PYTHON_3_9"
  "@aws-cdk/core"
  "@aws-cdk/aws-"
  "extends cdk\.Construct"
  "BucketEncryption\.Kms[^_]"
)

for file in $TS_FILES; do
  for pattern in "${PATTERNS[@]}"; do
    grep -n -E "$pattern" "$file"
  done
done
```

---

## Common Deprecated Constructs Mapping

### Overview

This section provides a comprehensive mapping of deprecated CDK constructs to their modern replacements. Each entry includes the deprecated construct, its replacement, and any parameter changes required.

### Lambda Runtime Deprecations

Lambda runtimes are frequently deprecated as AWS ends support for older versions.

| Deprecated Runtime | Replacement | Notes |
|-------------------|-------------|-------|
| `Runtime.NODEJS_14_X` | `Runtime.NODEJS_20_X` | Node.js 14 EOL |
| `Runtime.NODEJS_16_X` | `Runtime.NODEJS_20_X` | Node.js 16 EOL |
| `Runtime.NODEJS_18_X` | `Runtime.NODEJS_20_X` | Still supported, but 20 is latest LTS |
| `Runtime.PYTHON_3_8` | `Runtime.PYTHON_3_12` | Python 3.8 EOL |
| `Runtime.PYTHON_3_9` | `Runtime.PYTHON_3_12` | Python 3.9 approaching EOL |
| `Runtime.DOTNET_CORE_3_1` | `Runtime.DOTNET_8` | .NET Core 3.1 EOL |
| `Runtime.JAVA_8` | `Runtime.JAVA_21` | Java 8 (non-AL2) deprecated |
| `Runtime.JAVA_11` | `Runtime.JAVA_21` | Java 11 approaching EOL |

**Migration example**:
```typescript
// Before (deprecated)
new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.NODEJS_16_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
});

// After (updated)
new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
});
```

### S3 Bucket Encryption Changes

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `BucketEncryption.Kms` | `BucketEncryption.KMS_MANAGED` | Clearer naming |
| `BucketEncryption.S3Managed` | `BucketEncryption.S3_MANAGED` | Consistent naming |
| `BucketEncryption.Unencrypted` | `BucketEncryption.UNENCRYPTED` | Consistent naming |

**Migration example**:
```typescript
// Before (deprecated)
new s3.Bucket(this, 'MyBucket', {
  encryption: s3.BucketEncryption.Kms,
});

// After (updated)
new s3.Bucket(this, 'MyBucket', {
  encryption: s3.BucketEncryption.KMS_MANAGED,
});
```

### EC2 Instance Type Changes

| Deprecated Pattern | Replacement | Notes |
|-------------------|-------------|-------|
| `InstanceType.of(InstanceClass.T2, InstanceSize.MICRO)` | `InstanceType.of(InstanceClass.T3, InstanceSize.MICRO)` | T2 is older generation |
| `InstanceType.of(InstanceClass.M4, ...)` | `InstanceType.of(InstanceClass.M6I, ...)` | M4 is older generation |
| `InstanceType.of(InstanceClass.C4, ...)` | `InstanceType.of(InstanceClass.C6I, ...)` | C4 is older generation |

**Note**: Instance type changes are recommendations, not strict deprecations. Older instance types still work but may not be available in all regions.

### VPC and Networking Changes

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `SubnetType.ISOLATED` | `SubnetType.PRIVATE_ISOLATED` | Clearer naming |
| `SubnetType.PRIVATE` | `SubnetType.PRIVATE_WITH_EGRESS` | Explicit egress indication |
| `securityGroup` (singular prop) | `securityGroups` (array) | Consistent with multiple SGs |

**Migration example**:
```typescript
// Before (deprecated)
new ec2.Instance(this, 'MyInstance', {
  vpc,
  securityGroup: mySecurityGroup,  // Singular
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE },
});

// After (updated)
new ec2.Instance(this, 'MyInstance', {
  vpc,
  securityGroups: [mySecurityGroup],  // Array
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
});
```

### API Gateway Changes

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `RestApi` default endpoint | Explicit `endpointConfiguration` | Must specify endpoint type |
| `LambdaIntegration` without proxy | `LambdaIntegration` with explicit `proxy` | Explicit proxy setting |

**Migration example**:
```typescript
// Before (deprecated - implicit defaults)
new apigateway.RestApi(this, 'MyApi');

// After (updated - explicit configuration)
new apigateway.RestApi(this, 'MyApi', {
  endpointConfiguration: {
    types: [apigateway.EndpointType.REGIONAL],
  },
});
```

### DynamoDB Changes

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `BillingMode.Provisioned` without capacity | Explicit `readCapacity`/`writeCapacity` | Must specify capacity |
| `Table` without removal policy | Explicit `removalPolicy` | Best practice |

**Migration example**:
```typescript
// Before (deprecated - missing explicit settings)
new dynamodb.Table(this, 'MyTable', {
  partitionKey: { name: 'id', type: dynamodb.AttributeType.STRING },
});

// After (updated - explicit settings)
new dynamodb.Table(this, 'MyTable', {
  partitionKey: { name: 'id', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  removalPolicy: cdk.RemovalPolicy.RETAIN,
});
```

### Construct Base Class Changes

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `import { Construct } from '@aws-cdk/core'` | `import { Construct } from 'constructs'` | Separate constructs package |
| `extends cdk.Construct` | `extends Construct` | Direct import from constructs |
| `cdk.Construct` | `Construct` | Simplified reference |

**Migration example**:
```typescript
// Before (deprecated)
import * as cdk from 'aws-cdk-lib';

export class MyConstruct extends cdk.Construct {
  constructor(scope: cdk.Construct, id: string) {
    super(scope, id);
  }
}

// After (updated)
import { Construct } from 'constructs';

export class MyConstruct extends Construct {
  constructor(scope: Construct, id: string) {
    super(scope, id);
  }
}
```

### CloudWatch Alarm Changes

| Deprecated | Replacement | Notes |
|------------|-------------|-------|
| `Alarm` with `period` as number | `period: Duration.minutes(X)` | Use Duration class |
| `evaluationPeriods` implicit | Explicit `evaluationPeriods` | Best practice |

**Migration example**:
```typescript
// Before (deprecated)
new cloudwatch.Alarm(this, 'MyAlarm', {
  metric: myMetric,
  threshold: 100,
  period: 300,  // Number in seconds
});

// After (updated)
new cloudwatch.Alarm(this, 'MyAlarm', {
  metric: myMetric,
  threshold: 100,
  evaluationPeriods: 1,
  // period is now on the metric itself
});
```

### Parameter Change Requirements

Some deprecated constructs require parameter changes beyond simple renaming:

#### Lambda Function Props Changes

```typescript
// Before: Separate timeout and memorySize
{
  timeout: 30,           // Number (seconds)
  memorySize: 256,       // Number (MB)
}

// After: Use Duration class for timeout
{
  timeout: Duration.seconds(30),  // Duration object
  memorySize: 256,                // Still number
}
```

#### S3 Bucket Lifecycle Rules

```typescript
// Before: expiration as number
{
  lifecycleRules: [{
    expiration: 30,  // Number (days)
  }],
}

// After: Use Duration class
{
  lifecycleRules: [{
    expiration: Duration.days(30),  // Duration object
  }],
}
```

### Deprecated Constructs Database

The CDK Upgrader maintains a mapping database of deprecated constructs. This database is updated with each CDK release.

**Database structure**:
```json
{
  "deprecations": [
    {
      "pattern": "Runtime\\.NODEJS_16_X",
      "replacement": "Runtime.NODEJS_20_X",
      "category": "lambda-runtime",
      "autoMigrate": true,
      "cdkVersion": "2.150.0",
      "notes": "Node.js 16 reached EOL"
    },
    {
      "pattern": "BucketEncryption\\.Kms(?!_)",
      "replacement": "BucketEncryption.KMS_MANAGED",
      "category": "s3-encryption",
      "autoMigrate": true,
      "cdkVersion": "2.100.0",
      "notes": "Naming consistency update"
    }
  ]
}
```

---

## Custom Construct Handling

### Overview

Custom constructs are user-defined classes that extend CDK base classes. During upgrades, these constructs need special handling to update CDK-related imports and inheritance while preserving all custom business logic.

### Identifying Custom Constructs

Custom constructs typically follow these patterns:

```typescript
// Pattern 1: Extends Construct directly
export class MyCustomConstruct extends Construct {
  // Custom logic
}

// Pattern 2: Extends a CDK L2 construct
export class MyCustomBucket extends s3.Bucket {
  // Custom logic
}

// Pattern 3: Extends Stack
export class MyCustomStack extends Stack {
  // Custom logic
}
```

**Detection criteria**:
1. Class definition with `extends` keyword
2. Extends a CDK class (`Construct`, `Stack`, or any `aws-cdk-lib` class)
3. Located in user's source files (not in `node_modules`)

### Updating CDK Base Class Imports

#### Step 1: Identify Import Statements

Scan for CDK-related imports at the top of the file:

```typescript
// Current imports to check
import { Construct } from '@aws-cdk/core';           // ❌ Deprecated
import { Construct } from 'aws-cdk-lib';             // ❌ Deprecated location
import * as cdk from 'aws-cdk-lib';                  // ✓ OK, but check usage
import { Stack, StackProps } from 'aws-cdk-lib';     // ✓ OK
import { Construct } from 'constructs';              // ✓ Correct
```

#### Step 2: Update Construct Import

**Before**:
```typescript
import * as cdk from 'aws-cdk-lib';
// or
import { Construct } from '@aws-cdk/core';
```

**After**:
```typescript
import { Construct } from 'constructs';
import * as cdk from 'aws-cdk-lib';
```

#### Step 3: Verify Import Changes

After updating imports, verify:
1. `Construct` is imported from `'constructs'` package
2. Other CDK classes remain imported from `'aws-cdk-lib'`
3. No duplicate imports exist

### Preserving Custom Logic

**Critical Rule**: Never modify custom business logic within constructs.

#### What to Preserve

- Custom properties and methods
- Business logic implementation
- Custom validation code
- Resource configurations specific to the project
- Comments and documentation

#### What to Update

- Import statements for CDK classes
- Base class references (e.g., `cdk.Construct` → `Construct`)
- Deprecated CDK method calls
- Deprecated property names

**Example - Preserving custom logic**:

```typescript
// Before
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';

export class MyCustomLambda extends cdk.Construct {
  public readonly function: lambda.Function;
  
  constructor(scope: cdk.Construct, id: string, props: MyCustomLambdaProps) {
    super(scope, id);
    
    // PRESERVE: All custom logic below
    this.function = new lambda.Function(this, 'Function', {
      runtime: lambda.Runtime.NODEJS_16_X,  // UPDATE: Runtime only
      handler: props.handler,                // PRESERVE: Custom prop
      code: lambda.Code.fromAsset(props.codePath),  // PRESERVE: Custom prop
      environment: this.buildEnvironment(props),     // PRESERVE: Custom method
      timeout: cdk.Duration.seconds(props.timeout),  // PRESERVE: Custom prop
    });
    
    // PRESERVE: Custom post-processing
    this.addCustomPermissions(props.permissions);
  }
  
  // PRESERVE: All custom methods
  private buildEnvironment(props: MyCustomLambdaProps): Record<string, string> {
    return {
      STAGE: props.stage,
      ...props.additionalEnv,
    };
  }
  
  private addCustomPermissions(permissions: Permission[]): void {
    // Custom permission logic
  }
}

// After
import { Construct } from 'constructs';  // UPDATED: Import source
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';

export class MyCustomLambda extends Construct {  // UPDATED: Base class reference
  public readonly function: lambda.Function;
  
  constructor(scope: Construct, id: string, props: MyCustomLambdaProps) {  // UPDATED: Scope type
    super(scope, id);
    
    // PRESERVED: All custom logic
    this.function = new lambda.Function(this, 'Function', {
      runtime: lambda.Runtime.NODEJS_20_X,  // UPDATED: Runtime version
      handler: props.handler,
      code: lambda.Code.fromAsset(props.codePath),
      environment: this.buildEnvironment(props),
      timeout: cdk.Duration.seconds(props.timeout),
    });
    
    this.addCustomPermissions(props.permissions);
  }
  
  // PRESERVED: All custom methods unchanged
  private buildEnvironment(props: MyCustomLambdaProps): Record<string, string> {
    return {
      STAGE: props.stage,
      ...props.additionalEnv,
    };
  }
  
  private addCustomPermissions(permissions: Permission[]): void {
    // Custom permission logic
  }
}
```

### Updating Inheritance Chain for Deprecated Base Classes

When a custom construct extends a deprecated CDK class, update the inheritance:

#### Scenario 1: Extends cdk.Construct

```typescript
// Before
export class MyConstruct extends cdk.Construct {

// After
export class MyConstruct extends Construct {
```

#### Scenario 2: Extends Deprecated L2 Construct

If the base L2 construct has been deprecated and replaced:

```typescript
// Before (hypothetical deprecated construct)
import { DeprecatedFunction } from 'aws-cdk-lib/aws-lambda';

export class MyFunction extends DeprecatedFunction {

// After
import { Function } from 'aws-cdk-lib/aws-lambda';

export class MyFunction extends Function {
```

**Note**: L2 construct deprecations are rare. Most deprecations are at the property/method level, not the class level.

#### Scenario 3: Multiple Inheritance Levels

For constructs with multiple levels of custom inheritance:

```typescript
// Custom base construct
export class BaseConstruct extends Construct {
  // Base functionality
}

// Extended custom construct
export class ExtendedConstruct extends BaseConstruct {
  // Extended functionality
}
```

**Update strategy**:
1. Update the root custom construct (`BaseConstruct`) first
2. Child constructs (`ExtendedConstruct`) inherit the fix automatically
3. Verify all levels compile correctly

### Custom Construct Update Output

```
=== Custom Construct Updates: <project_name> ===

File: lib/constructs/my-custom-lambda.ts
  Class: MyCustomLambda
  
  Updates applied:
    ✓ Import: Added 'import { Construct } from "constructs"'
    ✓ Base class: cdk.Construct → Construct
    ✓ Constructor param: cdk.Construct → Construct
    ✓ Runtime: NODEJS_16_X → NODEJS_20_X
  
  Preserved (not modified):
    - Custom properties: function, config
    - Custom methods: buildEnvironment, addCustomPermissions
    - Business logic: 45 lines unchanged

File: lib/constructs/custom-api.ts
  Class: CustomApiGateway
  
  Updates applied:
    ✓ Import: Added 'import { Construct } from "constructs"'
    ✓ Base class: Construct (already correct)
  
  Preserved (not modified):
    - Custom properties: api, authorizer
    - Custom methods: addRoutes, configureAuth
    - Business logic: 120 lines unchanged

Total custom constructs processed: 2
  - Successfully updated: 2
  - Failed: 0
```

### Handling Update Failures

If a custom construct cannot be automatically updated:

```
=== Custom Construct Update Failed ===

File: lib/constructs/complex-construct.ts
Class: ComplexConstruct

✗ Unable to automatically update this construct.

Reason: Complex inheritance pattern detected
  - Extends multiple CDK classes through mixins
  - Contains dynamic class generation

Manual migration required:

1. Update import statement:
   - Add: import { Construct } from 'constructs';

2. Update class definition:
   - Change: extends cdk.Construct
   - To: extends Construct

3. Update constructor parameter types:
   - Change: scope: cdk.Construct
   - To: scope: Construct

4. Review and test all custom logic after changes.

The file has NOT been modified. Please apply changes manually.
```

---

## Third-Party Library Handling

### Overview

CDK projects often use third-party construct libraries that build on top of `aws-cdk-lib`. These libraries must be compatible with the target CDK version. This section covers how to identify, upgrade, and handle conflicts with third-party CDK libraries.

### Identifying Third-Party CDK Libraries

#### Common Third-Party Library Patterns

Third-party CDK libraries typically follow these naming patterns:

| Pattern | Examples | Description |
|---------|----------|-------------|
| `@aws-solutions-constructs/*` | `@aws-solutions-constructs/aws-lambda-dynamodb` | AWS Solutions Constructs |
| `@cdklabs/*` | `@cdklabs/cdk-validator-cfnguard` | CDK Labs experimental |
| `cdk-*` | `cdk-nag`, `cdk-pipelines-github` | Community constructs |
| `@*-cdk/*` | `@datadog-cdk/datadog-cdk-constructs` | Vendor-specific constructs |

#### Detecting Third-Party Libraries in package.json

Scan `package.json` for third-party CDK dependencies:

```bash
# Extract dependencies that look like CDK libraries
cat package.json | jq '.dependencies + .devDependencies | 
  to_entries | 
  map(select(.key | test("cdk|constructs|@aws-solutions"))) | 
  from_entries'
```

**Example output**:
```json
{
  "aws-cdk-lib": "^2.150.0",
  "constructs": "^10.0.0",
  "@aws-solutions-constructs/aws-lambda-dynamodb": "^2.50.0",
  "cdk-nag": "^2.28.0"
}
```

### Upgrading to Latest Compatible Versions

#### Step 1: Query Latest Versions

For each third-party library, query npm for the latest version:

```bash
npm view @aws-solutions-constructs/aws-lambda-dynamodb version
```

#### Step 2: Check Peer Dependencies

Third-party CDK libraries declare `aws-cdk-lib` as a peer dependency. Check compatibility:

```bash
npm view @aws-solutions-constructs/aws-lambda-dynamodb peerDependencies --json
```

**Example output**:
```json
{
  "aws-cdk-lib": "^2.150.0",
  "constructs": "^10.0.0"
}
```

#### Step 3: Determine Compatibility

Compare the library's peer dependency range with the target CDK version:

| Library Peer Dep | Target CDK | Compatible? |
|------------------|------------|-------------|
| `^2.150.0` | `2.175.3` | ✓ Yes (2.175.3 satisfies ^2.150.0) |
| `>=2.100.0 <2.160.0` | `2.175.3` | ✗ No (2.175.3 > 2.160.0) |
| `^2.170.0` | `2.175.3` | ✓ Yes |

#### Step 4: Update Compatible Libraries

For compatible libraries, update to the latest version:

```json
// Before
{
  "@aws-solutions-constructs/aws-lambda-dynamodb": "^2.50.0"
}

// After
{
  "@aws-solutions-constructs/aws-lambda-dynamodb": "^2.55.0"
}
```

### Version Conflict Detection and Reporting

#### Conflict Scenarios

**Scenario 1: Library requires older CDK version**
```
Library: some-cdk-construct@1.5.0
Peer dependency: aws-cdk-lib@">=2.50.0 <2.150.0"
Target CDK: 2.175.3

Conflict: Target CDK version exceeds library's maximum supported version.
```

**Scenario 2: Library requires newer CDK version**
```
Library: new-cdk-construct@3.0.0
Peer dependency: aws-cdk-lib@"^2.180.0"
Target CDK: 2.175.3

Conflict: Target CDK version is below library's minimum required version.
```

**Scenario 3: Multiple libraries with conflicting requirements**
```
Library A: requires aws-cdk-lib@"^2.150.0"
Library B: requires aws-cdk-lib@">=2.100.0 <2.160.0"
Target CDK: 2.175.3

Conflict: Library B is incompatible with target CDK version.
```

#### Conflict Reporting Format

```
=== Third-Party Library Compatibility Check ===

Target CDK Version: aws-cdk-lib@2.175.3

Checking 3 third-party libraries...

✓ @aws-solutions-constructs/aws-lambda-dynamodb
  Current: ^2.50.0
  Latest: ^2.55.0
  Peer dependency: aws-cdk-lib@"^2.150.0"
  Status: Compatible
  Action: Update to ^2.55.0

✓ cdk-nag
  Current: ^2.28.0
  Latest: ^2.32.0
  Peer dependency: aws-cdk-lib@"^2.100.0"
  Status: Compatible
  Action: Update to ^2.32.0

✗ legacy-cdk-construct
  Current: ^1.5.0
  Latest: ^1.5.0 (no newer version)
  Peer dependency: aws-cdk-lib@">=2.50.0 <2.150.0"
  Status: INCOMPATIBLE
  Action: Cannot update - library does not support CDK 2.175.x

Summary:
  Compatible: 2
  Incompatible: 1
```

### Handling Incompatible Libraries

When a third-party library is incompatible with the target CDK version:

#### Option 1: Check for Newer Library Version

```
=== Incompatible Library: legacy-cdk-construct ===

The current version (1.5.0) is incompatible with CDK 2.175.3.

Checking for newer versions...

Available versions:
  1.5.0 - aws-cdk-lib@">=2.50.0 <2.150.0" ✗
  1.4.0 - aws-cdk-lib@">=2.50.0 <2.140.0" ✗
  1.3.0 - aws-cdk-lib@">=2.50.0 <2.130.0" ✗

No compatible version found.
```

#### Option 2: Suggest Alternatives

```
=== Suggested Alternatives ===

The library 'legacy-cdk-construct' appears to be unmaintained.

Consider these alternatives:
1. @aws-solutions-constructs/aws-lambda-sqs - Similar functionality, actively maintained
2. Build custom construct - Implement the specific functionality you need
3. Fork and update - Fork the library and update CDK dependencies

For more information:
- AWS Solutions Constructs: https://docs.aws.amazon.com/solutions/latest/constructs/
- CDK Patterns: https://cdkpatterns.com/
```

#### Option 3: Keep Current Version (With Warning)

```
=== Library Version Unchanged ===

Library: legacy-cdk-construct
Version: ^1.5.0 (unchanged)

⚠ WARNING: This library may cause runtime errors with CDK 2.175.3

The library will NOT be updated because no compatible version exists.

Potential issues:
- Peer dependency warnings during npm install
- Runtime errors if library uses deprecated CDK APIs
- Synthesis failures if construct definitions are incompatible

Recommendations:
1. Test thoroughly after upgrade
2. Monitor for library updates
3. Consider replacing with alternative solution
```

### Third-Party Library Update Output

```
=== Third-Party Library Updates: <project_name> ===

Libraries processed: 3

Updated:
  ✓ @aws-solutions-constructs/aws-lambda-dynamodb: ^2.50.0 → ^2.55.0
  ✓ cdk-nag: ^2.28.0 → ^2.32.0

Unchanged (incompatible):
  ⚠ legacy-cdk-construct: ^1.5.0 (no compatible version)

package.json changes:
  dependencies:
    @aws-solutions-constructs/aws-lambda-dynamodb: "^2.55.0"
  devDependencies:
    cdk-nag: "^2.32.0"

Note: Run 'npm install' to apply dependency changes.
```

### Source Code Preservation

**Critical Rule**: Never modify third-party library source code.

Third-party libraries are installed in `node_modules` and should never be modified:
- Changes would be lost on `npm install`
- Modifications could break library functionality
- Updates would overwrite any changes

If a third-party library has issues:
1. Report the issue to the library maintainer
2. Fork the library if urgent fixes are needed
3. Consider alternative libraries
4. Implement custom constructs as a workaround

---

## Migration Guidance Format

### Overview

When deprecated constructs cannot be automatically migrated, the CDK Upgrader provides detailed migration guidance. This section defines the format for migration guidance to ensure consistency and clarity.

### Migration Guidance Structure

Each migration guidance entry follows this structure:

```
╔══════════════════════════════════════════════════════════════════╗
║                    MANUAL MIGRATION REQUIRED                     ║
╠══════════════════════════════════════════════════════════════════╣
║ Construct: [Construct Name]                                      ║
║ File: [File Path]                                                ║
║ Line: [Line Number]                                              ║
╠══════════════════════════════════════════════════════════════════╣
║ Issue: [Description of the deprecation]                          ║
║                                                                  ║
║ Current Code:                                                    ║
║   [Code snippet showing current usage]                           ║
║                                                                  ║
║ Recommended Fix:                                                 ║
║   [Code snippet showing recommended replacement]                 ║
║                                                                  ║
║ Migration Steps:                                                 ║
║   1. [Step 1]                                                    ║
║   2. [Step 2]                                                    ║
║   3. [Step 3]                                                    ║
║                                                                  ║
║ Additional Notes:                                                ║
║   [Any additional context or warnings]                           ║
║                                                                  ║
║ Documentation:                                                   ║
║   [Link to relevant CDK documentation]                           ║
╚══════════════════════════════════════════════════════════════════╝
```

### Example Migration Guidance Entries

#### Example 1: Complex Lambda Configuration

```
╔══════════════════════════════════════════════════════════════════╗
║                    MANUAL MIGRATION REQUIRED                     ║
╠══════════════════════════════════════════════════════════════════╣
║ Construct: Lambda Function with VPC Configuration                ║
║ File: lib/api-stack.ts                                           ║
║ Line: 45-62                                                      ║
╠══════════════════════════════════════════════════════════════════╣
║ Issue: Lambda VPC configuration uses deprecated subnet selection ║
║        pattern that requires manual review.                      ║
║                                                                  ║
║ Current Code:                                                    ║
║   new lambda.Function(this, 'ApiHandler', {                      ║
║     runtime: lambda.Runtime.NODEJS_16_X,                         ║
║     vpc: vpc,                                                    ║
║     vpcSubnets: {                                                ║
║       subnetType: ec2.SubnetType.PRIVATE,                        ║
║       availabilityZones: ['us-east-1a', 'us-east-1b'],           ║
║     },                                                           ║
║     securityGroup: lambdaSg,                                     ║
║   });                                                            ║
║                                                                  ║
║ Recommended Fix:                                                 ║
║   new lambda.Function(this, 'ApiHandler', {                      ║
║     runtime: lambda.Runtime.NODEJS_20_X,                         ║
║     vpc: vpc,                                                    ║
║     vpcSubnets: {                                                ║
║       subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,            ║
║       availabilityZones: ['us-east-1a', 'us-east-1b'],           ║
║     },                                                           ║
║     securityGroups: [lambdaSg],                                  ║
║   });                                                            ║
║                                                                  ║
║ Migration Steps:                                                 ║
║   1. Update runtime from NODEJS_16_X to NODEJS_20_X              ║
║   2. Change SubnetType.PRIVATE to SubnetType.PRIVATE_WITH_EGRESS ║
║   3. Change securityGroup (singular) to securityGroups (array)   ║
║   4. Verify Lambda code is compatible with Node.js 20            ║
║   5. Test Lambda function after deployment                       ║
║                                                                  ║
║ Additional Notes:                                                ║
║   - Node.js 20 has breaking changes from Node.js 16              ║
║   - Review Lambda code for deprecated Node.js APIs               ║
║   - Consider testing in a non-production environment first       ║
║                                                                  ║
║ Documentation:                                                   ║
║   https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda.Function.html
╚══════════════════════════════════════════════════════════════════╝
```

#### Example 2: Custom Construct with Complex Inheritance

```
╔══════════════════════════════════════════════════════════════════╗
║                    MANUAL MIGRATION REQUIRED                     ║
╠══════════════════════════════════════════════════════════════════╣
║ Construct: CustomDatabaseCluster                                 ║
║ File: lib/constructs/database-cluster.ts                         ║
║ Line: 1-150                                                      ║
╠══════════════════════════════════════════════════════════════════╣
║ Issue: Custom construct uses mixin pattern with deprecated       ║
║        CDK base classes that cannot be automatically migrated.   ║
║                                                                  ║
║ Current Code:                                                    ║
║   import * as cdk from 'aws-cdk-lib';                            ║
║                                                                  ║
║   function Taggable<T extends cdk.Construct>(Base: T) {          ║
║     return class extends Base {                                  ║
║       // Mixin implementation                                    ║
║     };                                                           ║
║   }                                                              ║
║                                                                  ║
║   export class CustomDatabaseCluster extends                     ║
║     Taggable(cdk.Construct) {                                    ║
║     // Implementation                                            ║
║   }                                                              ║
║                                                                  ║
║ Recommended Fix:                                                 ║
║   import { Construct } from 'constructs';                        ║
║   import * as cdk from 'aws-cdk-lib';                            ║
║                                                                  ║
║   function Taggable<T extends new (...args: any[]) => Construct> ║
║     (Base: T) {                                                  ║
║     return class extends Base {                                  ║
║       // Mixin implementation                                    ║
║     };                                                           ║
║   }                                                              ║
║                                                                  ║
║   export class CustomDatabaseCluster extends                     ║
║     Taggable(Construct) {                                        ║
║     // Implementation                                            ║
║   }                                                              ║
║                                                                  ║
║ Migration Steps:                                                 ║
║   1. Add import for Construct from 'constructs' package          ║
║   2. Update mixin type constraint to use Construct               ║
║   3. Update base class reference in mixin call                   ║
║   4. Update any constructor parameter types                      ║
║   5. Verify TypeScript compilation succeeds                      ║
║   6. Run unit tests to verify functionality                      ║
║                                                                  ║
║ Additional Notes:                                                ║
║   - Mixin patterns require careful type handling                 ║
║   - The Construct class is now in a separate 'constructs' package║
║   - TypeScript generics may need adjustment                      ║
║                                                                  ║
║ Documentation:                                                   ║
║   https://docs.aws.amazon.com/cdk/v2/guide/constructs.html       ║
╚══════════════════════════════════════════════════════════════════╝
```

#### Example 3: Deprecated API Gateway Configuration

```
╔══════════════════════════════════════════════════════════════════╗
║                    MANUAL MIGRATION REQUIRED                     ║
╠══════════════════════════════════════════════════════════════════╣
║ Construct: REST API with Custom Authorizer                       ║
║ File: lib/api-gateway-stack.ts                                   ║
║ Line: 78-95                                                      ║
╠══════════════════════════════════════════════════════════════════╣
║ Issue: API Gateway authorizer uses deprecated configuration      ║
║        pattern that requires structural changes.                 ║
║                                                                  ║
║ Current Code:                                                    ║
║   const authorizer = new apigateway.RequestAuthorizer(           ║
║     this, 'Authorizer', {                                        ║
║       handler: authorizerFn,                                     ║
║       identitySource: 'method.request.header.Authorization',     ║
║     }                                                            ║
║   );                                                             ║
║                                                                  ║
║ Recommended Fix:                                                 ║
║   const authorizer = new apigateway.RequestAuthorizer(           ║
║     this, 'Authorizer', {                                        ║
║       handler: authorizerFn,                                     ║
║       identitySources: [                                         ║
║         apigateway.IdentitySource.header('Authorization'),       ║
║       ],                                                         ║
║     }                                                            ║
║   );                                                             ║
║                                                                  ║
║ Migration Steps:                                                 ║
║   1. Change identitySource (string) to identitySources (array)   ║
║   2. Use IdentitySource helper class for type safety             ║
║   3. Update any references to the authorizer configuration       ║
║   4. Test API Gateway authorization flow                         ║
║                                                                  ║
║ Additional Notes:                                                ║
║   - The new pattern supports multiple identity sources           ║
║   - IdentitySource class provides better type checking           ║
║   - Existing deployed authorizers will continue to work          ║
║                                                                  ║
║ Documentation:                                                   ║
║   https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_apigateway.RequestAuthorizer.html
╚══════════════════════════════════════════════════════════════════╝
```

### Migration Guidance Summary Report

After processing all files, provide a summary of migration guidance:

```
=== Migration Guidance Summary ===

Total constructs requiring manual migration: 5

By Category:
  Lambda Runtime Updates: 2
  VPC Configuration Changes: 1
  Custom Construct Patterns: 1
  API Gateway Configuration: 1

By Complexity:
  Simple (< 5 minutes): 3
  Moderate (5-15 minutes): 1
  Complex (> 15 minutes): 1

Files Affected:
  lib/api-stack.ts - 2 items
  lib/constructs/database-cluster.ts - 1 item
  lib/api-gateway-stack.ts - 1 item
  lib/lambda-stack.ts - 1 item

Recommended Order:
  1. Simple changes first (runtime updates)
  2. Configuration changes (VPC, API Gateway)
  3. Complex patterns last (custom constructs)

Detailed guidance has been saved to:
  .cdk-upgrade/migration-guidance.md

After completing manual migrations:
  1. Run 'npm run build' to verify TypeScript compilation
  2. Run 'npm test' to verify unit tests pass
  3. Run 'cdk synth' to verify CDK synthesis
  4. Deploy to a test environment before production
```

### Migration Guidance File Output

For complex projects, save detailed migration guidance to a file:

```markdown
# CDK Upgrade Migration Guidance

Generated: 2025-01-02T10:30:00Z
Project: my-cdk-project
Target CDK Version: 2.175.3

## Summary

- Total items requiring manual migration: 5
- Estimated time: 30-45 minutes

## Migration Items

### 1. Lambda Runtime Update (lib/api-stack.ts:45)

**Priority**: High
**Complexity**: Simple
**Estimated Time**: 5 minutes

[Detailed guidance...]

### 2. VPC Configuration (lib/api-stack.ts:52)

**Priority**: High
**Complexity**: Moderate
**Estimated Time**: 10 minutes

[Detailed guidance...]

[Additional items...]

## Post-Migration Checklist

- [ ] All TypeScript files compile without errors
- [ ] All unit tests pass
- [ ] CDK synth completes successfully
- [ ] CloudFormation diff reviewed
- [ ] Deployed to test environment
- [ ] Integration tests pass
- [ ] Ready for production deployment
```

---

## Quick Reference

### Auto-Migratable Constructs

These constructs can be automatically updated:

| Pattern | Replacement | Auto-Migrate |
|---------|-------------|--------------|
| `Runtime.NODEJS_16_X` | `Runtime.NODEJS_20_X` | ✓ |
| `Runtime.PYTHON_3_9` | `Runtime.PYTHON_3_12` | ✓ |
| `BucketEncryption.Kms` | `BucketEncryption.KMS_MANAGED` | ✓ |
| `SubnetType.PRIVATE` | `SubnetType.PRIVATE_WITH_EGRESS` | ✓ |
| `import { Construct } from '@aws-cdk/core'` | `import { Construct } from 'constructs'` | ✓ |

### Manual Migration Required

These patterns require manual review:

| Pattern | Reason |
|---------|--------|
| Complex mixin patterns | Type inference complexity |
| Dynamic construct generation | Cannot statically analyze |
| Custom L2 construct extensions | May have custom logic dependencies |
| Third-party library conflicts | Requires version compatibility check |
| VPC with custom subnet selection | May have environment-specific config |

### Common Migration Commands

```bash
# Verify TypeScript compilation
npm run build

# Run unit tests
npm test

# Synthesize CDK application
npx cdk synth

# Compare with deployed stack
npx cdk diff

# Deploy to test environment
npx cdk deploy --profile test
```
