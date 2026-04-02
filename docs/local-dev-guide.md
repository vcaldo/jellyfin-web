# Local Development & Testing Guide

How to build and test the custom Jellyfin Web (with the "Play on Discord" feature) locally.

---

## Prerequisites

| Tool | Required version |
|---|---|
| Node.js | >= 24.0.0 |
| npm | >= 11.0.0 |
| Docker + Docker Compose | Any recent version |

> **Note:** The project enforces Node 24+. If your system Node is older (check with `node --version`), use [nvm](https://github.com/nvm-sh/nvm) to install the correct version:
> ```bash
> nvm install 24
> nvm use 24
> ```

---

## Option 1 — Dev Server (fastest, hot reload)

Best for UI iteration. Webpack serves the app in memory with live reload; no Docker needed.

### 1. Install dependencies

```bash
cd jellyfin-web
npm install
```

### 2. Set the Discord bot URL (optional)

```bash
export DISCORD_BOT_URL=http://localhost:3000
```

If omitted it defaults to `http://discord-bot:3000`, which only resolves inside Docker. Set it to `localhost` when testing the bot outside of Docker.

### 3. Start the dev server

```bash
npm run serve
```

Webpack starts at `http://localhost:8080`. The app proxies API calls to a Jellyfin server — point your browser to:

```
http://localhost:8080/?serverAddress=http://YOUR_JELLYFIN_HOST:8096
```

Replace `YOUR_JELLYFIN_HOST` with the IP or hostname of your Jellyfin server.

---

## Option 2 — Production Build + Docker Compose (closest to real deployment)

Use this to test the full stack including the Discord bot and Jellyfin server together.

### 1. Install dependencies and build

```bash
cd jellyfin-web
npm install
DISCORD_BOT_URL=http://discord-bot:3000 npm run build:production
```

The compiled output lands in `dist/`.

### 2. Write a Docker Compose file

Create a `docker-compose.yml` in a directory **above** `jellyfin-web/` (e.g. `~/lab/`):

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    volumes:
      - ./jellyfin-web/dist:/jellyfin/jellyfin-web   # mount our custom build
      - jellyfin-config:/config
      - jellyfin-cache:/cache
      - /path/to/your/media:/media:ro                 # point to your media library
    ports:
      - "8096:8096"
    restart: unless-stopped

  discord-bot:
    image: your-discord-bot:latest   # replace with your bot image
    ports:
      - "3000:3000"
    restart: unless-stopped

volumes:
  jellyfin-config:
  jellyfin-cache:
```

> The key line is the `jellyfin-web/dist` volume mount — it replaces the default Jellyfin web UI with our custom build.

### 3. Start the stack

```bash
docker compose up -d
```

Open `http://localhost:8096` in your browser and log in.

---

## Testing the "Play on Discord" Feature

1. Navigate to any **Movie**, **Episode**, or **Video** item in the Jellyfin UI
2. The **headphones icon** ("Play on Discord") button appears next to the standard play controls
3. Optionally select a subtitle track from the dropdown
4. Click the button — open the browser DevTools console (`F12`) and look for `[Play on Discord]` log lines:

```
[Play on Discord] Sending payload: { ... }
[Play on Discord] Bot response: { status: "ok", ... }
```

5. On a successful bot response, two new buttons appear:
   - **Pause/Resume** (toggles icon and calls `/api/pause` or `/api/resume`)
   - **Stop** (calls `/api/stop` and hides the controls)

### Testing without a bot (stub server)

If you don't have the Discord bot running yet, you can use `npx json-server` or a simple stub to accept the requests and return a valid response:

```bash
# Quick stub with netcat — accepts one request and responds OK
while true; do
  echo -e 'HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n\r\n{"status":"ok","message":"stub"}' \
    | nc -l 3000
done
```

Or with Node:

```bash
node -e "
const http = require('http');
http.createServer((req, res) => {
  let body = '';
  req.on('data', d => body += d);
  req.on('end', () => {
    console.log(req.method, req.url, body ? JSON.parse(body) : '');
    res.writeHead(200, {'Content-Type': 'application/json'});
    res.end(JSON.stringify({status:'ok', message:'stub'}));
  });
}).listen(3000, () => console.log('Stub bot listening on :3000'));
"
```

Set `DISCORD_BOT_URL=http://localhost:3000` before running the dev server or build, then click the button — the stub will print the full payload to its console.

---

## Rebuilding After Code Changes

When you change `src/` files:

- **Dev server**: changes are picked up automatically (hot reload)
- **Production build**: re-run `npm run build:production` then restart the Docker stack:

```bash
DISCORD_BOT_URL=http://discord-bot:3000 npm run build:production
docker compose restart jellyfin
```

---

## Useful Commands

| Command | What it does |
|---|---|
| `npm run serve` | Start webpack dev server with hot reload |
| `npm run build:production` | Production build to `dist/` |
| `npm run build:development` | Development build to `dist/` (no minification) |
| `npm run build:check` | TypeScript type-check only (no output files) |
| `npm run lint` | Run ESLint |
| `npm test` | Run unit tests |
