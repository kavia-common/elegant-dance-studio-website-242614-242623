# Supabase setup: Gallery Images (DB + Storage + RLS)

This project uses Supabase/Postgres to store **gallery image metadata** and Supabase Storage to store the **image files**.

---

## 1) Database schema (applied in this container)

Table: `public.gallery_images`

Columns:
- `id` (uuid, PK, default `gen_random_uuid()`)
- `image_url` (text, not null) — the public URL (or signed URL) to the image in Supabase Storage
- `alt_text` (text, nullable) — accessibility description
- `created_at` (timestamptz, not null, default `now()`)

Indexes:
- `gallery_images_created_at_idx` on `(created_at desc)` for sorting newest-first.

Notes:
- `pgcrypto` extension is enabled to support `gen_random_uuid()`.

### Local dev DB connection (this repo container)

The connection command is stored here:
- `supabase_db/db_connection.txt`

Example usage:
```bash
# from supabase_db/
$(cat db_connection.txt) -c "select now();"
```

---

## 2) Supabase Storage bucket

Create a bucket (recommended name): `gallery`

Suggested bucket settings:
- **Public bucket**: ON (recommended for simple public gallery display)
  - If you prefer a private bucket, you must generate signed URLs server-side; update the app accordingly.

Upload path suggestion:
- `gallery/<uuid>.<ext>` (or any consistent naming scheme)

---

## 3) RLS policies (table: `public.gallery_images`)

Important note about this repository's backend:
- The `nextjs_backend_admin` service uses **SUPABASE_SERVICE_ROLE_KEY** for DB/storage operations via Supabase REST.
- The service-role key **bypasses RLS** policies.
- Admin access for the app is enforced by the backend's own JWT (`/api/admin/auth/login`) and request filter for `/api/admin/**`.

Therefore:
- RLS is **optional** for this repo’s current backend implementation.
- You may still enable RLS as a defense-in-depth measure, but it will not affect calls made with the service-role key.

If you want to enforce policies at the DB level, you must:
- Stop using the service-role key for normal operations, and
- Use Supabase Auth JWTs / anon key + user JWTs, and
- Rework the backend accordingly.

### Optional: Public read policy (when using anon/auth JWTs)
If you implement a JWT-based Supabase Auth flow later, a common policy is:
```sql
alter table public.gallery_images enable row level security;

create policy "Public read gallery images"
on public.gallery_images
for select
to anon, authenticated
using (true);
```

---

## 4) Storage policies (bucket: `gallery`)

Goal:
- Public can **read** images
- Only admin can **write/delete** images

### 4.1 Public read

If the bucket is public, reads are already public. If using RLS policies for `storage.objects`, you can use:

```sql
create policy "Public read gallery objects"
on storage.objects
for select
to anon, authenticated
using (bucket_id = 'gallery');
```

### 4.2 Admin write/delete

```sql
create policy "Admin upload gallery objects"
on storage.objects
for insert
to authenticated
with check (
  bucket_id = 'gallery'
  and (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin'
);

create policy "Admin update gallery objects"
on storage.objects
for update
to authenticated
using (
  bucket_id = 'gallery'
  and (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin'
)
with check (
  bucket_id = 'gallery'
  and (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin'
);

create policy "Admin delete gallery objects"
on storage.objects
for delete
to authenticated
using (
  bucket_id = 'gallery'
  and (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin'
);
```

---

## 5) Admin user expectation

At least one Supabase user must have:
- `app_metadata.role = 'admin'`

How you set that depends on your auth flow:
- via Supabase Dashboard (manual) + admin tooling, or
- via a secure server-side process using the Supabase Admin API.

The frontend/public site should only require **read** access.

---

Troubleshooting:
- If inserts fail with RLS errors: verify the user is authenticated and has `app_metadata.role='admin'`.
- If images don’t load publicly: verify the bucket is public OR the `storage.objects` select policy exists for `anon`.
