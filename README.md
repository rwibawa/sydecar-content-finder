# Sydecar Content Finder

A lightweight sales enablement app that helps reps match prospect context to the most relevant Sydecar content assets.

This repo contains:
- A static frontend app (`index.html`) with a curated asset library and AI-assisted recommendations
- A simple add-asset page (`add.html`) for triggering asset ingestion
- Python automation scripts to keep the asset library updated
- A Cloudflare Worker proxy (`worker/`) for HubSpot, GitHub Actions, Anthropic, and Asana integrations

## Repository Structure

- `index.html`: Main Content Finder app (React via CDN, inline Babel)
- `add.html`: Minimal UI for adding a new asset
- `add_asset.py`: Add one asset into `index.html` (with redirect resolution + Claude auto-tagging)
- `sync_assets.py`: Daily sync from `https://sydecar.io/sitemap.xml` into `index.html`
- `apps-script-proxy.js`: Legacy Google Apps Script proxy implementation
- `worker/index.js`: Cloudflare Worker proxy (current backend)
- `worker/wrangler.toml`: Worker config
- `.github/workflows/add-asset.yml`: Manual workflow for adding a single asset
- `.github/workflows/sync-assets.yml`: Daily sitemap sync workflow
- `.github/workflows/deploy-worker.yml`: Auto deploy Worker on `worker/**` changes

## How It Works

### 1) Frontend (`index.html`)
- Loads a local `ASSETS` array (source of truth in this repo)
- Collects prospect context (deal stage, persona, competitor, challenges)
- Uses Anthropic in-browser for recommendation rationale and tag matching
- Calls the proxy URL for:
  - HubSpot search (`action: "search"`)
  - Deal/contact context enrichment (`action: "get_deal"`, `action: "get_contact"`)
  - Asset add workflow trigger (`action: "add_asset"`)
  - Content request routing (`action: "content_request"`)

### 2) Asset automation
- `add_asset.py` inserts one asset into `index.html`
- `sync_assets.py` discovers new content URLs from the sitemap and appends missing assets
- Both scripts use Anthropic to generate tags across:
  - Deal stage
  - Persona
  - Competitor
  - Challenge

### 3) Worker backend (`worker/index.js`)
Handles CORS-safe server-side operations and keeps tokens out of the browser:
- HubSpot API calls
- Anthropic API proxy calls
- GitHub Actions workflow dispatch to `add-asset.yml`
- Asana task creation for content-gap requests

## Local Development

No build step is required for the frontend.

### Prerequisites
- Python `3.12+` (for scripts)
- `requests` and `anthropic` Python packages
- A modern browser (for `index.html` / `add.html`)

### Run the app locally
1. Open `index.html` in your browser.
2. Enter your Anthropic API key in the UI when prompted.
3. Configure `APPS_SCRIPT_URL` in `index.html` if you want HubSpot/proxy features enabled.

Tip: If `APPS_SCRIPT_URL` is empty/unreachable, basic in-page behavior still works, but proxy-backed actions will fail.

### Run automation scripts locally
Install dependencies:

```bash
pip install requests anthropic
```

Add a single asset:

```bash
ANTHROPIC_API_KEY=your_key \
ASSET_URL="https://hubs.ly/your-link" \
ASSET_TITLE="Optional title" \
python add_asset.py
```

Sync from sitemap:

```bash
ANTHROPIC_API_KEY=your_key python sync_assets.py
```

Both scripts modify `index.html` directly.

## GitHub Actions

### Add one asset manually
Workflow: `.github/workflows/add-asset.yml`
- Trigger: manual (`workflow_dispatch`)
- Inputs: `url`, optional `title`
- Secret required: `ANTHROPIC_API_KEY`
- Output: commits and pushes updated `index.html` if changed

### Daily sync
Workflow: `.github/workflows/sync-assets.yml`
- Trigger: daily cron + manual dispatch
- Secret required: `ANTHROPIC_API_KEY`
- Output: commits and pushes updated `index.html` if new URLs are found

### Worker deploy
Workflow: `.github/workflows/deploy-worker.yml`
- Trigger: push to `main` affecting `worker/**` + manual dispatch
- Uses Wrangler Action to deploy `worker/`

## Cloudflare Worker Setup

Config file: `worker/wrangler.toml`

### Required repo/environment secrets for deploy workflow
- `CLOUDFLARE_API_TOKEN`
- `CLOUDFLARE_ACCOUNT_ID`
- `HUBSPOT_TOKEN`
- `ASANA_TOKEN`
- `ANTHROPIC_API_KEY`

### Runtime Worker variables/secrets
In Worker runtime, ensure:
- `GITHUB_REPO` (configured in `wrangler.toml`, default `halle-ka/sydecar-content-finder`)
- `HUBSPOT_TOKEN`
- `ASANA_TOKEN`
- `ANTHROPIC_API_KEY`
- `GH_PAT` (GitHub token used by Worker when dispatching add-asset workflow)

Note: `worker/index.js` currently expects `GH_PAT` for GitHub dispatch.

## API Actions Exposed by Worker

POST JSON to Worker URL with one of:
- `{"action":"search","query":"..."}`
- `{"action":"get_deal","dealId":"..."}`
- `{"action":"get_contact","contactId":"..."}`
- `{"action":"add_asset","url":"...","title":"..."}`
- `{"action":"claude","messages":[...],"max_tokens":1024}`
- `{"action":"content_request","topic":"...", ...}`

## Notes and Caveats

- `index.html` is intentionally monolithic and self-contained for zero-build deployment.
- `add.html` posts directly to `PROXY_URL`; Worker defaults to `add_asset` when no `action` is provided.
- Asset source of truth for the app is the `ASSETS` array inside `index.html`.

## License

Add a license file if this repository is intended for open distribution.
