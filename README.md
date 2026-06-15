# Intelligent AI & Smart Agriculture Research Lab

A production-ready research-lab platform for **Dr. Pradeep Gupta**'s lab. It auto-syncs
publications from **Google Scholar (SerpAPI)**, **Scopus** and **ORCID**, merges and
deduplicates them into MongoDB, and presents the lab's team, projects, datasets, live
research metrics, a collaboration portal, a markdown news section, a secure admin panel,
and a RAG-based AI research assistant.

**Stack:** Next.js 14 (App Router) · Tailwind CSS · Framer Motion · MongoDB (Mongoose) ·
NextAuth · Recharts · OpenAI · Vercel.

---

## 1. System architecture

```
                         ┌──────────────────────────────────────┐
                         │             Vercel (Next.js)          │
                         │                                       │
  Browser ──────────────▶  App Router pages (RSC + client)      │
                         │   /  /about  /team  /publications     │
                         │   /projects  /metrics  /datasets      │
                         │   /collaborate  /blog  /admin/*        │
                         │                                       │
                         │  API routes (/app/api/*)              │
                         │   publications · team · projects      │
                         │   datasets · metrics · collaborate    │
                         │   chat (RAG) · auth · cron/sync        │
                         └───────┬───────────────┬───────────────┘
                                 │               │
            ┌────────────────────┘               └───────────────────────┐
            ▼                                                             ▼
   ┌──────────────────┐                                        ┌───────────────────┐
   │  Sync pipeline    │  fetch → merge/dedup → upsert          │   MongoDB Atlas    │
   │  (src/lib/sync.js)│ ─────────────────────────────────────▶│  publications      │
   └───────┬───────────┘                                        │  teammembers       │
           │ parallel fetch                                     │  projects          │
   ┌───────┼─────────────┐                                      │  datasets          │
   ▼       ▼             ▼                                      │  inquiries · users │
 SerpAPI  Scopus       ORCID                                    └───────────────────┘
 (Scholar)(Elsevier)   (public)
                                 ▲
        Vercel Cron (weekly) ────┘  GET /api/cron/sync  (CRON_SECRET protected)
        Admin "Sync now" ─────────  POST /api/publications/sync  (session protected)
```

**Data flow for publications**

1. `fetchScholar` / `fetchScopus` / `fetchOrcid` each return a normalized `baseRecord[]`.
2. `mergePublications` computes a deterministic dedup key (DOI, else normalized
   title+year), then merges duplicates — union of authors/topics/sources, max citation
   count, highest-priority index, longest title/abstract.
3. `runSync` bulk-upserts by `key`. Source-derived fields are `$set`; admin-editable
   fields (`index`, `topics`) are `$setOnInsert` so manual curation is preserved.
4. `computeMetrics` derives totals, h-index and i10-index from the cached corpus.

---

## 2. Folder structure

```
ai-smart-agri-lab/
├── package.json  next.config.mjs  tailwind.config.js  postcss.config.js
├── vercel.json                 # weekly cron schedule
├── .env.example                # all required env vars (documented)
├── scripts/
│   ├── seed-admin.mjs           # create the admin login
│   ├── seed-projects.mjs        # seed the 4 flagship project records
│   ├── sync-publications.mjs    # CLI trigger of the sync endpoint
│   └── embed-publications.mjs   # generate OpenAI embeddings for the assistant
└── src/
    ├── middleware.js            # protects /admin/*
    ├── content/blog/*.md        # markdown CMS for News & Events
    ├── lib/
    │   ├── site.js              # lab identity + nav (single source of truth)
    │   ├── mongodb.js           # cached Mongoose connection
    │   ├── auth.js              # NextAuth config (credentials + optional Google)
    │   ├── sync.js              # pipeline orchestrator + metrics
    │   ├── citation.js          # BibTeX / RIS export
    │   ├── blog.js  fetcher.js
    │   └── sources/             # serpapi.js scopus.js orcid.js merge.js normalize.js
    ├── models/                  # Publication TeamMember Project Dataset Inquiry User
    ├── components/              # Navbar Footer Hero Reveal ThemeProvider AssistantWidget …
    └── app/
        ├── layout.jsx  page.jsx  globals.css  not-found.jsx
        ├── about/ team/ publications/ projects/[slug]/ metrics/
        ├── datasets/ collaborate/ blog/[slug]/
        ├── admin/ (login, dashboard, team, projects, datasets)
        └── api/ (auth, publications[/sync], cron/sync, team, projects,
                  datasets, metrics, collaborate, chat)
```

---

## 3. Database schema (MongoDB / Mongoose)

| Collection      | Key fields |
|-----------------|------------|
| **publications**| `key` (unique dedup hash), `title`, `authors[]`, `venue`, `year`, `doi`, `url`, `citationCount`, `index` (SCIE\|Scopus\|Conference\|Other), `topics[]`, `sources[]`, `abstract`, `embedding[]` (optional), `lastSyncedAt` |
| **teammembers** | `name`, `role` (director\|faculty\|phd\|mtech\|btech\|alumni), `title`, `photoUrl`, `email`, `interests[]`, `links{scholar,orcid,github,linkedin,website}`, `order`, `active` |
| **projects**    | `slug` (unique), `name`, `summary`, `abstract`, `status`, `topics[]`, `githubUrl`, `datasetUrl`, `architectureDiagramUrl`, `relatedPublicationKeys[]` |
| **datasets**    | `name`, `category` (agriculture\|medical\|benchmark), `description`, `downloadUrl`, `sizeLabel`, `license`, `bibtex` |
| **inquiries**   | `type` (join\|collaborate\|internship), `name`, `email`, `affiliation`, `interests`, `message`, `handled` |
| **users**       | `email` (unique), `name`, `passwordHash` (bcrypt), `role` |

Indexes: `publications.key` (unique), text index on `title/abstract/venue`, plus
`year`, `index`, `topics`; `projects.slug` (unique); `users.email` (unique).

---

## 4. Local setup

```bash
npm install
cp .env.example .env.local      # fill in the values (see section 6)

# one-time data setup
node --env-file=.env.local scripts/seed-admin.mjs
node --env-file=.env.local scripts/seed-projects.mjs

npm run dev                      # http://localhost:3000
```

Then sign in at `/admin/login` and click **Sync now** to pull publications.
> Requires Node 18.18+ (20+ recommended for the `--env-file` flag used by the scripts).

---

## 5. API integration steps

### SerpAPI (Google Scholar)
1. Create an account at https://serpapi.com and copy the key from
   **Dashboard → Your API Key**.
2. Find your Scholar author id: open your Google Scholar profile; the URL contains
   `?user=XXXXXXXX` — that value is your id.
3. Set `SERPAPI_KEY` and `GOOGLE_SCHOLAR_ID` in env.
   The client paginates the `google_scholar_author` engine and also caches the
   citations / h-index / i10-index from the profile.

### Scopus (Elsevier)
1. Register an app at https://dev.elsevier.com and request an **API Key** (Scopus
   Search API). Institutional / VPN access is usually required for live queries.
2. (Optional) If your institution issued an **Insttoken**, set `SCOPUS_INST_TOKEN`.
3. Set `SCOPUS_API_KEY`; `SCOPUS_AUTHOR_ID` defaults to `58710685300`.
   The client queries `AU-ID(<id>)` and paginates 25 results at a time.

### ORCID
No key needed for public works. Set `ORCID_ID` (default `0000-0003-3142-6373`).
The client reads `https://pub.orcid.org/v3.0/<orcid>/works`.

### OpenAI (AI research assistant)
1. Set `OPENAI_API_KEY` (and optionally `OPENAI_CHAT_MODEL`, `OPENAI_EMBED_MODEL`).
2. After a publication sync, run `node --env-file=.env.local scripts/embed-publications.mjs`
   to generate embeddings. Until then the assistant falls back to keyword retrieval.

---

## 6. Environment variables

See `.env.example` for the full annotated list. Required at minimum:
`MONGODB_URI`, `NEXTAUTH_SECRET`, `NEXTAUTH_URL`, plus the source keys you intend to use
(`SERPAPI_KEY` + `GOOGLE_SCHOLAR_ID`, `SCOPUS_API_KEY`, `ORCID_ID`), `CRON_SECRET`, and
`OPENAI_API_KEY` for the assistant.

---

## 7. Deploying to Vercel

1. Push this folder to a Git repo and **Import Project** in Vercel.
2. In **Project → Settings → Environment Variables**, add every variable from
   `.env.example` (set `NEXTAUTH_URL` to your production URL).
3. Deploy. `vercel.json` registers the weekly cron:
   `GET /api/cron/sync` every Monday 03:00 UTC. Vercel sends
   `Authorization: Bearer <CRON_SECRET>` automatically — make sure `CRON_SECRET` is set.
4. After first deploy, run the seed scripts against your Atlas DB (locally with the
   production `MONGODB_URI`, or via a one-off job), then sign in to `/admin` and **Sync now**.

### Auto-update engine
- **Scheduled:** Vercel Cron → `/api/cron/sync` (weekly; edit `vercel.json` for daily `0 3 * * *`).
- **Manual fallback:** Admin dashboard **Sync now** → `POST /api/publications/sync`.
- **External scheduler:** `GET /api/cron/sync?secret=<CRON_SECRET>` from any cron service.

---

## 8. Feature checklist

| Module | Status |
|--------|--------|
| Landing page + animated AI/Agri hero | ✅ |
| About (vision/mission/philosophy) | ✅ |
| Team management + admin CRUD | ✅ |
| Publications auto-sync (Scholar+Scopus+ORCID, merge/dedup) + filters | ✅ |
| Research projects + detail pages (abstract, diagram placeholder, links) | ✅ |
| Datasets hub (categories, download, BibTeX) | ✅ |
| Metrics dashboard (citations, h-index, i10, per-year) | ✅ |
| Collaboration portal (join/collaborate/internship forms) | ✅ |
| News/Events (markdown CMS) | ✅ |
| Secure admin panel (NextAuth) | ✅ |
| Auto-update engine (cron + manual) | ✅ |
| AI research assistant (RAG over publications, OpenAI) | ✅ |
| SEO + schema.org (Person/ResearchOrganization) | ✅ |
| Dark mode · BibTeX/RIS export | ✅ |

### Documented extension points (wire-up left to you)
- **PDF viewer**: store `pdfUrl` on publications and embed `<iframe>`/`react-pdf` on a detail page.
- **GitHub repo stats**: call the GitHub API in the project page to show stars/last commit.
- **Collaboration world map**: add a `country` field to `inquiries` and render with a map lib.
- **Google sign-in**: set `GOOGLE_CLIENT_ID`/`SECRET`; only emails present in `users` are allowed.

---

## Notes
No mock/fake data is committed. Project records are seeded by **name only**
(PLANTDetectNet, PlantVitGNet, AppleViT, Federated Learning Systems); abstracts,
links and diagrams are added through the admin panel. All publication content comes
from the live source APIs.
