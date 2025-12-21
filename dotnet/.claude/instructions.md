# Global Instructions

## Language Policy

All instructions and prompts in this repository must be written in English.

## Development Workflow

When working with C# code, follow these instructions carefully.

### Implementation Steps

1. **Consult relevant skill files** and list which ones guided the implementation (e.g. `Skills used: [clean-architecture.md, testing.md]`).

2. **Follow TDD** when possible. Start new changes by writing or updating test cases first.
   See [Testing Rules](./skills/testing.md) for details.

3. **Verify before committing**: Run `dotnet test` or `dotnet build` to ensure all tests pass.
   Use `dotnet watch test` for automatic test execution during development.

4. **Fix all warnings and errors** before proceeding.

### Path Placeholders

When you see paths like `/[project]/features/[feature]/` in rules, replace:

-   `[project]` with the project name (e.g. `Ordering`)
-   `[feature]` with the feature name (e.g. `VerifyOrAddPayment`)

## Code Formatting

CSharpier is enforced via MSBuild (`CSharpier.MSBuild`).

```bash
# Format all files
dotnet csharpier .

# Check only (CI)
dotnet csharpier . --check
```

-   IDE0055 (Roslyn formatting) is disabled in `.editorconfig`
-   CSharpier handles all formatting automatically
-   Do not manually format code
