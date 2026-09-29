# Travel blog: plan

## Goal
Find every holiday in my Apple Photos library, pick the best photos from each one, and have Claude write a draft post per trip. I review each draft and merge it to publish.

## Decisions (2026-09-29)
- **Photos:** iCloud / Apple Photos. Apple has no public API, so we export locally with `osxphotos`.
- **Stack:** Next.js (App Router) + MDX on Vercel. Published images live in Vercel Blob.
- **Publishing:** Claude drafts each post as a PR. Merging the PR publishes it.
- **Repo:** `jghazally/atozcss`, wiped and started again from scratch.

## How it fits together

```
Mac (local)                         Repo / cloud
───────────                         ────────────
Apple Photos
  │ osxphotos export (+ JSON sidecars: GPS, date, place, labels)
  ▼
export/  ──► 1. detect-trips  ──► trips.json  (you review/rename/merge)
                 │
                 ▼
             2. curate  (dedupe bursts, drop screenshots/blurry,
                 │        Claude vision picks 8–15 per trip)
                 ▼
             3. publish-images (resize, strip EXIF/GPS) ──► Vercel Blob
                 │
                 ▼
             4. draft-post (Claude API) ──► content/trips/<slug>.mdx ──► PR
                                                                          │
                                                        Next.js site ◄────┘ merge
```

The pipeline runs on the Mac. The library is too big to upload, and the raw photos never leave the machine. Only the resized photos you pick, with location data stripped, get published.

## Phases

### 0. Reset and scaffold
- [x] Wipe the old A to Z CSS files
- [ ] Scaffold Next.js + TypeScript + Tailwind + MDX
- [ ] Pages: home (trip grid + map), `/trips/[slug]`, about
- [ ] Hook up the Vercel project

### 1. Export (you, one-off, ~1 hr + download time)
- `brew install pipx && pipx install osxphotos`
- In Photos > Settings > iCloud, turn on "Download Originals to this Mac" (or use `--download-missing`)
- `osxphotos export ~/travel-export --download-missing --sidecar json --convert-to-jpeg --skip-edited`
- Optional: `--from-date`/`--to-date` to start small, e.g. one year

### 2. Trip detection (`scripts/detect-trips`)
- Read the sidecars: timestamp, GPS, place name, albums
- Home base = Wellington. A trip is ≥2 consecutive days with photos more than ~100 km from home. Allow gaps of up to 2 days before closing a trip.
- Use albums you've already made (e.g. "Japan 2019") as a strong signal
- Output `trips.json`: dates, places, photo count, a suggested title. **You confirm it before anything else runs.**

### 3. Curation (`scripts/curate`)
- Drop screenshots, receipts, documents and near-duplicates (perceptual hash)
- Score sharpness and exposure, then ask Claude (vision) to pick a hero shot and 8–15 gallery shots with captions
- Skip photos where people are the main subject unless you allow them

### 4. Publish images
- Resize (e.g. 2400px + thumbnails), convert to WebP/AVIF, **strip all EXIF/GPS**, upload to Vercel Blob
- Save the resulting URLs in `content/trips/<slug>.json`

### 5. Draft posts (`scripts/draft-post`)
- Send Claude the trip metadata, places, dates, chosen photos and captions, plus any notes from you ("rained all week", "best ramen ever")
- It writes MDX with frontmatter (title, dates, countries, hero, gallery) in a consistent voice
- Each trip gets its own branch and PR, which you edit and merge

### 6. Launch
- Domain, OG images, RSS, sitemap
- Later: a "new trip" flow for future holidays (drop photos in, rerun the pipeline)

## Open questions
- Blog name and domain?
- Should this repo stay public? It'll hold post text and image URLs only, never raw photos.
- Can people (family, friends) appear in published photos?
- Voice: first person, how chatty, and roughly how long per post?
- Include the trip map / exact places, or keep it at city level?
