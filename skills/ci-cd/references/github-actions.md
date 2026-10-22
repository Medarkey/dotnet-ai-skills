# GitHub Actions

Platform details for the rules in [SKILL.md](../SKILL.md). Read SKILL.md first: this file only shows how to apply it in a workflow under `.github/workflows/`.

`@<version>` in the examples is a placeholder: use the latest major version of each action from its releases page (`actions/checkout@v<major>`), and keep the version already used in the repository unless an update is requested.

## 1. Application workflow

One job, steps in the order of SKILL.md section 1. Replace the values in `env` and the registry login.

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

env:
  BUILD_CONFIGURATION: Release
  SOLUTION: MyApp.slnx
  ENTRY_PROJECT: src/MyApp.Api/MyApp.Api.csproj
  APP_NAME: MyApp.Api
  DOCKERFILE: deploy/Dockerfile
  REGISTRY: myregistry.azurecr.io
  IMAGE_NAME: myapp-api
  STAGING: ${{ github.workspace }}/staging

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write # federated login to Azure, image push only
    steps:
      - uses: actions/checkout@<version>

      - name: Install SDK from global.json
        uses: actions/setup-dotnet@<version>
        with:
          global-json-file: global.json

      - name: Set version
        run: |
          version="0.$(date -u +%Y).$(date -u +%-m%d).${{ github.run_number }}"
          echo "VERSION=$version" >> "$GITHUB_ENV"
          echo "VERSION_ARGS=-p:Version=$version -p:AssemblyVersion=$version -p:FileVersion=$version" >> "$GITHUB_ENV"

      # Add --locked-mode when the projects have packages.lock.json files
      - name: Restore
        run: dotnet restore "$SOLUTION"

      - name: Build
        run: >
          dotnet build "$SOLUTION" -c "$BUILD_CONFIGURATION" --no-restore
          -p:ContinuousIntegrationBuild=true $VERSION_ARGS

      - name: Test
        run: >
          dotnet test "$SOLUTION" -c "$BUILD_CONFIGURATION" --no-build
          --logger trx --results-directory "$RUNNER_TEMP/TestResults"
          --collect:"XPlat Code Coverage"

      - name: Upload test results and coverage
        if: ${{ !cancelled() }}
        uses: actions/upload-artifact@<version>
        with:
          name: test-results
          path: ${{ runner.temp }}/TestResults

      - name: Publish app
        run: >
          dotnet publish "$ENTRY_PROJECT" -c "$BUILD_CONFIGURATION" --no-build
          -o "$STAGING/$APP_NAME" $VERSION_ARGS

      - name: Upload app artifact
        uses: actions/upload-artifact@<version>
        with:
          name: ${{ env.APP_NAME }}
          path: ${{ env.STAGING }}/${{ env.APP_NAME }}

      # Container image: build and scan in every build, push from the default branch only (SKILL.md section 6)
      - name: Build image
        run: docker build --pull -f "$DOCKERFILE" -t "$REGISTRY/$IMAGE_NAME:$VERSION" "$STAGING/$APP_NAME"

      - name: Scan image
        run: <scan command> "$REGISTRY/$IMAGE_NAME:$VERSION"

      - name: Log in to Azure
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        uses: azure/login@<version>
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Push image
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: |
          az acr login --name "${REGISTRY%%.*}"
          docker push "$REGISTRY/$IMAGE_NAME:$VERSION"
```

- **Version:** GitHub has no daily run counter. `github.run_number` counts the runs of the workflow and never resets, and a re-run keeps the same number, so the version stays unique and stable. It must stay below 65534 (SKILL.md section 2): when it gets close, use a new workflow file name, which restarts the counter.
- `<scan command>` is the team's image scanner. It must exit with a non-zero code on critical findings that have a fix (for example `trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed`), and must be installed on the runner first if the runner image doesn't have it.
- If the test projects use Microsoft.Testing.Platform (SKILL.md section 4), replace the test step with `dotnet test --solution "$SOLUTION" ...` and the options of the repository's MTP extensions (for example `--report-trx` for test results).
- The Azure login uses a federated credential (OIDC): no password stored in GitHub. It needs `id-token: write` and a federated credential on the Azure identity for this repository and branch. For another registry, replace the login and push steps with that registry's login action and `docker push`.
- `if: ${{ !cancelled() }}` uploads the test results also when tests fail.
- The test results (`.trx`) and coverage (`coverage.cobertura.xml`) are uploaded as an artifact. To show them in the run summary, add a test reporter action or a coverage report step that the team approves (third-party actions follow the rules in section 5).

### Parameterized Dockerfile

When the image uses the parameterized Dockerfile of the docker skill, derive `DOTNET_VERSION` from the project:

```yaml
      - name: Build image
        run: |
          tfm=$(dotnet msbuild "$ENTRY_PROJECT" -getProperty:TargetFramework)
          docker build --pull -f "$DOCKERFILE" \
            --build-arg DOTNET_VERSION="${tfm#net}" \
            --build-arg DLL_NAME="$DLL_NAME" \
            -t "$REGISTRY/$IMAGE_NAME:$VERSION" "$STAGING/$APP_NAME"
```

Add `DLL_NAME: MyApp.Api.dll` to `env`. `-getProperty` returns nothing for a multi-targeted project (`TargetFrameworks`): pass the framework explicitly in that case.

## 2. Library workflow (NuGet packages)

A separate workflow file (for example `.github/workflows/library.yml`). This example is for a library in a repository shared with an app: it builds the library's test project, and its path filter follows SKILL.md section 9. In a library-only repository, use the solution instead of `TEST_PROJECT` and remove `paths`.

```yaml
name: library

on:
  push:
    branches: [main]
    tags: ['v*']
    paths:
      - 'src/MyApp.Client/**'
      - 'tests/MyApp.Client.Tests/**'
      - 'Directory.Build.props'
      - 'Directory.Packages.props'
      - 'global.json'
      - 'nuget.config'
  pull_request:
    branches: [main]
    paths:
      - 'src/MyApp.Client/**'
      - 'tests/MyApp.Client.Tests/**'
      - 'Directory.Build.props'
      - 'Directory.Packages.props'
      - 'global.json'
      - 'nuget.config'

permissions:
  contents: read

env:
  BUILD_CONFIGURATION: Release
  TEST_PROJECT: tests/MyApp.Client.Tests/MyApp.Client.Tests.csproj
  LIBRARY_PROJECT: src/MyApp.Client/MyApp.Client.csproj
  FEED_URL: https://nuget.pkg.github.com/my-org/index.json
  STAGING: ${{ github.workspace }}/staging

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write # push to GitHub Packages
    steps:
      - uses: actions/checkout@<version>

      - name: Install SDK from global.json
        uses: actions/setup-dotnet@<version>
        with:
          global-json-file: global.json

      - name: Set package version
        run: |
          prefix=$(dotnet msbuild "$LIBRARY_PROJECT" -getProperty:VersionPrefix)
          if [ "$GITHUB_REF_TYPE" = "tag" ]; then
            if [ "$GITHUB_REF_NAME" != "v$prefix" ]; then
              echo "::error::Tag $GITHUB_REF_NAME doesn't match VersionPrefix $prefix"
              exit 1
            fi
            version="$prefix"
          else
            version="$prefix-ci.$(date -u +%Y%m%d).$GITHUB_RUN_NUMBER"
          fi
          echo "VERSION_ARGS=-p:Version=$version -p:AssemblyVersion=${prefix%%.*}.0.0.0 -p:FileVersion=$prefix.$GITHUB_RUN_NUMBER" >> "$GITHUB_ENV"

      - name: Restore
        run: dotnet restore "$TEST_PROJECT"

      - name: Build
        run: >
          dotnet build "$TEST_PROJECT" -c "$BUILD_CONFIGURATION" --no-restore
          -p:ContinuousIntegrationBuild=true $VERSION_ARGS

      - name: Test
        run: >
          dotnet test "$TEST_PROJECT" -c "$BUILD_CONFIGURATION" --no-build
          --logger trx --results-directory "$RUNNER_TEMP/TestResults"
          --collect:"XPlat Code Coverage"

      # Upload test results and coverage as in section 1

      - name: Pack
        run: >
          dotnet pack "$LIBRARY_PROJECT" -c "$BUILD_CONFIGURATION" --no-build
          -o "$STAGING/packages" $VERSION_ARGS

      - name: Upload packages
        uses: actions/upload-artifact@<version>
        with:
          name: packages
          path: ${{ env.STAGING }}/packages

      - name: Push packages
        if: github.event_name == 'push' && (github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/v'))
        env:
          NUGET_API_KEY: ${{ secrets.GITHUB_TOKEN }}
        run: dotnet nuget push "$STAGING/packages/*.nupkg" --source "$FEED_URL" --api-key "$NUGET_API_KEY"
```

- **Release:** push a tag `v<major>.<minor>.<patch>` on a commit of `main`. The version step fails when the tag doesn't match `VersionPrefix`.
- Path filters are not evaluated for tag pushes, so a release tag always starts a run. Keep `branches` next to `tags`: with `tags` alone, pushes to `main` no longer trigger the workflow.
- The tag name and run number are read from the `GITHUB_REF_NAME` and `GITHUB_RUN_NUMBER` environment variables, not from `${{ }}` expressions inside the script (section 5).
- Don't make this workflow a required status check: a pull request that doesn't touch its paths skips it, and a skipped required check blocks the pull request (section 5).
- Building the test project also builds the library through its `ProjectReference`, with the same version properties, so `dotnet pack --no-build` finds its output.
- The example pushes to GitHub Packages with `GITHUB_TOKEN`, which needs `packages: write`. For another feed, use its URL and an API key from a repository secret.
- A feed rejects a version that already exists. A release run for an existing version fails, which is the expected behavior.
- After a release, bump `VersionPrefix` by pull request before the next CI package (SKILL.md section 7).
- `FileVersion` uses the run number as its last part, so the 65534 limit applies.

## 3. Private NuGet feeds

`nuget.config` lists the feeds; the workflow adds the credentials before the restore:

```yaml
      - name: Authenticate NuGet feeds
        run: >
          dotnet nuget update source MyFeed
          --username ci --password "$FEED_TOKEN" --store-password-in-clear-text
        env:
          FEED_TOKEN: ${{ secrets.FEED_TOKEN }}
```

- `MyFeed` is the key of the feed in `nuget.config`. The change stays in the runner's checkout and is never committed.
- `--store-password-in-clear-text` is required on Linux, where NuGet can't encrypt passwords. The runner is discarded after the job.
- For GitHub Packages of the same organization, use `secrets.GITHUB_TOKEN` with `packages: read` instead of a personal token.
- Store tokens as repository or organization secrets with the least scope (read packages only).

## 4. Shared workflows

Call a reusable workflow from the templates repository:

```yaml
jobs:
  ci:
    uses: my-org/pipeline-templates/.github/workflows/dotnet-ci.yml@v1
    with:
      solution: MyApp.slnx
      entry-project: src/MyApp.Api/MyApp.Api.csproj
      app-name: MyApp.Api
      image-name: myapp-api
    permissions:
      contents: read
      id-token: write
```

Reusable workflow in the templates repository (`.github/workflows/dotnet-ci.yml`):

```yaml
on:
  workflow_call:
    inputs:
      solution:
        type: string
        required: true
      entry-project:
        type: string
        required: true
      app-name:
        type: string
        required: true
      image-name:
        type: string
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<version>
      # ... the steps of section 1, using ${{ inputs.<name> }}
```

- Reference your organization's reusable workflows by major version tag (`@v1`), moved to each compatible release; a breaking change gets a new major tag. Never reference a branch.
- To freeze a pipeline on an exact version, use a full commit SHA. Reusable workflows from outside the organization are third-party code: always pin them to a SHA.
- Mark project inputs `required: true` without a default: a missing value fails before the job starts.
- Pass secrets explicitly (`secrets:` with named secrets). Use `secrets: inherit` only for workflows of the same organization that need all of them.
- The called workflow gets at most the permissions of the caller job: set them on the caller.
- A reusable workflow in a private repository must allow access from other repositories (repository settings → Actions → Access).
- For a few shared steps inside a job, use a composite action instead of a reusable workflow.

## 5. Workflow rules

- Set `permissions` at the top of the workflow (`contents: read`) and add write permissions only on the job that needs them. Don't rely on the repository default for `GITHUB_TOKEN`.
- Use actions from `actions/*` and the cloud provider's official organization by major version tag. Pin any other third-party action to a full commit SHA, with the version in a comment: a tag can be moved.
- Pass secrets and untrusted values (branch names, pull request titles) to `run` through `env`, not by writing `${{ ... }}` inside the script: an expression in a script is pasted before the shell runs and can inject commands.
- Pull requests from forks get no secrets and a read-only `GITHUB_TOKEN`: keep steps that need secrets behind the default-branch condition.
- Don't use `pull_request_target` to build pull request code: it runs with the base repository's secrets.
- `on.push.paths` and `on.pull_request.paths` filter by changed files. If a required status check is skipped by a path filter, the pull request stays blocked: don't combine path filters with required checks on the same workflow.

## 6. Anti-patterns

| Avoid | Do instead |
| --- | --- |
| `setup-dotnet` with `dotnet-version: 8.0.x` | `global-json-file: global.json` |
| No `permissions` block | `contents: read` at the top, more only per job |
| Third-party action by tag | Full commit SHA |
| `${{ secrets.X }}` or `${{ github.head_ref }}` inside `run` | Pass it through `env` |
| Push steps without an event and branch condition | `if: github.event_name == 'push' && github.ref == 'refs/heads/main'` |
| Registry password in a secret when OIDC is available | Federated login (`id-token: write`) |
| Personal token in `nuget.config` | `dotnet nuget update source` with a secret at run time |
| Reusable workflow referenced by branch | Major version tag, or SHA for third-party |
| `pull_request_target` for builds | `pull_request` |
| Image build and scan limited to the default branch | Build and scan in every run, limit only login and push |

## Review checklist

- [ ] `setup-dotnet` reads `global.json`.
- [ ] Version computed once, each part below 65534, stored in `GITHUB_ENV`, passed to every producing command.
- [ ] Build with `ContinuousIntegrationBuild=true`.
- [ ] `permissions` set at workflow level, write permissions only where needed.
- [ ] Feed credentials added at run time from secrets.
- [ ] Test results and coverage uploaded with `if: ${{ !cancelled() }}`.
- [ ] Image built and scanned in every run; login and image push limited to pushes on the default branch.
- [ ] Library workflow: test project built instead of the shared solution, paths include tests and shared build files, `branches` next to `tags`.
- [ ] Library versions: stable only from a `v*` tag matching `VersionPrefix`, `AssemblyVersion` at `<major>.0.0.0`; packages pushed from `main` and release tags only.
- [ ] Third-party actions pinned to a SHA; own reusable workflows referenced by major version tag.
- [ ] No `${{ }}` expression with secrets or user input inside `run` scripts.
