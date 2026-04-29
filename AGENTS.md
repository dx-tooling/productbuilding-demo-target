# AGENTS.md — Project Guide for AI Coding Agents

## Project Overview

**Acme Directory** is a single-page service-provider directory: a searchable, filterable, grid/list-toggleable listing of providers ("Acme") with type-ahead filtering and a switchable view mode. It exists primarily as a **demo target for the productbuilding orchestration system** — a small, framework-free HTML/CSS/JS app that exercises the orchestrator's PR-preview pipeline end-to-end without any build step or runtime dependency surface.

The repo is the canonical shape for "minimum viable orchestration target": every PR opened against it gets a live preview at `https://productbuilding-demo-target-pr-<n>.<preview-domain>/`.

## Architecture

A single file does the job:

- `index.html` (~1450 lines) — the entire application. HTML markup, embedded CSS (custom-property design tokens, Flexbox + Grid layout, Inter + Source Serif 4 from Google Fonts), and inline JavaScript (vanilla, no framework, no transpiler).

The orchestration contract is in `.productbuilding/preview/`:

- `config.yml` — declares `version: 1`, points at the docker-compose file, names the main service `app`, exposes port 8080, healthchecks `/`.
- `docker-compose.yml` — single `app` service that builds from the Dockerfile in this directory.
- `Dockerfile` — `nginx:1-alpine`, copies `nginx.conf` and `index.html` into the image, exposes 8080.
- `nginx.conf` — listens on 8080, SPA-style `try_files` fallback (handy for adding client-side routing later without revisiting the contract).

OpenCode workflow at `.github/workflows/opencode.yml` triggers on `/opencode` mentions in issue/PR comments.

## Development

There's no build, no package manager, no test runner. Two ways to iterate:

```bash
# Quickest: open the file directly. Fonts load from Google Fonts; everything else is local.
open index.html

# Closer to production: run the preview container locally (matches what the orchestrator builds).
docker compose -f .productbuilding/preview/docker-compose.yml up --build
# Then: http://localhost:8080/  (or whatever Compose surfaces)
```

For PR previews, just push: the orchestrator handles clone → build → deploy → comment-with-URL automatically. Health is determined by `/` returning 2xx within 30 s.

## Coding Standards

- **Vanilla everything.** No framework, no bundler, no TypeScript, no preprocessor. The single-file constraint is intentional — keep it so unless there's a strong reason otherwise.
- **CSS via design tokens.** Use the existing `--*` custom properties at the top of the `<style>` block; don't introduce raw colour/spacing literals further down. If you need a new token, add it to the token block, not inline.
- **Semantic HTML.** Real `<button>` elements for buttons, real `<a>` for links, ARIA only where semantics genuinely fall short.
- **Two spaces for indentation** in HTML/CSS/JS (matches the existing file).
- **No external runtime dependencies** beyond Google Fonts (already wired). Don't pull in a CDN-hosted JS library to add a feature that vanilla JS can do in twenty lines.

## Important Rules

- **Don't introduce a build step.** This repo's value as an orchestration target depends on it being trivially deployable as-is. If a feature genuinely warrants a build pipeline, raise it as an issue first — adopting one changes the orchestrator preview shape (`Dockerfile` and `config.yml` would both need updates).
- **Don't change the orchestration contract files** (`.productbuilding/preview/*`) without understanding the implications. They're managed in coordination with the orchestrator deployment; the `internal_port`, `healthcheck_path`, and compose file path are part of a bilateral contract. If you need to adjust them, the orchestrator-side counterpart is in `productbuilding-orchestration/orchestrator-app/internal/preview/`.
- **Keep `index.html` self-contained.** Splitting into multiple files is fine *if* the splitting is also reflected in the Dockerfile's `COPY` lines and the nginx config still serves them. Easy to forget the second half.
- **No secrets in the repo.** This is a demo public repo; treat it as such.

## AI Integration

This project uses OpenCode with FireworksAI (Kimi K2.5). Trigger with `/opencode` in GitHub issues and pull requests — the workflow at `.github/workflows/opencode.yml` picks it up. The `FIREWORKS_API_KEY` Actions secret is managed by the productbuilding orchestrator's `targetadmin` reconciler; you don't need to set or rotate it manually.

The orchestrator also posts PR-preview status as comments on every PR it processes, and (when configured) mirrors PR/issue activity into a dedicated Slack channel. `@ProductBuilder` mentions in that Slack channel can drive issue creation, PR delegation, and questions about this repo.
