# AI Skills for .NET

A library of focused, tool-agnostic skills that teach AI coding agents how to work on modern .NET projects: solution setup, Clean Architecture, Minimal APIs, Docker, and CI/CD. Every skill assumes a .NET codebase, so skill names don't repeat it.

Each skill covers **one topic**, uses the open `SKILL.md` format, and works with **Claude Code**, **OpenAI Codex**, and **GitHub Copilot**.

## Principles

- **One topic per skill.** Small skills load only when relevant and are easy to review.
- **Version-agnostic.** Skills tell the agent to read versions from the project (target framework, `global.json`) instead of hard-coding them.
- **Modern defaults, respectful of existing code.** New code follows current recommendations. Existing projects keep their conventions unless a migration is requested.
- **No company specifics.** No internal packages, names, feeds, or URLs.
- **Reviewable.** Every skill ends with a review checklist.

## Skills

| Skill | Covers | Status |
| --- | --- | --- |
| [solution-setup](skills/solution-setup/SKILL.md) | Repository layout (`src/`, `tests/`, `deploy/`), `.slnx`, `global.json`, `Directory.Build.props`, central package management, `nuget.config` with package source mapping, `.editorconfig`, `.gitignore`, created with the SDK templates | Draft |
| [clean-architecture](skills/clean-architecture/SKILL.md) | Domain, Application, Infrastructure, entry project; the dependency rule by project references, folders by feature, aggregates, use-case handlers and records, repositories and unit of work, service registration per layer, validation of Application records, tests per layer | Draft |
| [minimal-api](skills/minimal-api/SKILL.md) | New HTTP APIs with Minimal APIs: one extension method per feature with `MapGroup`, thin endpoint methods, pagination, `TypedResults`, built-in validation, problem details, OpenAPI, group authorization, CORS, health checks (live/ready), forwarded headers, stdout logging | Draft |
| [docker](skills/docker/SKILL.md) | Choosing the production base image (chiseled, Alpine, full Ubuntu, `-extra`), Dockerfile for a published output, optional parameterized Dockerfile, non-root | Draft |
| [ci-cd](skills/ci-cd/SKILL.md) | Continuous integration on Azure Pipelines and GitHub Actions: build-number versioning, SDK from `global.json`, restore/build/test/publish with `--no-build`, coverage, container image build-scan-push, NuGet packages, shared templates | Draft |

## Repository layout

```text
skills/
  <skill-name>/
    SKILL.md          # instructions loaded by the agent
    references/       # optional: longer material the skill links to
templates/
  SKILL.template.md   # starting point for new skills
```

## Installation

Copy a skill folder (for example `skills/docker/`) into the location your tool reads:

| Tool | In a project | For all your projects |
| --- | --- | --- |
| Claude Code | `.claude/skills/<skill-name>/` | `~/.claude/skills/<skill-name>/` |
| OpenAI Codex | `.agents/skills/<skill-name>/` | `~/.agents/skills/<skill-name>/` |
| GitHub Copilot | `.github/skills/<skill-name>/` | `~/.copilot/skills/<skill-name>/` |

Restart the tool or start a new session if the skill does not appear. Agents pick a skill from its `description`; you can also name the skill in your request to use it explicitly.

## Contributing

Start from [templates/SKILL.template.md](templates/SKILL.template.md) and follow the authoring rules in it.

## License

[MIT](LICENSE)
