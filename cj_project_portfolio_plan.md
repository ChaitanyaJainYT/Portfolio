# PROJECT ARCADE — Portfolio Website Build Plan

> **Audience:** An AI coding agent that will build this site end-to-end.
> **Owner:** Chaitanya (engineer: AI, automation, Azure, React, Python, Power Platform).
> **One-line pitch:** A developer portfolio that looks and feels like a game launcher (Steam / Epic / Xbox Game Pass). Every project is a "game" in a library, with its own store-style detail page.

---

## 0. READ THIS FIRST — Hard Constraints

These override anything else in this document if there is a conflict.

1. **100% static site.** There is NO backend, NO server, NO database, NO API routes, NO server actions, NO middleware, NO ISR/SSR. Everything must be pre-rendered at build time or run in the browser.
2. **Hosted on GitHub Pages**, deployed automatically by **GitHub Actions** on every push to `main`.
3. **Must work as a project site** (`https://<user>.github.io/<repo>/`) **and** as a user site (`https://<user>.github.io/`). The base path must be configurable by one environment variable. Every asset/link must respect it.
4. **Fully responsive.** Mobile-first. Must be excellent on a 360px-wide phone, a tablet, a laptop, and a 4K monitor. Touch devices have no hover, so every hover interaction needs a tap/focus equivalent.
5. **All content lives in JSON files in `/content`** (projects, profile, skills, timeline, achievements, activity, site settings). The owner adds or edits a project by editing `content/projects.json` and dropping images in a folder, **with zero changes to source code** (see Section 5). JSON is validated at build time; invalid content fails the build and nothing is deployed. No CMS, no external DB, no runtime fetching of content.
6. **No secrets in the repo or the bundle.** Any third-party keys used (form service, analytics) must be public-by-design identifiers only.
7. **Performance budget:** Lighthouse mobile ≥ 90 Performance, ≥ 95 Accessibility, ≥ 95 Best Practices, ≥ 95 SEO.
8. **Respect `prefers-reduced-motion`.** Heavy effects (particles, parallax, 3D) must disable or simplify automatically.

---

## 1. Goals & Non-Goals

### Goals
- Memorable, recruiter-friendly, instantly understandable ("oh, projects as games, nice").
- Fast to scan: a recruiter should find the best 3 projects and the resume within 10 seconds.
- Easy to maintain: add/edit projects by editing one data file.
- Polished UI with tasteful gaming-style motion, not a gimmick that hurts usability.

### Non-Goals
- No user accounts, login, comments, or real "launching" of games.
- No server-side contact form handling (use a third-party form endpoint or `mailto:`).
- No heavy 3D scene that blocks content or kills mobile battery.

---

## 2. Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Framework | **Next.js 15 (App Router)** with `output: 'export'` | Static export only. |
| Language | **TypeScript** (strict mode) | |
| Styling | **Tailwind CSS** (v4, fall back to v3.4 if tooling issues) + CSS variables for themes | |
| Animation | **Framer Motion** (`framer-motion` or `motion`) | Use `LazyMotion` + `domAnimation` to reduce bundle size. |
| Icons | **lucide-react** | Brand icons (GitHub, LinkedIn) via `react-icons/si` or inline SVG, because Lucide dropped brand icons. |
| Search / filter | **fuse.js** (client-side fuzzy search) | |
| Content validation | **zod** + `zod-to-json-schema` | JSON content is validated at build time; generated JSON Schemas give editor autocomplete. |
| Particles background | Custom lightweight `<canvas>` implementation (preferred) | Avoid heavy libs. Pause when tab hidden. |
| 3D (optional, Phase 8 only) | `three` + `@react-three/fiber` + `@react-three/drei`, **lazy-loaded** | Only on desktop, only if Lighthouse budget still passes. |
| Carousel | Custom with Framer Motion + CSS scroll-snap | No heavy carousel libs. |
| Fonts | `next/font/google`: **Rajdhani** or **Orbitron** (headings), **Inter** (body), **JetBrains Mono** (stats/code) | Self-hosted at build time by `next/font`. |
| Analytics (optional) | Umami Cloud or Plausible via `<script>` tag | Controlled by env var; off if unset. |
| Lint/format | ESLint (next config), Prettier, `prettier-plugin-tailwindcss` | |
| Tests | Vitest + React Testing Library (unit), Playwright (smoke e2e on built site) | |
| CI/CD | GitHub Actions → GitHub Pages | See Section 14. |
| Package manager | `pnpm` (or npm if the owner prefers; keep lockfile committed) | |

### Static-export rules the agent must follow
- `next.config.ts` must set `output: 'export'`, `images: { unoptimized: true }`, `trailingSlash: true`.
- Use `<Image>` only with `unoptimized` (already global) or plain `<img>` with a `withBase()` helper. Pre-optimize images manually (WebP/AVIF, multiple sizes).
- Dynamic routes (`/projects/[slug]`) **must** export `generateStaticParams()` and `export const dynamicParams = false`.
- No `cookies()`, `headers()`, `useSearchParams()` without a Suspense boundary, no `route.ts` handlers, no `middleware.ts`.
- Anything that reads `window`, `localStorage`, `matchMedia` must be inside `useEffect` or a client component with a hydration-safe default.
- Add `public/.nojekyll` (so GitHub Pages serves the `_next/` folder).
- `404.html` is generated from `app/not-found.tsx`; make it a themed "Game Over / 404" page.

---

## 3. Repository Structure

```
project-arcade/
├─ .github/
│  └─ workflows/
│     └─ deploy.yml
├─ content/                          (ALL editable content as JSON; see §5)
│  ├─ projects.json
│  ├─ profile.json
│  ├─ skills.json
│  ├─ timeline.json
│  ├─ achievements.json
│  ├─ activity.json
│  └─ site.json
├─ schemas/                          (generated *.schema.json, committed)
├─ .vscode/settings.json             (maps content/*.json → schemas)
├─ public/
│  ├─ .nojekyll
│  ├─ favicon.ico / icon.svg / apple-touch-icon.png
│  ├─ og-default.png                  (1200x630)
│  ├─ resume/Chaitanya-Resume.pdf
│  ├─ avatar/avatar.webp
│  ├─ projects/<slug>/
│  │  ├─ cover.webp                   (460x215 library capsule, plus 2x)
│  │  ├─ hero.webp                    (1920x620 banner)
│  │  ├─ logo.webp                    (optional transparent logo)
│  │  └─ shots/01.webp ... 0N.webp    (screenshots, 1280x720)
│  └─ sounds/ (optional, tiny .mp3/.ogg; OFF by default)
├─ src/
│  ├─ app/
│  │  ├─ layout.tsx
│  │  ├─ page.tsx                     (home)
│  │  ├─ not-found.tsx
│  │  ├─ projects/
│  │  │  ├─ page.tsx                  (full library view)
│  │  │  └─ [slug]/page.tsx           (project detail)
│  │  ├─ about/page.tsx               ("Engineer Profile")
│  │  ├─ sitemap.ts
│  │  ├─ robots.ts
│  │  └─ globals.css
│  ├─ components/
│  │  ├─ layout/ (Navbar, MobileTabBar, Footer, CommandPalette, ThemeSwitcher, BackgroundFX)
│  │  ├─ home/ (Hero, FeaturedCarousel, LibraryPreview, SkillsSection, Timeline, StatsDashboard, ActivityFeed, ContactSection)
│  │  ├─ library/ (LibraryGrid, GameTile, FilterBar, SortMenu, TagChip, LibrarySidebar, EmptyState)
│  │  ├─ project/ (ProjectHero, ProjectMeta, ScreenshotGallery, Lightbox, AchievementList, TechBadges, LinkButtons, RelatedProjects)
│  │  └─ ui/ (Button, Badge, Card, Modal, Tooltip, Skeleton, ProgressBar, ScrollReveal, Counter)
│  ├─ data/
│  │  └─ activity.generated.json      (build-time generated, see §9.5)
│  ├─ lib/
│  │  ├─ paths.ts                     (withBase(), absoluteUrl())
│  │  ├─ schemas.ts                   (Zod schemas + inferred types)
│  │  ├─ content.ts                   (server-only loader: read JSON → validate → normalize)
│  │  ├─ projects.ts                  (getAll, getBySlug, getFeatured, getRelated; built on content.ts)
│  │  ├─ search.ts                    (Fuse config)
│  │  ├─ themes.ts
│  │  └─ utils.ts
│  ├─ hooks/ (useMediaQuery, useReducedMotion, useInView, useHotkey, useTheme, useLocalStorage)
│  └─ types/index.ts
├─ scripts/
│  ├─ generate-activity.mjs           (build-time GitHub activity fetch, optional)
│  ├─ validate-content.mjs            (Zod validation + file/asset cross-checks; runs in CI)
│  ├─ generate-schemas.mjs            (Zod → /schemas/*.schema.json)
│  ├─ new-project.mjs                 (CLI scaffold: pnpm new:project "Name")
│  └─ optimize-images.mjs             (sharp; optional helper)
├─ tests/ (unit + e2e)
├─ .env.example
├─ next.config.ts
├─ tailwind.config.ts (if v3) / postcss config
├─ tsconfig.json
├─ README.md
└─ plan.md  (this file)
```

---

## 4. Base Path & Asset Handling (critical for GitHub Pages)

```ts
// next.config.ts
import type { NextConfig } from 'next';

const basePath = process.env.NEXT_PUBLIC_BASE_PATH ?? ''; // '' for user site, '/repo-name' for project site

const nextConfig: NextConfig = {
  output: 'export',
  basePath,
  assetPrefix: basePath || undefined,
  trailingSlash: true,
  images: { unoptimized: true },
  reactStrictMode: true,
};
export default nextConfig;
```

```ts
// src/lib/paths.ts
export const BASE_PATH = process.env.NEXT_PUBLIC_BASE_PATH ?? '';
export const SITE_URL = process.env.NEXT_PUBLIC_SITE_URL ?? 'http://localhost:3000';

/** Use for every string path to something in /public (img src, pdf href, json fetch). */
export const withBase = (p: string) => `${BASE_PATH}${p.startsWith('/') ? p : `/${p}`}`;
export const absoluteUrl = (p: string) => `${SITE_URL}${withBase(p)}`;
```

**Rules**
- `next/link` handles basePath automatically. Plain `<a href="/resume/...pdf">`, `<img src>`, CSS `url()` pointing to `/public` do **not**; always wrap with `withBase()`.
- In CSS, avoid absolute `url(/...)` paths for public assets. Use inline `style` with `withBase()` or import the image.
- The deploy workflow sets `NEXT_PUBLIC_BASE_PATH` automatically from the repo name (see §14).
- Metadata `metadataBase` must be built from `NEXT_PUBLIC_SITE_URL` + base path.

---

## 5. Content Model — JSON-Driven (no code changes to add or edit projects)

### 5.1 Principle

**All site content lives in JSON files under `/content`. Component code must never contain project data.** To add a project, the owner adds one JSON object, drops images in a folder, and pushes. Nothing in `/src` changes.

Because hosting is static (GitHub Pages), "configurable" means:

1. JSON is read **at build time**, validated, and pre-rendered into static HTML.
2. A push to `main` triggers GitHub Actions, which rebuilds and redeploys (about 1–2 minutes).
3. If the JSON is invalid, **validation fails the build and nothing is deployed**, so the live site is never broken by a typo. The error message must name the file, the entry, and the field.

**Explicitly rejected:** fetching the JSON in the browser at runtime. It adds loading states, hurts SEO, and gains nothing, since a deploy is needed to change files anyway.

### 5.2 Content files

```
content/
├─ projects.json        # all projects (the main file the owner edits)
├─ profile.json         # name, headline, bio, socials, resume path, stats overrides
├─ skills.json          # skill groups for the skill tree + icon/label lookup for tech badges
├─ timeline.json        # career journey / quest log
├─ achievements.json    # profile-level achievements (unlocked / in progress)
├─ activity.json        # hand-written "Recent Releases" feed (see §9.5)
└─ site.json            # default theme, section on/off toggles, nav labels, section titles, SEO defaults
schemas/                # generated JSON Schemas (committed) for editor autocomplete
.vscode/settings.json   # maps content/*.json to schemas/*.schema.json
```

Every content file begins with a `"$schema"` key (for example `"$schema": "../schemas/projects.schema.json"`) so VS Code and GitHub-compatible editors give autocomplete and inline errors. The loader ignores `$schema`.

### 5.3 How content is loaded (build-time, server-only)

- `src/lib/content.ts` is **server-only** (`import 'server-only'`). It reads `/content/*.json` with `fs`, parses with **Zod**, normalizes (fills defaults, derives fields), and returns typed objects. This runs only during `next build` (Server Components, `generateStaticParams`, `generateMetadata`, `sitemap.ts`), which is fully compatible with `output: 'export'`.
- Client components (library filtering, carousel, command palette) receive **slim, serializable props** from a server page component. Do not import the full JSON into client bundles. For the search index, pass only `{ slug, title, tagline, tags, tech, categories }`.
- Results are memoized per build so each file is read and parsed once.
- Unknown keys are **errors** (`.strict()`), so typos like `"tehc"` are caught instead of silently ignored.
- JSON syntax errors must be reported with file name, line, and column.

### 5.4 `projects.json` — design goals

**The minimum viable entry is tiny; everything else has a sensible default or is derived.**

**Required fields:** `title`, `description`, `startDate`, `tech`.

| Field | Type | Default / behavior |
|---|---|---|
| `title` | string | required |
| `slug` | string (kebab-case) | derived from `title` if omitted; must be unique. Changing it changes the URL, so the owner should set it explicitly once published. |
| `tagline` | string ≤ 90 chars | falls back to the first sentence of `description` |
| `description` | string, 20–300 chars | required; used for tiles, meta description, OG tags |
| `longDescription` | string or string[] | optional; paragraphs for "About This Project" (a plain string is split on blank lines). Falls back to `description`. |
| `status` | `"Released"` \| `"In Development"` \| `"Early Access"` \| `"Prototype"` \| `"Archived"` | `"Released"` |
| `startDate` | `"YYYY-MM"` | required |
| `endDate` | `"YYYY-MM"` | omitted means ongoing, shown as "Jan 2025 – Present". Must not be earlier than `startDate`. |
| `role` | string | `""` (row hidden if empty) |
| `categories` | string[] | `[]`. **Free strings**, not a hardcoded enum. The library sidebar is generated from whatever categories are used. Known names (AI, Automation, Web, Cloud, Tools, Mobile, Data, Game) get an icon and color; unknown ones get a generic style. |
| `tags` | string[] | `[]`; searchable |
| `tech` | string[] | required. Each entry is matched case-insensitively against `skills.json` for its icon and label. **Unknown tech renders a generic badge and prints a build warning; it does not fail the build.** |
| `impactScore` | 1–5 | `3` |
| `featured` | boolean | `false` |
| `featuredOrder` | number | featured projects sort by this, then file order |
| `pinned` | boolean | `false`; pinned projects come first in the library |
| `hidden` | boolean | `false`; if true, excluded from every page, route, search, stat, and sitemap (acts as draft mode) |
| `version` | string | optional (e.g. `"v2.1"`) |
| `playtime` | string | optional (e.g. `"320 hrs"`) |
| `confidential` | boolean | `false`; if true, shows an "Enterprise / NDA" badge and notice |
| `demo` | URL | shorthand for the live-demo link (powers the "Launch Project" button) |
| `repo` | URL | shorthand for the GitHub link |
| `links` | `{ label, url, type? }[]` | extra links (case study, docs, video). `type` is inferred from the URL if omitted. |
| `video` | string (YouTube URL or `.mp4` path) | optional; rendered as a click-to-load facade |
| `achievements` | `(string \| { title, description?, icon?, rarity? })[]` | a string becomes `{ icon: "🏆", title, rarity: "common" }` |
| `metrics` | `{ [label: string]: string }` | e.g. `{ "Users": "100+", "Hours saved / week": "20+" }` |
| `related` | slug[] | falls back to computed similarity (shared tags, tech, categories) |
| `cover`, `hero`, `logo` | path under `/public` | **convention first** (below) |
| `screenshots` | `(string \| { src, alt?, caption? })[]` | **auto-discovered** if omitted (below) |

**Image conventions (so the JSON doesn't need to list files):**
- Folder per project: `public/projects/<slug>/`.
- Looks for `cover.webp|png|jpg`, `hero.webp|png|jpg`, `logo.webp|png|svg` automatically.
- Screenshots: if `screenshots` is omitted, the loader scans `public/projects/<slug>/shots/` at build time and uses the files sorted by filename (`01-dashboard.webp`, `02-report.webp`, ...). The alt text defaults to a humanized filename plus the project title. The JSON overrides this to add captions and alt text.
- **Missing images never break the page.** If there's no `cover` or `hero`, generate a **fallback cover** (deterministic gradient from the slug + category color + large title text, rendered as inline SVG/CSS). A project with zero images must still look intentional.
- Explicit paths in JSON are written relative to `/public` (for example `"/projects/my-app/cover.webp"`), never with the base path. The code applies `withBase()`.

**Derived at build time (never written by hand):** duration text ("3 months"), `projectsBuilt` count, top-technologies chart, library category counts, prev/next ordering, sitemap entries, JSON-LD.

**Ordering rules:** the default "Featured" sort is `pinned` first, then `featured` by `featuredOrder`, then **the order of entries in the file** (so the owner controls order by moving objects). Other sorts (Newest, Oldest, Impact, A–Z) are computed.

#### Example `content/projects.json`

```json
{
  "$schema": "../schemas/projects.schema.json",
  "projects": [
    {
      "title": "AI Incident Analyzer",
      "slug": "ai-incident-analyzer",
      "tagline": "Turns production incidents into RCA reports in minutes.",
      "description": "Support engineers spent hours writing root-cause analyses by hand. This tool ingests incident data and drafts a structured RCA with an LLM, cutting turnaround dramatically.",
      "longDescription": [
        "The problem: RCA documents were slow, inconsistent, and often skipped.",
        "The solution: a pipeline on Azure that collects logs and tickets, summarizes them with Azure OpenAI, and produces a reviewed draft."
      ],
      "status": "Released",
      "version": "v2.0",
      "startDate": "2025-01",
      "endDate": "2025-03",
      "role": "Solo developer",
      "categories": ["AI", "Automation", "Cloud"],
      "tags": ["RCA", "incident management", "LLM"],
      "tech": ["Azure", "Azure OpenAI", "Python", "React"],
      "impactScore": 5,
      "featured": true,
      "featuredOrder": 1,
      "demo": "https://example.com/demo",
      "repo": "https://github.com/your-username/ai-incident-analyzer",
      "links": [{ "label": "Case study", "url": "https://example.com/case-study" }],
      "achievements": [
        "Reduced manual effort by 80%",
        { "title": "Saved 20+ engineering hours/week", "rarity": "epic" },
        { "title": "Used by 100+ support engineers", "rarity": "legendary" }
      ],
      "metrics": { "Users": "100+", "Hours saved / week": "20+" },
      "related": ["ai-rca-generator"]
    },
    {
      "title": "Quick Demo Bot",
      "description": "A minimal entry: only the four required fields are needed for a project to appear.",
      "startDate": "2024-06",
      "tech": ["Node.js", "Teams"]
    }
  ]
}
```

### 5.5 Schema, types, and validation

- Define **Zod schemas** in `src/lib/schemas.ts` (input shape with defaults) and export TypeScript types with `z.infer`. The **normalized `Project` type** (all defaults applied, all derived fields present, image paths resolved, `links` always an array, `achievements` always objects) is what components consume.
- Key schema rules (sketch):

```ts
import { z } from 'zod';

const YearMonth = z.string().regex(/^\d{4}-(0[1-9]|1[0-2])$/, 'Use YYYY-MM, e.g. 2025-03');
const HttpsUrl = z.string().url().refine(u => u.startsWith('https://'), 'URL must start with https://');
const Slug = z.string().regex(/^[a-z0-9]+(?:-[a-z0-9]+)*$/, 'Use lower-case-kebab-case');

export const ProjectInput = z.object({
  title: z.string().min(1),
  slug: Slug.optional(),
  description: z.string().min(20).max(300),
  startDate: YearMonth,
  endDate: YearMonth.optional(),
  tech: z.array(z.string().min(1)).min(1),
  status: z.enum(['Released','In Development','Early Access','Prototype','Archived']).default('Released'),
  impactScore: z.number().int().min(1).max(5).default(3),
  demo: HttpsUrl.optional(),
  repo: HttpsUrl.optional(),
  // ...all other fields from the table in §5.4
}).strict();
```

- **`scripts/validate-content.mjs`** (run by `pnpm validate`, and as a CI step before build) performs:
  - **Errors (fail the build):** invalid JSON; schema violations; duplicate slugs (including derived ones); `related` pointing to a missing slug; `endDate` before `startDate`; an explicitly referenced image file that does not exist; non-https URLs; a `featured` project that is also `hidden`.
  - **Warnings (print, don't fail):** unknown `tech` key; missing screenshots; image over 300 KB; missing alt text; `tagline` over 90 chars; no `demo` and no `repo` (the tile's "Launch" will be disabled).
- Error format must be human-friendly, for example:
  `✖ content/projects.json → projects[2] ("Teams Bot") → startDate: Use YYYY-MM, e.g. 2025-03 (got "March 2025")`
- **`scripts/generate-schemas.mjs`** (`pnpm schemas`) converts the Zod schemas into `schemas/*.schema.json` using `zod-to-json-schema`. Commit the output. CI fails if the committed schemas are out of date (run the generator and `git diff --exit-code`).
- `.vscode/settings.json` maps `content/projects.json` to `schemas/projects.schema.json` and so on, with `json.schemaDownloadEnable` off.

### 5.6 Other content files (shapes)

```json
// content/profile.json
{
  "$schema": "../schemas/profile.schema.json",
  "name": "Chaitanya",
  "handle": "CHAITANYA.JS",
  "headline": "Building Products, Automations & AI",
  "subheadline": "Engineer shipping AI-powered tools, cloud apps and automations.",
  "nowPlaying": "Now building: AI RCA Generator v3",
  "location": "India",
  "email": "you@example.com",
  "resume": "/resume/Chaitanya-Resume.pdf",
  "avatar": "/avatar/avatar.webp",
  "bio": ["Short paragraph one.", "Short paragraph two."],
  "socials": [
    { "type": "github", "url": "https://github.com/your-username" },
    { "type": "linkedin", "url": "https://linkedin.com/in/your-username" }
  ],
  "githubUsername": "your-username",
  "stats": { "yearsExperience": 6, "technologies": null, "contributions": 100 }
}
```
(`technologies: null` means "compute from skills.json". `projectsBuilt` is always computed from non-hidden projects, but may be overridden with an explicit number.)

```json
// content/skills.json
{
  "groups": [
    {
      "name": "Cloud",
      "icon": "cloud",
      "skills": [
        { "key": "azure", "label": "Azure", "level": 5, "yearsUsed": 5, "icon": "azure" },
        { "key": "azure-openai", "label": "Azure OpenAI", "aliases": ["aoai"], "level": 4 }
      ]
    }
  ]
}
```
(Tech badges look up `key`, `label`, and `aliases` case-insensitively.)

```json
// content/timeline.json
{ "items": [
  { "type": "work", "title": "Software Engineer", "org": "Company", "start": "2022-01",
    "summary": "One or two sentences.", "highlights": ["Did X", "Shipped Y"] }
] }
```

```json
// content/achievements.json
{ "items": [
  { "title": "Azure Expert", "description": "Designed and shipped N Azure solutions.", "icon": "🏆", "rarity": "epic", "unlocked": true },
  { "title": "Kubernetes", "icon": "☸️", "rarity": "rare", "unlocked": false, "progress": 40 }
] }
```

```json
// content/activity.json  (hand-written feed; GitHub-generated entries are merged in if present)
{ "items": [
  { "title": "AI RCA Generator v2", "date": "2025-03-14", "project": "ai-rca-generator", "version": "v2.0" }
] }
```

```json
// content/site.json
{
  "defaultTheme": "steam",
  "sections": { "featured": true, "library": true, "skills": true, "timeline": true, "stats": true, "activity": true, "contact": true },
  "sectionTitles": { "featured": "Featured Projects", "library": "Game Library", "skills": "Skills & Achievements", "timeline": "Quest Log", "stats": "Engineer Profile", "activity": "Recent Releases", "contact": "Let's Co-op" },
  "seo": { "titleTemplate": "%s · Chaitanya — Project Arcade", "defaultDescription": "Portfolio of Chaitanya: AI, automation and cloud projects." }
}
```

All of these get Zod schemas, `$schema` files, and validation just like `projects.json`. Section toggles in `site.json` must actually hide the section, its nav/tab entry, and its anchor link.

### 5.7 No hardcoded content in code (enforced)

- Components, pages, and helpers must not contain project titles, slugs, tech names, category lists, socials, stats numbers, nav section titles, or copy that is in the JSON. Everything is read from the content loader.
- Add an ESLint rule or a simple CI grep check that fails if any seeded project slug appears under `/src` (excluding tests and `src/lib/content*`).
- Filters, counts, charts, sidebar categories, related projects, sitemap, and static routes are all **derived** from the loaded JSON.
- Static routes: `generateStaticParams()` returns the slugs of all non-hidden projects, so adding a JSON entry automatically creates `/projects/<slug>/`.

### 5.8 Owner workflows (must be documented in the README)

1. **Easiest, no local setup:** on GitHub, open `content/projects.json` → pencil icon → add an object → *Commit*. Upload images with *Add file → Upload files* into `public/projects/<slug>/` (screenshots in `shots/`). The deploy runs automatically; if validation fails, the Actions log shows the exact error and the live site stays unchanged.
2. **Local, with a helper:** `pnpm new:project "My New Project"` runs `scripts/new-project.mjs`, which creates `public/projects/<slug>/shots/` (with a README note on image sizes) and appends a stub entry (title, slug, required fields with `TODO` values, `hidden: true` so unfinished work isn't published) to `content/projects.json` while preserving formatting. Then the owner fills it in, removes `hidden`, runs `pnpm validate && pnpm dev`, and pushes.
3. **Hide a project temporarily:** set `"hidden": true`.
4. **Reorder:** move the object in the file, or set `pinned` / `featuredOrder`.
5. **Optional later upgrade (out of scope for v1):** a Git-backed CMS UI such as Decap or Sveltia could edit these same JSON files through a form. Keep file formats CMS-friendly (flat objects, no comments) to allow this.

### 5.9 Seed content

Create realistic placeholder content **in the JSON files** (not in code): at least **8 projects** (2 confidential, 5 featured), 6 skill groups, 6+ timeline items, 9 achievements (3 locked), 4+ activity items. Seed projects: *AI Incident Analyzer, AI Meeting Bot, Teams Productivity Bot, Azure Monitoring Dashboard, AI RCA Generator, Portfolio Website, Power Platform Approval Suite, Support Knowledge Search*. Include one deliberately minimal 4-field entry (hidden) as a documented example. Generate gradient placeholder images locally (no hotlinking). Mark placeholders with a `"_todo": "replace"` convention that the schema allows (an optional underscore-prefixed `_todo` string on any object), and list them in the README.

---

## 6. Design System

### 6.1 Theme system
Implement themes as CSS variables on `[data-theme="..."]` on `<html>`. Default = **Steam**. Persist choice in `localStorage` (guard for SSR; avoid flash by setting the attribute with a tiny inline script in `<head>` that reads `localStorage` before paint).

| Token | Steam (default) | Cyberpunk | Xbox |
|---|---|---|---|
| `--bg` | `#171A21` | `#0A0A0A` | `#0B0B0B` |
| `--bg-elevated` | `#1B2838` | `#12121A` | `#107C10` at 15% over bg |
| `--surface` | `#1F2F44` | `#16161F` | `#141A14` |
| `--border` | `#2A475E` | `#00F5FF33` | `#9BF00B33` |
| `--accent` | `#66C0F4` | `#00F5FF` | `#9BF00B` |
| `--accent-2` | `#1A9FFF` | `#FF00AA` | `#107C10` |
| `--text` | `#FFFFFF` | `#F5F5F5` | `#FFFFFF` |
| `--text-muted` | `#8F98A0` | `#9A9AB0` | `#9DB09D` |
| `--success/--warn/--danger` | green/amber/red tuned per theme | | |

- Ensure **WCAG AA** text contrast in all three themes. Verify programmatically in a unit test for `--text` / `--text-muted` on `--bg` and `--surface`.
- Theme switcher: small palette icon in navbar → popover with 3 swatches. Keyboard accessible.
- Tailwind: map tokens to utilities (`bg-bg`, `text-muted`, `border-border`, `text-accent`, etc.).

### 6.2 Typography
- Headings: Rajdhani/Orbitron, uppercase, letter-spaced (`tracking-wider`) for section titles. Body: Inter. Numbers/stats: JetBrains Mono.
- Fluid type via `clamp()`: hero title `clamp(2.25rem, 7vw, 5rem)`.

### 6.3 Visual language
- Rounded-sm corners (4–8px), thin 1px borders, subtle inner glow on hover (`box-shadow: 0 0 0 1px var(--accent), 0 8px 30px -8px var(--accent)/40%`).
- Section headers styled like launcher categories: `── FEATURED ──`, with a small accent bar.
- Subtle scanline / noise overlay at 3–4% opacity (static CSS, no animation by default).
- Status "pills" like game states: `Released` (green), `In Development` (amber), `Early Access` (blue), `Archived` (grey).
- Impact score = 5 stars with half-star not needed; show `4.0 / 5 Impact`.
- Rarity colors for achievements: common grey, rare blue, epic purple, legendary gold. Legendary gets a one-time shimmer sweep when scrolled into view (disabled under reduced motion).

### 6.4 Breakpoints & layout
- Mobile-first. Use Tailwind defaults: `sm 640`, `md 768`, `lg 1024`, `xl 1280`, `2xl 1536`.
- Content max width `1280px` (`max-w-screen-xl`) with `px-4 sm:px-6 lg:px-8`.
- Respect safe-area insets on notched phones (`env(safe-area-inset-*)`), especially for the bottom tab bar.
- Use `min-h-[100svh]` not `100vh` for the hero (mobile browser chrome).

---

## 7. Pages & Routes

| Route | Purpose |
|---|---|
| `/` | Home: hero → featured carousel → library preview → skills/achievements → timeline → stats → activity → contact |
| `/projects/` | Full library with sidebar filters, search, sort, grid/list toggle |
| `/projects/[slug]/` | Store-style detail page |
| `/about/` | "Engineer Profile": bio, stats, skill tree, timeline, resume button |
| `/404` | Themed "GAME OVER – Page not found" with button back to library |

Add a **sitemap.xml** and **robots.txt** via `app/sitemap.ts` / `app/robots.ts` (both must be statically exportable: add `export const dynamic = 'force-static'`).

---

## 8. Global Layout Components

### 8.1 Navbar (desktop ≥ md)
- Left: logo mark + `CHAITANYA.JS`.
- Center links: `Store` (home), `Library` (/projects), `Profile` (/about), `Contact` (scroll to section / `/#contact`).
- Right: search button (`Ctrl/⌘+K` hint), theme switcher, `Resume` button (download).
- Sticky, translucent blur background, shrinks slightly on scroll. Active route indicator with an animated underline (`layoutId`).
- Hidden on scroll-down / shown on scroll-up is optional; keep it always visible if it risks jank.

### 8.2 Mobile navigation (< md)
- **Top bar:** logo + search icon + theme icon.
- **Bottom tab bar** (fixed, safe-area aware): `Home`, `Library`, `Profile`, `Contact`. Icons + labels, 44px+ touch targets, active tab highlighted.
- Add bottom padding to `<main>` so content isn't hidden behind the bar.

### 8.3 Command Palette / Search (signature feature)
- Opens via `Ctrl/⌘+K`, `/` key, or the search button/icon. Full-screen sheet on mobile, centered modal on desktop.
- Searches projects (title, tagline, tags, tech, categories) with Fuse.js; also lists quick actions (Go to Library, Open Resume, Switch theme, Open GitHub, Email me).
- Arrow keys + Enter navigation, `Esc` closes, focus trapped, `aria-modal`, announces result count to screen readers.
- Recent selections stored in `localStorage` (optional).

### 8.4 Footer
- Short sign-off ("Thanks for playing 🎮"), social icons, "Built with Next.js · Hosted on GitHub Pages", link to repo, back-to-top button, current year.

### 8.5 BackgroundFX
- Fixed full-viewport canvas **behind** content (`z-index: -1`, `pointer-events: none`).
- Particles: count scales with viewport (`≈ width*height/18000`, max 90 desktop, max 35 mobile), slow drift, faint connecting lines only on desktop, accent-colored using current theme token.
- Pause on `document.hidden`, cap to 30–45 fps on low-power devices, and disable completely when `prefers-reduced-motion` is set or `navigator.hardwareConcurrency <= 2`.
- Optional subtle mouse parallax on desktop only.
- Must not cause layout shift or hurt LCP (mount after first paint with `requestIdleCallback`).

---

## 9. Home Page Sections (in order)

### 9.1 Hero — "Press Start"
- Full-height (svh) hero with BackgroundFX visible.
- Left/center: small label `PLAYER ONE`, big glitch-free gradient title `CHAITANYA.JS` (typing/“boot” reveal animation, ≤ 1.2s, skipped under reduced motion), subtitle "Building Products, Automations & AI".
- Buttons: **[▶ Explore Library]** (scrolls to library / navigates to `/projects`), **[⬇ Resume]**.
- Right (desktop) / above title (mobile): **futuristic avatar** — avatar image in a hexagon/angled frame with animated neon border (CSS conic-gradient rotation), small floating "Level 6 · Engineer" chip. Provide a placeholder avatar SVG.
- A "scroll" hint chevron at the bottom.
- Optional: a mini "now playing" ticker (marquee) of current focus, e.g., "Now building: AI RCA Generator v3" (data-driven from `profile.nowPlaying`).

### 9.2 Featured Project Carousel (Steam "Featured & Recommended")
- Large banner carousel for projects where `featured: true` ordered by `featuredOrder`.
- Desktop layout: big hero image left (≈ 70%) + right column showing 3–4 screenshot thumbnails and key details (title, short description, tech chips, status, impact stars, buttons). Hovering over a right-hand "queue" item previews it in the main panel, as Steam does.
- Mobile layout: single swipeable card (CSS scroll-snap or Framer drag), dots indicator, buttons stacked.
- Autoplay every 6s, **pauses on hover/focus/touch and when offscreen**, no autoplay under reduced motion. Visible pause/play button (accessibility).
- Buttons: `[View Details]` → detail page, `[Live Demo]` and `[GitHub]` if available (external).
- Arrow buttons + keyboard left/right + swipe. Add progress bar for the current slide timer.
- ARIA: `role="region" aria-roledescription="carousel"`, slides labelled "n of N".

### 9.3 Game Library Preview (top 8 projects + "View all")
- Grid of `GameTile`s: 2 cols mobile, 3 cols `md`, 4 cols `xl`. Cover aspect ratio ≈ 460:215 (Steam capsule) or 3:4 portrait (user-switchable on `/projects`).
- **GameTile (resting):** cover image, title overlay at the bottom gradient, status dot, category icon.
- **GameTile (hover/focus on desktop):** tile lifts + scales ~1.04, glow border, overlay expands to show: title, tagline, tech chips (max 3 + "+n"), impact stars, and two actions: `▶ Launch Project` (opens live demo/primary link in new tab; if none, disabled with tooltip "Source only") and `ℹ View Details`.
- **Touch devices:** no hover. First tap = open the overlay (same content as hover) and show a small ✕; second tap on `View Details` navigates. Alternatively always show a compact info row under the cover on mobile. **Default decision: always show title + status + 2 tech chips under the cover on mobile, and tapping the card navigates to details directly** (simplest, most usable). Provide a "Launch" icon button inside the tile for the live demo.
- Detect hover capability with `@media (hover: hover) and (pointer: fine)`; never rely on user-agent sniffing.
- "View full library →" button to `/projects`.
- Staggered entrance animation using `whileInView` with `once: true`.

### 9.4 Skills & Achievements
Two views via tabs: **Skill Tree** and **Achievements**.
- **Skill Tree:** grouped cards (Frontend, Backend, Cloud, AI/ML, Automation/Low-code, DevOps & Tools). Each group is an expandable tree (`├──`, `└──` styling with CSS pseudo-elements, not ASCII text), each skill shows icon, label, and a level bar (5 pips). On mobile, groups are accordions (first one open).
- **Achievements:** grid of badges (e.g. "Azure Expert", "React Developer", "Python Automation", "Generative AI", "Incident Management", "Power Platform"). Locked-style design for unlocked ones is a checkmark ✅ + rarity glow; optionally include 2–3 "locked" ones labelled "In progress" with progress bars (e.g., "Kubernetes — 40%") to show growth. Add a global counter "Achievements unlocked: 6 / 9" with a progress bar.
- Unlock animation when the section scrolls into view (stagger + subtle pop). Optional tiny sound if user enabled sound (default off).

### 9.5 Recent Activity Feed ("Recent Releases")
- Vertical list styled like a game activity feed: icon, title, version, relative date ("2 weeks ago"), link to project.
- **Primary source:** `content/activity.json` (hand-edited JSON).
- **Optional enhancement (still static):** `scripts/generate-activity.mjs` runs at build time in GitHub Actions, fetches the owner's latest public GitHub events/releases using the built-in `GITHUB_TOKEN` (server-side in CI, never shipped to the client) and writes `src/data/activity.generated.json`. If the fetch fails or is absent, the site silently falls back to the hand-edited list. A scheduled workflow (`cron: '0 3 * * *'`) rebuilds daily so it stays fresh.
- Do **not** call the GitHub API from the browser (rate limits).

### 9.6 Timeline / Career Journey
- Vertical timeline (alternating left/right on desktop, single column left-rail on mobile).
- Each node: date range, role/title, org, summary, highlight bullets. Node icons by type (briefcase, graduation cap, flag, award).
- Scroll-linked "progress line" that fills as the user scrolls (Framer `useScroll` + `scaleY`); static under reduced motion.
- Styled as "Quest Log": `COMPLETED`, `IN PROGRESS` tags.

### 9.7 Stats Dashboard ("Engineer Profile")
- Row/grid of stat cards: **Projects Built** (computed from data), **Years Experience**, **Technologies**, **Contributions**.
- Animated counters triggered once on scroll into view, using `requestAnimationFrame` with easing, ≤ 1.5s. Show the final value immediately when reduced motion is on or JS is not hydrated (render the real number in the HTML, then animate from 0 on view; avoid showing "0" to crawlers).
- Add a "Top technologies" horizontal bar chart (pure CSS/SVG, no chart library) computed from `projects[].tech` frequency.
- Optional: GitHub contribution graph image is **not** to be hotlinked; skip or build from generated data.

### 9.8 Contact / Social Links
- Section titled "Let's Co-op" / "Join My Party" with: email button (`mailto:`), copy-email-to-clipboard with toast, social icon buttons (GitHub, LinkedIn, X, etc.), resume button.
- **Contact form (optional, static-friendly):** posts to a third-party endpoint (Formspree / Web3Forms / Getform). Endpoint URL comes from `NEXT_PUBLIC_CONTACT_FORM_ENDPOINT`. If the env var is empty, hide the form and show only the mailto/social buttons. Include a honeypot field, client-side validation (name, email, message), loading/success/error states, and accessible error text. No custom backend.

---

## 10. Library Page (`/projects/`)

Layout mimics the Steam library window.

**Desktop (≥ lg):**
- Left sidebar (collapsible, ~260px): search input, category list as "collections" with counts (`All Projects (12)`, `AI (5)`, `Automation (4)`, …), status filter checkboxes, tech filter (multi-select chips, top 12 + "show more"), "Clear filters".
- Main area: top toolbar with result count, sort dropdown (`Featured`, `Newest`, `Oldest`, `Impact`, `A–Z`), view toggle (Grid / List / Compact), and active filter chips.
- Grid uses the same `GameTile`. List view shows a wide row: small cover, title, tagline, tech chips, status, dates, impact, playtime.

**Mobile (< lg):**
- Sticky filter bar with a search field and a `Filters` button that opens a **bottom sheet** (drag-to-close, focus trap) containing the sidebar content. Horizontal scrollable category chips under the search bar.

**Behavior**
- Filtering/sorting is fully client-side. Reflect state in the URL **hash or query string** for shareable links: because this is a static export, read it with `useSearchParams` **inside a `<Suspense>` boundary** in a client component, and sync with `router.replace` (shallow, scroll: false). Verify it works after export.
- Animated reorder using Framer `layout` (disable under reduced motion and for > 40 items).
- Empty state: "No games match your filters" with Clear Filters button.
- Keyboard: `/` focuses the search; filter controls are native inputs/buttons with visible focus rings.

---

## 11. Project Detail Page (`/projects/[slug]/`)

Looks like a store page.

1. **Hero banner:** large `hero` image with gradient overlay, project logo/title, tagline, status pill, version. On mobile the banner is shorter (≈ 40svh) with the title below.
2. **Top-right "purchase box" (sticky on desktop, stacked on mobile):** cover capsule, short description, **Impact** stars with numeric score, key meta rows — `Status`, `Role`, `Duration` (computed, e.g. "Jan 2025 – Mar 2025 · 3 months"), `Version`, `Playtime`, `Categories` — and CTA buttons: `[▶ Launch / View Live Demo]`, `[GitHub]`, plus additional links. Missing links are omitted; if `confidential`, show an "Enterprise project — source and demo under NDA" notice.
3. **Media gallery:** primary large viewer + thumbnail strip (horizontal scroll, scroll-snap, drag/swipe on touch, arrow buttons on desktop). Clicking opens a **Lightbox** (keyboard arrows, swipe, pinch-to-zoom is optional, `Esc` closes, focus trap, captions, lazy-loaded). Video (YouTube) must be a click-to-load facade (thumbnail + play button) to avoid loading third-party iframes by default; use `youtube-nocookie.com`.
4. **About This Project:** `longDescription` paragraphs, key metrics as stat tiles.
5. **Technologies:** badges with icons, each linking to the library filtered by that tech.
6. **Achievements:** list/grid with rarity styling (🏆 icons), e.g. "Reduced manual effort by 80%".
7. **Tags / Genres:** chips linking to filtered library.
8. **Related / "More like this":** 3–4 tiles from `related` or computed by overlapping tags/tech.
9. **Prev / Next project** navigation at the bottom.
10. **Breadcrumb:** `Library › Project name` (with JSON-LD `BreadcrumbList`).

SEO per page: unique `<title>`, description, canonical, OpenGraph/Twitter using the project's `hero` (or `cover`) image, JSON-LD `CreativeWork`/`SoftwareSourceCode` where relevant.

---

## 12. About / Profile Page (`/about/`)
- Gaming-profile header: avatar frame, handle, "Level" (years of experience), location, tagline, bio (2 short paragraphs), resume + contact buttons.
- Stats Dashboard (reuse component), Skill Tree, Achievements, Timeline (reuse).
- "Loadout" section: current tools/setup (editor, OS, favorite stack), optional and fun.

---

## 13. Motion, Interaction & Polish

- Animation principles: fast (150–300ms) for UI feedback; ≤ 700ms for entrances; ease-out curves; animate `transform` and `opacity` only.
- Page transitions: lightweight fade/slide using `template.tsx` + Framer (keep route change instant; no blocking exit animations).
- Scroll reveal via `whileInView` with `viewport={{ once: true, margin: '-80px' }}`.
- Buttons: hover glow, press scale 0.97, focus-visible ring using `--accent`.
- Cursor glow effect on desktop only (radial gradient following the mouse), disabled on touch/reduced motion.
- **Easter eggs (cheap, delightful):**
  - Konami code (↑↑↓↓←→←→BA) switches to a temporary "Rainbow/Retro" theme and shows "Cheat code activated!" toast.
  - Console message in DevTools greeting recruiters with contact info.
  - 404 page includes a tiny playable-ish element (e.g., blinking "INSERT COIN" text) — no heavy game logic.
- **Optional sound:** a global "Sound: OFF/ON" toggle in the navbar, default OFF, never autoplay, tiny UI blips for tile hover/click, preloaded lazily only after the user enables it.
- **Loading screen:** NOT a blocking splash screen (hurts LCP and recruiters hate it). If included, make it ≤ 600ms, skippable, and shown only on first visit per session.

---

## 14. Deployment — GitHub Pages via GitHub Actions

### 14.1 Workflow (`.github/workflows/deploy.yml`)

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:
  schedule:
    - cron: '0 3 * * *'   # daily rebuild to refresh generated GitHub activity

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm

      - run: pnpm install --frozen-lockfile

      - name: Compute base path
        id: base
        run: |
          REPO="${GITHUB_REPOSITORY#*/}"
          OWNER="${GITHUB_REPOSITORY_OWNER}"
          if [ "$REPO" = "${OWNER}.github.io" ]; then
            echo "path=" >> "$GITHUB_OUTPUT"
            echo "url=https://${OWNER}.github.io" >> "$GITHUB_OUTPUT"
          else
            echo "path=/${REPO}" >> "$GITHUB_OUTPUT"
            echo "url=https://${OWNER}.github.io" >> "$GITHUB_OUTPUT"
          fi

      - name: Validate content
        run: pnpm validate && pnpm schemas && git diff --exit-code schemas/

      - name: Generate GitHub activity (optional, non-fatal)
        run: node scripts/generate-activity.mjs || echo "activity generation skipped"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Lint, typecheck, test
        run: |
          pnpm lint
          pnpm typecheck
          pnpm test --run

      - name: Build static site
        run: pnpm build          # next build with output: 'export' → ./out
        env:
          NEXT_PUBLIC_BASE_PATH: ${{ steps.base.outputs.path }}
          NEXT_PUBLIC_SITE_URL: ${{ steps.base.outputs.url }}
          NEXT_PUBLIC_CONTACT_FORM_ENDPOINT: ${{ vars.CONTACT_FORM_ENDPOINT }}
          NEXT_PUBLIC_ANALYTICS_ID: ${{ vars.ANALYTICS_ID }}

      - name: Add SPA-style fallback
        run: cp out/404.html out/404.html || true

      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./out

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

### 14.2 One-time owner setup (document this in `README.md`)
1. Create the GitHub repo and push the code to `main`.
2. Repo **Settings → Pages → Build and deployment → Source: GitHub Actions**.
3. (Optional) Add repository **Variables**: `CONTACT_FORM_ENDPOINT`, `ANALYTICS_ID`.
4. (Optional) Custom domain: add `public/CNAME` containing the domain, configure DNS, and set `NEXT_PUBLIC_SITE_URL` to the custom domain and `NEXT_PUBLIC_BASE_PATH` to empty.
5. Local dev: `pnpm dev` (no base path). To test the production base path locally: `NEXT_PUBLIC_BASE_PATH=/repo-name pnpm build && npx serve out` (serve at the matching sub-path or use a small script in `package.json` named `preview:pages`).

### 14.3 Static-hosting gotchas the agent must verify
- Deep links (`/projects/some-slug/`) load directly (thanks to `trailingSlash: true` creating `index.html` per route).
- `_next/` assets load (`.nojekyll` present).
- Case sensitivity: GitHub Pages is case-sensitive — keep file names and slugs lower-case and match references exactly.
- Unknown URLs show the themed `404.html`.
- No requests to `localhost` or absolute `/` paths that ignore `basePath`.

---

## 15. Accessibility (WCAG 2.2 AA)

- Semantic landmarks: `header`, `nav`, `main`, `section` with headings in logical order (one `h1` per page), `footer`.
- "Skip to content" link as the first focusable element.
- Visible `:focus-visible` outlines in all themes; never remove outlines.
- All interactive elements are real `<button>`/`<a>`; icon-only buttons have `aria-label`s; tiles are a single focusable link (avoid nested interactive elements; use a stretched-link pattern with secondary buttons positioned above via z-index and separate tab stops).
- Images have meaningful `alt`; decorative effects use `aria-hidden`.
- Carousel, tabs, accordions, dialogs, bottom sheets and command palette follow WAI-ARIA authoring patterns (focus trap, `Esc`, return focus to trigger).
- Respect `prefers-reduced-motion`, `prefers-color-scheme` (initial theme default is Steam dark either way; do not auto-switch to a light theme), and `prefers-contrast` where cheap.
- Minimum touch target 44×44px. No content conveyed by color alone (status pills include text/icons).
- Run `axe` checks in Playwright on Home, Library, and a Detail page; fix all serious/critical findings.

---

## 16. Performance Plan

- Server-render/pre-render everything; keep client components small and leaf-level (`"use client"` only where needed).
- Images: WebP/AVIF, covers ≤ 60KB, heroes ≤ 200KB, screenshots ≤ 150KB; always set `width`/`height` (or aspect-ratio) to prevent CLS; `loading="lazy"` + `decoding="async"` except the LCP image (hero/featured first slide: `fetchpriority="high"`).
- Provide `scripts/optimize-images.mjs` (using `sharp`) to convert/resize images dropped into `public/projects/<slug>/raw/` into the required sizes.
- Dynamic imports (`next/dynamic`, `ssr: false` where required) for: Lightbox, CommandPalette, BackgroundFX, three.js scene, contact form.
- Fonts: `display: 'swap'`, subset to Latin, limit weights.
- Avoid large icon imports (import individual Lucide icons).
- Third-party scripts (analytics) loaded with `strategy="afterInteractive"` (or plain deferred script), only when configured.
- Track bundle size: fail CI if first-load JS for `/` exceeds **~180 KB gzipped** (use `@next/bundle-analyzer` locally and a simple size check script in CI).

---

## 17. SEO, Sharing & Meta

- `metadataBase`, default title template `%s · Chaitanya — Project Arcade`.
- OpenGraph/Twitter cards (`summary_large_image`) with a static `og-default.png` and per-project images (use hero). Since there is no dynamic OG generation without a server, pre-generate images or reuse `hero` images; do **not** use `ImageResponse` routes unless they are statically exportable.
- JSON-LD `Person` on home/about (name, jobTitle, sameAs socials).
- `sitemap.xml` includes all project routes; `robots.txt` points to it.
- Semantic URLs: `/projects/ai-incident-analyzer/`.
- Add `manifest.webmanifest` + theme color meta (matches the active theme's `--bg`) so the mobile browser chrome blends in.

---

## 18. Privacy & Analytics

- Analytics optional and privacy-friendly (Umami/Plausible). Load only if `NEXT_PUBLIC_ANALYTICS_ID` (and script URL) are provided.
- No cookies beyond what the analytics provider needs (prefer cookieless). If any non-essential cookies/third-party embeds are added, show a minimal consent notice. YouTube is click-to-load for this reason.
- `localStorage` usage limited to theme, sound toggle, and recent searches; wrap in try/catch (Safari private mode).

---

## 19. Testing & QA

**Unit (Vitest + RTL)**
- Content loader/normalizer (defaults, slug derivation, shorthand `demo`/`repo`/`achievements`/`metrics`, screenshot auto-discovery, fallback covers, `hidden` filtering, readable error messages for bad JSON), `withBase`, project helpers (`getBySlug`, `getRelated`), duration formatting, search ranking, filter/sort logic, stat counters (renders final value without JS animation under reduced motion), theme contrast ratios.

**E2E (Playwright, run against `out/` served statically)**
- Home loads, hero buttons work, carousel next/prev/pause, command palette opens with `Ctrl+K` and navigates.
- Library filter + search + sort update results and URL; reload preserves state.
- Detail page deep-link works with direct navigation; gallery and lightbox keyboard controls work.
- 404 page appears for an unknown route.
- Run on projects: Desktop Chrome, Mobile Chrome (Pixel 7), Mobile Safari (iPhone 14), and a tablet viewport.
- `axe-core` scan on key pages.
- Base-path test: build once with `NEXT_PUBLIC_BASE_PATH=/test-repo`, serve under that prefix, assert no broken asset/link requests (monitor `page.on('response')` for 404s).

**Manual checklist (document in `README.md`)**
- iOS Safari: bottom bar vs. home indicator, `100svh` hero, no horizontal scroll, tap targets.
- Android Chrome: swipe carousels do not hijack vertical page scroll (`touch-action: pan-y` on carousels).
- Landscape phones; zoomed 200% text; slow 4G throttling; reduced-motion on.

---

## 20. Build Phases (execute in order; commit after each)

Each phase ends with: `pnpm lint && pnpm typecheck && pnpm build` passing, plus the acceptance criteria listed.

### Phase 0 — Scaffolding
- Init Next.js 15 + TS + Tailwind, ESLint/Prettier, pnpm, strict TS, path alias `@/*`.
- Configure `next.config.ts` per §4, add `public/.nojekyll`, `.env.example`, scripts (`dev`, `build`, `start`, `lint`, `typecheck`, `test`, `validate`, `schemas`, `new:project`, `preview:pages`).
- **Accept:** `pnpm build` produces `out/` with `index.html`.

### Phase 1 — Design system & layout shell
- Theme tokens + switcher (no-flash script), fonts, global CSS, `ui/` primitives, Navbar, MobileTabBar, Footer, skip link.
- **Accept:** All three themes switch instantly; layout is correct at 360 / 768 / 1280 / 1920px; keyboard nav works.

### Phase 2 — JSON content layer
- Zod schemas, the server-only content loader/normalizer (defaults, derived fields, image conventions, fallback covers, `hidden`), JSON files in `/content` with seed data (§5.9), generated JSON Schemas, `.vscode/settings.json`, `validate-content.mjs`, `generate-schemas.mjs`, `new-project.mjs`.
- Build a temporary `/dev/content` debug page (excluded from production export) that lists all loaded projects and any warnings.
- **Accept:** Adding a 4-field project to `projects.json` makes it appear on the home page, library, search, sitemap, and `/projects/<slug>/` with no source edits. Breaking a field (bad date, duplicate slug, typo'd key, missing image) fails `pnpm validate` with a readable file/entry/field message. `hidden: true` removes a project everywhere. `pnpm new:project "Test"` scaffolds correctly.

### Phase 3 — Home: Hero, Library Preview, GameTile
- Hero with avatar frame and BackgroundFX, `GameTile` with hover (fine pointer) and touch behavior, library preview grid.
- **Accept:** Tiles behave per §9.3 on desktop and touch emulation; no CLS; LCP image prioritized.

### Phase 4 — Featured Carousel
- Desktop Steam-style layout + mobile swipe layout, autoplay with pause logic, a11y.
- **Accept:** Autoplay pauses on hover/focus/touch/offscreen and with reduced motion; swipe doesn't block vertical scroll.

### Phase 5 — Library page
- Sidebar/bottom-sheet filters, Fuse search, sort, view toggle, URL sync inside Suspense, empty state.
- **Accept:** Works after static export, shareable URLs restore state, mobile bottom sheet is accessible.

### Phase 6 — Project detail pages
- `generateStaticParams`, hero, meta box, gallery, lightbox, YouTube facade, achievements, related, prev/next, JSON-LD.
- **Accept:** Every project route exists in `out/`; deep link refresh works; lightbox keyboard + swipe work.

### Phase 7 — Skills, Achievements, Timeline, Stats, Activity, Contact, About page
- Build remaining sections and compose the Home + About pages.
- **Accept:** Counters animate once; timeline progress line works; contact form hidden when no endpoint, functional with one (test with a mock).

### Phase 8 — Polish & extras
- Command palette, Easter eggs, optional sound toggle, page transitions, cursor glow, 404 page.
- (Optional, only if budget allows) lazy-loaded desktop-only R3F scene behind the hero (e.g., floating low-poly controller/cubes). Must be skipped on mobile, low-power devices and reduced-motion; must not alter Lighthouse mobile scores.
- **Accept:** Command palette is fully keyboard/screen-reader operable; extras are all individually disableable.

### Phase 9 — SEO, A11y, Performance hardening
- Metadata, sitemap, robots, manifest, JSON-LD, axe fixes, image optimization, bundle checks, Lighthouse runs (mobile + desktop).
- **Accept:** Lighthouse targets from §0 met; no axe serious/critical issues.

### Phase 10 — CI/CD, tests & docs
- Workflow per §14, Playwright suite, unit tests, README with owner guide (below).
- **Accept:** Push to `main` deploys; live URL works with deep links and assets under the correct base path.

---

## 21. README Requirements (owner-facing)

The agent must write a `README.md` that explains, in plain language:
1. What the site is + screenshot placeholders.
2. Quick start (`pnpm i`, `pnpm dev`).
3. **How to add or edit a project using only JSON** (Section 5.8): via the GitHub web UI (no local setup) and via `pnpm new:project`; the full field reference table; image folder conventions; `hidden`, `pinned`, `featuredOrder`; how to read validation errors in the Actions log; a copy-paste minimal example and a full example.
4. How to edit the other JSON files: `profile.json`, `skills.json`, `timeline.json`, `achievements.json`, `activity.json`, `site.json` (including section toggles), and how to replace the resume file.
5. How to change theme colors / default theme.
6. How to deploy (Section 14.2) and set up a custom domain.
7. How to enable the contact form and analytics.
8. Image guidelines (sizes, formats, `optimize-images` script).
9. List of all `TODO(owner)` placeholders to replace before going live.
10. Troubleshooting (blank page = wrong base path, missing CSS = missing `.nojekyll`, 404 on refresh = missing `trailingSlash`, etc.).

---

## 22. Content Guidance for the Owner's Projects

For each project, the agent should structure placeholder copy using this formula so the owner can swap in real data easily:
- **Tagline (≤ 70 chars):** outcome-oriented, e.g. "Turns production incidents into RCA reports in minutes."
- **Description (≤ 220 chars):** problem → solution → result.
- **Achievements (2–4):** measurable (%, hours saved, users).
- **Impact score:** reflect real impact, not effort.
- **Confidential projects:** describe the problem and outcome without naming clients; use `confidential: true`.

---

## 23. Definition of Done

- [ ] `pnpm build` generates a fully static `out/` directory; no server features used.
- [ ] GitHub Actions deploys to GitHub Pages successfully on push to `main`; works both at a sub-path and root.
- [ ] All routes (`/`, `/projects/`, `/projects/[slug]/`, `/about/`, 404) work on direct load and refresh.
- [ ] Looks and works great on mobile, tablet, desktop; no horizontal scroll; hover alternatives for touch.
- [ ] Three themes work, with no flash on load; Steam is the default.
- [ ] Library search, filter, sort, and shareable URLs work.
- [ ] Every home section from §9 is implemented and data-driven.
- [ ] **A project can be added, edited, reordered, hidden, or removed by editing only `content/projects.json` (+ images); no `/src` changes needed.**
- [ ] Invalid JSON fails CI with a clear file/entry/field message and does not deploy.
- [ ] No hardcoded project data or copy in `/src` (CI check passes); generated JSON Schemas are committed and up to date.
- [ ] A project with no images still renders well (fallback cover).
- [ ] Lighthouse, axe, and Playwright targets met (§0, §15, §19).
- [ ] `README.md` complete; all placeholders listed.
- [ ] `reduced-motion` and keyboard-only usage verified.

---

## 24. Instructions to the Building Agent

1. Work phase by phase (Section 20). After each phase, run lint, typecheck, build, and relevant tests; fix before moving on. Commit with clear messages (`feat: ...`, `chore: ...`).
2. Prefer simple, dependency-light solutions. Justify any new dependency in the PR/commit message.
3. If something in this plan conflicts with static hosting (Section 0), choose the static-compatible alternative and note it in `README.md`.
4. If a requirement is ambiguous, pick the most mobile-friendly and accessible option and document the decision in a `DECISIONS.md` file.
5. Do not invent real personal data (employers, metrics, emails). Use clearly marked placeholders.
6. Do not hotlink external images, fonts (outside `next/font`), or scripts (outside the optional analytics).
7. **Never hardcode content in components.** Everything the owner might change (projects, copy, socials, stats, skills, section titles, theme default) must come from `/content/*.json` through the loader. Keep the JSON easy for a non-developer: minimal required fields, forgiving shorthands, sensible defaults, and error messages that say exactly what to fix.
8. Finish by producing a short summary: what was built, how to deploy, which placeholders need replacing, and any known limitations.
