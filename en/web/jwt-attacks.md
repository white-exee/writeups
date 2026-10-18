# JWT attacks

A JSON Web Token is just `header.payload.signature`, three base64url chunks. The
header and payload are **not encrypted** — anyone can read and edit them. The
only thing standing between a user and admin is the **signature**. Every JWT
attack is really one question: can I get the server to accept a token I changed?

All examples are generic. Nothing here is the solution to a specific challenge.

## 1. Anatomy

```
eyJhbGciOiJIUzI1NiJ9 . eyJ1c2VyIjoiYm9iIiwiYWRtaW4iOmZhbHNlfQ . <signature>
   header (alg)              payload (claims)                      sig
```

Base64url-decode the first two parts to read them. Tampering means re-encoding a
modified payload and producing a signature the server will trust.

## 2. The "is it even checked?" test

Before anything clever: change a claim (`"admin":false` -> `true`), re-encode,
and send it with a garbage signature. Some apps decode without verifying. Free
win when it happens.

## 3. alg: none

The spec allows `alg: none` (no signature). If the library honours it, forge at
will:

```
header:  {"alg":"none","typ":"JWT"}
payload: {"user":"bob","admin":true}
token:   base64url(header) . base64url(payload) .     <- empty signature, keep the trailing dot
```

Try `none`, `None`, `NONE` — naive allow-lists miss the case variants.

## 4. Weak HMAC secret (HS256)

If it's signed with HMAC (`HS256`) and the secret is weak, crack it offline and
then you can sign anything:

```
hashcat -a 0 -m 16500 token.jwt rockyou.txt
# or
python3 jwt_tool.py <token> -C -d rockyou.txt
```

Found the secret -> re-sign your tampered payload with it (jwt_tool, jwt.io, or a
few lines of code) and you have a valid admin token.

## 5. Algorithm confusion (RS256 -> HS256)

Asymmetric tokens (`RS256`) are verified with a **public** key. The classic bug:
the server calls a verify function that trusts the token's own `alg`. Switch it
to `HS256` and sign the token using the **public key as the HMAC secret** — the
server HMAC-verifies with that same public key it already has, and it matches.

```
python3 jwt_tool.py <token> -X k -pk public.pem
```

(The public key is often downloadable — `/jwks.json`, a cert, TLS — which is the
whole point: public by design.)

## 6. Header injection: kid / jku / jwk

Trust can also leak through header fields:

- **`kid`** points to the key. If it's used to read a file or hit a DB, it's
  injectable: `"kid":"../../../../dev/null"` (sign with an empty key), or SQLi in
  the lookup to return a key you control.
- **`jku`** is a URL to the key set — if the server fetches it without an
  allow-list, point it at your own JWKS.
- **`jwk`** embeds a key in the header — if accepted, embed *your* key and sign
  with it.

## 7. Tools

`jwt_tool` is the swiss-army knife: `-C -d <wordlist>` to crack HMAC, `-X a` for
alg-none, `-X k -pk key.pem` for RS256->HS256, `-X i`/`-X s` for header
injection. `hashcat -m 16500` for fast cracking. `jwt.io` to decode and hand-craft.

## 8. Defending

- Pin the algorithm server-side (don't trust the token's `alg`); reject `none`.
- Use a long random HMAC secret, or asymmetric keys with strict algorithm
  handling.
- Validate `kid`/`jku`/`jwk` against an allow-list; never fetch or load keys from
  attacker-controlled input.
- Verify `exp`, `iss`, `aud` — not just the signature.

---

Built by **White** · [github.com/white-exee](https://github.com/white-exee)
