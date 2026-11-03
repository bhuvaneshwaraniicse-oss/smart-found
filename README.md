# TraceBack

A full-stack **smart found-item** app: one board for found reports, offline visual/text duplicate detection, map pins, and human-reviewed claim verification so items get back to their owners.

Built as a self-contained demo — no external AI services, no cloud database, no API keys. Everything runs locally.

## Features

- **Report found items** with photo, description, category, colour, date, location pin and contact detail.
- **AI duplicate detection without an API**: perceptual colour + layout fingerprints (128-d) and hashed text embeddings (256-d) are compared with category, distance and timing.
- **Duplicate radar** showing the strongest similar found reports for each of your open reports, with score breakdowns you can inspect.
- **Map-first browsing** with Leaflet/OpenStreetMap, found markers, filters and a sticky map preview.
- **Verified claims on found items**: the finder hides a secret detail; claimants answer it and describe the item.
- **Transparent claim scoring** guides a human decision — the finder or an admin approves or rejects; contact details unlock only after approval.
- **Admin review queue** with engine stats and claim history.
- **Security basics**: bcrypt passwords, 7-day JWTs, AES-256-GCM encryption of verification answers at rest.

## Stack

| Layer | Tech |
| --- | --- |
| Frontend | React 19, Vite 7, React Router 7, TypeScript, Leaflet 1.9, custom CSS |
| Backend | Node.js, Express 5, TypeScript (`tsx`) |
| Data | SQLite via `better-sqlite3`, uploads on disk |
| Images | `sharp` for processing + offline feature extraction |
| Auth | `jsonwebtoken`, `bcryptjs` |
| Validation | `zod` |

No Tailwind and no external fonts — the UI uses system/display font stacks.

## Quick start

```bash
npm install
npm run dev
```

Open http://localhost:5173. The API runs on http://localhost:8787 and Vite proxies `/api` and `/uploads`.

On first start the API creates `data/traceback.db`, generates demo item artwork with `sharp`, and seeds demo users, found reports and claims around Bengaluru.

### Demo accounts

| Role | Email | Password |
| --- | --- | --- |
| Admin | `admin@traceback.dev` | `admin1234` |
| User | `maya@traceback.dev` | `demo1234` |
| User | `arjun@traceback.dev` | `demo1234` |
| User | `lena@traceback.dev` | `demo1234` |
| User | `kiran@traceback.dev` | `demo1234` |

### Production-style run

```bash
npm run build
npm start
```

`npm start` serves the built SPA from `dist` plus the API on port 8787.

## Scripts

| Script | Purpose |
| --- | --- |
| `npm run dev` | Run API + Vite together |
| `npm run dev:api` | API only (`tsx watch`) |
| `npm run dev:web` | Vite only |
| `npm run build` | Type-check-free Vite production build |
| `npm start` | Serve API + `dist` as production |
| `npm run seed` | Reset database and uploads, then reseed demo data |
| `npm run typecheck` | `tsc --noEmit` |

## Configuration

Copy `.env.example` to `.env` if you want to override defaults:

```ini
PORT=8787
APP_SECRET=replace-with-a-long-random-string
# DATA_DIR=./data
```

`APP_SECRET` is used for JWT signing and AES-256-GCM key derivation. Change it before deploying anywhere real.

## How matching works

When an item is created or its details change, the server stores two vectors:

1. **Image — 128 dimensions**
   - 64-bin RGB colour histogram
   - 64 values from an 8×8 grayscale layout thumbnail
   - L2-normalized, cosine similarity
2. **Text — 256 dimensions**
   - Hashed unigrams, bigrams and character trigrams with a synonym map
   - FNV-1a hashing into buckets, L2-normalized, cosine similarity

A candidate score combines available signals with weights:

| Signal | Weight |
| --- | --- |
| Image similarity | 0.44 |
| Text similarity | 0.30 |
| Category match | 0.10 |
| Location proximity | 0.11 |
| Timing proximity | 0.05 |

Each open found report is compared against every other open found report to surface possible duplicates of the same item. The UI shows the weighted score plus a breakdown and human-readable reasons.

## How claim verification works

1. A finder can attach a **verification question** and answer when posting a found report. The answer is encrypted with AES-256-GCM.
2. A signed-in user files a claim with a description and answer.
3. The server scores the claim:
   - answer similarity × 0.55
   - description text overlap × 0.45
   - recommendation: approve ≥ 72 (and answer ≥ 0.55), review ≥ 42, otherwise reject
4. The finder or an admin reviews the full message, answer and score. Only an approval marks the report resolved and unlocks contact details.

Scores are guidance, never automatic approval.

## Project layout

```
server/
  config.ts        env + paths + categories
  db.ts            SQLite schema, row types, serializers
  security.ts      bcrypt/JWT/AES helpers
  auth.ts          auth middleware
  embeddings.ts    image/text vectors, match + claim scoring
  images.ts        sharp processing and storage
  seedImages.ts    generated demo artwork
  geocode.ts       local place search/reverse geocoding
  serialize.ts     client-safe item shaping
  seed.ts          demo data
  routes/
    auth.ts        register/login/me
    items.ts       item CRUD, upload, matches, claims list
    claims.ts      claim create/list/review
    misc.ts        stats, meta, geo, admin queue, health
  index.ts         app wiring + production SPA serving
src/
  components/      Navbar, cards, map, score visuals, forms, etc.
  pages/           Home, Browse, ItemDetail, PostItem, Radar,
                   Dashboard, Admin, Login, Register, NotFound
  lib/             api client, auth context, types, formatting, toasts
  styles.css       full design system and responsive layout
```

## API overview

| Method | Route | Notes |
| --- | --- | --- |
| `POST` | `/api/auth/register` | Create account |
| `POST` | `/api/auth/login` | Get JWT |
| `GET` | `/api/auth/me` | Current user |
| `GET` | `/api/items` | Filters: `category`, `status`, `q`, `limit` (found reports only) |
| `POST` | `/api/items` | Multipart found-report creation |
| `GET` | `/api/items/:id` | Item detail (contact if allowed) |
| `PATCH` | `/api/items/:id` | Owner/admin update |
| `DELETE` | `/api/items/:id` | Owner/admin delete |
| `GET` | `/api/items/:id/matches` | Ranked similar found reports |
| `POST` | `/api/items/:id/claims` | File a claim |
| `GET` | `/api/items/:id/claims` | Owner/admin claim queue |
| `GET` | `/api/claims/:id` | Claimant/reviewer detail |
| `PATCH` | `/api/claims/:id` | Approve/reject |
| `GET` | `/api/me/items` | My reports |
| `GET` | `/api/me/claims` | My filed claims |
| `GET` | `/api/admin/claims` | Admin queue |
| `GET` | `/api/stats` | Public stats |
| `GET` | `/api/meta` | Categories, demo accounts |
| `GET` | `/api/geo/search` | Local place search |
| `GET` | `/api/geo/reverse` | Nearest place label |
| `GET` | `/api/health` | Health check |

## Notes and limitations

- Demo geocoding is a curated Bengaluru place list, not a live geocoder.
- Image fingerprints are perceptual but lightweight; they are not a neural embedding model.
- SQLite and disk uploads are intended for local/demo use. For production, move to Postgres/object storage, rotate `APP_SECRET`, and add rate limiting.
- Map tiles require internet access (OpenStreetMap).
