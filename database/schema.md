# Notes App PostgreSQL Schema (applied)

This container uses PostgreSQL (running on port `5000` by default in `startup.sh`).

**Connection command** (authoritative source): `db_connection.txt`

Example:

- `psql postgresql://appuser:dbuser123@localhost:5000/myapp`

## Container rule compliance

All schema operations were executed:
- using the `psql ... -c "SQL_STATEMENT"` format, and
- **one SQL statement at a time** (no multi-line statements), and
- using the connection string read from `database/db_connection.txt`.

No `.sql` migration files were created.

## Extensions enabled

The following extensions are enabled for UUIDs and search quality/performance:

- `pgcrypto` (UUID generation via `gen_random_uuid()`)
- `citext` (case-insensitive text for emails/tag names)
- `unaccent` (normalize content for FTS)
- `pg_trgm` (trigram indexes for fast `ILIKE` / fuzzy matching)

## Tables

### `users` (optional)

Columns:
- `id uuid primary key default gen_random_uuid()`
- `email citext unique`
- `display_name text`
- `password_hash text`
- `is_active boolean default true`
- `created_at timestamptz default now()`
- `updated_at timestamptz default now()`
- `last_login_at timestamptz`

### `sessions` (optional)

For refresh-token based session storage:

- `id uuid primary key default gen_random_uuid()`
- `user_id uuid references users(id) on delete cascade`
- `refresh_token_hash text not null`
- `user_agent text`
- `ip_address inet`
- `created_at timestamptz default now()`
- `expires_at timestamptz not null`
- `revoked_at timestamptz`

### `notes`

Core note entity including pin/favorite + soft-delete flags:

- `id uuid primary key default gen_random_uuid()`
- `user_id uuid references users(id) on delete set null`
- `title text not null default ''`
- `content text not null default ''`
- `content_format text not null default 'markdown'`
- `is_archived boolean not null default false`
- `is_deleted boolean not null default false`
- `pinned_at timestamptz`
- `favorited_at timestamptz`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`

Full-text search support:
- `search_tsv tsvector` (stored column, maintained by trigger)

### `tags`

- `id uuid primary key default gen_random_uuid()`
- `user_id uuid references users(id) on delete set null`
- `name citext not null`
- `color text`
- `created_at timestamptz not null default now()`
- Unique constraint: `unique(user_id, name)` (per-user tag names)

### `note_tags` (join)

- `note_id uuid not null references notes(id) on delete cascade`
- `tag_id uuid not null references tags(id) on delete cascade`
- `created_at timestamptz not null default now()`
- Primary key: `(note_id, tag_id)`

### `note_shares`

Supports both direct user shares and token-based sharing links:

- `id uuid primary key default gen_random_uuid()`
- `note_id uuid not null references notes(id) on delete cascade`
- `owner_user_id uuid references users(id) on delete set null`
- `shared_with_user_id uuid references users(id) on delete cascade`
- `share_token uuid not null default gen_random_uuid()`
- `permission text not null default 'read'`
- `expires_at timestamptz`
- `created_at timestamptz not null default now()`
- `revoked_at timestamptz`

Constraints:
- `permission` check: `('read','comment','edit')`
- `unique(note_id, shared_with_user_id)`
- `unique(share_token)`

## Full-text search details

A trigger maintains `notes.search_tsv` from `title` (weight A) and `content` (weight B):

- Trigger function: `notes_search_tsv_trigger()`
- Trigger: `trg_notes_search_tsv` (`BEFORE INSERT OR UPDATE OF title, content ON notes`)

Search vector generation uses:
- `to_tsvector('simple', unaccent(coalesce(...)))`

## Indexes

Sessions:
- `idx_sessions_user_id` on `sessions(user_id)`
- `idx_sessions_refresh_token_hash` on `sessions(refresh_token_hash)`

Notes:
- `idx_notes_user_updated_at` on `notes(user_id, updated_at desc)`
- `idx_notes_pinned_at` on `notes(pinned_at desc)` where `pinned_at is not null`
- `idx_notes_favorited_at` on `notes(favorited_at desc)` where `favorited_at is not null`
- `idx_notes_not_deleted` on `notes(is_deleted)` where `is_deleted = false` (partial)
- `idx_notes_search_tsv` GIN on `notes(search_tsv)` (FTS)
- `idx_notes_title_trgm` GIN trigram on `notes(title gin_trgm_ops)`

Tags:
- `idx_tags_user_name` on `tags(user_id, name)`
- `idx_tags_name_trgm` GIN trigram on `tags(name gin_trgm_ops)`

Join + sharing:
- `idx_note_tags_tag_id` on `note_tags(tag_id)`
- `idx_note_shares_note_id` on `note_shares(note_id)`
- `idx_note_shares_shared_with_user_id` on `note_shares(shared_with_user_id)`

## Seed data inserted (minimal)

Seed tags (global tags with `user_id NULL`):
- `inbox` (`#3b82f6`)
- `todo` (`#06b6d4`)
- `idea` (`#64748b`)

Seed note (global note with `user_id NULL`):
- Title: `Welcome`
- Content: `This is your first note. Use tags, pin, favorite, and search to organize.`
- `pinned_at = now()`
- `favorited_at = now()`

Seed relationship:
- The `Welcome` note is associated with the `inbox` tag.

All inserts were done with `ON CONFLICT DO NOTHING` to keep the operation idempotent.
