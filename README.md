# jammi

**The fastest way to catch the Jammie.** jammi is a premium, minimalist web app that shows UCT students the next UCT Shuttle ("Jammie") departure the instant the page loads — no taps, no menus, no waiting. The most relevant bus is always one glance away.

> Scheduled times are based on official UCT PDFs and are subject to delays.

---

## Why it feels instant

The home page renders a single hero — the **Next Jammie** card with a live countdown — before any network request is required. The frontend bundles a fallback timetable and a preview sign-in, so it works fully standalone; when the backend and Firebase are wired up it transparently upgrades to live timetable data and per-user personalization. Search is lazy-loaded and filters as you type, so the first paint stays light.

The look is deliberately *not* the usual purple glassmorphism: deep Cape Town night-blues with a single warm amber accent — the Jammie's headlight at dusk — set in Sora + Inter, with motion that respects `prefers-reduced-motion`.

---

## Architecture — strict layered (Presentation · Business · Data)

```
jammi/
├── presentation/          PRESENTATION LAYER — the frontend the student sees
│   ├── public/
│   │   ├── index.html
│   │   ├── css/jammi.css
│   │   ├── js/{app,search,auth,routes,firebase-config}.js
│   │   └── assets/lottie/jammie-bus.json
│   └── tailwind.config.js          (production Tailwind build config)
│
├── business/              BUSINESS / LOGIC LAYER — Node + Express API
│   ├── server.js                   serves the frontend + mounts /api
│   ├── routes/{timetable,user,admin}.routes.js
│   ├── controllers/{timetable,user,admin}.controller.js
│   ├── services/{timetable,firestore}.service.js
│   ├── middleware/verifyToken.js   Firebase ID-token verification
│   └── config/firebaseAdmin.js     Admin SDK bootstrap (graceful preview mode)
│
├── data/                  DATA LAYER — automated Python pipeline + schema
│   ├── pipeline/{scrape,fetch_pdfs,parse_pdf,format_llm,validate,
│   │             upload_firestore,admin_rescrape,config}.py
│   ├── schema/timetable.schema.json
│   └── output/timetable.sample.json   (committed seed / fallback)
│
├── .github/workflows/scrape.yml   weekly auto-refresh + manual dispatch
├── .env.example
├── package.json
└── README.md
```

Each layer only talks to the one below it: the frontend calls the API, the API reads what the pipeline publishes, the pipeline is the only thing that touches UCT's PDFs.

---

## Quick start

### 1. Run the app (works immediately, no config)

```bash
npm install
npm start
# → http://localhost:4000
```

This serves the frontend and the API together. With no Firebase service account present, the backend runs in **preview mode**: timetables come from the committed sample, and "My Routes" personalization is kept in memory so you can click through the whole experience locally.

### 2. Refresh real timetable data (data layer)

```bash
npm run pipeline:install          # one-time: pip install the pipeline deps
export ANTHROPIC_API_KEY=sk-...   # the LLM only reshapes text into JSON
npm run pipeline:dry              # parse UCT PDFs without writing anywhere
npm run pipeline                  # parse + publish data/output/timetable.json
```

The server picks up a freshly published `timetable.json` automatically (it watches the file's mtime) — no restart needed.

### 3. Go live (optional)

Drop a Firebase **web** config into `presentation/public/js/firebase-config.js` (replace the `REPLACE_*` placeholders) to enable real Email/Google/Apple sign-in, and provide a Firebase **service account** (`FIREBASE_SERVICE_ACCOUNT` or `FIREBASE_SERVICE_ACCOUNT_PATH`) so the backend can verify tokens and persist personalization to Firestore. Copy `.env.example` to `.env` and fill in what you need.

---

## The data automation pipeline

The pipeline turns UCT's human-facing PDFs into clean, structured JSON the app can trust — and the LLM is strictly a *formatter*, never a source of times.

1. **fetch_pdfs** — scrapes the official timetables page for `*.pdf` links (so a filename change doesn't break us), or uses explicit `UCT_PDF_URLS` overrides, and downloads them to a gitignored cache.
2. **parse_pdf** — extracts raw text and tables with `pdfplumber`.
3. **format_llm** — sends that extracted text to the LLM with a system prompt that **forbids inventing or interpolating times**; it only normalizes what's there into 24-hour `HH:MM` and the canonical shape.
4. **validate** — checks the result against `data/schema/timetable.schema.json` (Draft 7) and refuses to publish a timetable with zero departures.
5. **upload_firestore** — writes a `current` doc plus a dated snapshot for audit (skipped on dry runs).

`scrape.py` orchestrates these and aborts *before* publishing if any step fails, so a bad parse can never overwrite good data.

### Scheduling & manual override

`.github/workflows/scrape.yml` runs the pipeline every **Monday at 03:00 UTC** and can also be dispatched manually with optional PDF URLs and a dry-run toggle. When UCT changes their PDF layout out of cycle, you have two manual paths:

```bash
# CLI (data layer):
npm run pipeline:rescrape -- --pdf https://uct.ac.za/path/to/new.pdf --dry-run

# HTTP (business layer), guarded by ADMIN_TOKEN:
curl -X POST http://localhost:4000/api/admin/rescrape \
  -H "x-admin-token: $ADMIN_TOKEN" -H "Content-Type: application/json" \
  -d '{"pdfUrls":["https://uct.ac.za/path/to/new.pdf"],"dryRun":true}'
```

---

## API surface

| Method & path | Auth | Purpose |
|---|---|---|
| `GET /api/health` | — | Liveness check |
| `GET /api/timetable` | — | Full published timetable (used by search) |
| `GET /api/timetable/next` | — | Home payload + next-departure-per-route summary |
| `GET /api/users/:uid/routes` | Bearer | A user's saved "My Routes" |
| `PUT /api/users/:uid/routes` | Bearer | Replace the saved set |
| `POST /api/users/:uid/routes` | Bearer | Pin one route |
| `POST /api/admin/rescrape` | `x-admin-token` | Trigger a manual pipeline run |

Authenticated routes expect `Authorization: Bearer <Firebase ID token>`. In preview mode (no service account) they accept an `x-jammi-uid` header instead, purely for local development.

---

## Roadmap

This scaffold delivers the full data layer, the complete frontend, and a runnable business layer. Natural next steps: harden auth with admin custom-claims for the re-scrape endpoint, expand personalization (trip history, smart "leave now" nudges based on the live countdown), add service-worker offline caching of the latest timetable, and ship a real LottieFiles bus animation in place of the bundled placeholder.

---

## License

MIT.

*jammi is an independent student tool and is not affiliated with or endorsed by the University of Cape Town. Always confirm critical departures against official UCT communications.*
