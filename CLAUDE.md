# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This is a freshly-scaffolded Rails 8 app (`rails new`) — no models, controllers, or views beyond
defaults exist yet. `README.md` is the spec for what's being built: read it in full before starting
work, since it describes the data model, storage architecture, and page-by-page requirements that
aren't yet reflected in the code. Solo developer project (Tomas).

## Stack

- Ruby 3.3.5, Rails ~8.1.3, PostgreSQL, Propshaft (asset pipeline)
- Hotwire only — Turbo + Stimulus, importmap-rails. **No React, no JS bundler, no JS frameworks.**
- Devise for auth (single admin user), Pundit optional at this scale
- Solid Queue / Solid Cache / Solid Cable (DB-backed, no Redis)
- Active Storage with per-attachment services (see Storage below)
- Deploy target is Render with GitHub Actions CI (per README) — note the repo also has Kamal/Docker
  files (`config/deploy.yml`, `Dockerfile`, `.kamal/`) from the Rails default generator; confirm
  with the user which deploy path is actually in use before relying on either.

## Commands

```bash
bin/setup                          # bundle install, db:prepare, clear logs/tmp, start bin/dev
bin/dev                            # start dev server (bin/rails server)
bin/rails db:prepare                # create/migrate dev db

bin/rails test                      # run unit/integration tests
bin/rails test test/models/foo_test.rb          # single file
bin/rails test test/models/foo_test.rb:12       # single test at line
bin/rails test:system               # system tests (Capybara + Selenium)

bin/rubocop                         # lint (rubocop-rails-omakase style)
bin/rubocop -a                      # autocorrect
bin/brakeman --no-pager             # static security scan
bin/bundler-audit                   # gem vulnerability scan
bin/importmap audit                 # JS dependency vulnerability scan

bin/rails credentials:edit           # edit encrypted credentials (Backblaze/Cloudinary/Listmonk creds live here)
```

CI (`.github/workflows/ci.yml`) runs, per push/PR: `brakeman`, `bundler-audit`, `importmap audit`,
`rubocop`, `test`, and `test:system` (against a real Postgres service container). Match these
locally before pushing.

## Architecture (per README — target state, not yet implemented)

### Data model
- **Event**: title, description, event_date (datetime), dice_url, slug (unique, from
  `title + date` via `parameterize`, set once at publish and never changed — used on flyers/ticket
  URLs). Attachments: one `hero_image`, many `recordings` (audio). No archived flag — "next event"
  is the soonest future-dated row; everything else is the archive, shown descending.
- **Release**: title, artist, description, bandcamp_url, slug (same rules). Attachments: one
  `cover_image`, many `previews` (audio).
- **SiteSetting**: singleton row holding `landing_video` attachment and editable text blocks.
- Use `to_param` + `find_by!(slug:)` for routing — no FriendlyId gem needed at this scale.

### Storage (Active Storage, per-attachment services — see `config/storage.yml`)
- `backblaze` (B2, S3-compatible, served via Cloudflare CDN) → `landing_video`, `recordings`, `previews`
- `cloudinary` (URL-based transforms for responsive/AVIF/WebP variants — do not build a libvips
  variant pipeline for these) → `hero_image`, `cover_image`
- `local`/`test` → dev/test only
- Audio/video streams directly from the CDN with range-request support; **never proxy media
  through the Rails app**. Use direct uploads in the admin for large audio/video files.

### Pages
- **Landing** — full-viewport background video, minimal typographic nav (Events / Label / Newsletter)
- **Events** (`/events/:slug`) — next event hero on top, archive below as Turbo Frames that
  lazy-load from the same `show` action on expand
- **Label** (`/label/:slug`) — release cards with cover, audio previews, Bandcamp buy link
- **Newsletter** — signup posts server-side to a self-hosted Listmonk instance (credentials in
  Rails credentials, never client-side); inline success/error via Turbo Stream; double opt-in via
  Listmonk + Amazon SES

### Admin CMS
- Hand-rolled under `Admin::` namespace, Devise-protected, no admin gems
- CRUD for Events/Releases with image/audio upload previews; single edit screen for SiteSetting
- Functional over fancy, but must use the same design tokens as the public site

### Audio
Plain HTML5 `<audio>` elements, styled minimally; a small Stimulus controller only if custom UI
controls are needed. No JS audio libraries.

### Deliberately deferred (design for, don't build yet)
Stripe Checkout → webhook → `Order`/`Ticket` models → `rqrcode` QR codes for on-site ticketing.
Keep `dice_url`/`bandcamp_url` as plain model attributes so they can be swapped for internal
purchase flows later without a redesign.

## Conventions

- CSS: design tokens live as custom properties in `app/assets/stylesheets/tokens.css` — always
  reference tokens, never hardcode colours/spacing/type sizes.
- ERB partials / ViewComponents for reusable UI (release card, event row).
- Turbo Frames for lazy/expandable content; Stimulus reserved for small interactions only.
- Keep controllers RESTful and thin.
