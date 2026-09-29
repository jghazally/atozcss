# Travel blog: plan

## Goal
Build a side income that pays for more holidays. Money comes from affiliate links (places we stayed, tours, gear we used), then display ads, then our own products.

To get there: find every holiday in the Apple Photos library, pick the best photos, and have Claude draft **search-focused** posts (stay reviews, itineraries, where-to-stay guides, packing lists) from what we actually did. We review each draft and add our own details before merging to publish.

## Monetisation strategy

### Why first-hand content matters
Google has cut traffic to thin and AI-mass-produced travel content since 2023, and AI answers in search take more clicks every year. What still ranks and converts is proof we were really there: our own photos, what we actually paid, specific likes and gripes. So Claude writes the first draft and we add the details only we know. Every post needs at least one of these before it's published.

### One trip → many posts
Each trip should become several posts aimed at things people search for, not one diary entry:
| Post type | Example | Search intent | Money |
|---|---|---|---|
| Stay review | "Hotel X review: honest thoughts after 4 nights" | booking | accommodation affiliate (highest) |
| Where to stay | "Where to stay in Queenstown: areas + picks" | booking | accommodation + map widget |
| Itinerary | "5 days in Rarotonga itinerary" | planning | tours, car hire, insurance |
| Things to do | "Best day trips from Kyoto" | planning | tours (GetYourGuide/Viator) |
| Gear / packing | "What we packed for Japan in winter" | buying | Amazon & retailer affiliates |
| Trip story | the diary-style post | brand/email | internal links to the above |

### Income streams, in the order to add them
1. **Accommodation affiliates:** Stay22 (one embed covers Booking.com, Expedia, Hotels.com and more, good for small sites), Booking.com and Expedia Group partner programs, Agoda. *Airbnb no longer runs an affiliate program.*
2. **Tours and activities:** GetYourGuide, Viator, Klook (strong in Asia).
3. **Travel services:** travel insurance, car hire (e.g. DiscoverCars), eSIMs (e.g. Airalo). These often pay a flat fee per sale.
4. **Products:** Amazon Associates (pick the store your readers buy from: .com / .com.au), plus direct programs from gear brands we actually use.
5. **Display ads**, once traffic justifies it: AdSense or Ezoic (no traffic minimum) → Mediavine / Raptive (need tens of thousands of monthly sessions; check their current minimums).
6. **Own products:** paid itineraries or Google Maps lists, printable guides, Lightroom presets made from our editing. Best margin, add later.
7. **Owned audience:** an email list from day one, plus Pinterest (still a big traffic source for travel) and short videos made from the same photos.

### Built into the site
- Structured `stays` / `products` / `activities` data in each post's frontmatter → `<StayCard>`, `<ProductCard>` components with affiliate buttons
- `/go/<slug>` redirect links → click tracking in one place, and a dead or changed affiliate link gets fixed once
- schema.org `Review` / `LodgingBusiness` markup, fast pages (Core Web Vitals), sitemap
- Affiliate disclosure on every page with links (required by NZ Fair Trading Act and FTC rules for US readers)
- Analytics: which posts and links earn money, so we write more of those

### Finding what we stayed at and used
- **Photos:** the GPS location of photos taken between ~10pm and 8am usually shows where we slept
- **Gmail (connector is available):** booking confirmations (Booking.com, Airbnb, hotels, airlines, tours) and Amazon orders give exact names, dates and prices paid. That's the best source for stay reviews and gear lists. Only with explicit OK, and only reading receipts and bookings.

### Realistic expectations
Most travel blogs earn little in year one. Search traffic usually takes 6–12+ months to build. Aim to cover hosting in year 1, then grow from there. A focus helps you rank sooner, e.g. NZ/Pacific trips from a Kiwi point of view, or couples' mid-range stays. Declare the income to IRD.

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

### 4b. Match stays and products (`scripts/find-stays`)
- Work out where we slept each night from the GPS of night-time photos, then cross-check against booking emails in Gmail
- Look up affiliate links for each stay, tour and product; leave a gap where none exist

### 5. Draft posts (`scripts/draft-post`)
- Send Claude the trip metadata, stays, products, chosen photos and captions, plus our notes ("rained all week", "best ramen ever", what we paid)
- It plans a set of search-focused posts for the trip (see the table above), then writes MDX with frontmatter (title, dates, countries, hero, gallery, stays, products)
- Each draft marks the places that need our own details with `TODO(us):` markers. Nothing merges while those are left.
- Each trip gets its own branch and PR, which you edit and merge

### 6. Launch
- Domain, OG images, RSS, sitemap
- Later: a "new trip" flow for future holidays (drop photos in, rerun the pipeline)

## Open questions
- Focus: what's the angle (NZ/Pacific, couples, budget vs. luxury)?
- OK to scan Gmail for booking confirmations and orders?
- Blog name and domain?
- Should this repo stay public? It'll hold post text and image URLs only, never raw photos.
- Can people (family, friends) appear in published photos?
- Voice: first person, how chatty, and roughly how long per post?
- Include the trip map / exact places, or keep it at city level?
