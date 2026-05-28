# Shopify Token CLI

A Windows CLI tool to extract **permanent offline access tokens** from Shopify stores. No coding required — just run the `.exe`, answer a few prompts, and get your token.

***

## What's in this zip

```
shopify-token-cli/
├── shopify-token-cli.exe   ← the tool you run
├── package.json            ← required by Shopify CLI (do not delete or move)
└── README.md               ← this file
```

> **Keep all three files in the same folder.** Do not move the `.exe` out on its own.

***

## Prerequisites

Install these before running the tool. The tool will tell you if something is missing.

### 1. Tunnel tool (ngrok or Cloudflare Tunnel)

The tool needs one of these to create a public HTTPS URL so Shopify can redirect back to your machine after OAuth. **You only need one** — the tool auto-detects whichever is installed. If both are installed, it will ask you to choose.

#### Option A — Cloudflare Tunnel (Recommended)

Easier to set up — no account or auth token required.

- Download and install from: developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads
- Verify: run `cloudflared --version` in your terminal

That's it. No login or token needed.

#### Option B — ngrok

Requires a free account and auth token setup.

- Download and install from: ngrok.com/download
- Verify: run `ngrok version` in your terminal
- Create a free account at ngrok.com, then authenticate:

```
ngrok config add-authtoken YOUR_TOKEN_HERE
```

Your auth token is on the ngrok dashboard under Getting Started.

> **Tip — ngrok browser warning:** The first time you open the install URL with ngrok, you may see a warning page titled something like "You are about to visit..." with an orange/yellow header. This is a normal ngrok safety screen — it is **not an error**. Just click the **Visit Site** button to continue. The tool attempts to bypass this automatically, but if it appears, clicking Visit Site is safe.

### 2. Shopify CLI

Pushes the app configuration (redirect URLs, scopes) to Shopify Partner Dashboard automatically.

Install via npm (Node.js must be installed first):

```
npm install -g @shopify/cli @shopify/theme
```

Verify with `shopify version`, then log in to your Partner account:

```
shopify auth login
```

This opens a browser to authenticate. Do this once and it stays logged in.

### 3. A Shopify Partner App

The tool works with apps already created in Shopify Partner Dashboard. Have these ready before you run:

| What you need | Where to find it |
|---|---|
| Client ID | Partner Dashboard → Your App → API credentials |
| Client Secret | Partner Dashboard → Your App → API credentials |
| Scopes | The permissions needed, e.g. `read_products,write_orders` |
| Shop domain | The store to install on, e.g. `mystore.myshopify.com` |

This tool works with existing apps only — it does not create new Partner apps.

***

## How to run

Open a terminal in the folder containing the `.exe` and run:

```
shopify-token-cli.exe
```

Or double-click it from File Explorer.

***

## Steps walkthrough

### Step 1 — Tunnel Setup

The tool checks which tunneling tool you have installed and starts a tunnel on port 5000 automatically.

| Situation | What happens |
|---|---|
| Only Cloudflare installed | Auto-starts Cloudflare tunnel silently |
| Only ngrok installed | Auto-starts ngrok tunnel silently |
| Both installed | Asks you to choose which to use |
| Neither installed | Prints install instructions and exits |

After a few seconds you will see a public URL like:
- `https://xxxx.ngrok-free.app` (ngrok)
- `https://xxxx.trycloudflare.com` (Cloudflare)

**Requires:** At least one tunnel tool installed (see Prerequisites).

### Step 2 — App Configuration

You will be prompted to enter your app details one by one:

- **App Name** — your app's display name (anything descriptive)
- **Client ID** — from Partner Dashboard API credentials
- **Client Secret** — from Partner Dashboard API credentials *(input is hidden while typing)*
- **Scopes** — comma-separated permissions, e.g. `read_products,write_orders`
- **Shop domain** — the store to install on, e.g. `mystore.myshopify.com`

Nothing is saved to disk — all credentials are held in memory only for the current session.

### Step 3 — Deploy Config to Shopify

The tool writes a `shopify.app.toml` file with your tunnel URL registered as the app URL and redirect URL, then runs:

```
shopify app deploy --config shopify.app.toml --allow-updates
```

This tells Shopify to accept the OAuth redirect to your tunnel URL. **This step must succeed** — if it fails, Shopify will block the OAuth flow with a `redirect_uri is not whitelisted` error.

**Requires:** Shopify CLI installed and logged in via `shopify auth login`.

### Step 4 — Server starts

A local Express server starts on port 5000 and waits for the OAuth callback from Shopify.

### Step 5 — Install URL

The tool prints a URL like:

```
https://xxxx.ngrok-free.app/start?shop=mystore.myshopify.com
```

Copy and open this in your browser. You will briefly see a "Connecting..." screen, then be taken straight to the Shopify install page. Click **Install** or **Update** — Shopify redirects back to the tool automatically.

> **Note:** The URL uses `/start` instead of `/auth/install` directly. This is intentional — it bypasses the ngrok browser warning interstitial automatically so you land straight on the Shopify screen.

***

## Getting your token

After clicking Install in Shopify, your browser loads a results page showing:

- Store Name *(pulled live from Shopify API)*
- Shop Domain
- Client ID
- Client Secret
- Scopes
- **Permanent Access Token** (highlighted in green, with a ✓ Verified badge)

Each field has a **Copy** button. The **Copy All as .env Format** button copies everything formatted and ready to paste into a `.env` file.

The token is also printed in your terminal in `.env` format.

***

## What is a permanent offline token?

The token you receive is a Shopify offline access token. It:

- Does **not** expire
- Gives full API access to the store within the scopes you requested
- Remains valid until the app is uninstalled or the API secret is rotated
- Should be treated like a password — do not share it publicly or commit it to git

***

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `ngrok not found` and `cloudflared not found` | No tunnel tool installed | Install Cloudflare Tunnel (recommended) or ngrok |
| Tunnel URL not found after startup | ngrok auth token missing | Run `ngrok config add-authtoken YOUR_TOKEN` |
| ngrok warning page appears in browser | Normal ngrok interstitial | Click **Visit Site** — it is safe |
| `shopify not found` | Shopify CLI not installed | Run `npm install -g @shopify/cli` |
| Deploy exited with code 1 | Not logged in to Shopify CLI | Run `shopify auth login` |
| `redirect_uri is not whitelisted` | Deploy step failed, URL not registered | Fix the deploy error and re-run the tool |
| `HMAC validation failed` | OAuth request expired or tampered | Re-open the install URL from Step 5 |
| `Missing required OAuth parameters` | Shopify redirect was incomplete | Check scopes are valid and retry |
| Port 5000 already in use | Another process on port 5000 | Kill the other process or restart your machine |

***

## Important notes

- **Run the tool fresh each time** — both ngrok and Cloudflare Tunnel generate a new URL on every run (free plan), so the deploy step re-registers the new URL automatically each session.
- **The tunnel closes when you exit** — when you press Ctrl+C or close the terminal, the tunnel is automatically terminated. No background processes are left running.
- **Do not close the terminal** until you have your token — the local server must stay running to receive the Shopify callback.
- **One token per store** — each store installation gives one offline token. Run the tool once per store.
- **Port 5000 must be free** — make sure nothing else is running on port 5000 before starting.