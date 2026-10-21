# Analyse & rejeu de signaux RF (SDR)

Énormément de matériel du quotidien communique en radio sub-GHz sans aucune
authentification : télécommandes, sonnettes, stations météo, capteurs de
pression de pneus (TPMS), vieux ouvre-portails et portes de garage. Avec une
radio logicielle (SDR), tu peux observer ce spectre, capturer une transmission,
comprendre son encodage, et — sur les appareils à code statique — la rejouer. La
méthode est toujours la même : trouver la fréquence, capturer, démoduler,
décoder, rejouer.

Tous les exemples sont génériques. Rien ici n'est la solution d'un challenge précis.

> **Le légal d'abord.** *Recevoir* et analyser de la radio est généralement
> libre. *Émettre* — rejeu, brouillage — est strictement réglementé presque
> partout et illégal contre du matériel qui n'est pas le tien. Tout ce qui suit
> concerne tes propres appareils sur un banc de test, idéalement blindé. Connais
> la réglementation locale avant de jamais passer en émission.

## 1. Le matériel

- **RTL-SDR** — pas cher, **réception seule**, ~24 MHz–1,7 GHz. Parfait pour
  apprendre, capturer et analyser.
- **HackRF / LimeSDR** — **émission et réception**, large plage. Nécessaire pour
  le rejeu, donc c'est aussi là qu'est la limite légale.
- Logiciels : **GQRX** / **SDR++** (spectre en direct), **Universal Radio Hacker
  (URH)** (capture + démod + décodage), **rtl_433** (décode automatiquement des
  centaines d'appareils connus).

## 2. Trouver la fréquence

La plupart du matériel grand public vit dans les bandes ISM : **433,92 MHz**
(Europe), **315 MHz** (US), **868 MHz**, **915 MHz**. Deux façons de la cerner :

- **FCC ID** — imprimé sur l'appareil ; cherche-le, le dossier indique la
  fréquence exacte de fonctionnement.
- **Regarde le waterfall** — ouvre GQRX autour de 433,92 MHz, appuie sur la
  télécommande, et repère le pic qui n'apparaît que pendant l'appui.

## 3. Capturer la salve

Dans URH (ou `rtl_433 -A` pour une identification rapide), accorde-toi sur la
fréquence, règle un taux d'échantillonnage adapté, et enregistre en déclenchant
l'appareil plusieurs fois. Tu veux plusieurs copies propres de la même salve
pour les comparer.

```
rtl_433 -f 433.92M -A        # identifie + montre les impulsions décodées
```

## 4. Lire la modulation

Zoome dans la capture. Les télécommandes sub-GHz sont presque toujours l'une de :

- **OOK / ASK** — la porteuse est simplement présente ou absente. Tu verras des
  salves et des silences nets ; décode en largeurs d'impulsion.
- **FSK** — la fréquence bascule entre deux valeurs pour le 0 et le 1.

URH montre la forme d'onde brute, te laisse régler la longueur de bit, et
convertit les salves en chaîne de bits. Aligne plusieurs captures pour trouver
le sync/préambule et la charge utile.

## 5. Code statique vs code tournant

Compare deux captures du *même bouton* :

- **Identique à chaque fois → code statique.** L'appareil n'a aucune protection
  contre le rejeu. Les bits capturés rejoués tels quels le déclenchent.
- **Différent à chaque fois → code tournant.** Chaque appui utilise une nouvelle
  valeur issue d'un compteur synchronisé + secret (KeeLoq et compagnie). Un
  rejeu direct est rejeté parce que ce code est désormais « utilisé ».

## 6. Rejeu (statique uniquement, ton propre appareil)

Avec un SDR capable d'émettre, URH peut retransmettre exactement les
échantillons capturés :

```
# conceptuellement : charger la capture -> même fréquence/modulation -> envoyer
```

Si c'est un appareil à code statique, le portail/la sonnette/la prise réagit
comme si la vraie télécommande avait été pressée. C'est toute l'attaque — et
toute la raison pour laquelle ça ne devrait pas être statique.

## 7. Pourquoi le code tournant n'est pas magique

Le rejeu échoue, mais la conception a des faiblesses connues documentées :
**capture-et-brouillage** (« rolljam ») — brouiller pour que l'appareil
n'entende jamais le premier appui, le capturer en douce, laisser l'utilisateur
rappuyer, transmettre le premier code et garder le second inutilisé pour soi. Ça
demande d'émettre (brouiller) et c'est franchement dans l'illégal contre
autrui ; comprends-le pour défendre, ne le fais pas dans la nature.

## 8. Se défendre

- **Ne livre pas de codes statiques** pour quoi que ce soit qui contrôle un
  accès. Code tournant au minimum, authentification/challenge-réponse
  idéalement.
- Préfère des protocoles avec de la vraie crypto et de la fraîcheur
  (nonces/timestamps), pas juste un compteur incrémental.
- Pour les accès à forte valeur (voitures, portails), passe au
  **challenge-réponse bidirectionnel** ou au distance bounding UWB, qui bat le
  capture-rejeu simple et le relais.
- Traite le canal RF comme n'importe quelle autre entrée : une radio non
  authentifiée est une porte ouverte.

---

Fait par **White** · [github.com/white-exee](https://github.com/white-exee)
