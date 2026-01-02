# CDK Upgrader - Kiro Power

Automatically upgrade TypeScript AWS CDK v2 projects to the latest stable version with intelligent merging, deprecated construct handling, and git integration.

## Installation

### Installing in Kiro

The CDK Upgrader is a Kiro Power that can be installed directly into your Kiro workspace.

#### Option 1: Install from Repository

1. Open Kiro and navigate to the Powers panel
2. Click "Add Power" or use the command palette (`Cmd+Shift+P` / `Ctrl+Shift+P`)
3. Select "Install Power from Repository"
4. Enter the repository URL: `https://github.com/frankhefeng/kiro-power-cdk-upgrader`
5. Click "Install"


#### Option 2: Clone and Install from Local Path


```bash
# Clone the repository
git clone https://github.com/frankhefeng/kiro-power-cdk-upgrader.git

# Open the folder in Kiro
# The power will be automatically detected from POWER.md
```

The power will be automatically detected from the POWER.md file and installed into your Kiro workspace.

### Prerequisites

Before using the CDK Upgrader, ensure your environment meets these requirements:

| Requirement | Minimum Version | Check Command |
|-------------|-----------------|---------------|
| Node.js | 18.x | `node --version` |
| npm | 9.x | `npm --version` |
| Git | Any recent version | `git --version` |

#### Quick Environment Check

```bash
node --version && npm --version && git --version
```

All three commands should succeed before using the upgrader.


## Quick Start

### 1. Navigate to Your CDK Workspace

Open Kiro in a directory containing one or more TypeScript CDK v2 projects.

### 2. Start the Upgrade

Ask Kiro to upgrade your projects:

> "Upgrade my CDK projects to the latest version"

### 3. Review and Confirm

The upgrader will:
1. **Discover** all CDK projects in your workspace
2. **Validate** your environment meets requirements
3. **Preview** planned changes for your review
4. **Apply** changes after your confirmation
5. **Validate** upgrades with `npm install` and `cdk synth`
6. **Commit** changes with semantic commit messages

### Example Workflow

```
You: Upgrade my CDK projects

Kiro: Found 2 CDK projects:
      - ./infrastructure (CDK v2.150.0)
      - ./services/api-stack (CDK v2.145.0)
      
      Environment check passed ✓
      
      Planned changes for ./infrastructure:
      - Update aws-cdk-lib: 2.150.0 → 2.175.3
      - Update aws-cdk: 2.150.0 → 2.1005.0
      - Replace tsconfig.json with latest template
      - Update Lambda runtime: nodejs18.x → nodejs20.x
      
      Proceed with upgrade? [Yes/No]

You: Yes

Kiro: ✓ ./infrastructure upgraded successfully
      ✓ ./services/api-stack upgraded successfully
      
      Created commits:
      - feat(cdk): upgrade CDK to version 2.175.3 (infrastructure)
      - feat(cdk): upgrade CDK to version 2.175.3 (api-stack)
```

### Preview Mode (Dry Run)

To see what changes would be made without modifying any files:

> "Preview CDK upgrade changes"

or

> "Dry-run CDK upgrade"


## Features

### Multi-Project Discovery
- Automatically scans directories to find all TypeScript CDK v2 projects
- Validates git tracking for each project
- Reports project locations and current CDK versions
- Allows selective project upgrades

### Smart Version Selection
- **5-step algorithm**:
  - Step 1: Select CLI version (use latest stable if > 7 days old, otherwise previous stable)
  - Step 2: Record CLI release date as compatibility constraint
  - Step 3: Select Lib version (use latest stable if > 7 days old, otherwise previous stable)
  - Step 4: MANDATORY compatibility check (CLI release date >= Lib release date)
  - Step 5: Final validation
- **Automatic backward search**: When incompatible versions detected, searches for earlier compatible version
- Example: When latest Lib 2.233.0 is incompatible with CLI 2.1100.1, automatically finds compatible 2.232.2
- Always uses latest CDK CLI version (2.1000.x+ series) that meets stability requirements
- Cross-platform date calculation for macOS and Linux compatibility

### Intelligent File Merging
- **tsconfig.json**: Complete replacement with latest template
- **jest.config.js**: Complete replacement with latest template
- **cdk.json**: Preserves ONLY "app" field, replaces all other fields with template
- **package.json**: Updates CDK dependencies, preserves user scripts and non-CDK packages
- **.gitignore**: Adds new CDK entries, preserves project-specific entries

### Deprecated Construct Handling
- Scans TypeScript files for deprecated CDK constructs
- Automatically replaces deprecated constructs with modern equivalents
- Updates parameters when construct signatures change
- Reports constructs requiring manual migration

### Lambda Runtime Upgrades
- Detects all Lambda function definitions
- Upgrades Node.js runtimes (e.g., nodejs18.x → nodejs20.x)
- Upgrades Python runtimes (e.g., python3.9 → python3.12)
- Supports Java and .NET runtime upgrades
- Handles both inline and external Lambda code

### Git Integration
- Verifies clean working directory before upgrades
- Requires feature branch (not main/master)
- Creates semantic commits: `feat(cdk): upgrade CDK to version X.Y.Z`
- Separate commits per project in multi-project workspaces
- Never auto-pushes to remote repositories

### Dry-Run Mode
- Preview all changes without modifying files
- Shows files to be modified or replaced
- Displays dependency version changes
- Highlights deprecated construct updates
- Lists Lambda runtime upgrades

### Post-Upgrade Validation
- Executes `npm install` to update package-lock.json
- Runs `cdk synth` to validate TypeScript compilation
- Reports specific errors with suggested fixes
- Provides detailed upgrade summary

### Error Handling
- Retry mechanisms with exponential backoff for network issues
- Clear error messages with suggested solutions
- Project isolation - one failure doesn't stop other upgrades
- Disk space and permission checks before operations


## Contributing

We welcome contributions to the CDK Upgrader! Here's how you can help:

### Reporting Issues

1. Check existing issues to avoid duplicates
2. Use the issue template when available
3. Include:
   - CDK version you're upgrading from/to
   - Node.js and npm versions
   - Error messages or unexpected behavior
   - Steps to reproduce

### Submitting Changes

1. **Fork the repository**
   ```bash
   git clone https://github.com/<your_name>/kiro-power-cdk-upgrader.git
   cd kiro-power-cdk-upgrader
   ```

2. **Create a feature branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```

3. **Make your changes**
   - Follow existing documentation style
   - Update relevant steering files
   - Test with real CDK projects

4. **Submit a pull request**
   - Describe what your changes do
   - Reference any related issues

### Areas for Contribution

- **Deprecated Construct Mappings**: Add new deprecated construct → replacement mappings in `steering/deprecated-constructs.md`
- **Lambda Runtime Updates**: Update runtime upgrade paths in `steering/lambda-runtimes.md`
- **Troubleshooting**: Add solutions for common issues in `steering/troubleshooting.md`
- **Documentation**: Improve clarity and add examples

### Code of Conduct

- Be respectful and inclusive
- Focus on constructive feedback
- Help others learn and grow

## Steering Files

| File | Description |
|------|-------------|
| `steering/upgrade-workflow.md` | Complete upgrade workflow instructions |
| `steering/version-strategy.md` | Version selection strategy and compatibility |
| `steering/file-processing.md` | File merge and replace rules |
| `steering/deprecated-constructs.md` | Deprecated construct mappings |
| `steering/lambda-runtimes.md` | Lambda runtime upgrade guide |
| `steering/troubleshooting.md` | Common issues and solutions |

## License

MIT License - see LICENSE file for details.

## Author

Feng He

## Links

- [Repository](https://github.com/frankhefeng/kiro-power-cdk-upgrader)
- [AWS CDK Documentation](https://docs.aws.amazon.com/cdk/latest/guide/)
- [Kiro Documentation](https://kiro.dev/docs)
