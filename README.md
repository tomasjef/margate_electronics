# Margate Electronics

Brochure/portfolio/archival site for Margate Electronics — events, label releases, newsletter. Typography-led, minimal aesthetic. Custom admin CMS. Solo developer project.

## Stack

- Ruby on Rails (latest stable), PostgreSQL
- Hotwire (Turbo + Stimulus) — **no React, no JS frameworks**
- Devise (single admin user), Pundit optional at this scale
- Solid Queue for background jobs
- Deploy: Render, CI via GitHub Actions
- CSS: design tokens as CSS custom properties in `app/assets/stylesheets/tokens.css` — always reference tokens, never hardcode colours/spacing/type sizes

## Data model

- **Event** — title, description (rich text ok), event_date (datetime), dice_url (string), slug (string, unique, indexed). Attachments: `hero_image` (one), `recordings` (many audio).
  - No archived flag. "Next event" = soonest future-dated event. Archive = past events, descending.
  - Every event has its own show page at `/events/:slug` (slug from title + date via parameterize, set once and never changed after publish — these URLs go on flyers and listings, and will become the ticket booking page when Stripe lands). Archive expansion Turbo Frames lazy-load from the same show action.
- **Release** — title, artist, description, bandcamp_url, slug (string, unique, indexed). Attachments: `cover_image` (one), `previews` (many audio). Show page at `/label/:slug`, same slug rules.
- Use `to_param` + `find_by!(slug:)`; no FriendlyId gem needed at this scale.
- **SiteSetting** — singleton row. Attachment: `landing_video` (one). Editable text blocks as columns or jsonb.

## Storage (Active Storage, per-attachment services)

Three services in `config/storage.yml`:

- `backblaze` — Backblaze B2 via S3-compatible API (custom `endpoint`, B2 application key). Used for: `landing_video`, `recordings`, `previews`. Served through Cloudflare CDN (free egress via Bandwidth Alliance).
- `cloudinary` — official `cloudinary` gem. Used for: `hero_image`, `cover_image`. Use URL-based transformations for variants (responsive sizes, auto WebP/AVIF) — do not build libvips variant pipeline for these.
- `local` — dev/test.

Audio/video must stream directly from CDN with range-request support (public or long-lived signed URLs) — never proxy media through the Rails app.
Use direct uploads in the admin for large audio/video files.

## Pages

- **Landing** — full-viewport muted background video (compressed mp4/webm ~1080p), minimal typographic menu: Events, Label, Newsletter.
- **Events** — next event on top (hero image, title, description, "Buy tickets" → dice_url). Below: archived events list. Each archived event is a lazy-loaded Turbo Frame that expands on click to show details + HTML5 audio players for recordings.
- **Label** — cards/list of releases: cover image, audio previews, "Buy" → bandcamp_url.
- **Newsletter** — signup form. POSTs server-side to self-hosted Listmonk public subscription API (credentials/instance URL in Rails credentials, never exposed client-side). Inline success/error via Turbo Stream. Double opt-in handled by Listmonk + Amazon SES.

## Admin CMS (hand-rolled, no admin gems)

- `Admin::` namespace, Devise-protected.
- CRUD for Events and Releases (image/audio upload with previews).
- Single edit screen for SiteSetting (replace landing video, edit text blocks).
- Functional over fancy, but consistent with site tokens.

## Audio players

Plain HTML5 `<audio>`, styled minimal/typographic; small Stimulus controller if custom UI needed. No JS audio libraries.

## Future (design for, don't build yet)

- On-site ticket purchases: Stripe Checkout → webhook → `Order`/`Ticket` models → `rqrcode` QR codes on ticket email/PDF.
- Keep all buy URLs as model attributes so DICE/Bandcamp can be swapped for internal flows without redesign.

## Conventions

- ERB partials / ViewComponents for reusable UI (release card, event row).
- Turbo Frames for lazy/expandable content; Stimulus for small interactions only.
- Keep controllers RESTful and thin.
