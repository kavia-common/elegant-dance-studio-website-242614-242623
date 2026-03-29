# Supabase setup: Gallery Images (DB + Storage + RLS)

This project uses Supabase/Postgres to store **gallery image metadata** and Supabase Storage to store the **image files**.

## 1) Database schema (already applied in this container)

Table: `public.gallery_images`

Columns:
- `id` (uuid, PK, default `gen_random_uuid()`)
- `image_url` (text, not null) — the public URL (or signed URL) to the image in Supabase Storage
- `alt_text` (text, nullable) — accessibility description
- `created_at` (timestamptz, not null, default `now()`)

Notes:
- `pgcrypto` extension is enabled to support `gen_random_uuid()`.
- Index: `gallery_images_created_at_idx` on `(created_at desc)` for sorting newest-first.

## 2) Supabase Storage bucket

Create a bucket (recommended name): `gallery`

Suggested bucket settings:
- **Public bucket**: ON (recommended for simple public gallery display)
  - If you prefer a private bucket, you must generate signed URLs server-side; update the app accordingly.

Upload path suggestion:
- `gallery/<uuid>.<ext>` (or any consistent naming scheme)

## 3) RLS policies (table: `public.gallery_images`)

The desired access model is:
- **Public read**: anyone can list and view gallery metadata
- **Admin write/delete**: only admin users can insert/update/delete rows

### 3.1 Enable RLS
In Supabase SQL editor:
```sql
alter table public.gallery_images enable row level security;
```

### 3.2 Public read policy
```sql
create policy "Public read gallery images"
on public.gallery_images
for select
to anon, authenticated
using (true);
```

### 3.3 Admin write/delete policies

This project expects an “admin” concept to be enforced by **JWT claims** or a **dedicated admin role**.

Recommended approach: **JWT claim** `app_metadata.role = 'admin'`.

Create policies:
```sql
create policy "Admin insert gallery images"
on public.gallery_images
for insert
to authenticated
with check ((auth.jwt() -> 'app_metadata' ->> 'role') = 'admin');

create policy "Admin update gallery images"
on public.gallery_images
for update
to authenticated
using ((auth.jwt() -> 'app_metadata' ->> 'role') = 'admin')
with check ((auth.jwt() -> 'app_metadata' ->> 'role') = 'admin');

create policy "Admin delete gallery images"
on public.gallery_images
for delete
to authenticated
using ((auth.jwt() -> 'app_metadata' ->> 'role') = 'admin');
```

If you instead use a different claim (e.g. `auth.jwt() ->> 'role'`) or use Supabase “teams”/RBAC, update these policies accordingly.

## 4) Storage policies (bucket: `gallery`)

Goal:
- Public can **read** images
- Only admin can **write/delete** images

In Supabase Dashboard → Storage → Policies (or SQL), implement:

### 4.1 Public read
If bucket is public, reads are already public. If using RLS policies for storage objects, you can use:

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

## 5) Admin user expectation

You must ensure at least one Supabase user has:
- `app_metadata.role = 'admin'`

How you set that depends on your auth flow:
- via Supabase Dashboard (manual) + admin tooling, or
- via a secure server-side process using the Supabase Admin API.

The frontend/public site should only require **read** access.

---
Troubleshooting:
- If inserts fail with RLS errors: verify the user is authenticated and has `app_metadata.role='admin'`.
- If images don’t load publicly: verify bucket is public OR storage select policy exists for anon users.
