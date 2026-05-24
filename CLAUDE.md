# Momobook Codebase Guide

Momobook is a personal fork of [Immich](https://immich.app) — a self-hosted photo and video management platform. The operator is self-hosting this for personal use and adding custom features on top of the Immich base.

## License & Attribution

### License
This repo is licensed under the **GNU Affero General Public License v3.0 (AGPL-3.0)**. The `LICENSE` file at the repo root must never be removed or modified.

Immich was created by [Alex Tran](https://github.com/alextran1502) and the [Immich contributors](https://github.com/immich-app/immich/graphs/contributors). The upstream project lives at https://github.com/immich-app/immich.

### What AGPL-3.0 requires of this fork

**Always:**
- Keep the `LICENSE` file intact — never delete or alter it
- Keep any existing license/copyright notices in source files — do not strip headers
- All new code added to this fork must also be licensed under AGPL-3.0

**When making modifications (AGPL Section 5):**
- Modified files must carry a prominent notice stating they were changed and the date
- The preferred way to do this in this repo is via clear git commit messages (e.g. `feat(momobook): add not-in-album sidebar view`) rather than inline file comments, so the history is auditable

**Network use clause (AGPL Section 13 — the key difference from GPL):**
- If any user other than the operator interacts with this software over a network (e.g. a family member is given access), the modified source code must be made available to them
- Since the repo is on GitHub at `linnalihe/momobook`, this is already satisfied as long as the repo remains accessible. Do not make the repo private if others are given access to the running instance.

### Practical rules for adding code

1. **New files** — add a brief comment at the top identifying the file as a momobook addition:
   ```ts
   // momobook addition
   ```
2. **Modified upstream files** — no inline comment needed; the git history documents the change. Write commit messages that clearly distinguish momobook work from upstream (use `feat(momobook):` or `fix(momobook):` prefix).
3. **Never** remove the upstream README attribution, the contributors image, or the star history section — these credit the original Immich project and must stay.
4. **Do not relicense** any part of the codebase under a different license, even for small utilities added to the repo.

## Tech Stack

- **Backend:** NestJS (TypeScript), Kysely ORM
- **Frontend:** SvelteKit (TypeScript)
- **Database:** PostgreSQL 14 with `pgvectors` and `vectorchord` extensions (required for ML/similarity search)
- **Cache / Queue:** Redis (Valkey)
- **ML:** Separate `immich-machine-learning` container (CLIP embeddings, face recognition)
- **Deployment:** Docker Compose

## Architecture

Four-layer structure, strictly followed throughout the codebase:

```
HTTP Request
    ↓
Controller   — routing, auth decorators (@Authenticated), parameter parsing
    ↓
Service      — business logic, access checks, calls repositories
    ↓
Repository   — SQL queries via Kysely, one file per domain entity
    ↓
Filesystem / PostgreSQL
```

Key entry points:
- `server/src/controllers/` — HTTP layer
- `server/src/services/` — business logic
- `server/src/repositories/` — data access
- `server/src/cores/storage.core.ts` — file path logic (singleton)
- `server/src/utils/access.ts` — access control logic
- `server/src/repositories/access.repository.ts` — access SQL queries
- `server/src/middleware/auth.guard.ts` — authentication middleware
- `server/src/enum.ts` — all enums (Permission, StorageFolder, AssetVisibility, etc.)
- `server/src/config.ts` — SystemConfig type and defaults

## Storage System

### Folder layout on disk

```
$UPLOAD_LOCATION/           (set in .env, bind-mounted into container)
  upload/                   ← uploaded originals, UUID-named
    <userId>/ab/cd/
      <uuid>.jpg
  thumbs/                   ← generated thumbnails and previews
    <userId>/ab/cd/
      <assetId>_thumbnail.webp
  encoded-video/            ← transcoded video files
  library/                  ← files after storage template migration
    <userId-or-storageLabel>/
      2024/2024-06-15/IMG_4821.jpg
  profile/                  ← user profile images
  backups/                  ← database backups
```

The two-level nesting (`ab/cd/`) uses the first 4 characters of the UUID to avoid large flat directories. See `StorageCore.getNestedPath()` in `storage.core.ts:332`.

### File lifecycle for uploaded assets

1. **On arrival:** saved immediately as `<uuid>.<ext>` under `upload/` — original filename is stored only in the database (`asset.originalFileName`)
2. **After EXIF extraction (async):** if Storage Template is enabled, a background job physically moves the file to the template path under `library/`
3. **Display name:** always comes from `asset.originalFileName` in the DB — never changes regardless of what happens on disk

### Storage Template

**Disabled by default** (`config.ts:319`). Controlled in Admin settings.

Uses Handlebars. Default template: `{{y}}/{{y}}-{{MM}}-{{dd}}/{{filename}}`

Available tokens: `{{filename}}`, `{{assetId}}`, `{{assetIdShort}}`, `{{y}}`, `{{yy}}`, `{{MM}}`, `{{MMM}}`, `{{MMMM}}`, `{{dd}}`, `{{WW}}`, `{{HH}}`, `{{mm}}`, `{{ss}}`, `{{filetype}}`, `{{filetypefull}}`, `{{ext}}`, `{{make}}`, `{{model}}`, `{{lensModel}}`, `{{album}}`, `{{album-startDate-*}}`, `{{album-endDate-*}}`

Supports Handlebars conditionals: `{{#if album}}{{album}}{{else}}Other{{/if}}`

Duplicate filenames get a suffix: `IMG_4821+1.jpg`, `+2`, etc.

Extension normalisation happens here: `.jpeg`→`.jpg`, `.tif`→`.tiff`, `.mpeg`→`.mpg`, `.m2ts`→`.mts`.

**External library assets are never moved or renamed** — `moveAsset()` in `storage-template.service.ts:222` returns early for external assets.

### External Libraries

Files scanned from an external library path keep their original path and filename forever. Immich only records the path in the database. Manual changes to files outside Immich require a library rescan to stay in sync.

## Access Control

Four layers, applied on every image request:

### 1. Authentication (`auth.guard.ts`)
Runs before every controller method. Accepts:
- Session cookie
- Bearer token (JWT or API key)
- OAuth 2.0 / OIDC

Resolves to an `AuthDto` containing the user identity and session flags.

### 2. Per-endpoint permissions (`enum.ts:104`)
~100 granular `Permission` values (e.g. `asset.view`, `asset.download`, `album.share`). Each controller method declares its required permission via `@Authenticated({ permission: Permission.XxxYyy })`.

### 3. Asset visibility (`enum.ts:1097`)
Per-asset `AssetVisibility` field:
- `timeline` — visible in main timeline (default)
- `archive` — hidden from timeline, still accessible
- `hidden` — video half of Live Photos
- `locked` — requires elevated session (PIN code) even for the owner

### 4. Resource-level access (`access.repository.ts` + `utils/access.ts`)
Before any operation, `requireAccess()` queries the DB to check the requesting user actually has rights to the specific asset IDs. For `AssetView`, it checks three paths in order, stopping early:
1. Owner (`asset.ownerId = userId`) — single indexed query, fast
2. Album member — joins `album_asset`, `album_user`
3. Partner sharing — checks `partner` table

Shared link access is checked separately via `checkSharedLinkAccess()`.

## Image Request Performance

For every thumbnail request, two DB round-trips fire before the file is read:
1. Access check (owner / album / partner)
2. File path lookup (`assetRepository.getForThumbnail`)

File delivery uses `res.sendFile()` which calls the OS `sendfile(2)` syscall — zero-copy from disk to socket, no data passes through Node.js memory. Cache headers: `private, max-age=86400, stale-while-revalidate=2592000`.

**No caching of access decisions** — every request hits Postgres. Fine for personal use; would need Redis-based caching of access results to scale.

## Known Missing Features (to be added)

- **"Not in any album" dedicated view** — the backend filter (`isNotInAlbum`) exists in the search DTO and SQL, but there is no sidebar link or standalone page. Only accessible today via Search → filter panel → Display Options → "Not in any album" checkbox.

## Self-Hosting Setup

Recommended directory structure on external storage:

```
/mnt/photos/
  immich/
    library/        ← UPLOAD_LOCATION in .env
    postgres/       ← DB_DATA_LOCATION in .env (ideally on SSD)
  originals/        ← external library mount (read-only), operator's own files
```

Docker Compose wiring:
```yaml
immich-server:
  volumes:
    - /mnt/photos/immich/library:/data
    - /mnt/photos/originals:/mnt/originals:ro

database:
  volumes:
    - /mnt/photos/immich/postgres:/var/lib/postgresql/data
```

Access is restricted to the operator only — no public exposure. Remote access via Tailscale VPN (no port forwarding, nothing exposed to public internet). Immich login as second auth layer.

### Host user

The dedicated host user is named **`momobookuser`** (UID 1000). Any setup instructions, scripts, or documentation that reference a Linux username for running the server or owning data directories must use `momobookuser` — not `momobook` or any other name.
