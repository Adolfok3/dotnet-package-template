# dotnet-package-template
An awesome structured template for .NET packages.

## Overview

This repository is a `dotnet new` template for bootstrapping .NET class library packages that are ready to build, test, statically analyze, and publish to NuGet from day one. It ships with a pre-wired solution layout, centrally managed package versions, a curated set of Roslyn analyzers, a test project with common testing libraries, and GitHub Actions workflows for CI and release.

## Getting started

Install the template from source and scaffold a new package:

```bash
dotnet new install .
dotnet new dotnetpkg -n YourPackage.Name -o ../YourPackage.Name
```

Available template options:

| Option | Description | Default |
|---|---|---|
| `-n`, `--name` | Name used for the solution, projects, and namespaces | *(required)* |
| `-P`, `--PackageDescription` | One-line description written to `README.md` | `An awesome structured template for .NET packages.` |
| `-p`, `--packageAuthor` | Author name written to `LICENSE` | `Luiz Adolfo` |

## Project structure

```
.
├── .github/workflows/       # CI and release pipelines
├── assets/                  # Package icon and other static assets
├── src/
│   ├── Directory.Build.props
│   └── DotnetPackageTemplate/
│       └── DotnetPackageTemplate.csproj
├── tests/
│   ├── Directory.Build.props
│   └── DotnetPackageTemplate.Tests/
│       └── DotnetPackageTemplate.Tests.csproj
├── Directory.Build.props    # Solution-wide build settings
├── Directory.Packages.props # Centrally managed package versions
├── DotnetPackageTemplate.slnx
├── .editorconfig
└── LICENSE
```

### `assets/`
Static, non-code files shipped alongside the repository — currently the package icon (`icon.png`). Wire it into the library's `.csproj` (`PackageIcon` + a packed `None` include) once you're ready to publish.

### `src/`
Contains the library project(s) that get packed and published to NuGet. Every project here has `IsPackable` and `GenerateDocumentationFile` enabled, and automatically grants `InternalsVisibleTo` to its matching test project and to `DynamicProxyGenAssembly2` (needed by NSubstitute to proxy `internal` types).

### `tests/`
Contains the corresponding test project(s), one per `src` project, following the `<ProjectName>.Tests` naming convention. Test projects are non-packable (`IsPackable = false`) and reference the matching `src` project automatically by convention (`src/<ProjectName>/<ProjectName>.csproj`) — a new test project only needs to follow the naming pattern to be wired up, no manual `ProjectReference` needed.

### Solution folders (`.slnx`)
The solution uses the newer [SLNX solution format](https://devblogs.microsoft.com/dotnet/introducing-slnx-support-dotnet-cli/) and groups content into virtual folders for a cleaner Solution Explorer / Rider view:

| Folder | Contents |
|---|---|
| `1 - Solution Items` | Root-level configuration files (`.editorconfig`, `.gitignore`, `Directory.Build.props`, `Directory.Packages.props`, `README.md`) |
| `1 - Solution Items/src` | `src/Directory.Build.props` |
| `1 - Solution Items/tests` | `tests/Directory.Build.props` |
| `2 - src` | Library project(s) |
| `3 - tests` | Test project(s) |

## Configuration files

| File | Purpose |
|---|---|
| `Directory.Build.props` (root) | Applies to every project in the solution: targets `net11.0` with `LangVersion=preview`, enables nullable reference types and implicit usings, sets the latest recommended analysis level, treats warnings as errors, enforces code style during build, and turns on NuGet audit (`moderate` level, across all dependency types). |
| `Directory.Packages.props` | [Central Package Management](https://learn.microsoft.com/nuget/consume-packages/central-package-management) — every package version used across the solution is pinned once here, so individual `.csproj` files only reference package names, without versions. |
| `src/Directory.Build.props` | Imports the root props, marks `src/` projects as packable with XML doc generation enabled, and adds `InternalsVisibleTo` for the matching test project and for NSubstitute's dynamic proxy assembly. |
| `tests/Directory.Build.props` | Imports the root props, marks `tests/` projects as non-packable test projects, adds the test/mocking/assertion package references and their analyzers, wires the `ProjectReference` to the matching `src` project by naming convention, and adds global usings for `Xunit`, `NSubstitute`, and `FluentAssertions`. |
| `.editorconfig` | Repository-wide C# formatting and naming-convention rules (e.g. interfaces prefixed with `I`, PascalCase types/members), enforced at build time via `EnforceCodeStyleInBuild`. |
| `DotnetPackageTemplate.slnx` | Solution file (SLNX format) with the solution folder layout described above. |

## Analyzers

Enabled on every project via `Directory.Build.props` + `Directory.Packages.props`:

| Package | Purpose |
|---|---|
| `Microsoft.CodeAnalysis.NetAnalyzers` | Official .NET code quality/style analyzers (`AnalysisLevel=latest`, `AnalysisMode=Recommended`) |
| `Roslynator.Analyzers` | General-purpose Roslyn refactoring/analysis rules |
| `Roslynator.CodeAnalysis.Analyzers` | Analyzers for Roslyn-analyzer authors |
| `Roslynator.Formatting.Analyzers` | Formatting-focused analyzers |
| `StyleCop.Analyzers` | C# style/consistency rules |
| `Meziantou.Analyzer` | Additional best-practice/performance analyzers |
| `SonarAnalyzer.CSharp` | SonarSource code quality and security rules |

All analyzer packages are `PrivateAssets="all"` (dev-time only, never propagated to consumers of the package), and every warning fails the build (`TreatWarningsAsErrors=true`).

## Testing

Test projects (`tests/DotnetPackageTemplate.Tests`) use **xUnit** as the test framework, with:

| Package | Purpose |
|---|---|
| `Microsoft.NET.Test.Sdk` | Test host/runner infrastructure |
| `xunit` / `xunit.runner.visualstudio` | Test framework and Visual Studio test discovery |
| `coverlet.collector` / `coverlet.msbuild` | Code coverage collection (used by the CI workflow to produce Cobertura reports) |
| `NSubstitute` | Mocking library |
| `FluentAssertions` | Fluent assertion syntax |

Plus analyzers to keep the test code itself in good shape:

| Package | Purpose |
|---|---|
| `xunit.analyzers` | xUnit usage analyzers |
| `FluentAssertions.Analyzers` | FluentAssertions usage analyzers |
| `Meziantou.FluentAssertionsAnalyzers` | Additional FluentAssertions rules |
| `NSubstitute.Analyzers.CSharp` | NSubstitute usage analyzers |

`Xunit`, `NSubstitute`, and `FluentAssertions` are registered as global `Using`s, so test files don't need to repeat those imports.

## CI/CD

Two GitHub Actions workflows live under `.github/workflows/`, both on the `.NET 11` preview SDK (`dotnet-quality: preview`), matching the `TargetFramework` in `Directory.Build.props`.

### `main.yml` — CI
Runs on every push and pull request targeting `main`:
1. Restore, build, and run tests with Coverlet coverage collection (Cobertura + JSON, merged across test projects).
2. Upload the coverage report to [Codecov](https://about.codecov.io/) — requires a `CODECOV_TOKEN` repository secret.

### `release.yml` — Release
Runs on push of a `*.*.*` tag (e.g. `1.0.0`):
1. Reads the pushed tag as the release version.
2. Cleans, builds, tests, and packs the library in `Release` configuration, stamping the package with `/p:Version=<tag>`.
3. Pushes the resulting `.nupkg` to NuGet.org — requires a `NUGET_API_KEY` repository secret.

## Development recommendations

- **Cut a release by tagging**: bump the version by pushing a semver tag (`git tag 1.2.0 && git push --tags`) — the release workflow handles packing and publishing, there's no manual `dotnet pack`/`dotnet nuget push` step.
- **Add package metadata before your first release**: `src/DotnetPackageTemplate/DotnetPackageTemplate.csproj` currently has no NuGet metadata. Add `PackageId`, `Authors`, `Description`, `RepositoryUrl`, and `PackageLicenseExpression` (e.g. `MIT`), and wire up `assets/icon.png` via `PackageIcon` before publishing.
- **Keep warnings at zero**: `TreatWarningsAsErrors` is on — fix analyzer warnings rather than suppressing them; reach for `#pragma warning disable` with a justification comment only as a last resort.
- **Add every package version once**: new dependencies go in `Directory.Packages.props` (`PackageVersion`), then as an unversioned `PackageReference` in the consuming `.csproj`.
- **One test project per library project**: keep the `<ProjectName>` / `<ProjectName>.Tests` naming convention so `tests/Directory.Build.props`'s automatic `ProjectReference` and `InternalsVisibleTo` keep working without extra wiring.
- **Prefer `internal` + `InternalsVisibleTo` over reflection** when tests need access to implementation details — it's already configured for both the test project and NSubstitute's proxies.
- **Review `NuGetAudit` findings instead of silencing them**: `NuGetAuditLevel=moderate` surfaces known vulnerable dependencies during restore/build — update the dependency rather than lowering the audit level.
- **This repo doubles as a `dotnet new` template** (`.template.config/template.json`, short name `dotnetpkg`): when changing the folder/file layout, keep the `DotnetPackageTemplate` naming consistent so the template's `sourceName` substitution keeps working, and re-verify with `dotnet new install .` after structural changes.
