# Skill: /jca-publish

## Trigger

```
/jca-publish
```

## Description

Publishes the most recently generated JCA analysis report (`concurrency_analysis/report.md`) to the configured destination (default: Confluence). Reads publish settings from `jca-config.json`.

## Arguments

None. Uses the latest report in `concurrency_analysis/` and publish configuration in `jca-config.json`.

## Example Usage

```
/jca-publish
```

## Prerequisites

1. A completed `/jca-analyze` run must exist; `concurrency_analysis/report.md` must be present.
2. `jca-config.json` must contain valid `publish` settings:
   - `publish.destination` — target system (e.g., `"confluence"`)
   - `publish.spaceKey` — Confluence space key (e.g., `"ANDROID"`)
   - `publish.parentPageTitle` — parent page title
3. Credentials must be available in the environment (e.g., `CONFLUENCE_TOKEN`).

## Behavior

1. Reads `concurrency_analysis/report.md` and `concurrency_analysis/report.json`.
2. Resolves publish destination and credentials from `jca-config.json` and environment.
3. Creates or updates the target page with report content.
4. Prints the URL of the published page on success.

## Configuration

```json
"publish": {
  "destination": "confluence",
  "spaceKey": "ANDROID",
  "parentPageTitle": "JCA Analysis Reports",
  "reportFormat": ["markdown", "json"]
}
```

## Execution Principles

- Read-only with respect to the AOSP source tree.
- Only writes to the external destination system.
- No source files are modified.
