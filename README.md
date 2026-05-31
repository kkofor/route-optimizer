# Route Optimizer

A lightweight delivery route planner for dense city dispatch workflows.

This repo contains two static browser tools:

- `index.html`: paste parsed stops, optimize the route, and generate Google Maps navigation links
- `app.html`: upload dispatch screenshots, extract stops with an Anthropic-compatible API, then optimize and track the run

Both tools are client-side only and can be hosted on GitHub Pages.

## Why this project exists

Most delivery tools are either too heavy for solo drivers or too generic for real pickup-before-delivery constraints. This project focuses on a narrow workflow:

- fast paste-in or screenshot-driven dispatch intake
- pickup-before-delivery ordering
- practical city routing
- same-day order tracking in the browser
- mobile-friendly Google Maps handoff

## Features

- Multi-start greedy routing with local search improvements
- Pickup and delivery dependency handling
- OpenRouteService-based travel-time and directions lookup
- Mapbox geocoding for better address resolution
- Google Maps deep links with chunking for mobile navigation
- Local order history stored in the browser
- End-of-day summary and lightweight stats

## Open Source Setup

No secrets are stored in this repository.

To run the tools, open the HTML file in a browser and provide your own API settings in the built-in `Open Source Setup` panel:

- `OpenRouteService API key`
- `Mapbox public token` for `index.html`
- `AI Base URL`, `AI API token`, and `AI model` for `app.html`

These values are stored only in your browser via `localStorage`.

## Usage

### `index.html`

1. Open `index.html`
2. Paste lines in this format:

```text
PICKUP|job-1|123 Pickup Ave, Winnipeg, MB
DELIVER|job-1|500 Delivery Rd, Winnipeg, MB
DELIVER_ONLY|job-2|42 Example Crescent, Winnipeg, MB
HOME|123 Example St, Winnipeg, MB
```

3. Click `解析`
4. Click `优化并生成导航`

### `app.html`

1. Open `app.html`
2. Fill in the `Open Source Setup` values
3. Upload one or more dispatch screenshots
4. Run AI extraction
5. Confirm results and generate the route

## Deployment

Because the project is static, the simplest deployment path is GitHub Pages.

If you want a live demo:

1. Push this repo to GitHub
2. Enable GitHub Pages from the `main` branch
3. Use `index.html` or `app.html` as your entry point

## Privacy and Security

- This repository intentionally contains no live API keys
- User-supplied keys stay in the local browser only
- Local order history is stored in browser `localStorage`
- If you fork or redeploy publicly, use your own restricted API credentials

## Tech Notes

- Pure HTML, CSS, and JavaScript
- No build step
- No backend included
- Optimized for dispatch-style usage on desktop and mobile browsers

## Roadmap Ideas

- Import and export daily runs as JSON
- Better batch editing for parsed stops
- Optional hosted proxy for teams that do not want browser-side API setup
- Sharable demo dataset for easier onboarding

## Contributing

Issues and pull requests are welcome, especially around:

- routing heuristics
- mobile UX
- dispatch-specific parsing workflows
- better onboarding for self-hosting
