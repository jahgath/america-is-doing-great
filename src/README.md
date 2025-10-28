## Web3 is Going Just Great – src/ Codebase Guide

A guided walkthrough of the `src/` application to help you modify and extend the project confidently. This is a Next.js app backed by Firebase (Firestore + Storage), Algolia search, and some server-side pages/APIs.

### Quick start

- Install and run:
```bash
cd src
npm install
npm run dev
```
- Build and start:
```bash
npm run build
npm start
```
- Lint:
```bash
npm run lint
```
- Docker (from `src/`):
```bash
docker build -t w3igg .
docker run -p 3000:3000 w3igg
```

### Tech stack

- Next.js 14, React 18, SASS styles
- Firebase (Firestore, Storage, Auth)
- Algolia search
- react-query for data fetching/cache
- Accessibility-first UI with JS and no-JS fallbacks
- Deployed as a standard Next app (Dockerfile included)

---

## Project structure (src/)

### Top-level configuration

- `package.json`: Scripts and dependencies for the frontend.
  - Scripts: `dev`, `build`, `start`, `lint`.
- `next.config.js`:
  - Enables Next Strict Mode.
  - Configures remote images from `**.web3isgoinggreat.com`.
  - Redirects: `/feed` → `/feed.xml`, `/suggest` → `/contribute`, `/charts` → `/charts/top`.
- `.eslintrc.json`, `.prettierrc`: Linting and formatting rules.
- `.dockerignore`, `Dockerfile`, `Docker-compose.yaml`:
  - Multi-stage build and a production runner (`npm start`).
  - Compose maps port 3000.

### Public assets (served statically)

- `public/`: Favicons, manifest, pinned tab icon, robots.txt.
- Images referenced across the app are served from CDNs (see `constants/urls.js`).

### Global app shell

- `pages/_app.js`:
  - Initializes Google Analytics (ReactGA).
  - Adds default meta tags and social cards.
  - Provides `react-query` client.
  - Wraps app in `AppProvider` context and `Layout`.
- `pages/_document.js`:
  - Adds link tags for icons, manifest, RSS alt link.
- `components/Layout.js`:
  - Applies theme classes and font choices from `AppContext`.

### Global state and hooks

- `context/AppContext.js`:
  - App-wide UI preferences:
    - `theme`: "system" | "dark" | "light"
    - `useSansSerif`: boolean
  - Persists to `localStorage` via `js/localStorage.js`.
- Hooks:
  - `hooks/useIsBrowserRendering.js`: Returns `true` after mount (SSR-safe conditional UI).
  - `hooks/useWindowWidth.js`: Returns `"xs" | "sm" | "md" | "lg" | "xl"` based on breakpoints; throttled resize.
  - `hooks/useGA.js`: Sends a GA pageview on mount.

### Constants

- `constants/breakpoints.js`: Pixel breakpoints.
- `constants/colorModes.js`: Light/dark/system definitions (legacy list).
- `constants/filters.js`:
  - Allowed filter vocabularies (`theme`, `tech`, `blockchain`).
  - `EMPTY_FILTERS_STATE`, `FILTER_CATEGORIES`, `FiltersPropType` PropTypes.
- `constants/icons.js`:
  - Mapping for icon options (FontAwesome and custom PNGs).
  - `ICON_PATHS` maps logical icon names to CDN filenames.
- `constants/navigation.js`:
  - Top navigation config (groups, children).
  - Social links list with icons.
- `constants/urls.js`:
  - `STORAGE_URL` and `STATIC_STORAGE_URL` CDNs.

### Client utilities

- `js/utilities.js`:
  - Formatting and DOM string helpers:
    - `humanizeDate`, `formatDollarString`, `truncateToNearestWord`
    - `stripHtml`, `isWrappedInParagraphTags`, `sentenceCase`
    - URL helpers: `getPermalink`, `removeQueryParamsFromUrl`
    - `generateReadableId` for entry slugs
    - `fallback`, `copy`, `getCollectionName`
- `js/localStorage.js`:
  - Keys for persisted preferences (theme, grift counter).
  - Safe get/set wrappers (SSR-friendly).
- `js/datepicker.js`:
  - Presets and helpers for date picking, range parsing from query.
- `js/images.js`:
  - CDN URL generators and `src/srcset` builders for entry images.
- `js/admin.js`:
  - Admin auth and upload orchestration (calls Firestore + attribution writers).
- `js/entry.js`:
  - `EMPTY_ENTRY` template and `EntryPropType`.
  - `trimEmptyFields`, `isValidEntry` validation used by admin form.

### Data layer (Firebase and external services)

- `db/db.js`:
  - Initializes Firebase app.
  - Exports `db` (Firestore), `storage`, and `staticStorage`.
  - Note: Firebase config is hardcoded here.
- `db/entries.js`:
  - Core entry fetching with Firestore queries + pagination.
  - Filters by `collection` or filter categories (one category active at a time), `starred`, sort direction.
  - Helpers:
    - `getNumericId(idOrReadableId)` resolves `readableId` → numeric ID (YYYY-MM-DD-…).
    - `getEntries(options)`
    - `getAllEntries({ cursor, direction })` for Web1 mode pagination.
- `db/singleEntry.js`:
  - `getEntry(idOrReadableId)` with validation and not-found handling.
- `db/leaderboard.js`:
  - Fetches all entries with `scamAmountDetails.hasScamAmount == true`.
  - Date range filtering, sorting by `date` or `amount`, then paginates in-memory for the leaderboard view.
  - Computes date-range-specific `scamTotal` when applicable.
- `db/glossary.js`:
  - Retrieves glossary entries:
    - `getGlossaryEntries()` returns keyed object.
    - `getSortedGlossaryEntries()` returns array sorted by term.
- `db/attribution.js`:
  - Fetches attribution lists for images and entries pages.
- `db/metadata.js`:
  - Retrieves site metadata doc (includes `griftTotal` and `collections` map).
- `db/money.js`:
  - Retrieves donation/expense figures for the contribute page.
- `db/utils.js`: `compareCaseInsensitive` for alpha sorting.
- `db/admin.js` (write operations used by admin):
  - `uploadEntry(entry)`: Creates a unique `id` by incrementing date suffix.
  - `addImageAttribution`, `addEntryAttribution`: Inserts in alpha order.

- `db/searchEntries.js`:
  - Algolia search client (`web3` index).
  - `search(query, filters)` builds facetFilters from active filters.
  - Note: Uses an Algolia search key embedded in the repo; replace as needed.

### API routes (Next.js API)

- `pages/api/griftTotal.js`: Returns `{ total }` from metadata.
- `pages/api/oembed.js`:
  - Validates `url` param; supports JSON (default) and `format=xml`.
  - When `id` present (or `/single/:id` or `/embed/:id`), builds embed iframe for that entry and adds thumbnail dimensions by fetching image bytes and running `image-size`.
  - Otherwise, returns a generic site embed.

### Pages (routing and SSR strategy)

- Timeline and navigation
  - `pages/index.js` (Home):
    - SSR: determines initial filters from query, fetches first page of entries, glossary, metadata.
    - Client: `useInfiniteQuery` fetches more as you scroll, using entry `_key` as cursor.
    - Supports:
      - Filtering by one category at a time (`theme`, `tech`, `blockchain`), starred-only, or `collection`.
      - Deep links via `?id=readableId` (scrolls/highlights).
  - `components/timeline/*`:
    - `Header`: Responsive header with navigation, logo, social links, and skip link.
    - `Filters`: React-select based multi-select filters, starred toggle, sort direction.
    - `Timeline`: Orchestrates header, filter bar visibility, infinite scroll sentinel, grift counter, and `Entry` rendering. Adds “Start from the top” CTA when deep-linked or searched.
    - `Entry`: Renders an entry card, with:
      - Permalink copy button (client-side), star indicator, optional social links.
      - Entry image with lightbox.
      - Tags (theme/tech/blockchain) and related collection links.
      - Uses `TimelineEntryContent` to add popover glossary definitions for `.define-target` buttons inside the HTML body.
    - `TimelineEntryContent`: Hydrates glossary popovers with `react-popper`, wraps punctuation to avoid line breaks, accessible controls and no-JS fallback links to glossary.
    - `FixedAtBottom`: Settings panel toggler, “scroll to top”, and Grift Counter status panel.
    - `GriftCounter`: Animated corner display of running scam totals as user scrolls.
    - `ScrollToTop`: Button to return to top.
  - Navigation
    - `components/navigation/*`: Desktop bar, dropdown menus (`downshift`), mobile toggle menu, and no-JS navigation strip.

- Entry views
  - `pages/single/[id].js`: Full entry view with site chrome; SSR fetches entry, glossary, metadata.
  - `pages/embed/[id].js`: Embed-friendly single-entry layout; SSR fetches entry, glossary, metadata; used by `oembed`.
  - `pages/archive/[[...id]].js`: Archived tweet assets view (screenshots and linked assets) for sources that referenced now-removed tweets.

- Charts
  - `pages/charts/top.js`: Leaderboard of largest hacks/scams
    - SSR resolves date range from query and metadata.
    - Client uses `react-query` and `getEntriesForLeaderboard` results.
    - Date range via `components/charts/DatePicker` (custom presets + calendar).
    - Pagination using `LeaderboardPaginator`.

- Static content
  - `pages/about.js`, `pages/what.js`, `pages/faq.js`, `pages/contribute.js`, `pages/attribution.js`
    - Mixed SSR for data-driven pages (e.g., `contribute`, `attribution`) and static pages.
  - Errors: `pages/404.js`, `pages/500.js`.

- No-JS mode
  - `pages/web1.js`: Plain “Web 1.0” experience for users without JS; server-paginated.

- Feed passthrough
  - `pages/feed.xml.js`: Streams the `rss.xml` file from `staticStorage` so feed readers can fetch it.

### Components (general)

- `components/BackBar.js`: A sub-navigation with back link or custom action.
- `components/CustomHead.js`: Reusable `<Head>` helper for generic pages.
- `components/CustomEntryHead.js`: `<Head>` for entry-specific OpenGraph/Twitter meta with dynamic image dimensions (fallbacks if load fails).
- `components/ExternalLink.js`: External link with `noopener` and target.
- `components/Footer.js`: Footer with license and links.
- `components/Loader.js`: Inline loading indicator.
- `components/Checkbox.js`: Simple checkbox with label and PropTypes.

### Admin (entry authoring UI)

- Route: `pages/admin.js`
  - Uses Firebase Auth; listens with `onAuthStateChanged`.
  - Shows `Login` or `Form` based on session.
- `components/admin/Login.js`:
  - Calls `signIn(password)` for a fixed email address (see `js/admin.js`).
- `components/admin/Form.js`:
  - Full entry composer:
    - Title, short title, readable ID (auto-generated), date, icon picker, starred.
    - Scam amount inputs with optional lower/upper bounds, recovered, pre-recovery, and text overrides.
    - Body with tag insertion buttons (`EntryTextArea`), link array (`LinkField`), filters (`FilterSelector`).
    - Image fields (src, alt, caption, class toggles such as `on-dark`/`on-light`, logo flag, link).
    - Attribution inputs (image and entry).
  - Validates via `isValidEntry` and transforms data via `trimEmptyFields`.
  - Uploads via `js/admin.upload(...)` which calls:
    - `db/admin.uploadEntry` (writes the entry)
    - `db/admin.addImageAttribution` / `addEntryAttribution` as needed
  - After a successful upload, copies the new `entryId` to clipboard.

- `components/admin/IconSelector.js`:
  - `react-select` tied to `constants/icons.js`; writes either `faicon` or `icon`.
- `components/admin/EntryTextArea.js`:
  - Textarea with helpers to insert `.define-target` glossary button markup or links at cursor/selection.
- `components/admin/FilterSelector.js`:
  - Filter checkbox groups (theme, tech, blockchain).
- `components/admin/LinkField.js`:
  - Link editor for `href`, `linkText`, and `extraText`.

### Styles

- `styles/main.sass` and partials:
  - `_navigation.sass`, `_filters.sass`, `_loading.sass`, `_fixed-at-bottom.sass`, `_charts.sass`, `_popovers.sass`, `_admin.sass`, `_react-date-range-theme-overrides.sass`, `_tokens.sass`
  - Loaded once in `_app.js`.
- Uses normalize.css and SASS. Classnames are semantically named and referenced in components.

---

## Data models

- Entry (`EntryPropType` in `js/entry.js`):
  - `id` (numeric key `YYYY-MM-DD-#`), `readableId` (slug), `date` (YYYY-MM-DD), `title`, `shortTitle`, `body` (HTML), `filters` (arrays), `collection` (array), `starred` (optional), `color` (optional).
  - `image` (optional): `{ src, alt, link?, caption?, class?, isLogo? }`.
  - `links` (array): `{ linkText, href, extraText?, archiveHref?, archiveTweetPath?, archiveTweetAlt?, archiveTweetAssets? }`.
  - `scamAmountDetails`: `{ total, hasScamAmount, preRecoveryAmount, lowerBound?, upperBound?, recovered?, textOverride? }`.
  - `socialPostIds` (optional): `{ twitter?, mastodon?, bluesky? }`.

- Metadata (`db/metadata.js`):
  - `griftTotal`: site-wide total (used by grift counter and charts).
  - `collections`: map of collection slug → human-readable label.

- Glossary (`db/glossary.js`):
  - Keyed object `entries` (id → { id, term, definition }).
  - Sorted list for display pages.

- Money (`db/money.js`):
  - Donations, expenses, remaining/used credits, next month estimate.

---

## UI patterns and flows

- Infinite scroll:
  - `Timeline` uses `react-intersection-observer` to observe a sentinel above the last entry, then tells `react-query` to fetch the next page via the last entry `_key`.
- Filtering strategy:
  - Only one category (theme | tech | blockchain) is applied at once due to Firestore index constraints; starred is mutually exclusive with category filters.
- Date range handling (charts):
  - URL can specify a preset via `?dateRange=all|month|YYYY` etc., or `startDate/endDate`. The SSR and client both use `getDateRangeFromQueryParams`.
- Permalinks:
  - Clicking the link icon in an entry copies a URL with `?id=readableId` to clipboard (client-only behavior).
- Accessibility:
  - Skip links, ARIA attributes, semantic controls, no-JS fallbacks for nav and glossary definitions.

---

## Config you will likely customize

- Firebase:
  - `db/db.js` contains the config for your project; replace with your credentials.
  - Firestore collections used: `entries`, `metadata`, `glossary`, `attribution`, `money`.
- Algolia:
  - `db/searchEntries.js`: Update Application ID, Search key, and index name (`web3`).
- CDN/Assets:
  - `constants/urls.js`: Point `STORAGE_URL` and `STATIC_STORAGE_URL` to your buckets/CDN.
  - Icon assets referenced via `constants/icons.js` and `STORAGE_URL`.
- Google Analytics:
  - `_app.js`: Replace `ReactGA.initialize("UA-215114522-1")` with your GA ID or remove GA if not needed.
- Navigation:
  - `constants/navigation.js`: Add/remove/rename top-level nav items and social links.
- Filters:
  - `constants/filters.js`: Add new tags or categories (ensure Firestore indexes support new queries).

---

## Common modifications

- Add a new page:
  - Create `pages/yourpage.js`; optionally fetch server-side data using `getServerSideProps`.
  - Add link in `constants/navigation.js`.
- Add a new leaderboard or chart:
  - Create a page similar to `pages/charts/top.js`.
  - Add a data function under `db/` to fetch and shape the dataset.
  - Reuse `DatePicker`, `LeaderboardPaginator`, or build new components.
- Change entry schema:
  - Update `EntryPropType`, admin form (`components/admin`), `trimEmptyFields`, `isValidEntry`, and any display logic in `Entry`/`Timeline`/`Leaderboard`.
- Customize the timeline UI:
  - Edit `components/timeline/*` and corresponding SASS partials.
- Change image behavior:
  - Update `js/images.js` for URL/size logic and `next.config.js` for remotePatterns.

---

## Deployment notes

- Production build uses the `src/Dockerfile`:
  - Multi-stage: install deps, build, run with `next start`.
- Ensure Firestore indexes:
  - Composite or array-contains indexes may be required for combined filters; the code currently queries only one category at a time to keep indexes simple.
- SEO/OpenGraph:
  - Generic tags in `_app.js`, entry-specific tags in `CustomEntryHead`.
  - Feed available via `/feed.xml`.

---

## Troubleshooting

- Blank glossary popovers:
  - Ensure glossary `entries` object contains IDs matching `.define-target` button `id`s inside entry body HTML.
- Images not loading:
  - Confirm `next.config.js` allows your image domains and your `STORAGE_URL`/file paths are correct.
- Algolia search returns nothing:
  - Check index name, credentials, and that facet attributes are configured to match `theme`, `tech`, `blockchain`.

---

## Security and privacy

- External links use `rel="noopener"` and `target="_blank"`.
- Admin route uses Firebase Auth; the login flow signs in to a fixed email with password only. Replace with your own auth model if necessary.
- GA is opt-in via script; consider removing or replacing to match your privacy posture.

---

## Where to start modifying

- Branding and nav: `constants/navigation.js`, `components/SimpleHeader.js`, `components/navigation/*`, styles.
- Content model: `js/entry.js` and admin form components.
- Data sources: `db/*` collection functions; adjust Firestore data model as needed.
- Theming: `context/AppContext.js`, `components/Layout.js`, and SASS tokens.

---

- Summary:
  - Mapped the entire `src/` structure, explaining each directory and key file.
  - Covered data flow (Firestore, Algolia), page routing/SSR, components, admin UI, and styles.
  - Highlighted configuration points (Firebase, Algolia, CDNs, GA) and common extension paths.