# Repository Guidelines

## Project Structure & Module Organization

This repository builds a custom Hermes Agent container image. The main files are:

- `Dockerfile`: multi-stage image definition that installs Go, apt, pipx, and npm CLIs into the Hermes base image.
- `README.md`: operational documentation for the image, local builds, `spogo` authentication, and CI behavior.
- `.github/workflows/docker-weekly.yml`: scheduled and manual GitHub Actions workflow for building and publishing the image.

There is no application source tree, test suite, or static asset directory at present. Keep future additions minimal and document any new top-level directories here.

## Build, Test, and Development Commands

Use Docker as the primary validation tool:

```bash
docker build -t hermes-agent-custom:local .
```

Builds the image with the default `HERMES_BASE`.

```bash
docker build -t hermes-agent-custom:local \
  --build-arg HERMES_BASE=nousresearch/hermes-agent:latest \
  .
```

Builds while explicitly selecting the upstream Hermes base image.

```bash
docker run --rm hermes-agent-custom:local gh --version
docker run --rm hermes-agent-custom:local spogo --help
```

Smoke-tests installed tools after a successful build.

## Coding Style & Naming Conventions

Use standard Dockerfile formatting: uppercase instructions, one package per line in long install lists, and backslash continuations aligned for readability. Keep comments short and focused on maintenance context. Prefer pinned image digests or explicit build arguments when reproducibility matters.

Markdown should use sentence-case headings where practical, fenced code blocks with language hints, and relative links such as `[Dockerfile](Dockerfile)`.

## Testing Guidelines

There is no dedicated automated test framework. Treat `docker build` as the required baseline check for Dockerfile changes. For tool additions, add at least one smoke command that verifies the executable is on `PATH`, such as `tool --help` or `tool --version`. Avoid tests that require personal credentials, browser profiles, or external interactive auth.

## Commit & Pull Request Guidelines

Recent commits use concise imperative summaries. Follow that style: start with an action verb, name the touched area, and keep the subject specific.

Pull requests should include a short description, the reason for the image change, the local `docker build` result, and any relevant smoke-test output. Link issues when applicable. For CI or publishing changes, describe tag, registry, or schedule impact.

## Security & Configuration Tips

Do not commit secrets, Spotify cookies, registry tokens, or host-specific configuration. Keep credentials in runtime-mounted files or platform secret stores. When documenting auth workarounds, use placeholders and restrictive file permissions, matching the README examples.
