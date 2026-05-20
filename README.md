# ImageHosting

## Initial Installation

1. **Copy Example Config** (in the `config/` folder):
   - Copy `config/secret.sample.php` to `config/secret.php`
   - Copy `config/admin_ids_example.txt` to `config/admin_ids.txt`

   Example:

   ```bash
   cp config/secret.sample.php config/secret.php
   cp config/admin_ids_example.txt config/admin_ids.txt
   ```

2. **Set Secrets** in `config/secret.php`:
   - `admin_hmac_secret`: long, random string (32+ characters).
   - `admin_login_token`: token used for the admin login form.

3. **Set Admin IDs** in `config/admin_ids.txt`:
   - One user ID (`ih_uid`) per line.
   - Comments are supported — use `#`.

> Important: `config/admin_ids.txt` and `config/secret.php` contain sensitive data and should only be readable by the operator.

## Admin Panel

1. **Open the login page:** Access `/admin_login.php`.
2. **Enter the admin token:** Use the `admin_login_token` from `config/secret.php`.
3. **Redirect:** After a successful login, you will be redirected to `/admin.php` (Admin Panel).

Access to the admin panel is granted only if **both conditions** are met:
1. The anonymous user ID (`ih_uid`) is present in `config/admin_ids.txt`.
2. A valid `ih_admin` cookie has been set via `/admin_login.php`.

## Security Hardening

### Storage Execution Hardening

Apache users should keep the bundled `public/storage/.htaccess` in place to disable PHP execution in the storage directory:

```apache
php_flag engine off
RemoveHandler .php .phtml .phar .php3 .php4 .php5 .php7 .php8
SetHandler None
Options -ExecCGI

<FilesMatch "\.(php|phtml|phar|php[0-9])$">
    Require all denied
</FilesMatch>
```

For nginx, add a deny rule to the storage location in your vhost:

```nginx
location ~* ^/storage/.*\.(php|phtml|phar|php[0-9])$ {
  deny all;
  return 403;
}
```

### Admin Authentication Setup

Create `config/secret.php` based on `config/secret.sample.php` and set strong values for `admin_hmac_secret` and `admin_login_token`. Keep `config/admin_ids.txt` populated with admin user IDs (one per line).

Admins must authenticate via `/admin_login.php` using the admin token to mint an `ih_admin` cookie. Admin access is granted only when both `ih_uid` and `ih_admin` are valid.

### Security Headers

Set the `X-Content-Type-Options: nosniff` header to prevent MIME sniffing in the event of misconfiguration.

**Apache (global or vHost):**

```apache
Header always set X-Content-Type-Options "nosniff"
```

Optional — limit to `/storage` only:

```apache
<Location "/storage/">
    Header always set X-Content-Type-Options "nosniff"
</Location>
```

**Nginx:**

```nginx
add_header X-Content-Type-Options "nosniff" always;
```

Optional — limit to `/storage` only:

```nginx
location /storage/ {
  add_header X-Content-Type-Options "nosniff" always;
}
```

### Directory Structure & Permissions

The application requires write access for upload metadata, logs, rate limits, and served files:

- `data/uploads/` — upload metadata
- `public/storage/` — served image files
- `storage/logs/` — application logs
- `storage/ratelimit/` — rate limit state

Recommended permissions (adjust to your web server user):

```bash
chown -R www-data:www-data data public/storage storage
chmod 750 data public/storage storage
chmod 770 storage/logs storage/ratelimit
```

> Important: `public/storage/` must never execute PHP.

### PHP Settings (Recommended)

- `upload_max_filesize = 10M` (matches `IH_MAX_BYTES_PER_FILE`)
- `post_max_size = 50M` (matches `IH_MAX_BYTES_TOTAL`)
- `max_file_uploads = 20` (matches `IH_MAX_FILES_PER_REQUEST`)
- `memory_limit = 128M` (sufficient for MIME detection without unnecessary overhead)
- `display_errors = Off` (production — no internal details exposed)

## Security Model

- The anonymous user ID (`ih_uid`) is the access key (no password, no recovery).
- Capability links (`/v.php?id=...`, `/u.php?id=...`) grant access independent of the user.
- Uploads bound to a `user_id` are only visible/deletable by that user or an admin.
- TTL is enforced server-side; expired uploads are no longer retrievable.

## Known Limitations / Non-Goals

- No IP bans or IP blocklists.
- No content scanning (e.g. SVG or script analysis beyond the allowlist).
- No malware/AV scanning.
- No abuse automation or moderation workflow.

## Self-Tests / curl Examples

Blocked MIME (SVG should fail):

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -F "file=@tests/fixtures/bad.svg" \
  http://localhost:8000/api/upload.php
```

Too many files (should return 413 + `too_many_files`):

```bash
curl -s -X POST \
  -F "file[]=@tests/fixtures/a.jpg" \
  -F "file[]=@tests/fixtures/b.jpg" \
  -F "file[]=@tests/fixtures/c.jpg" \
  -F "file[]=@tests/fixtures/d.jpg" \
  -F "file[]=@tests/fixtures/e.jpg" \
  -F "file[]=@tests/fixtures/f.jpg" \
  -F "file[]=@tests/fixtures/g.jpg" \
  -F "file[]=@tests/fixtures/h.jpg" \
  -F "file[]=@tests/fixtures/i.jpg" \
  -F "file[]=@tests/fixtures/j.jpg" \
  -F "file[]=@tests/fixtures/k.jpg" \
  -F "file[]=@tests/fixtures/l.jpg" \
  -F "file[]=@tests/fixtures/m.jpg" \
  -F "file[]=@tests/fixtures/n.jpg" \
  -F "file[]=@tests/fixtures/o.jpg" \
  -F "file[]=@tests/fixtures/p.jpg" \
  -F "file[]=@tests/fixtures/q.jpg" \
  -F "file[]=@tests/fixtures/r.jpg" \
  -F "file[]=@tests/fixtures/s.jpg" \
  -F "file[]=@tests/fixtures/t.jpg" \
  -F "file[]=@tests/fixtures/u.jpg" \
  http://localhost:8000/api/upload.php
```

Oversized file (should return 413 + `file_too_large`):

```bash
curl -s -X POST \
  -F "file=@tests/fixtures/oversize.jpg" \
  http://localhost:8000/api/upload.php
```

## Pre-Launch Security Checklist

1. SVG uploads are rejected (`415`/`400` depending on client) and not stored.
2. Files larger than 10 MB return `413 file_too_large`.
3. More than 20 files return `413 too_many_files`.
4. Upload rate limiting is active (HTTP `429`).
5. `/api/admin_uploads.php` without an admin cookie returns `403`.
6. Deleting another user's upload returns `403`.
7. Direct access to `/storage/*.php` returns `403`.
