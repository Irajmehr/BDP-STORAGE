# BDP-STORAGE — Shared CDN for Static Assets

**CDN URL:** `https://dynaapi.netlify.app`
**Local Repo:** `C:\Users\Admin\AppData\Local\Temp\BDP-STORAGE`
**GitHub:** `https://github.com/Irajmehr/BDP-STORAGE`
**Netlify Site:** `dynaapi`

---

## Purpose

Shared CDN for storing and serving static assets (audio, images, videos, PDFs) across all BDP apps. Deployed via Netlify from local directory — no build step required.

## Folder Structure

Every app MUST use its own root folder. Within that, organize by asset type and entity ID.

```
BDP-STORAGE/
├── CDN-STORAGE-GUIDE.md          # This file
├── netlify.toml                  # Cache headers per path
│
├── litcasepro/                   # LitCasePro app
│   ├── simulations/              # Hearing simulation audio
│   │   └── {simulation-id}/      # One folder per simulation
│   │       ├── {segment-id}.mp3  # TTS audio per segment
│   │       └── ...
│   ├── videos/                   # Tutorial/demo videos
│   │   ├── LitCasePro-Full-Tour.webm
│   │   └── ...
│   └── images/                   # Generated images (avatars, backgrounds)
│       └── {entity-id}.png
│
├── ediscovery/                   # eDiscovery app (example)
│   ├── exports/                  # Exported production sets
│   └── thumbnails/               # Document preview thumbnails
│
├── jobq/                         # JobQ app (example)
│   └── reports/                  # Generated report PDFs
│
└── doqa/                         # DOQA app (example)
    └── screenshots/              # Captured screenshots
```

## How to Add Assets from Your App

### 1. Copy files to the local repo

```javascript
import fs from 'fs';
import path from 'path';

const CDN_REPO = process.env.CDN_LOCAL_REPO || 'C:/Users/Admin/AppData/Local/Temp/BDP-STORAGE';
const APP_NAME = 'litcasepro'; // YOUR app name

// Create folder structure
const destDir = path.join(CDN_REPO, APP_NAME, 'simulations', simulationId);
fs.mkdirSync(destDir, { recursive: true });

// Copy file
fs.copyFileSync(localAudioPath, path.join(destDir, `${segmentId}.mp3`));
```

### 2. Deploy to CDN

```bash
cd /tmp/BDP-STORAGE
npx netlify-cli deploy --prod --dir . --no-build
```

Or from your app's publish endpoint:

```javascript
import { execSync } from 'child_process';
execSync(`cd "${CDN_REPO}" && npx netlify-cli deploy --prod --dir . --no-build`);
```

### 3. Build the CDN URL

```
https://dynaapi.netlify.app/{app-name}/{asset-type}/{entity-id}/{filename}
```

Example:
```
https://dynaapi.netlify.app/litcasepro/simulations/cbf08c32/seg-001.mp3
https://dynaapi.netlify.app/litcasepro/videos/Full-Tour.webm
https://dynaapi.netlify.app/ediscovery/exports/production-set-42.zip
```

### 4. Cache-busting

For assets that may be updated (audio, images), always append `?v={timestamp}`:

```javascript
const cacheBust = Math.floor(Date.now() / 1000);
const cdnUrl = `https://dynaapi.netlify.app/${appName}/simulations/${simId}/${segId}.mp3?v=${cacheBust}`;
```

## Cache Headers (netlify.toml)

Configure per-path cache rules in `netlify.toml`:

```toml
# Videos — immutable, cache forever (change filename to bust)
[[headers]]
  for = "/*/videos/*"
  [headers.values]
    Access-Control-Allow-Origin = "*"
    Cache-Control = "public, max-age=31536000, immutable"

# Audio/simulations — short cache, use ?v= to bust
[[headers]]
  for = "/*/simulations/*"
  [headers.values]
    Access-Control-Allow-Origin = "*"
    Cache-Control = "public, max-age=300, must-revalidate"

# Images — medium cache
[[headers]]
  for = "/*/images/*"
  [headers.values]
    Access-Control-Allow-Origin = "*"
    Cache-Control = "public, max-age=86400, must-revalidate"
```

## Rules

1. **App isolation** — Never write to another app's folder
2. **Entity-based paths** — Use UUIDs or entity IDs as folder names, not dates or sequential numbers
3. **No secrets** — This is a PUBLIC CDN. Never store API keys, credentials, or PII
4. **File naming** — Use the entity ID as filename (e.g., `{segment-id}.mp3`), not descriptive names that may collide
5. **Git** — Commit and push changes after deploying so GitHub serves as backup
6. **Size limit** — Netlify free tier: 100GB bandwidth/month, 500MB deploy size. Keep assets reasonable
7. **CORS** — All paths have `Access-Control-Allow-Origin: *` so any frontend can fetch assets

## Deployment

```bash
# From the repo directory:
cd /tmp/BDP-STORAGE
npx netlify-cli deploy --prod --dir . --no-build

# Output:
# Deployed to production URL: https://dynaapi.netlify.app
```

No build step. No GitHub push required for CDN to work (Netlify deploys from local). GitHub push is for backup only.

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Old audio still playing | Add `?v={timestamp}` cache-bust to URL |
| 404 on CDN | File not in local repo or deploy not run |
| Git push denied | Run `gh auth login` to refresh credentials |
| Deploy fails | Check `npx netlify-cli status` — may need `npx netlify-cli login` |
