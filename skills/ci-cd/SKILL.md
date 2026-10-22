---
name: ci-cd
description: Build, test, and package applications and libraries in a CI pipeline on Azure Pipelines or GitHub Actions. Use when creating, reviewing, or fixing azure-pipelines.yml or .github/workflows files; versioning builds from the build number; restoring from private NuGet feeds; running tests with code coverage; publishing an app to an artifact; building, scanning, and pushing a container image; packing and pushing NuGet packages; or sharing pipeline templates across repositories. Covers continuous integration only, not deployment to environments. The Dockerfile itself is covered by the docker skill.
---

# CI/CD

Aim for a pipeline that **builds once, tests what it built, and ships exactly that output**: one version for every output, one restore, one build, no rebuild in later steps, and nothing pushed unless every gate passed.

Platform-specific templates and task rules:
- Azure Pipelines: read [references/azure-pipelines.md](references/azure-pipelines.md) before writing or editing `azure-pipelines.yml`.
- GitHub Actions: read [references/github-actions.md](references/github-actions.md) before writing or editing a workflow in `.github/workflows/`.

## 1. Pipeline shape

Use **one pipeline per deliverable type**: an application pipeline for apps (APIs, workers) and a library pipeline for NuGet packages. Don't pack a library in an application pipeline: version properties passed on the command line apply to every project of the build, and apps and libraries follow different version rules (sections 2 and 7).

Where a library goes:

| The library is | Do |
| --- | --- |
| Used only by apps of the same repository | No package: `ProjectReference` from the apps |
| A client or contract package of one API | Same repository, separate library pipeline triggered by the library's folder |
| Shared by several services | Separate repository with its own library pipeline |

Run each pipeline in one job, on a hosted Linux agent, in this order.

**Application pipeline**

| # | Step | Command | Pull request builds |
| --- | --- | --- | --- |
| 1 | Install the SDK | From `global.json` | Yes |
| 2 | Restore | `dotnet restore <solution>` | Yes |
| 3 | Build | `dotnet build <solution> -c Release --no-restore -p:ContinuousIntegrationBuild=true <version properties>` | Yes |
| 4 | Test | Section 4 | Yes |
| 5 | Publish | `dotnet publish <entry project> -c Release --no-build -o <staging>/<project name> <version properties>` | Yes |
| 6 | Upload the artifact | Publish folder (section 5) | Yes |
| 7 | Build and scan the image | Section 6 | Yes |
| 8 | Push the image | Section 6 | **No** |

**Library pipeline**

| # | Step | Command | Pull request builds |
| --- | --- | --- | --- |
| 1 to 4 | SDK, restore, build, test | As in the application pipeline, with the library version properties (section 7). In a repository shared with apps, use the library's test project instead of the solution | Yes |
| 5 | Pack | `dotnet pack <library project> -c Release --no-build -o <staging>/packages <version properties>` | Yes |
| 6 | Upload the packages | Packages folder | Yes |
| 7 | Push the packages | Section 7 | **No** |

- Push only from builds of the default branch or a release tag (section 7), never from pull request builds.
- Keep `--no-restore` and `--no-build` on every step after the one that does the work: a later step that rebuilds produces outputs that were not tested.
- Use `-c Release` on every step: `--no-build` looks for the output of the same configuration.
- Restore and build the solution file, not a `**/*.csproj` glob: the solution defines what belongs to the build and restores in one pass.
- Exception: a library pipeline in a repository shared with apps restores, builds, and tests the library's test project. It builds the library through its `ProjectReference`, and the apps are neither built with the library's version nor able to block its release.
- Publish and pack only the projects that ship, by path. Don't publish the solution with `-o`: the outputs of all projects, test projects included, are copied into the same folder.
- Work that doesn't depend on the build (for example packaging infrastructure files) can run as a separate job in parallel.

## 2. Version

Use one version per run for every output of the pipeline: assemblies and artifact plus image tag (application pipeline), or assemblies and packages (library pipeline).

Requirements for the build number:
- Four numeric parts, **each below 65534**: `AssemblyVersion` rejects bigger values, so `yyyyMMdd` can't be one part.
- Unique per run and increasing over time.

Examples: `0.<yyyy>.<Mdd>.<run of the day>` (`0.2026.1016.3`) on Azure Pipelines, `0.<yyyy>.<Mdd>.<run number>` on GitHub Actions. If the repository already uses a Git-based versioning tool, keep it instead.

- For apps, pass the build number as properties to every `dotnet` command that produces output:

  ```bash
  -p:Version=$BUILD_NUMBER -p:AssemblyVersion=$BUILD_NUMBER -p:FileVersion=$BUILD_NUMBER
  ```

  Command-line properties override values set in project files, so all projects get the same version.
- For libraries, follow section 7.
- Don't commit version bumps from the pipeline: the version comes from the run, not from the repository.

## 3. SDK, restore, and build

- Install the SDK from `global.json`. Don't write an SDK version in the pipeline: `global.json` keeps the local and CI versions the same. If the repository has no `global.json`, propose adding one.
- Restore with the repository's `nuget.config`. It lists the feeds; it never contains credentials.
- Authenticate to private feeds with the platform's mechanism (see the reference file), using the pipeline's identity or a secret. Don't put tokens in `nuget.config` or in the command line of a logged step.
- When the projects have `packages.lock.json` files, restore with `--locked-mode`: the restore fails instead of resolving different packages than the developer machine. Recommended: lock files for apps only, since a library's lock file doesn't apply to the projects that consume it. The SDK adds implicit package references, so with locked mode set `"rollForward": "disable"` in `global.json`.
- Build with `-p:ContinuousIntegrationBuild=true`: it turns on the settings meant for official builds (deterministic paths, Source Link). Don't set it for local builds.
- Fail on vulnerable packages of high and critical severity, in apps and libraries. NuGet audits the packages during restore and reports `NU1901` to `NU1904` (low to critical); turn the last two into errors in `Directory.Build.props`, so local builds and CI behave the same:

  ```xml
  <PropertyGroup>
    <WarningsAsErrors>$(WarningsAsErrors);NU1903;NU1904</WarningsAsErrors>
  </PropertyGroup>
  ```

  - Below .NET 10, also set `<NuGetAuditMode>all</NuGetAuditMode>`: the audit otherwise checks direct packages only.
  - The audit needs a package source with vulnerability data: nuget.org, or an `<auditSources>` entry in `nuget.config` when the feeds are private only.
  - Fix a finding by updating the package (`dotnet nuget why <package>` shows which dependency brings a transitive one). Suppress an advisory with `<NuGetAuditSuppress Include="<advisory URL>" />` only as a last resort, with a comment giving the reason.
  - Optional: run a restore of the default branch on a schedule, to catch advisories published after the last build.

## 4. Tests and coverage

- Run the tests with the test runner the repository uses, with `--no-build`, and fail the pipeline when a test fails:

  | Runner | Selected by | Command |
  | --- | --- | --- |
  | VSTest | Default | `dotnet test <solution> -c Release --no-build` |
  | Microsoft.Testing.Platform | `"test": { "runner": "Microsoft.Testing.Platform" }` in `global.json` | `dotnet test --solution <solution> -c Release --no-build` |

- Publish test results to the platform so failures show in the run summary, also when the test step fails.
- Coverage with VSTest: `--collect:"XPlat Code Coverage"`, which needs the `coverlet.collector` package in each test project. It writes Cobertura by default. Add other formats only for a tool that reads them:

  ```bash
  dotnet test <solution> -c Release --no-build --collect:"XPlat Code Coverage" \
    -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=cobertura,opencover
  ```

- Coverage with Microsoft.Testing.Platform: keep the coverage extension the test projects already reference. `--collect` is a VSTest option.
- Static analysis, when the project uses a tool for it: prepare the analysis before the build and analyse after the tests, so it sees the build and the coverage. Fail the pipeline on the tool's quality gate: a gate that doesn't fail is ignored.

## 5. Application artifact

- Publish framework-dependent and portable (no `-r`), as the docker skill expects. Use `-o <staging>/<project name>` so each app gets its own folder.
- Exception: a self-contained or Native AOT app (`runtime-deps` image in the docker skill) needs a runtime identifier that matches the image's distribution (`linux-x64` for chiseled and full Ubuntu, `linux-musl-x64` for Alpine). Set `<RuntimeIdentifier>` in its project file, so restore, build, and publish all use it; don't pass `-r` to `publish` alone, or `--no-build` finds no matching build output.
- Upload the publish folder as the pipeline artifact. Zip it only when the deployment target needs a zip.
- Upload infrastructure files used by the deployment (for example Bicep) as a separate artifact of the same run, so a deployment uses matching code and infrastructure.

## 6. Container image

Build the image from the publish folder, as described in the docker skill. Keep the Dockerfile rules there; this section covers what the pipeline does.

```bash
docker build --pull -f <path to Dockerfile> -t <registry>/<image name>:$BUILD_NUMBER <staging>/<project name>
```

- Build and scan the image in every build, pull requests included: a broken Dockerfile, build argument, or build context fails before the merge. Push only from the default branch or a release.
- Keep the image name and the DLL name as pipeline variables next to the project, not computed in shared templates.
- Tag the image with the build number. Add moving tags (for example `latest`) only if a consumer needs them, and never deploy from them.
- For the parameterized Dockerfile, derive `DOTNET_VERSION` from the entry project's target framework (`net10.0` → `10.0`) instead of a separate variable that can drift.
- Scan the image after the build and **before the push**. Fail the pipeline on critical findings **that have a fix available**: findings without a fix would block every build with nothing to do.
- Record accepted findings in the scanner's ignore file, each with a justification and an expiry date.
- An image that passes today can have a critical finding next month: scan the images in use on a schedule, outside this pipeline.
- Push with the registry credentials from a service connection or a federated identity, not a stored password.

## 7. NuGet packages (libraries)

| Build | Package version | Example |
| --- | --- | --- |
| Default branch, every run | Prerelease: `<major>.<minor>.<patch>-ci.<yyyyMMdd>.<run>` | `2.1.0-ci.20261016.3` |
| Release tag `v<major>.<minor>.<patch>` | Stable: `<major>.<minor>.<patch>` | `2.1.0` |

- Keep `<major>.<minor>.<patch>` in one place in the repository (for example `<VersionPrefix>` in `Directory.Build.props`) and change it by pull request.
- Release by pushing a tag `v<major>.<minor>.<patch>` on a commit of the default branch: the release is traceable in Git. The pipeline fails when the tag doesn't match `<VersionPrefix>`.
- **Bump `<major>.<minor>.<patch>` right after a release**: a prerelease ranks below the stable version with the same number (`2.1.0-ci.*` < `2.1.0`), so CI packages built after the release would look older than it.
- Version properties for a library:
  - `Version`: the package version from the table.
  - `AssemblyVersion`: `<major>.0.0.0`. Changing it only with the major version reduces binding conflicts for consumers.
  - `FileVersion`: `<major>.<minor>.<patch>.<run>`, numbers only. It is informational only: when the run counter restarts every day (Azure Pipelines), it isn't increasing, which is acceptable.
- Don't put a prerelease suffix in `AssemblyVersion` or `FileVersion`: they accept numbers only.
- Apps that ship reference stable package versions only. Prerelease packages are for testing before a release.
- A stable version can be pushed once only: a release build for a version that already exists must fail, not overwrite.
- Push `*.nupkg` and exclude `*.symbols.nupkg` when the feed doesn't take symbol packages.

## 8. Reuse across repositories

- Put shared steps in a templates repository and reference it from each service pipeline (Azure Pipelines `resources.repositories` + `template`, GitHub reusable workflows or composite actions).
- Reference the templates by **major version**, not by the default branch: compatible fixes reach every pipeline, breaking changes go to a new major version that each pipeline adopts by pull request. See the reference files for the syntax per platform.
- Pin third-party actions and templates to a full commit SHA: a tag can be moved to other code.
- Pass project-specific values (project path, DLL name, image name) as template parameters. Templates contain no project names.

## 9. Triggers and security

- Trigger on the default branch and on pull requests to it; library pipelines also on release tags.
- Add path filters only when a repository holds several independently built deliverables (services, or a library next to an app). Include every path that changes the output: the project folder, its test folder, and the shared build files (`Directory.Build.props`, `Directory.Packages.props`, `global.json`, `nuget.config`).
- Keep secrets in the platform's secret store or a key vault. Don't echo them, and don't pass them to steps that don't need them.
- Give the pipeline the least permissions it needs (read for source and feeds, write only where it pushes).

## 10. Anti-patterns

| Avoid | Do instead |
| --- | --- |
| SDK version written in the pipeline | SDK from `global.json` |
| Version only in project files, or different versions per output | One version per run, passed as properties |
| A version part of 65534 or more (`yyyyMMdd`) | Split the date (`yyyy.Mdd`) |
| Packing a library in an application pipeline | Separate library pipeline, or a `ProjectReference` when no other repository consumes it |
| Library pipeline building the whole shared solution | Build and test the library's test project |
| Path filter on the library folder only | Add its tests and the shared build files |
| Vulnerable packages left as warnings | `NU1903;NU1904` in `WarningsAsErrors` |
| Stable release from a manual parameter | Release tag checked against `<VersionPrefix>` |
| Prerelease package referenced by an app that ships | Stable version |
| Restoring or building `**/*.csproj` | Restore and build the solution |
| Lock files committed without `--locked-mode` in CI | `dotnet restore --locked-mode` |
| `publish` or `pack` without `--no-build` | Build once, then `--no-build` |
| Publishing the whole solution with `-o` | Publish entry projects by path |
| `dotnet test <solution>` in Microsoft.Testing.Platform mode | `dotnet test --solution <solution>` |
| Credentials in `nuget.config` | Platform feed authentication |
| Image built only after the merge | Build and scan in pull requests, push from the default branch |
| Image tagged only `latest` | Build number tag |
| Pushing an image before scanning it | Build, scan, then push |
| Failing on critical findings without a fix | Fail on fixable critical findings, track the others |
| Images or packages pushed from pull request builds | Push from the default branch or a release only |
| Library `AssemblyVersion` changing on every minor or patch | `<major>.0.0.0` |
| Prerelease suffix in `AssemblyVersion` or `FileVersion` | Numeric version only |
| Same `<major>.<minor>.<patch>` kept after a release | Bump it right after the release |
| Shared templates taken from the default branch | Major version reference |
| Third-party actions or templates by tag | Full commit SHA |
| Project names inside shared templates | Template parameters |
| Committing version bumps from the pipeline | Version from the run |

## Review checklist

- [ ] Apps and packages built by separate pipelines; libraries used only in the repository referenced with `ProjectReference`.
- [ ] SDK installed from `global.json`.
- [ ] One restore (`--locked-mode` with lock files) and one build of the solution with `ContinuousIntegrationBuild=true`; later steps use `--no-restore` / `--no-build` and `-c Release`.
- [ ] One version per run, each part below 65534, passed to every producing command.
- [ ] Private feeds authenticated by the platform; no credentials in `nuget.config`.
- [ ] High and critical vulnerable packages fail the build (`NU1903;NU1904`).
- [ ] Tests run with the repository's test runner, fail the pipeline, results and coverage published.
- [ ] Only entry projects published, portable (or with the project's `RuntimeIdentifier` for self-contained and Native AOT apps), one folder per app, uploaded as an artifact.
- [ ] Image built from the publish folder with `--pull` in every build, tagged with the build number, scanned before push, failing on fixable critical findings.
- [ ] Packages: prerelease from CI, stable from a release tag matching `<VersionPrefix>`, `AssemblyVersion` at `<major>.0.0.0`, version bumped after each release.
- [ ] Shipping apps reference stable packages only.
- [ ] Path filters include tests and shared build files.
- [ ] Nothing pushed from pull request builds.
- [ ] Shared templates referenced by major version, third-party code by commit SHA, project values as parameters.
