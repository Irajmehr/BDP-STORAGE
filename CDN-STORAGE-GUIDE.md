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

## Privacy & Security

### Current Access Model

This CDN serves **public, unauthenticated content**. Anyone with the full URL can access a file. However, URLs contain **UUIDs** (e.g., `/simulations/cbf08c32-657b-49a3-aaef-1721d682c416/8421d9c4-a6f1-4d14-ad12-b384247a31c8.mp3`) which are not guessable and not indexed by search engines.

### Security Through Obscurity (Current)

- URLs are UUID-based — 2^122 possible values, effectively unguessable
- No directory listing — Netlify returns 404 for folder paths
- GitHub repo is public but files are buried in UUID folders
- No search engine indexing (no sitemap, no links from public pages)

### Protecting Sensitive Assets

For assets that require actual access control, use one of these approaches:

**Option 1: Signed URLs (Recommended for Production)**

Generate time-limited signed URLs from your backend:

```javascript
// Backend generates a signed URL with expiry
import crypto from 'crypto';

function signedCdnUrl(path, expiresInSeconds = 3600) {
  const expires = Math.floor(Date.now() / 1000) + expiresInSeconds;
  const signature = crypto
    .createHmac('sha256', process.env.CDN_SIGNING_SECRET)
    .update(`${path}:${expires}`)
    .digest('hex')
    .substring(0, 16);
  return `https://dynaapi.netlify.app${path}?expires=${expires}&sig=${signature}`;
}
```

Then validate via a Netlify Edge Function (requires Netlify Pro or above).

**Option 2: Serve Through Your Backend (Simple)**

Don't expose CDN URLs to the client. Instead, proxy through your API:

```javascript
// Backend route: GET /api/audio/:simulationId/:segmentId
router.get('/audio/:simId/:segId', authenticate, async (req, res) => {
  const cdnUrl = `https://dynaapi.netlify.app/simulations/${req.params.simId}/${req.params.segId}.mp3`;
  const response = await fetch(cdnUrl);
  res.set('Content-Type', 'audio/mpeg');
  response.body.pipe(res);
});
```

This ensures only authenticated users can access audio, but adds latency.

**Option 3: Netlify Password Protection (Quick)**

Netlify Pro ($19/mo) supports site-wide password protection. Add to `netlify.toml`:

```toml
[build.environment]
  # Requires Netlify Pro plan
  NETLIFY_AUTH_TOKEN = "your-token"
```

### What NOT to Store on CDN

| Content | Store on CDN? | Alternative |
|---------|--------------|-------------|
| Tutorial videos | Yes | Public content |
| Simulation audio (TTS) | Yes | UUID-obscured, not sensitive |
| Avatar images (AI-generated) | Yes | Not personal data |
| Client documents (PDFs) | **NO** | Serve through authenticated API |
| Confidential evidence | **NO** | Backend storage only |
| PII (names, addresses) | **NO** | Database only |
| API keys, credentials | **NEVER** | Environment variables |

### GitHub Repo Visibility

The `BDP-STORAGE` repo is **public**. This is fine because:
- Audio files are TTS-generated speech (not private recordings)
- Videos are tutorial walkthroughs (intended to be public)
- File paths use UUIDs not descriptive names

If you need the repo private, go to GitHub > Settings > Danger Zone > Change visibility. Netlify CLI deploys from local regardless.

---

## Netlify Image CDN

Netlify provides built-in image transformation at the edge — resize, crop, format conversion, and quality optimization without a build step.

**Docs:** https://docs.netlify.com/build/image-cdn/overview/

### URL Format

```
https://dynaapi.netlify.app/.netlify/images?url={source}&{transformations}
```

### Supported Transformations

| Parameter | Values | Purpose |
|-----------|--------|---------|
| `url` | relative path or remote URL | **Required** — source image |
| `w` | integer (pixels) | Width |
| `h` | integer (pixels) | Height |
| `fit` | `contain` (default), `cover`, `fill` | Resize behavior |
| `position` | `top`, `bottom`, `left`, `right`, `center` | Crop anchor for `fit=cover` |
| `fm` | `avif`, `jpg`, `png`, `webp`, `gif`, `blurhash` | Output format |
| `q` | 1-100 (default: 75) | Quality for lossy formats |

### Examples

```bash
# Resize avatar to 200x200 thumbnail
https://dynaapi.netlify.app/.netlify/images?url=/litcasepro/images/avatar-judge.png&w=200&h=200&fit=cover

# Convert to WebP for smaller file size
https://dynaapi.netlify.app/.netlify/images?url=/litcasepro/images/courtroom-bg.png&fm=webp&q=80

# Generate blurhash placeholder
https://dynaapi.netlify.app/.netlify/images?url=/litcasepro/images/courtroom-bg.png&fm=blurhash
```

### Remote Images

To transform images from external sources, configure allowed domains in `netlify.toml`:

```toml
[images]
remote_images = ["https://your-backend.up.railway.app/.*"]
```

### Rewrite Rules for Clean URLs

```toml
# Rewrite /thumbnails/200/{path} → Image CDN transform
[[redirects]]
from = "/thumbnails/200/*"
to = "/.netlify/images?url=/:splat&w=200&h=200&fit=cover"
status = 200

# Rewrite /webp/{path} → WebP conversion
[[redirects]]
from = "/webp/*"
to = "/.netlify/images?url=/:splat&fm=webp&q=80"
status = 200
```

### Use Cases for BDP Apps

| App | Use Case | URL |
|-----|----------|-----|
| LitCasePro | Avatar thumbnails | `/.netlify/images?url=/litcasepro/images/judge.png&w=80&h=80&fit=cover` |
| LitCasePro | Courtroom background (compressed) | `/.netlify/images?url=/litcasepro/images/courtroom.png&fm=webp&q=60` |
| eDiscovery | Document preview thumbnails | `/.netlify/images?url=/ediscovery/thumbnails/doc-123.png&w=300` |
| DOQA | Screenshot previews | `/.netlify/images?url=/doqa/screenshots/step-5.png&w=400&fm=webp` |

### Limitations

- No Split Testing support
- Not HIPAA-compliant
- Works with static sites (no build step needed)
- Response codes: 200 (new), 304 (cached), 404 (invalid params)

---

## Rules

1. **App isolation** — Never write to another app's folder
2. **Entity-based paths** — Use UUIDs or entity IDs as folder names, not dates or sequential numbers
3. **No secrets** — Never store API keys, credentials, or PII
4. **No confidential documents** — Only store generated/public assets (TTS audio, AI images, tutorial videos)
5. **File naming** — Use the entity ID as filename (e.g., `{segment-id}.mp3`), not descriptive names that may collide
6. **Git** — Commit and push changes after deploying so GitHub serves as backup
7. **Size limit** — Netlify free tier: 100GB bandwidth/month, 500MB deploy size. Keep assets reasonable
8. **CORS** — All paths have `Access-Control-Allow-Origin: *` so any frontend can fetch assets
9. **Privacy** — Use UUID-based paths. Never use real names, case numbers, or identifiable info in filenames

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
