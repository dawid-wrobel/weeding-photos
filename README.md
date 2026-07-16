# Wedding Photo Gallery

Serverless photo-sharing app for events, built on Cloudflare's edge stack. Guests scan a QR code and upload photos straight from their phone — no accounts, no app install. HEIC/HEIF from iPhones is converted client-side before upload.

Built to handle bursty uploads from many guests at once, without a traditional backend.

| Home | Upload | Gallery |
|---|---|---|
| ![Home](screenshots/home.png) | ![Upload](screenshots/upload.png) | ![Gallery](screenshots/gallery.png) |

## Architecture

```
Browser (guest)
     │
     │  POST /presign  →  Worker returns a signed PUT URL (1h TTL)
     ▼
Cloudflare Worker
     │
     │  PUT {file}  →  direct browser → R2 upload
     ▼
Cloudflare R2  (S3-compatible storage)
     │
     │  GET /photos  →  public CDN URLs
     ▼
Cloudflare Pages  (React SPA)
```

The Worker never touches file bytes — it just signs upload URLs. All uploads go browser → R2 directly, so Worker CPU/memory limits are never a bottleneck, and cost stays flat no matter how many people upload at once.

## Stack

| Layer | Tech |
|---|---|
| Frontend | React 18, Vite, CSS-in-JS |
| Edge API | Cloudflare Workers |
| Storage | Cloudflare R2 (S3-compatible) |
| Request signing | `aws4fetch` (Sig V4) |
| HEIC conversion | `heic2any` (client-side, WASM) |
| Hosting | Cloudflare Pages |

## Implementation notes

**Presigned uploads** — Worker signs an S3 PUT URL via `aws4fetch` against R2's endpoint. It never holds the file in memory, so upload size isn't limited by Worker resources.

**HEIC/HEIF** — detected by MIME/extension, converted to JPEG in-browser via WASM before the presign request. No server-side image processing needed.

**Upload progress** — used `XMLHttpRequest.upload.onprogress` instead of Fetch, since Fetch doesn't expose upload progress events. Tracked per-file and in aggregate.

**CORS** — permissive in dev, should be locked to the Pages domain in production.

## Project structure

```
wedding-frontend/
├── src/
│   ├── App.jsx       — Home, Upload, Gallery, Lightbox
│   └── api.js        — HEIC normalization, presign, XHR upload, photo fetch
├── .env.example
└── vite.config.js

wedding-worker/
├── src/
│   └── index.js      — GET /photos, POST /presign, CORS preflight
└── wrangler.jsonc
```

## Setup

**Requirements:** Node.js 18+, Cloudflare account with R2 enabled.

### R2 Bucket

1. Create bucket, enable public access
2. Apply CORS policy:
```json
[{
  "AllowedOrigins": ["*"],
  "AllowedMethods": ["GET", "PUT", "HEAD"],
  "AllowedHeaders": ["Content-Type", "Content-Length"],
  "ExposeHeaders": ["ETag"],
  "MaxAgeSeconds": 3600
}]
```
3. Generate an R2 API token with Object Read & Write scope

### Worker

```bash
cd wedding-worker
npm install
npx wrangler secret put R2_ACCESS_KEY_ID
npx wrangler secret put R2_SECRET_ACCESS_KEY
npx wrangler secret put BUCKET_PUBLIC_URL    # https://pub-xxxx.r2.dev
npx wrangler secret put ACCOUNT_ID
npx wrangler deploy
```

### Frontend

```bash
cd wedding-frontend
npm install
cp .env.example .env.local
# configure .env.local
npm run dev
npm run build     # → dist/
```

Deploy `dist/` to Cloudflare Pages (GitHub integration or direct upload). Set env vars in Pages project settings.

```
VITE_WORKER_URL=https://your-worker.your-subdomain.workers.dev
VITE_COUPLE_NAMES=Name & Name
VITE_WEDDING_DATE=DD MONTH YYYY
```

## Cost

| Resource | Usage | Cost |
|---|---|---|
| R2 Storage | 100 GB | ~$1.50/mo |
| R2 Operations | ~10k PUT, ~100k GET | < $0.50 |
| Workers | < 100k req/day | Free |
| Pages | Unlimited requests | Free |
| Egress | Unlimited (R2 → browser) | Free |
| **Total** | | **~$2/mo** |

## Downloading photos post-event

```bash
rclone config
# provider: S3, Cloudflare, R2 access key + secret + endpoint

rclone copy wedding:your-bucket-name/photos ./local-folder
```

## Security

- No auth — relies on the Pages subdomain not being guessable
- Presigned PUT URLs expire after 1h, scoped to a single key
- Worker checks `contentType` starts with `image/` before signing
- Guests can only PUT new keys, not overwrite existing ones
- CORS should be locked down for production

## License

MIT
