---
name: solution-setup
description: Set up the repository and solution of a new .NET codebase - folder layout (src, tests, deploy), the .slnx solution, global.json, Directory.Build.props, central package management (Directory.Packages.props), nuget.config with package source mapping, .editorconfig, and .gitignore. Use when creating a new repository or solution, adding a project to a solution, moving shared settings (TargetFramework, Nullable, ImplicitUsings, analyzers) out of project files, managing package versions centrally, configuring NuGet feeds, pinning the SDK, or fixing NU1008, NU1010, NU1100, or NU1101 restore errors. Doesn't cover how projects are layered (clean-architecture skill) or pipelines (ci-cd skill). Existing repositories keep their layout unless a restructuring is requested.
---

# Solution setup

Aim for a repository where **every shared setting lives in exactly one root file**: one SDK version, one target framework, one version per package, one list of trusted feeds. Project files only describe what is specific to the project, and a fresh clone builds the same way on every machine and in CI.

Existing repositories keep their layout and files. Don't move projects or rewrite configuration as a side effect of another change.

## 1. Repository layout

```text
MyApp/
  .editorconfig
  .gitignore
  global.json
  nuget.config
  Directory.Build.props
  Directory.Packages.props
  MyApp.slnx
  src/
    MyApp.Api/
    ...
  tests/
    MyApp.Api.Tests/
    ...
  deploy/
    Dockerfile
    (Kubernetes manifests, infrastructure as code)
```

- Put the shared files at the repository root: MSBuild, NuGet, and the SDK find them by walking up from each project.
- Production code in `src/`, tests in `tests/`, deployment files in `deploy/`: never inside a project folder, where they would be compiled or published.
- Pipeline files go where the platform reads them: `azure-pipelines.yml` at the root, GitHub workflows in `.github/workflows/` (ci-cd skill).

## 2. Naming

- Projects: `<Product>.<Part>` (`MyApp.Api`, `MyApp.Domain`). Tests: `<Project>.Tests` (`MyApp.Api.Tests`).
- The folder name, project file name, assembly name, and root namespace are the same. Don't set `AssemblyName` or `RootNamespace` to something else.
- The solution is named after the product: `MyApp.slnx`.

## 3. Create the files with the SDK templates

Generate the files instead of writing them by hand, then edit them as sections 4 to 8 describe:

```bash
dotnet new globaljson --sdk-version <installed SDK version> --roll-forward latestFeature
dotnet new gitignore
dotnet new editorconfig
dotnet new nugetconfig
dotnet new buildprops
dotnet new packagesprops
dotnet new sln --name MyApp
```

```bash
dotnet new <template> -o src/MyApp.Api --name MyApp.Api
dotnet sln MyApp.slnx add src/MyApp.Api/MyApp.Api.csproj
dotnet package add <PackageId> --project src/MyApp.Api/MyApp.Api.csproj
```

- `<installed SDK version>` is the full version shown by `dotnet --version` (for example `10.0.100`): `global.json` doesn't accept `10.0` or wildcards.
- `dotnet new sln` creates a `.slnx` solution from .NET 10. Keep an existing `.sln` unless a migration is requested (`dotnet sln migrate`).
- `dotnet sln add` creates solution folders from the paths (`src`, `tests`).
- Add packages with `dotnet package add`: with central package management it writes the version to `Directory.Packages.props` and a version-less reference to the project.
- After creating a project from a template, remove the properties now set centrally (section 5), and move any `Version` on a `<PackageReference>` to `Directory.Packages.props`: a version left in the project fails restore with NU1008.

## 4. global.json

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestFeature"
  }
}
```

- `version` is a placeholder: use the full version of the SDK the team installs. Without `rollForward`, the default is `patch`.
- `latestFeature` accepts newer patches and feature bands of the same major.minor version: developers and CI pick up SDK fixes without editing the file, and a new major version is never used by accident.
- Move to a new major SDK version by changing `version` in a pull request, together with `TargetFramework` (section 5).
- When the projects use NuGet lock files, set `"rollForward": "disable"`: the SDK adds implicit package references, and a different SDK breaks the locked restore (ci-cd skill).
- When the tests use Microsoft.Testing.Platform, add the runner here (from .NET 10): `"test": { "runner": "Microsoft.Testing.Platform" }`.

## 5. Directory.Build.props

Settings shared by every project:

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <WarningsAsErrors>$(WarningsAsErrors);NU1903;NU1904</WarningsAsErrors>
  </PropertyGroup>
</Project>
```

- `net10.0` is a placeholder: use the target framework of the solution. All projects move to a new version together.
- `AnalysisLevel` `latest-recommended` turns on the recommended .NET code analysis rules of the SDK in use.
- `NU1903;NU1904` make high and critical vulnerable packages fail the build, locally and in CI (ci-cd skill).
- Below .NET 10, also set `<NuGetAuditMode>all</NuGetAuditMode>`: the audit otherwise checks direct packages only, and a vulnerable transitive package doesn't fail the build.
- Library repositories also keep their `<VersionPrefix>` here (ci-cd skill).
- Remove these properties from the project files: `Directory.Build.props` is imported before the project body, so a value left in a `.csproj` silently overrides it and hides drift.
- A project that needs a different value (a multi-targeted library, a project-specific `NoWarn`) sets it in its own file, with a comment giving the reason.
- Project-specific settings stay in the project file: `UseAppHost` and `InvariantGlobalization` in the entry project (docker skill), `RestorePackagesWithLockFile` in apps that use lock files.

## 6. Central package management

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Microsoft.EntityFrameworkCore.SqlServer" Version="x.y.z" />
  </ItemGroup>
</Project>
```

```xml
<!-- In a project file -->
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" />
</ItemGroup>
```

- One `PackageVersion` per package for the whole repository: no project silently uses another version.
- Use exact versions, never floating ones (`1.*` is rejected with NU1011): every upgrade is a reviewed commit. Keep packages current with an automated update tool that opens pull requests.
- Use `VersionOverride` on a `PackageReference` only for a documented exception, never as the normal way to set a version.
- Don't enable transitive pinning (`CentralPackageTransitivePinningEnabled`) by default: in a library it adds the pinned packages to the published dependencies. Use a direct `PackageVersion` and `PackageReference` to fix a vulnerable transitive package.
  - Add a comment on the `PackageReference` naming the package that brings it (`dotnet nuget why <package>`) and the advisory. Without it, the reference looks unused and is removed, or stays after the parent package is fixed:

    ```xml
    <!-- Overrides the version brought by Microsoft.Data.SqlClient (<advisory URL>). Remove when SqlClient brings a fixed version. -->
    <PackageReference Include="System.Security.Cryptography.Pkcs" />
    ```
- Don't declare versions for packages the SDK references implicitly (NU1009).

## 7. nuget.config

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="MyFeed" value="https://<private feed>/v3/index.json" />
  </packageSources>
  <packageSourceMapping>
    <packageSource key="nuget.org">
      <package pattern="*" />
    </packageSource>
    <packageSource key="MyFeed">
      <package pattern="MyCompany.*" />
    </packageSource>
  </packageSourceMapping>
</configuration>
```

- `<clear />` removes the feeds of the user and machine configuration: every clone restores from the same feeds.
- Package source mapping decides which feed each package comes from, which prevents a public package from replacing a private one (dependency confusion). The most specific pattern wins: `MyCompany.Orders` comes from `MyFeed`, everything else from nuget.org.
- Once mapping exists, every package, transitive ones included, must match a pattern: a missing pattern fails restore (NU1100). Map private packages by a prefix the organization owns.
- Each `<packageSource key>` in the mapping must be the `key` of a feed in `<packageSources>`. `dotnet new nugetconfig` names nuget.org `nuget`: rename it to `nuget.org` as in the example, or use `nuget` in the mapping. A mismatch fails restore with NU1100 only for packages missing from the local cache, so it can pass locally and fail in CI.
- Without a private feed, keep only nuget.org with `<clear />` and the `*` mapping.
- When the repository restores from private feeds only, add an audit source for vulnerability data so the NU1903/NU1904 checks still work: `<auditSources><add key="nuget.org" value="https://api.nuget.org/v3/index.json" /></auditSources>`. If the network blocks `api.nuget.org`, use `https://data.nuget.org/v3/index.json` instead: it serves vulnerability data only, no packages.
- Never put credentials in `nuget.config`: CI and developers authenticate outside the file (ci-cd skill).
- Mapping applies to restore only: commands that query feeds (`dotnet package add`, `dotnet package list --outdated`) still send package names to every configured feed.

## 8. .editorconfig and .gitignore

- Keep the generated `.editorconfig` (`dotnet new editorconfig`): it holds the standard .NET formatting and naming rules. Adjust rules there, not with per-file suppressions.
- When test methods are named with underscores (`Cancel_twice_throws`), `AnalysisLevel` `latest-recommended` reports CA1707 on each of them. Turn the rule off for the tests only, at the end of the root `.editorconfig`, so production code keeps it:

  ```ini
  [tests/**.cs]
  dotnet_diagnostic.CA1707.severity = none
  ```
- Keep the generated `.gitignore` (`dotnet new gitignore`): it already ignores `bin/`, `obj/`, and `publish/` (docker skill).

## 9. Anti-patterns

| Avoid | Do instead |
| --- | --- |
| Shared files inside `src/` or a project folder | Repository root |
| `TargetFramework`, `Nullable`, `ImplicitUsings` repeated in each `.csproj` | `Directory.Build.props`, removed from project files |
| `Version` on `PackageReference` with central package management (NU1008) | `PackageVersion` in `Directory.Packages.props` |
| Floating versions (`1.*`) | Exact versions, updated by pull request |
| `VersionOverride` as the normal way to set versions | One central version, override only with a documented reason |
| No `global.json`, or `"version": "10.0"` | Full SDK version with `rollForward: latestFeature` |
| `nuget.config` without `<clear />` | `<clear />` and the repository's feeds |
| Public and private feeds without source mapping | `packageSourceMapping`, private packages by owned prefix |
| Credentials in `nuget.config` | Authentication outside the file |
| `AssemblyName` or `RootNamespace` different from the project name | Same name for folder, project, assembly, namespace |
| Hand-written config files | `dotnet new globaljson`, `buildprops`, `packagesprops`, `nugetconfig`, `editorconfig`, `gitignore`, `sln` |
| Restructuring an existing repository during another change | Keep the layout; restructure only on request |

## Review checklist

- [ ] Root has `global.json`, `nuget.config`, `Directory.Build.props`, `Directory.Packages.props`, `.editorconfig`, `.gitignore`, and the `.slnx` solution.
- [ ] `src/`, `tests/`, `deploy/`; projects named `<Product>.<Part>`, tests `<Project>.Tests`.
- [ ] `global.json` pins a full SDK version with `rollForward: latestFeature` (`disable` with lock files).
- [ ] `TargetFramework`, `Nullable`, `ImplicitUsings`, `AnalysisLevel`, and `NU1903;NU1904` only in `Directory.Build.props`.
- [ ] Central package management on; no `Version` in project files; exact versions only.
- [ ] `nuget.config` with `<clear />`, package source mapping for every feed, no credentials.
