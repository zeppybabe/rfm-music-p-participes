# Randomize.fm

**Version:** v1.0.0 &nbsp;|&nbsp; **Stack:** SvelteKit 2 · Svelte 5 · SQLite · Three.js · YouTube Data API v3

One button. Infinite music. A music discovery app that plays a random YouTube video on demand, driven by a genre-aware seed query system, a persistent SQLite cache, and a WebGL shader background that reacts to genre.

> **This repository is a public showcase of the Randomize.fm build.**
> The source code is maintained privately. This document covers the architecture, database schema, file structure, and systems design for reference and as a foundation for anyone building something similar.

---

## Live Demo

<!-- Add your live URL here -->
`https://randomize.fm`

---

## What It Does

- Authenticated users click **RANDOMIZE** to get a random YouTube video pulled from a curated seed query pool
- A 24-hour search result cache minimizes YouTube Data API quota consumption
- WebGL GLSL shaders crossfade between genre-matched visual presets (aurora, cosmic drift, neon pulse, etc.)
- Users can import YouTube playlists and randomize from their own track pool
- Listening history is persisted per user
- Sliding-window rate limiting enforced at the request level

---

## Tech Stack

| Layer | Package | Version |
|---|---|---|
| Framework | `@sveltejs/kit` | ^2.57.1 |
| UI | `svelte` | ^5.0.0 |
| Adapter | `@sveltejs/adapter-node` | ^5.5.4 |
| Auth | `lucia` | ^3.0.0 |
| Auth adapter | `@lucia-auth/adapter-drizzle` | latest |
| OAuth provider | `arctic` | ^2.0.0 |
| ORM | `drizzle-orm` | ^0.45.2 |
| Database | `better-sqlite3` | ^11.0.0 |
| 3D / WebGL | `three` | ^0.170.0 |
| CSS | `tailwindcss` | ^4.0.0 |
| Build | `vite` | ^8.0.0 |
| Type checking | `typescript` | ^5.7.0 |
| DB tooling | `drizzle-kit` | ^0.31.10 |

---

## File Structure

```
randomize/
├── src/
│   ├── app.css                          # Design tokens, global reset, SVG grain pattern
│   ├── app.d.ts                         # SvelteKit ambient type declarations
│   ├── app.html                         # HTML shell, font preloads
│   ├── hooks.server.ts                  # Session validation, rate limiting, CSP headers
│   │
│   ├── lib/
│   │   ├── components/
│   │   │   ├── AccessibilityAnnouncer.svelte
│   │   │   ├── CookieBanner.svelte
│   │   │   ├── CSSFallbackBG.svelte     # CSS-only animated background (non-GPU fallback)
│   │   │   ├── ErrorCard.svelte         # Typed error display with retry
│   │   │   ├── HistoryList.svelte       # Per-user listening history panel
│   │   │   ├── PhraseRotator.svelte     # Rotating landing page subtitles
│   │   │   ├── Player.svelte            # YouTube IFrame API wrapper
│   │   │   ├── PlaylistPanel.svelte     # Playlist import and selection
│   │   │   ├── RandomizeButton.svelte   # Main CTA button with pulse rings
│   │   │   ├── SettingsPanel.svelte     # Theme, effects, bg mode, repeat mode
│   │   │   ├── Sidebar.svelte           # Authenticated user sidebar
│   │   │   └── SiteNotice.svelte        # Dismissible global notice banner
│   │   │
│   │   ├── effects/
│   │   │   ├── canvas2d-visualizer.ts   # Canvas2D fallback renderer
│   │   │   ├── EffectsCanvas.svelte     # WebGL canvas mount, lifecycle, preset switching
│   │   │   ├── engine.ts                # Three.js renderer, fullscreen quad, texture loader
│   │   │   ├── preset-manager.ts        # Genre-to-preset mapping, crossfade transitions
│   │   │   └── shaders/
│   │   │       ├── shared.vert          # Shared vertex shader (passthrough UV)
│   │   │       ├── aurora-flow.glsl
│   │   │       ├── cosmic-drift.glsl    # Default idle preset
│   │   │       ├── fractal-fire.glsl
│   │   │       ├── gradient-drift.glsl
│   │   │       ├── heat-haze.glsl
│   │   │       ├── liquid-chrome.glsl
│   │   │       ├── neon-pulse.glsl
│   │   │       ├── particle-storm.glsl
│   │   │       ├── smoke-room.glsl
│   │   │       ├── velvet-wave.glsl
│   │   │       └── video-mirror.glsl    # VHS thumbnail mode — blurs cover art into bg
│   │   │
│   │   ├── server/                      # Server-only boundary ($lib/server/)
│   │   │   ├── auth/
│   │   │   │   ├── google.ts            # Arctic Google OAuth client
│   │   │   │   └── lucia.ts             # Lucia instance, session/user attributes
│   │   │   ├── db/
│   │   │   │   ├── index.ts             # better-sqlite3 connection, WAL mode, auto-migrate
│   │   │   │   ├── schema.ts            # Drizzle table definitions and type exports
│   │   │   │   └── seed.ts              # Phrase and seed query seed script
│   │   │   ├── config.ts                # Validated env var loader
│   │   │   ├── rate-limiter.ts          # Sliding-window rate limiter (SQLite-backed)
│   │   │   ├── youtube-cache.ts         # Search result cache layer
│   │   │   └── youtube.ts               # YouTube Data API v3 client
│   │   │
│   │   ├── stores/
│   │   │   ├── effects.svelte.ts        # WebGL state (preset name, gpu flag)
│   │   │   ├── player.svelte.ts         # Track state, randomize logic, playlist mode
│   │   │   └── user.svelte.ts           # User preferences, settings PATCH helper
│   │   │
│   │   ├── types/
│   │   │   └── youtube-player.d.ts      # YouTube IFrame Player API type declarations
│   │   │
│   │   └── utils/
│   │       ├── constants.ts             # App-wide constants (FPS cap, rate limit windows)
│   │       ├── debounce.ts              # tryRandomize — 800ms click debounce
│   │       ├── focus-trap.ts            # Modal focus trap utility
│   │       ├── genre-map.ts             # YouTube tag → genre → shader preset mapping
│   │       └── gpu-detect.ts            # WebGL capability detection (GPU vs software)
│   │
│   └── routes/
│       ├── +layout.server.ts            # Session load, user prefs, site notice
│       ├── +layout.svelte               # Root layout, sidebar, theme application
│       ├── +page.server.ts              # Landing page data (phrases)
│       ├── +page.svelte                 # Main page — hero, button, idle dissolve
│       ├── api/
│       │   ├── health/+server.ts        # GET /api/health — Docker health check
│       │   ├── history/+server.ts       # GET/POST /api/history
│       │   ├── playlists/
│       │   │   ├── +server.ts           # GET/DELETE /api/playlists
│       │   │   ├── import/+server.ts    # POST /api/playlists/import
│       │   │   └── random/+server.ts    # GET /api/playlists/random/:id
│       │   ├── randomize/+server.ts     # POST /api/randomize — core endpoint
│       │   └── user/settings/+server.ts # PATCH/DELETE /api/user/settings
│       ├── auth/
│       │   ├── google/+server.ts        # Redirect to Google OAuth
│       │   ├── google/callback/+server.ts # OAuth callback, session creation
│       │   └── signout/+server.ts       # Session invalidation
│       └── privacy/+page.svelte         # Privacy policy
│
├── static/                              # Public assets (favicons, robots.txt)
├── drizzle/                             # Auto-generated migration SQL (do not hand-edit)
├── scripts/
│   ├── entrypoint.sh                    # Docker entrypoint — runs migrations, starts server
│   └── backup.sh                        # SQLite WAL checkpoint + file backup
├── Dockerfile
├── docker-compose.yml
├── drizzle.config.ts
├── svelte.config.js
├── vite.config.ts
└── tsconfig.json
```

---

## Database Schema

SQLite database via Drizzle ORM. WAL mode enabled. Auto-migrates on startup from `drizzle/` migration files.

---

### `users`

Core user record created on first Google OAuth sign-in.

| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | Lucia-generated session ID |
| `email` | TEXT UNIQUE NOT NULL | From Google profile |
| `display_name` | TEXT NOT NULL | From Google profile |
| `avatar_url` | TEXT | Google profile photo URL |
| `google_id` | TEXT UNIQUE | Google sub claim |
| `theme_pref` | TEXT NOT NULL | `dark` · `light` · `auto` — default `dark` |
| `effects_pref` | TEXT NOT NULL | `on` · `off` · `reduced` — default `on` |
| `source_pref` | TEXT NOT NULL | `youtube` · `auto` · `alternate` — default `youtube` |
| `bg_mode_pref` | TEXT NOT NULL | `genre` · `video-mirror` — default `genre` |
| `created_at` | INTEGER NOT NULL | Unix timestamp |

---

### `sessions`

Lucia session records. Expired sessions are cleaned up by Lucia on access.

| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | Lucia session token |
| `user_id` | TEXT NOT NULL | FK → `users.id` |
| `expires_at` | INTEGER NOT NULL | Unix timestamp |

---

### `phrases`

Rotating subtitles displayed on the landing page hero.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `text` | TEXT UNIQUE NOT NULL | Displayed phrase |
| `category` | TEXT | `motivational` · `musical` · `philosophical` · `playful` |
| `active` | INTEGER (boolean) | Whether phrase is in rotation |

---

### `seed_queries`

Search terms used to query the YouTube Data API. A random active query is picked per randomize request when the search cache misses.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `query` | TEXT UNIQUE NOT NULL | e.g. `"jazz music"`, `"chill vibes music"` |
| `category` | TEXT | `genre` · `mood` · `decade` · `activity` · `wildcard` |
| `active` | INTEGER (boolean) | Whether query is eligible for selection |

---

### `video_cache`

Individual YouTube video metadata. Populated from search results and reused across sessions to avoid redundant API calls.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `video_id` | TEXT UNIQUE NOT NULL | YouTube video ID |
| `title` | TEXT NOT NULL | Video title |
| `channel_name` | TEXT NOT NULL | Channel display name |
| `thumbnail_url` | TEXT | `i.ytimg.com` URL |
| `duration_seconds` | INTEGER | For filtering live streams / shorts |
| `genre` | TEXT | Detected genre tag |
| `tags` | TEXT | JSON array of video tags |
| `seed_query` | TEXT | Which seed query surfaced this video |
| `cached_at` | INTEGER NOT NULL | Unix timestamp |

---

### `search_cache`

Caches the list of video IDs returned by a given seed query search. TTL: 24 hours. Prevents redundant YouTube API calls for the same query.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `seed_query` | TEXT NOT NULL | The search term used |
| `result_video_ids` | TEXT NOT NULL | JSON array of YouTube video IDs |
| `cached_at` | INTEGER NOT NULL | Unix timestamp |

---

### `listening_history`

Per-user record of every track played. Used to display history in the sidebar and power the no-repeat session mode.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `user_id` | TEXT NOT NULL | FK → `users.id` |
| `video_id` | TEXT | YouTube video ID |
| `title` | TEXT NOT NULL | Track title at time of play |
| `artist` | TEXT | Channel name |
| `source` | TEXT NOT NULL | `youtube` |
| `genre` | TEXT | Genre tag at time of play |
| `playlist_name` | TEXT | NULL = global pool, otherwise playlist name |
| `played_at` | INTEGER NOT NULL | Unix timestamp |

---

### `rate_limits`

Sliding-window rate limit counters. Keyed by IP address (unauthenticated) or user ID (authenticated).

| Column | Type | Notes |
|---|---|---|
| `key` | TEXT PK | IP address or user ID |
| `count` | INTEGER NOT NULL | Request count in current window |
| `window_start` | INTEGER NOT NULL | Unix timestamp of window open |

---

### `user_playlists`

YouTube playlists imported by authenticated users. Tracks are stored as a JSON array in `tracks`.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `user_id` | TEXT NOT NULL | FK → `users.id` (cascade delete) |
| `spotify_playlist_id` | TEXT NOT NULL | Legacy column name — stores YouTube playlist ID |
| `name` | TEXT NOT NULL | Playlist display name |
| `image_url` | TEXT | Playlist thumbnail |
| `owner_name` | TEXT | Playlist owner display name |
| `track_count` | INTEGER | Number of tracks |
| `tracks` | TEXT NOT NULL | JSON array of track objects |
| `imported_at` | INTEGER NOT NULL | Unix timestamp |

---

## Request Flow — Core Randomize

```
Browser
  └── POST /api/randomize
        │
        ├── hooks.server.ts
        │     ├── validate Lucia session
        │     └── sliding-window rate limit check (SQLite rate_limits)
        │
        └── api/randomize/+server.ts
              ├── if activePlaylistId → sample from user_playlists.tracks (JSON)
              │
              └── else (global pool)
                    ├── pick random active seed_query from SQLite
                    ├── check search_cache (TTL 24h)
                    │     ├── HIT  → use cached video_id list
                    │     └── MISS → YouTube Data API search → cache result
                    ├── pick random video_id from result list
                    ├── fetch/upsert video metadata into video_cache
                    └── return { videoId, title, channelName, thumbnailUrl, genre }

Client receives response
  ├── YouTube IFrame Player loads video
  ├── Three.js preset switches to genre-matched GLSL shader (2s crossfade)
  └── Listening history POST to /api/history
```

---

## Authentication Flow

Google OAuth 2.0 via Arctic + Lucia v3.

```
/auth/google            → Arctic generates Google OAuth URL → redirect
/auth/google/callback   → exchange code for tokens
                        → fetch Google profile (sub, email, name, picture)
                        → upsert user in SQLite
                        → create Lucia session
                        → set session cookie → redirect to /
/auth/signout           → invalidate Lucia session → clear cookie → redirect to /
```

---

## WebGL Effects Pipeline

```
EffectsCanvas.svelte
  └── onMount
        ├── gpu-detect.ts → detectRenderingCapability()
        │     ├── GPU   → initWebGL()  → EffectsEngine + PresetManager
        │     └── CPU   → initCanvas2D() → Canvas2DVisualizer (CSS fallback)
        │
        └── $effect: track changes
              ├── bgMode === 'video-mirror'
              │     └── engine.loadTexture(thumbnailUrl) → video-mirror.glsl
              └── bgMode === 'genre'
                    └── PresetManager.genreToPreset(genre) → switch shader
                          └── crossfade: u_transition uniform 0→1 over 2s
```

**GLSL Shader Presets**

| Preset | Genre Tags |
|---|---|
| `cosmic-drift` | Default / idle |
| `aurora-flow` | Ambient, lo-fi, classical, sleep |
| `neon-pulse` | Electronic, techno, house, dubstep |
| `velvet-wave` | R&B, soul, neo soul |
| `liquid-chrome` | Hip hop, trap, drill |
| `smoke-room` | Jazz, blues, bossa nova |
| `heat-haze` | Latin, reggaeton, afrobeats |
| `fractal-fire` | Metal, punk, grunge |
| `gradient-drift` | Pop, indie pop |
| `particle-storm` | Drum and bass, breakbeat |
| `video-mirror` | VHS mode — thumbnail behind CRT distortion |

---

## Deployment

Docker Compose on a self-hosted Linux machine. Traffic routed via Cloudflare Tunnel — no open inbound ports.

```
Internet
  └── Cloudflare Tunnel (cloudflared)
        └── localhost:3000
              └── Docker: randomize container
                    ├── Node.js SvelteKit server (adapter-node)
                    └── SQLite volume → /app/data/randomize.db
```

**Deploy / update:**
```bash
docker compose up -d --build
```

**Health check:**
```
GET http://127.0.0.1:3000/api/health
```
Checked every 30s. Uses `127.0.0.1` explicitly — Alpine Linux resolves `localhost` to `::1` (IPv6) while Node listens on IPv4.

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `YOUTUBE_API_KEY` | Yes | YouTube Data API v3 key (Google Cloud Console) |
| `GOOGLE_CLIENT_ID` | Yes | OAuth 2.0 client ID |
| `GOOGLE_CLIENT_SECRET` | Yes | OAuth 2.0 client secret |
| `GOOGLE_REDIRECT_URI` | Yes | Must match Google Console — e.g. `https://yourdomain.com/auth/google/callback` |
| `ENCRYPTION_KEY` | Yes | 64-char hex string (AES-256-GCM) — generate with `openssl rand -hex 32` |
| `DATABASE_URL` | No | Defaults to `./data/randomize.db` |
| `ORIGIN` | Yes (prod) | Full origin URL for CSRF protection — e.g. `https://yourdomain.com` |

---

## Diagrams

The following diagrams are referenced in this document. Each is labeled for the system or operational concern it covers.

---

**Diagram 1 — System Architecture**
*High-level overview: browser, SvelteKit server, SQLite, YouTube API, Cloudflare Tunnel, Docker.*

<!-- Add diagram here -->

---

**Diagram 2 — Database Entity-Relationship (ERD)**
*All tables, columns, primary keys, foreign keys, and relationships.*

<!-- Add diagram here -->

---

**Diagram 3 — Randomize API Request Flow**
*Detailed flow through rate limiter → seed query selection → cache check → YouTube API → response.*

<!-- Add diagram here -->

---

**Diagram 4 — Authentication Flow**
*Google OAuth handshake, Lucia session lifecycle, cookie management.*

<!-- Add diagram here -->

---

**Diagram 5 — WebGL Effects Pipeline**
*GPU detection → Three.js init → shader preset selection → crossfade → VHS thumbnail mode.*

<!-- Add diagram here -->

---

**Diagram 6 — Deployment Topology**
*Docker Compose service graph, volume mounts, Cloudflare Tunnel routing, health check path.*

<!-- Add diagram here -->

---

**Diagram 7 — YouTube API Quota Management**
*Seed query selection, 24-hour search cache, video metadata cache, quota cost per operation.*

<!-- Add diagram here -->

---

## Building on This

The architecture is intentionally modular. Common extension points:

- **Swap the video source** — replace `youtube.ts` and `youtube-cache.ts` with any API that returns a video ID and metadata. The `randomize` endpoint only cares about the shape of the response.
- **Add seed queries** — run `npm run db:seed` after adding entries to `seed.ts`. Categories: `genre`, `mood`, `decade`, `activity`, `wildcard`.
- **Add a shader preset** — drop a `.glsl` file into `src/lib/effects/shaders/`, register it in `preset-manager.ts`, and map genre tags to it in `genre-map.ts`.
- **Replace SQLite** — Drizzle ORM supports PostgreSQL and MySQL. Swap the `better-sqlite3` driver and update `drizzle.config.ts`.
- **Add OAuth providers** — Arctic supports GitHub, Apple, Discord, and others. Follow the same pattern as `auth/google/`.

---

## License

MIT — see LICENSE.

---

*Randomize.fm is independently built and maintained.*
