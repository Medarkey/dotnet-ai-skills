# Azure Pipelines

Platform details for the rules in [SKILL.md](../SKILL.md). Read SKILL.md first: this file only shows how to apply it in `azure-pipelines.yml`.

## 1. Application pipeline

One job, steps in the order of SKILL.md section 1. Replace the values in `variables` and the service connection name.

```yaml
name: '0.$(Date:yyyy).$(Date:Mdd).$(Rev:r)'

trigger:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

variables:
  buildConfiguration: Release
  solution: MyApp.slnx
  entryProject: src/MyApp.Api/MyApp.Api.csproj
  appName: MyApp.Api
  dockerfile: deploy/Dockerfile
  registry: myregistry.azurecr.io
  imageName: myapp-api
  image: $(registry)/$(imageName):$(Build.BuildNumber)
  versionArgs: -p:Version=$(Build.BuildNumber) -p:AssemblyVersion=$(Build.BuildNumber) -p:FileVersion=$(Build.BuildNumber)
  isMain: $[eq(variables['Build.SourceBranch'], 'refs/heads/main')]

steps:
  - task: UseDotNet@2
    displayName: Install SDK from global.json
    inputs:
      packageType: sdk
      useGlobalJson: true

  - task: NuGetAuthenticate@1
    displayName: Authenticate NuGet feeds

  # Add --locked-mode when the projects have packages.lock.json files
  - script: dotnet restore $(solution)
    displayName: Restore

  - script: >
      dotnet build $(solution) -c $(buildConfiguration) --no-restore
      -p:ContinuousIntegrationBuild=true $(versionArgs)
    displayName: Build

  - script: >
      dotnet test $(solution) -c $(buildConfiguration) --no-build
      --logger trx --results-directory $(Agent.TempDirectory)/TestResults
      --collect:"XPlat Code Coverage"
    displayName: Test

  - task: PublishTestResults@2
    displayName: Publish test results
    condition: succeededOrFailed()
    inputs:
      testResultsFormat: VSTest
      testResultsFiles: '**/*.trx'
      searchFolder: $(Agent.TempDirectory)/TestResults

  - task: PublishCodeCoverageResults@2
    displayName: Publish code coverage
    condition: succeededOrFailed()
    inputs:
      summaryFileLocation: $(Agent.TempDirectory)/TestResults/**/coverage.cobertura.xml

  - script: >
      dotnet publish $(entryProject) -c $(buildConfiguration) --no-build
      -o $(Build.ArtifactStagingDirectory)/$(appName) $(versionArgs)
    displayName: Publish app

  - task: PublishPipelineArtifact@1
    displayName: Upload app artifact
    inputs:
      targetPath: $(Build.ArtifactStagingDirectory)/$(appName)
      artifact: $(appName)

  # Container image: build and scan in every build, push from the default branch only (SKILL.md section 6)
  - script: docker build --pull -f $(dockerfile) -t $(image) $(Build.ArtifactStagingDirectory)/$(appName)
    displayName: Build image

  - script: <scan command> $(image)
    displayName: Scan image

  - task: Docker@2
    displayName: Log in to registry
    condition: and(succeeded(), eq(variables.isMain, 'true'))
    inputs:
      command: login
      containerRegistry: my-registry-connection

  - script: docker push $(image)
    displayName: Push image
    condition: and(succeeded(), eq(variables.isMain, 'true'))
```

- `<scan command>` is the team's image scanner. It must exit with a non-zero code on critical findings that have a fix (for example `trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed`), and must be installed on the agent first if the image doesn't have it.
- `my-registry-connection` is a Docker registry service connection. Write its name literally: service connections are authorized when the run is queued, before runtime variables exist.
- `isMain` is false for pull request builds (their source branch is `refs/pull/...`), so the login and push steps don't run for them.
- `$(Rev:r)` restarts at 1 when the rest of the build number changes, so it counts the runs of the day and stays far below 65534.
- If the test projects use Microsoft.Testing.Platform (SKILL.md section 4), replace the test step with `dotnet test --solution $(solution) ...` and the options of the repository's MTP extensions (for example `--report-trx` for test results).

### Parameterized Dockerfile

When the image uses the parameterized Dockerfile of the docker skill, derive `DOTNET_VERSION` from the project before the image build:

```yaml
  - script: |
      tfm=$(dotnet msbuild $(entryProject) -getProperty:TargetFramework)
      echo "##vso[task.setvariable variable=dotnetVersion]${tfm#net}"
    displayName: Read .NET version

  - script: >
      docker build --pull -f $(dockerfile)
      --build-arg DOTNET_VERSION=$(dotnetVersion)
      --build-arg DLL_NAME=$(dllName)
      -t $(image) $(Build.ArtifactStagingDirectory)/$(appName)
    displayName: Build image
```

Add `dllName: MyApp.Api.dll` to `variables`. `-getProperty` returns nothing for a multi-targeted project (`TargetFrameworks`): pass the framework explicitly in that case.

## 2. Library pipeline (NuGet packages)

A separate pipeline file (for example `azure-pipelines.library.yml`), registered as its own pipeline. This example is for a library in a repository shared with an app: it builds the library's test project, and its path filter follows SKILL.md section 9. In a library-only repository, use the solution instead of `testProject` and remove `paths`.

```yaml
name: '0.$(Date:yyyy).$(Date:Mdd).$(Rev:r)'

trigger:
  branches:
    include:
      - main
  tags:
    include:
      - v*
  paths:
    include:
      - src/MyApp.Client
      - tests/MyApp.Client.Tests
      - Directory.Build.props
      - Directory.Packages.props
      - global.json
      - nuget.config

pool:
  vmImage: ubuntu-latest

variables:
  buildConfiguration: Release
  testProject: tests/MyApp.Client.Tests/MyApp.Client.Tests.csproj
  libraryProject: src/MyApp.Client/MyApp.Client.csproj
  feed: MyFeed # key of the feed in nuget.config
  isMain: $[eq(variables['Build.SourceBranch'], 'refs/heads/main')]
  isRelease: $[startsWith(variables['Build.SourceBranch'], 'refs/tags/v')]

steps:
  - task: UseDotNet@2
    displayName: Install SDK from global.json
    inputs:
      packageType: sdk
      useGlobalJson: true

  - task: NuGetAuthenticate@1
    displayName: Authenticate NuGet feeds

  - script: |
      set -e
      prefix=$(dotnet msbuild $(libraryProject) -getProperty:VersionPrefix)
      IFS=. read -r _ yyyy mdd rev <<< "$(Build.BuildNumber)"
      if [ "${IS_RELEASE,,}" = "true" ]; then
        if [ "$TAG" != "v$prefix" ]; then
          echo "##vso[task.logissue type=error]Tag $TAG doesn't match VersionPrefix $prefix"
          exit 1
        fi
        version="$prefix"
      else
        version="$prefix-ci.$(printf '%s%04d' "$yyyy" "$mdd").$rev"
      fi
      echo "##vso[task.setvariable variable=versionArgs]-p:Version=$version -p:AssemblyVersion=${prefix%%.*}.0.0.0 -p:FileVersion=$prefix.$rev"
    displayName: Compute package version
    env:
      IS_RELEASE: $(isRelease)
      TAG: $(Build.SourceBranchName)

  - script: dotnet restore $(testProject)
    displayName: Restore

  - script: >
      dotnet build $(testProject) -c $(buildConfiguration) --no-restore
      -p:ContinuousIntegrationBuild=true $(versionArgs)
    displayName: Build

  - script: >
      dotnet test $(testProject) -c $(buildConfiguration) --no-build
      --logger trx --results-directory $(Agent.TempDirectory)/TestResults
      --collect:"XPlat Code Coverage"
    displayName: Test

  # PublishTestResults@2 and PublishCodeCoverageResults@2 as in section 1

  - script: >
      dotnet pack $(libraryProject) -c $(buildConfiguration) --no-build
      -o $(Build.ArtifactStagingDirectory)/packages $(versionArgs)
    displayName: Pack

  - task: PublishPipelineArtifact@1
    displayName: Upload packages
    inputs:
      targetPath: $(Build.ArtifactStagingDirectory)/packages
      artifact: packages

  - script: >
      dotnet nuget push "$(Build.ArtifactStagingDirectory)/packages/*.nupkg"
      --source $(feed) --api-key az
    displayName: Push packages
    condition: and(succeeded(), or(eq(variables.isMain, 'true'), eq(variables.isRelease, 'true')))
```

- **Release:** push a tag `v<major>.<minor>.<patch>` on a commit of `main`. `Build.SourceBranchName` is the tag name (`v2.1.0`); the version step fails when it doesn't match `VersionPrefix`.
- A trigger with branch and tag filters fires when either one matches. The documentation doesn't describe how path filters apply to a tag push: check that the first release tag starts a run.
- On Azure Repos, give the pull request build validation policy of this pipeline the same path filter.
- Building the test project also builds the library through its `ProjectReference`, with the same version properties, so `dotnet pack --no-build` finds its output.
- `--api-key az`: `dotnet nuget push` requires a non-empty key; Azure Artifacts ignores it and uses the credentials from `NuGetAuthenticate@1`.
- The build service identity needs the **Feed Publisher (Contributor)** role on the feed.
- Azure Artifacts never accepts the same package version twice, even after a delete. A release build for an existing version fails, which is the expected behavior.
- After a release, bump `VersionPrefix` by pull request before the next CI package (SKILL.md section 7).
- `FileVersion` ends with the run of the day, so it restarts every day: it is informational only.
- Azure Artifacts doesn't support `.snupkg` symbol packages. Don't set `IncludeSymbols` for an Azure Artifacts feed; if symbols are needed, publish them with `PublishSymbols@2`.

## 3. Shared templates

```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: MyProject/pipeline-templates
      ref: refs/heads/release/v1

steps:
  - template: steps/dotnet-ci.yml@templates
    parameters:
      solution: MyApp.slnx
      entryProject: src/MyApp.Api/MyApp.Api.csproj
      appName: MyApp.Api
      imageName: myapp-api
```

Template file in the templates repository (`steps/dotnet-ci.yml`):

```yaml
parameters:
  - name: solution
    type: string
  - name: entryProject
    type: string
  - name: appName
    type: string
  - name: imageName
    type: string

steps:
  - script: dotnet restore ${{ parameters.solution }}
    displayName: Restore
  # ... the remaining steps of section 1, using ${{ parameters.<name> }}
```

- `ref` selects the major version: one branch per major version (`release/v1`, `release/v2`) in the templates repository. Compatible fixes are merged into the branch and reach every pipeline; a breaking change starts a new branch. Without `ref`, `refs/heads/main` is used and every change to the templates reaches every pipeline.
- To freeze a pipeline on an exact version, use a tag (`refs/tags/v1.2.0`) or a commit SHA instead.
- To prevent pipelines from skipping the shared steps, use an `extends` template and a **required template** check on the protected resources (service connections, environments). This is optional and set up by the project administrators.
- Declare a `type` for each parameter and give no default to the required ones: a missing value fails when the pipeline is compiled.
- Triggers can be set only in the main pipeline file, not in templates.
- A templates repository in another project needs read access for the pipeline's build service.

## 4. Triggers

- **Azure Repos:** a `pr:` section in YAML is ignored. Run pull request builds with a **build validation branch policy** on `main`.
- **GitHub or Bitbucket repositories:** use `pr:` with the target branches. Without a `pr:` section, pull requests to every branch trigger a build.
- Variables can't be used in `trigger` or `pr`: they are evaluated before any variable exists.
- Trigger settings in the pipeline UI ("Override the YAML trigger") take precedence over the YAML. Check them when a trigger doesn't fire.

## 5. Task rules

- Use `script` steps that call `dotnet` instead of `DotNetCoreCLI@2`: the commands are the same as on a developer machine, and the `restore` and `push` commands of the task no longer get new features.
- In an existing pipeline that uses `DotNetCoreCLI@2`, keep it unless a migration is requested, but check its defaults:
  - `publish`: `publishWebProjects` defaults to `true` and then **ignores `projects`**. Set `publishWebProjects: false` to publish the listed project.
  - `publish`: `zipAfterPublish` defaults to `true` and deletes the publish folder. Set it to `false` for a container image build.
  - `publish`: `modifyOutputPath` defaults to `true` and adds the project name to the output path.
  - `test`: `publishTestResults` defaults to `true` and adds `--logger trx --results-directory $(Agent.TempDirectory)` itself.
- Use `PublishPipelineArtifact@1`, not `PublishBuildArtifacts@1`.
- Use `PublishCodeCoverageResults@2`, not `@1`. Version 2 has no `codeCoverageTool` input: it detects the format.
- Use `NuGetAuthenticate@1` before `dotnet restore` and `dotnet nuget push`. Add `nuGetServiceConnections` for feeds outside the organization.
- Don't use `Docker@2` with `command: buildAndPush`: it ignores `arguments` (so no `--pull`) and pushes before a scan can run. Use `login` with `docker build`, scan, and `docker push` scripts.
- Secret variables are not passed to scripts automatically. Map them explicitly with `env:` on the step that needs them.

## 6. Anti-patterns

| Avoid | Do instead |
| --- | --- |
| `UseDotNet@2` with `version: 8.0.x` | `useGlobalJson: true` |
| `projects: '**/*.csproj'` for restore and build | The solution file |
| `DotNetCoreCLI@2` publish with `projects` and the default `publishWebProjects` | `publishWebProjects: false`, or a `dotnet publish` script |
| `PublishBuildArtifacts@1` | `PublishPipelineArtifact@1` |
| `PublishCodeCoverageResults@1` | `PublishCodeCoverageResults@2` |
| `Docker@2` `buildAndPush` | `login` + `docker build --pull` + scan + `docker push` |
| `pr:` section in an Azure Repos pipeline | Build validation branch policy |
| Templates repository without `ref` | `ref: refs/heads/release/v<major>` |
| Push steps without a branch condition | `condition: and(succeeded(), eq(variables.isMain, 'true'))` |
| Image build and scan conditioned on the default branch | Build and scan in every build, condition only login and push |
| `NuGetCommand@2` push with a feed GUID | `NuGetAuthenticate@1` + `dotnet nuget push --source <feed key>` |

## Review checklist

- [ ] `name:` gives a build number with each part below 65534; `$(Build.BuildNumber)` used for every version.
- [ ] `UseDotNet@2` with `useGlobalJson: true`; `NuGetAuthenticate@1` before restore.
- [ ] `script` steps with `--no-restore` / `--no-build`, or `DotNetCoreCLI@2` defaults checked; build with `ContinuousIntegrationBuild=true`.
- [ ] Test results and coverage published with `condition: succeededOrFailed()`.
- [ ] `PublishPipelineArtifact@1` and `PublishCodeCoverageResults@2`.
- [ ] Image built and scanned in every build; login and image push conditioned on the default branch.
- [ ] Library pipeline: test project built instead of the shared solution, paths include tests and shared build files, `v*` tag trigger.
- [ ] Library versions: stable only from a `v*` tag matching `VersionPrefix`, `AssemblyVersion` at `<major>.0.0.0`; packages pushed from `main` and release tags only.
- [ ] Templates repository referenced with `ref` on a major version branch; project values passed as typed parameters.
- [ ] Azure Repos: pull request builds configured by branch policy, not by `pr:`.
