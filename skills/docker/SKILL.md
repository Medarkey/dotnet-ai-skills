---
name: docker
description: Containerize ASP.NET Core APIs, worker services, and console apps with Docker by packaging a published output, with the publish folder as build context. Use when creating, reviewing, or fixing a Dockerfile; choosing a production base image (Ubuntu chiseled first, then Alpine, then full Ubuntu, with or without the -extra variant for SQL Server and globalization); packaging a publish folder into an image; reducing image size or attack surface; running as non-root; configuring ports or the entrypoint; or using one parameterized Dockerfile for several projects.
---

# Docker

Aim for an image that is **as small and as secure as possible**: minimal attack surface (no shell or package manager unless needed), non-root, and free of secrets. Use the official images from `mcr.microsoft.com/dotnet/` and keep their defaults.

Linux containers only.

## 1. Choose the base image

Do the three steps. In step 1, take the row whose condition applies.

**Step 1: distribution**

| Choose | When | Tag pattern |
| --- | --- | --- |
| **Ubuntu chiseled** (default) | No shell and no OS package needed. The image has no shell and no package manager, and runs non-root by default. | `<version>-<codename>-chiseled` |
| **Alpine** | Shell or OS package needed, and no glibc-only dependency. | `<version>-alpine` |
| **Ubuntu (full)** | Shell or OS package needed, and a glibc-only dependency (see below). | `<version>` |

- A shell is needed for the parameterized Dockerfile (section 5). An OS package is needed when a dependency uses a system library the image lacks, for example Kerberos for SQL Server integrated authentication.
- `<version>` is the project's target framework: `net10.0` → `10.0`.
- `<codename>` is an Ubuntu release offered for that version in the official tag list (for example `noble`). One .NET version can be offered on several codenames: use the same one for all services.
- The `<version>` tag is Ubuntu from .NET 10 and Debian up to .NET 9. Debian images are no longer shipped from .NET 10.
- Glibc check before using Alpine: if a package in the publish folder's `runtimes/` has `linux-x64/native/` but no `linux-musl-x64/native/` (or the `arm64` equivalents), its native library fails to load on Alpine. Chiseled and full Ubuntu use glibc and are not affected.
  - It is a glibc-only dependency only if the app uses that library. Find the package that brings it (`dotnet nuget why <package>`) and check what it is for. Example: `libmsalruntime.so` comes with SQL Server packages (`Microsoft.Identity.Client.NativeInterop`), but it serves interactive sign-in through a desktop broker only, which a container doesn't use.

**Step 2: variant (chiseled and Alpine only, section 2)**

| The app | Add to the tag |
| --- | --- |
| Uses SQL Server, specific cultures, or named time zones | `-extra` |
| Uses none of them | Nothing |

Full Ubuntu has no variant: it already includes ICU and tzdata.

**Step 3: image type**

| App | Image |
| --- | --- |
| Web app or API | `aspnet` |
| Worker service or console app | `runtime` (no `EXPOSE` in the Dockerfile) |
| Self-contained or Native AOT | `runtime-deps`, which starts a native executable instead of `dotnet <dll>`. For Native AOT without globalization needs, use the `-aot` variant (`<version>-<codename>-chiseled-aot`), the smallest one. |

Examples: `aspnet:10.0-noble-chiseled`, `aspnet:10.0-noble-chiseled-extra`, `aspnet:10.0-alpine-extra`.

Tag rules:
- Use the `<major.minor>` tag to get patch updates, and rebuild images regularly with `--pull`. Without `--pull`, Docker reuses the cached base image and a rebuild doesn't pick up patches.
- Never use `latest`. Check that the tag exists for the version before using it.
- Use the same distribution and codename for all services. Switch a service only for a condition from step 1.

For information: a tag can be moved to a new image at any time. To use exactly the same image in every build, reference it by digest (`<image>:<tag>@sha256:<digest>`); it then receives patches only when the digest is updated.

## 2. Globalization and SQL Server

The plain `-chiseled` and `-alpine` images contain neither ICU (culture data) nor tzdata (time zone database). In them:
- creating any culture other than the invariant culture (`new CultureInfo("fr-FR")`) throws `CultureNotFoundException`;
- named time zones (`TimeZoneInfo.FindSystemTimeZoneById`) fail;
- `Microsoft.Data.SqlClient` is not supported, because it doesn't support globalization-invariant mode. This covers EF Core with SQL Server and Dapper with SQL Server.

The `-extra` variants add ICU and tzdata.

Rules:
- On chiseled or Alpine, use `-extra` when the app uses SQL Server, specific cultures, or named time zones.
- On plain chiseled or plain Alpine, set `<InvariantGlobalization>true</InvariantGlobalization>` in the entry project: the app then behaves the same on the developer machine and in the container, and problems show up locally.
- On full Ubuntu, don't set `InvariantGlobalization`: ICU and tzdata are already there.
- Don't install ICU or tzdata with `apk add`: use the `-extra` image, which Microsoft maintains and patches.

## 3. The Dockerfile

The same shape works for every `aspnet` image: only `FROM` and the DLL name change. For `runtime`, remove `EXPOSE`. For `runtime-deps`, the entrypoint is the native executable instead of `dotnet <dll>`.

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled-extra

WORKDIR /app
COPY . .

# Run as non-root: the official images already contain the "app" user,
# and APP_UID holds its numeric user ID. No adduser or chown needed.
USER $APP_UID

EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

Rules for this file:
- Set `FROM` with the image chosen in section 1, and the DLL name to the entry project's assembly name (`<AssemblyName>`, else the project name).
- Keep the exec form `ENTRYPOINT ["dotnet", "..."]`: it works without a shell, and the app receives `SIGTERM` for a graceful shutdown.
- Use `USER $APP_UID`, not `USER app`: Kubernetes `runAsNonRoot` requires a numeric user ID. Keep it even on chiseled, where it is already the default: it documents the intent and survives a base image change.
- Don't add `adduser`, `chown`, or `chmod`: the image already has the non-root `app` user, and root-owned files keep the app read-only.
- Use `COPY`, not `ADD`: `ADD` also extracts archives and fetches URLs.
- Don't add `RUN` instructions for a chiseled image: there is no shell to run them.
- Don't add a `# syntax=` line: it downloads a parser image from Docker Hub on every build, and this file needs no feature from it.
- Don't add a `.dockerignore`: the build context is the publish folder, which contains only the published app.

## 4. Build

```bash
dotnet publish src/MyApp.Api/MyApp.Api.csproj -c Release -o publish/MyApp.Api

docker build --pull -f deploy/Dockerfile -t myapp-api:1.0.0 publish/MyApp.Api
```

- The last argument is the build context: the publish folder. `-f` points to the Dockerfile, which stays in the repository outside that folder.
- Publish **framework-dependent and portable** (no `-r`). Portable output runs on any architecture, so the same folder also builds a multi-architecture image: `docker buildx build --platform linux/amd64,linux/arm64 ...`.
- For framework-dependent apps, set `<UseAppHost>false</UseAppHost>` in the entry project file: the image starts the app with `dotnet <dll>`, so the native launcher isn't needed. Setting it in the project, not on the command line, gives the same output locally and in CI. Don't set it for self-contained or Native AOT apps, which start through their executable.
- If a runtime identifier is used anyway, match the distribution: `linux-x64` / `linux-arm64` for chiseled and full Ubuntu, `linux-musl-x64` / `linux-musl-arm64` for Alpine.
- Keep the publish folder (`publish/`) out of Git: the generated `.gitignore` (`dotnet new gitignore`) already ignores it; add it only to a `.gitignore` that doesn't.

## 5. Optional: one parameterized Dockerfile for several projects

Use this only when several projects share the same base image **and that image has a shell** (Alpine or full Ubuntu). It is not possible with chiseled: the DLL name is expanded by `/bin/sh` at container start, and chiseled has no shell.

```dockerfile
# Build arguments (all required, no defaults)
# DOTNET_VERSION must be declared before FROM because FROM uses it.
ARG DOTNET_VERSION
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-alpine-extra
ARG DLL_NAME

WORKDIR /app
COPY . .

# Fail the build when DLL_NAME is missing or not in the publish folder
RUN test -n "$DLL_NAME" && test -f "$DLL_NAME" \
    || { echo "Check build arg DLL_NAME='$DLL_NAME' and the build context" >&2; exit 1; }

ENV DLL_NAME=${DLL_NAME}
USER $APP_UID
EXPOSE 8080
ENTRYPOINT ["/bin/sh", "-c", "exec dotnet \"$DLL_NAME\""]
```

```bash
docker build --pull -f deploy/Dockerfile \
  --build-arg DOTNET_VERSION=10.0 \
  --build-arg DLL_NAME=MyApp.Api.dll \
  -t myapp-api:1.0.0 \
  publish/MyApp.Api
```

Rules for this file:
- Keep `DOTNET_VERSION` before `FROM` (`FROM` uses it) and `DLL_NAME` right after `FROM` (an `ARG` declared before `FROM` is not usable after it).
- Don't give the arguments defaults: a missing argument must fail the build.
- Keep the check after `COPY`: it catches a wrong `DLL_NAME` or build context at build time, not at container start.
- Keep `exec` in the `ENTRYPOINT`: without it the shell can stay the main process and the app may not receive `SIGTERM`. Don't use plain shell form (`ENTRYPOINT dotnet $DLL_NAME`) for the same reason.
- Adjust the suffix to the image chosen in section 1 (`-alpine`, `-alpine-extra`, or none for full Ubuntu), keeping the same value for all projects that share the file.

## 6. Security

- **Run as non-root** with `USER $APP_UID`. The official images include an `app` user and set `APP_UID`. Don't create your own user or install `sudo`.
- **Keep app files read-only.** Don't `chown` or `chmod -R 775` the app folder. If the app needs to write, use a mounted volume.
- **No secrets in the image.** Values in `ARG`, `ENV`, and copied files stay in the image history. Everything in the publish folder goes into the image, so keep local settings files out of it in the project file:

  ```xml
  <ItemGroup>
    <Content Update="appsettings.Local.json" CopyToPublishDirectory="Never" />
  </ItemGroup>
  ```

  Provide runtime secrets through the orchestrator (environment variables, mounted secret files) or a managed identity with a secret store.
- **No TLS certificates in the image.** Terminate TLS at the ingress or load balancer and serve plain HTTP on 8080 inside the container.
- **OS packages (Alpine or full Ubuntu only):** install them before `USER`, in a single `RUN`, and only what the app needs. Clean up in the same `RUN`, otherwise the package index stays in the image:

  ```dockerfile
  # Alpine
  RUN apk add --no-cache <package>

  # Full Ubuntu
  RUN apt-get update \
   && apt-get install -y --no-install-recommends <package> \
   && rm -rf /var/lib/apt/lists/*
  ```

## 7. Runtime configuration

- **Port:** the images listen on `8080` (`ASPNETCORE_HTTP_PORTS`). Keep it and map it on the host (`-p 5000:8080`). When health checks use a dedicated port, list both at run time (`ASPNETCORE_HTTP_PORTS=8080;8081`) and expose only the API port through the ingress; the app binds its health checks to that port (minimal-api skill, "Hosting concerns").
- **Environment:** set `ASPNETCORE_ENVIRONMENT` / `DOTNET_ENVIRONMENT` at run time, not in the image (the default is `Production`).
- **Health:** don't add a `HEALTHCHECK` to the Dockerfile: Kubernetes ignores it, and chiseled has no tool to run it. Health probes are configured in the orchestrator.

## 8. Anti-patterns

| Avoid | Do instead |
| --- | --- |
| Alpine or full Ubuntu without a condition from section 1 | Ubuntu chiseled |
| Debian image for .NET 10 or later | Full Ubuntu (`<version>` tag) |
| Plain chiseled or Alpine for an app using SQL Server, cultures, or named time zones | `-extra` image |
| Plain chiseled or Alpine without `InvariantGlobalization` in the project | Set `<InvariantGlobalization>true</InvariantGlobalization>`, or use `-extra` |
| `-extra` for an app that doesn't need it | Plain image |
| `apk add icu-libs tzdata` | `-extra` image |
| Publishing with `-r linux-x64` for an Alpine image | Portable output or `-r linux-musl-*` |
| Different distributions or codenames across services without a reason | One standard for all services |
| Repository root as build context | Publish folder as build context, Dockerfile passed with `-f` |
| `ENTRYPOINT dotnet MyApp.Api.dll` (shell form) | `ENTRYPOINT ["dotnet", "MyApp.Api.dll"]` |
| Parameterized Dockerfile on a chiseled image | Project Dockerfile with the DLL name written in it |
| `ADD` for the publish folder | `COPY` |
| `# syntax=` line without a feature that needs it | No `# syntax=` line |
| Running as root, creating a custom user, or `USER app` | `USER $APP_UID` |
| `chmod -R 775` / `chown` on the app folder | Read-only app files, writes to a mounted volume |
| `apt-get install` without `--no-install-recommends` and cleanup in the same `RUN` | Install and clean up in one `RUN` |
| Secrets in `ARG`, `ENV`, or published settings files | `CopyToPublishDirectory="Never"` for local files, secrets from the orchestrator or a secret store |
| `latest` tag | `<major.minor>` tag of the chosen image |
| Building without `--pull` | `docker build --pull` |
| Ports 80/443 with a dev certificate in the image | Port 8080, TLS at the ingress |
| Baking `ASPNETCORE_ENVIRONMENT=Development` into the image | Set the environment at run time |

## Review checklist

- [ ] Base image chosen in order: chiseled, then Alpine, then full Ubuntu, each switch justified by a condition from section 1.
- [ ] Chiseled or Alpine: `-extra` if and only if the app uses SQL Server, specific cultures, or named time zones. Otherwise `InvariantGlobalization=true` in the entry project.
- [ ] Right image type: `aspnet`, `runtime`, or `runtime-deps`.
- [ ] Same distribution and codename as the other services.
- [ ] `<major.minor>` tag, no `latest`, built with `--pull`.
- [ ] Build context is the publish folder. Dockerfile passed with `-f`. No `# syntax=` line.
- [ ] Exec-form `ENTRYPOINT`, `USER $APP_UID`, port 8080.
- [ ] Published output is portable, or its runtime identifier matches the distribution; `UseAppHost=false` in the project for framework-dependent apps.
- [ ] No secrets in `ARG`, `ENV`, or the publish folder.
- [ ] Parameterized Dockerfile only on an image with a shell, with required args and the `sh -c "exec ..."` entrypoint.
