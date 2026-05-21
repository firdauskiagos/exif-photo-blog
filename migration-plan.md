# Migrate from Vercel Blob to Cloudflare R2

  ## Prerequisites

  - Your exif-photo-blog deployed on Vercel
  - A Cloudflare account (free tier works)
  - A domain on Cloudflare (optional but recommended for custom public URL)

  ---

  ## Step 1: Create an R2 Bucket

  1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com)
  2. Navigate to **R2 Object Storage** in the sidebar
  3. Click **Create bucket**
  4. Name it something like `photos`
  5. Choose a location hint closest to your visitors (or leave automatic)

  ## Step 2: Enable Public Access

  Your blog visitors need to load images directly from R2.

  **Option A — Custom domain (recommended):**
  1. In your bucket settings, go to **Settings** → **Public Access**
  2. Click **Connect Domain**
  3. Enter a subdomain like `photos-cdn.yourdomain.com`
  4. Cloudflare will handle SSL and caching automatically

  **Option B — R2.dev subdomain (quick but less clean):**
  1. In bucket settings, enable the **R2.dev subdomain**
  2. Note the URL (e.g., `pub-abc123.r2.dev`)

  ## Step 3: Create an API Token

  1. In the R2 overview page, click **Manage R2 API Tokens**
  2. Click **Create API token**
  3. Set permissions to **Object Read & Write**
  4. Scope it to your `photos` bucket only
  5. Click **Create API Token**
  6. Save the **Access Key ID** and **Secret Access Key** — you won't see the
  secret again

  ## Step 4: Find Your Account ID

  Your Cloudflare Account ID is visible in:
  - The URL when you're in the dashboard: `dash.cloudflare.com/<account-id>/...`
  - Or on the R2 overview page in the sidebar

  ## Step 5: Add Environment Variables to Vercel

  Go to your Vercel project → **Settings** → **Environment Variables** and add:

  | Variable | Value |
  |----------|-------|
  | `NEXT_PUBLIC_CLOUDFLARE_R2_BUCKET` | `photos` (your bucket name) |
  | `NEXT_PUBLIC_CLOUDFLARE_R2_ACCOUNT_ID` | Your Cloudflare account ID |
  | `NEXT_PUBLIC_CLOUDFLARE_R2_PUBLIC_DOMAIN` | `photos-cdn.yourdomain.com`
  (from Step 2) |
  | `CLOUDFLARE_R2_ACCESS_KEY` | Access Key ID from Step 3 |
  | `CLOUDFLARE_R2_SECRET_ACCESS_KEY` | Secret Access Key from Step 3 |
  | `NEXT_PUBLIC_STORAGE_PREFERENCE` | `cloudflare-r2` |

  **Important:** Keep `BLOB_READ_WRITE_TOKEN` for now — the migration needs it
  to read your existing files.

  ## Step 6: Apply the Migration Patch

  On the machine where you have your repo cloned:

  ```bash
  git apply migrate-vercel-to-r2.patch
  git add app/api/migration/vercel-to-r2/route.ts
  git commit -m "Add one-time migration endpoint for Vercel Blob to R2"
  git push

  Wait for Vercel to deploy.

  Step 7: Run the Migration

  1. Visit your blog and sign in as admin at /admin
  2. Then visit: https://yourdomain.com/api/migration/vercel-to-r2
  3. Wait for the JSON response — it will show how many photos were migrated

  Expected response:
  {
    "summary": { "total": 47, "migrated": 47, "skipped": 0, "errors": 0 },
    "results": [...]
  }

  If it times out (60s limit on Vercel Hobby), just hit the URL again — it skips
   photos already on R2.

  Step 8: Verify

  1. Browse your blog — check that images load correctly
  2. Visit /admin/storage — photos should show as complete
  3. Check your R2 bucket in Cloudflare dashboard — you should see all the files

  Step 9: Clean Up

  Once everything works:

  1. Remove BLOB_READ_WRITE_TOKEN from Vercel environment variables
  2. Delete the migration route:
  rm app/api/migration/vercel-to-r2/route.ts
  git add -A
  git commit -m "Remove migration endpoint"
  git push
  3. Optionally delete the files from Vercel Blob via the Vercel dashboard
  (Storage → Blob → your store)

  Step 10: Done

  All new uploads will now go to Cloudflare R2. Your free tier includes:
  - 10 GB storage
  - 10 million reads/month
  - 1 million writes/month
  - Zero egress fees

  ---
  Troubleshooting

  Images not loading after migration:
  - Check that your R2 bucket has public access enabled (Step 2)
  - Verify NEXT_PUBLIC_CLOUDFLARE_R2_PUBLIC_DOMAIN matches exactly what you
  configured

  Migration endpoint returns 401:
  - Make sure you're signed in as admin first at /admin

  Migration times out:
  - Just hit the endpoint again — it's idempotent (skips already-migrated
  photos)
  - If consistently timing out, upgrade to Vercel Pro (300s timeout) or run in
  smaller batches

  CORS errors:
  - In your R2 bucket settings, add a CORS rule:
    - Allowed Origins: https://yourdomain.com
    - Allowed Methods: GET, PUT
    - Allowed Headers: *
