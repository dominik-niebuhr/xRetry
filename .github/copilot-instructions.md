# xRetry Repository Instructions

## Overview
xRetry is a .NET library that provides retry functionality for flickering test cases in xUnit, Reqnroll, and SpecFlow. The repository consists of multiple NuGet packages that target different testing frameworks and versions.

**Project Type:** .NET Class Library (C#)  
**Target Framework:** .NET Standard 2.0, .NET 8.0+  
**Primary Language:** C#  
**Build System:** dotnet CLI with Makefile  
**Test Framework:** xUnit v2 and v3

## Repository Structure

### Key Directories
- **`src/`** - Source code for all packages
  - `src/xRetry/` - Core xRetry package for xUnit v2
  - `src/xRetry.v3/` - xRetry package for xUnit v3
  - `src/xRetry.SpecFlow/` - SpecFlow 3 integration
  - `src/xRetry.Reqnroll/` - Reqnroll 2 integration
- **`test/`** - All test projects
  - `test/UnitTests/` - Main xUnit v2 tests
  - `test/UnitTests.v3/` - Main xUnit v3 tests
  - `test/UnitTests.SingleThreaded/` - Single-threaded xUnit v2 tests (for deadlock scenarios)
  - `test/UnitTests.SingleThreaded.v3/` - Single-threaded xUnit v3 tests
  - `test/UnitTests.SpecFlow/` - SpecFlow integration tests
  - `test/UnitTests.Reqnroll/` - Reqnroll integration tests
- **`build/`** - Build scripts and Makefile
- **`docs/`** - Documentation source files (README is auto-generated from these)
- **`deploy/`** - Deployment scripts
- **`.github/workflows/`** - CI/CD pipeline definitions

### Key Files
- `build/Makefile` - Main build automation script
- `xRetry.sln` - Solution file containing all projects
- `.editorconfig` - Code formatting rules
- `Directory.Build.props` - MSBuild properties applied to all projects
- `README.md` - Auto-generated from `docs/` directory

## Build and Development

### Prerequisites
- .NET SDK 8.0 or higher (CI uses .NET 8.0 container, current local version is 10.0.102)
- `make` utility (optional but recommended)
- Docker (optional, for exact CI environment)

### Build Commands

**IMPORTANT:** Always work from the `build/` directory when using make commands.

#### Clean the repository
```bash
cd build
make clean
```
This removes all build artifacts, bin, and obj directories.

#### Lint check (code formatting)
```bash
cd build
make lint
```
**Critical:** This runs `dotnet format --verify-no-changes` and will FAIL the build if formatting is incorrect.

To automatically fix all formatting issues:
```bash
dotnet format
```
Always run from the repository root, not from the build directory.

#### Build all projects
```bash
cd build
make build
```
**IMPORTANT BUILD ORDER:** The build process has specific ordering requirements:
1. `xRetry.v3`, `xRetry.SpecFlow`, and `xRetry.Reqnroll` MUST be built with Release profile BEFORE test projects
2. This is because MSBuild needs the latest version when building test projects
3. The Makefile handles this automatically, but when building in an IDE, you must manually build these projects in Release mode first

#### Run all tests
```bash
cd build
make unit-tests-run
```
This runs all test suites in sequence.

To run individual test suites:
```bash
cd build
make unit-tests-run-main              # xUnit v2 main tests
make unit-tests-run-single-threaded   # xUnit v2 single-threaded tests (10s timeout)
make unit-tests-run-specflow          # SpecFlow tests
make unit-tests-run-reqnroll          # Reqnroll tests
make unit-tests-run-main-v3           # xUnit v3 main tests
make unit-tests-run-single-threaded-v3 # xUnit v3 single-threaded tests (10s timeout)
```

**Note about single-threaded tests:** These test deadlock scenarios and have a 10-second timeout. If they timeout (exit code 124), the tests have failed.

#### Generate documentation
```bash
cd docs
make all
```
The README.md is auto-generated from files in the `docs/` directory. After modifying docs, you MUST regenerate and commit the updated README.md.

#### Create NuGet packages
```bash
cd build
make nuget-create
```
Packages are created in `artefacts/nuget/`.

#### Complete CI pipeline
```bash
cd build
make ci
```
This runs: lint → build → test → docs → nuget-create

### Docker Build Environment
To build in the exact CI environment:
```bash
docker run --rm -it -v $(pwd):/src -w /src/build joshkeegan/dotnet-mixed-build:8.0
```
This mounts the xRetry source and gives you a terminal. Run `make ci` or other make commands from within the container.

## CI/CD Pipeline

### Workflows
- **`.github/workflows/cicd.yaml`** - Main CI/CD pipeline
  - Runs on push and pull requests
  - Steps: lint → build → unit tests (all 6 suites) → create packages
  - Uses container: `joshkeegan/dotnet-mixed-build:8.0`
  - CD step only runs on tags (format: `xRetry_v*`, `xRetry.SpecFlow_v*`, `xRetry.Reqnroll_v*`, `xRetry.v3_v*`)
  
- **`.github/workflows/test-report.yaml`** - Test result reporting

### Important CI Behaviors
1. Docs generation is checked separately - docs MUST be generated locally and checked in
2. All 6 test suites must pass
3. Lint must pass (no formatting issues)
4. Single-threaded tests have 10-second timeout

## Code Style and Conventions

### Formatting Rules
- **Enforced via:** `dotnet format` (checked in CI)
- **Configuration:** `.editorconfig`
- **Key rules:**
  - Indent style: 4 spaces for C#
  - End of line: LF
  - Charset: UTF-8
  - C# space after cast: true (Visual Studio default)

**Always run `dotnet format` from the repository root before committing.**

## Common Issues and Workarounds

### Build Issues
1. **SpecFlow/Reqnroll test build failures:** Ensure you build `src/xRetry.SpecFlow` and `src/xRetry.Reqnroll` in Release mode before building their test projects.

2. **Outdated assemblies:** Run `make clean` before building to ensure no stale artifacts.

### Test Issues
1. **Single-threaded tests timeout:** These tests verify deadlock scenarios. A timeout (exit code 124) means the test failed.

2. **Test discovery issues:** Ensure `xunit.runner.json` has `"diagnosticMessages": true` to see retry logs.

### Documentation Issues
1. **Modified README.md:** Never edit README.md directly. Edit files in `docs/` directory and run `cd docs && make all`.

2. **CI docs check fails:** You must regenerate docs locally and commit them. The CI verifies no files are modified after generation.

## Testing Strategy

### Test Organization
- **UnitTests / UnitTests.v3:** Main functionality tests (retry logic, attributes, etc.)
- **UnitTests.SingleThreaded / UnitTests.SingleThreaded.v3:** Deadlock and threading tests
- **UnitTests.SpecFlow:** SpecFlow 3 integration tests
- **UnitTests.Reqnroll:** Reqnroll 2 integration tests

### Running Tests
- Always build before testing (use `make build`)
- Tests should be run in Release configuration
- Test results are saved to `artefacts/testResults/*.trx`
- All tests must pass for CI to succeed

## Package Versioning
Versions are defined in `build/Makefile`:
- `XRETRY_VERSION` - xRetry (xUnit v2) package version
- `XRETRY_V3_VERSION` - xRetry.v3 (xUnit v3) package version
- `XRETRY_SPECFLOW_VERSION` - xRetry.SpecFlow package version
- `XRETRY_REQNOLL_VERSION` - xRetry.Reqnroll package version (note: variable name has typo, one 'L')

## Key Architectural Notes

### Strong Naming
All assemblies are strong-named using `build/keyPair.snk` (configured in `Directory.Build.props`).

### Multi-Version Support
The library supports both xUnit v2 and v3, which have different APIs. Changes to core retry logic may need to be replicated across both versions.

### Plugin Architecture
SpecFlow and Reqnroll packages work as test framework plugins that must be built before their test projects.

## Best Practices for Contributors

1. **Always lint before committing:** Run `dotnet format` from repository root
2. **Build order matters:** When working in IDE, build SpecFlow/Reqnroll packages in Release mode before test projects
3. **Test thoroughly:** Run full test suite with `make unit-tests-run`
4. **Update docs properly:** Edit files in `docs/` and regenerate README.md
5. **Clean between builds:** Use `make clean` to avoid stale artifacts
6. **Use Docker for validation:** Test in CI container if build issues occur locally
7. **Check single-threaded tests:** Ensure they complete within 10 seconds

## Quick Start for Common Tasks

### Make a code change
```bash
# 1. Clean and build
cd build
make clean
make build

# 2. Run tests
make unit-tests-run

# 3. Lint check
make lint
# If lint fails, fix formatting:
cd ..
dotnet format
cd build
make lint
```

### Update documentation
```bash
# 1. Edit files in docs/ directory
# 2. Regenerate README
cd docs
make all

# 3. Verify changes
cd ..
git diff README.md
```

### Create packages locally
```bash
cd build
make ci  # Runs everything: lint, build, test, docs, package
```

## Important Notes
- Never directly edit README.md - it's auto-generated
- Always verify docs are regenerated after editing docs/ files
- SpecFlow and Reqnroll packages must be built in Release before running their tests
- Single-threaded tests have a 10-second timeout by design
- Use the Docker container for exact CI environment matching
