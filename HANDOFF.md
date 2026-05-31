# Route Optimizer Handoff

This note is a sanitized maintainer handoff for the public repository.

## Repository shape

- `index.html`: paste-first route planner
- `app.html`: screenshot-first route planner with AI-assisted extraction
- `README.md`: public-facing overview and setup instructions

## Security posture

- No live secrets should be committed to this repository
- Browser-side API settings are provided by each user through the `Open Source Setup` panel
- Keys are stored locally in the browser only

If new integrations are added later, keep the same rule:

- never commit live tokens
- never commit personal addresses or operator-specific default values
- prefer placeholders and local browser storage for static builds

## Current product posture

The project is optimized for single-operator or small-team delivery workflows in a city environment:

- pickup before delivery constraints
- route optimization
- browser-local daily order tracking
- Google Maps navigation handoff

## Hosting

The repo is static and works well on GitHub Pages.

Recommended public setup:

- GitHub Pages for the frontend
- user-provided OpenRouteService and Mapbox credentials
- optional self-hosted Anthropic-compatible gateway for `app.html`

## Release checklist

Before publishing future updates:

1. Search for hardcoded tokens, private URLs, and personal addresses
2. Test both `index.html` and `app.html` with empty local storage
3. Verify the setup panel still explains required keys clearly
4. Confirm README setup steps still match the UI

## Suggested next improvements

- Add a sample dataset JSON for one-click demo loading
- Add a screenshot section to README
- Add a LICENSE once the preferred open-source license is chosen
- Add a tiny changelog section for public releases
