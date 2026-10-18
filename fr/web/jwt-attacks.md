# Attaques JWT

Un JSON Web Token, c'est juste `header.payload.signature`, trois blocs
base64url. L'en-tête et le payload **ne sont pas chiffrés** — n'importe qui peut
les lire et les modifier. La seule chose entre un utilisateur et l'admin, c'est
la **signature**. Toute attaque JWT revient à une question : puis-je faire
accepter au serveur un token que j'ai modifié ?

Tous les exemples sont génériques. Rien ici n'est la solution d'un challenge
précis.

## 1. Anatomie

```
eyJhbGciOiJIUzI1NiJ9 . eyJ1c2VyIjoiYm9iIiwiYWRtaW4iOmZhbHNlfQ . <signature>
   en-tête (alg)            payload (claims)                       sig
```

Décode en base64url les deux premières parties pour les lire. Modifier, c'est
ré-encoder un payload changé et produire une signature que le serveur acceptera.

## 2. Le test « est-ce seulement vérifié ? »

Avant toute finesse : change un claim (`"admin":false` -> `true`), ré-encode, et
envoie avec une signature bidon. Certaines applis décodent sans vérifier.
Victoire gratuite quand ça arrive.

## 3. alg: none

La spec autorise `alg: none` (pas de signature). Si la lib l'honore, tu forges à
volonté :

```
en-tête : {"alg":"none","typ":"JWT"}
payload : {"user":"bob","admin":true}
token   : base64url(en-tête) . base64url(payload) .     <- signature vide, garde le point final
```

Essaie `none`, `None`, `NONE` — les allow-lists naïves ratent les variantes de
casse.

## 4. Secret HMAC faible (HS256)

Si c'est signé en HMAC (`HS256`) et que le secret est faible, casse-le hors
ligne et tu peux ensuite tout signer :

```
hashcat -a 0 -m 16500 token.jwt rockyou.txt
# ou
python3 jwt_tool.py <token> -C -d rockyou.txt
```

Secret trouvé -> re-signe ton payload modifié avec (jwt_tool, jwt.io, ou
quelques lignes de code) et tu as un token admin valide.

## 5. Confusion d'algorithme (RS256 -> HS256)

Les tokens asymétriques (`RS256`) se vérifient avec une clé **publique**. Le bug
classique : le serveur appelle une fonction de vérif qui fait confiance au `alg`
du token. Passe-le en `HS256` et signe le token en utilisant la **clé publique
comme secret HMAC** — le serveur vérifie en HMAC avec cette même clé publique
qu'il possède déjà, et ça correspond.

```
python3 jwt_tool.py <token> -X k -pk public.pem
```

(La clé publique est souvent téléchargeable — `/jwks.json`, un certificat, le
TLS — c'est tout l'intérêt : publique par conception.)

## 6. Injection d'en-têtes : kid / jku / jwk

La confiance peut aussi fuir par des champs d'en-tête :

- **`kid`** pointe vers la clé. S'il sert à lire un fichier ou interroger une
  base, il est injectable : `"kid":"../../../../dev/null"` (signe avec une clé
  vide), ou une SQLi dans la recherche pour renvoyer une clé que tu contrôles.
- **`jku`** est une URL vers le jeu de clés — si le serveur la récupère sans
  allow-list, pointe-la vers ton propre JWKS.
- **`jwk`** embarque une clé dans l'en-tête — si c'est accepté, mets *ta* clé et
  signe avec.

## 7. Outils

`jwt_tool` est le couteau suisse : `-C -d <wordlist>` pour casser le HMAC,
`-X a` pour alg-none, `-X k -pk key.pem` pour RS256->HS256, `-X i`/`-X s` pour
l'injection d'en-têtes. `hashcat -m 16500` pour casser vite. `jwt.io` pour
décoder et fabriquer à la main.

## 8. Se défendre

- Fixer l'algorithme côté serveur (ne pas faire confiance au `alg` du token) ;
  refuser `none`.
- Utiliser un secret HMAC long et aléatoire, ou des clés asymétriques avec une
  gestion stricte de l'algorithme.
- Valider `kid`/`jku`/`jwk` contre une allow-list ; ne jamais charger de clé
  depuis une entrée contrôlée par l'attaquant.
- Vérifier `exp`, `iss`, `aud` — pas seulement la signature.

---

Fait par **White** · [github.com/white-exee](https://github.com/white-exee)
