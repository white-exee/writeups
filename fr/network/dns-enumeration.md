# Énumération DNS & transfert de zone

Le DNS est une carte de la surface d'attaque d'une cible : hôtes, serveurs mail,
services, et souvent des indices de nommage interne. Avant de toucher à quoi que
ce soit, je l'énumère. Et de temps en temps, un serveur mal configuré te donne
la carte complète via un transfert de zone.

Tous les exemples sont génériques. Rien ici n'est la solution d'un challenge
précis.

## 1. Les enregistrements à demander

```
dig target.com A           # IPv4
dig target.com AAAA        # IPv6
dig target.com MX          # serveurs mail
dig target.com NS          # serveurs de noms
dig target.com TXT         # SPF/DKIM, chaînes de vérif, parfois des fuites
dig target.com SOA         # métadonnées de zone
dig target.com ANY         # tout ce que le serveur veut bien donner (souvent filtré)
```

`NS` et `MX` pointent vers plus d'infrastructure ; les `TXT` révèlent souvent les
SaaS utilisés par une boîte (et parfois plus).

## 2. Transfert de zone (AXFR)

Un transfert de zone, c'est la façon dont un serveur secondaire récupère la zone
complète depuis le primaire. Si le serveur l'autorise à tout le monde, tu
obtiens **tous les enregistrements** — tous les sous-domaines, les hôtes
internes, tout :

```
dig axfr target.com @ns1.target.com
```

Essaie-le contre *chaque* serveur de noms de la requête `NS` — souvent un seul
est mal configuré. Un AXFR réussi est l'un des meilleurs gains de recon ; un
AXFR refusé veut juste dire « fais-le à l'ancienne ».

## 3. Énumération de sous-domaines (à l'ancienne)

Quand l'AXFR est fermé, construis la liste de sous-domaines toi-même :

- **Certificate transparency** — chaque certificat TLS est logué publiquement.
  `crt.sh` (`https://crt.sh/?q=%25.target.com`) liste les noms ayant eu un
  certificat. Rapide, passif, aucun paquet vers la cible.
- **Brute force** — `dnsx`, `gobuster dns`, `dnsrecon` sur une wordlist.
- **Sources passives** — amass, subfinder agrègent des données publiques.

(C'est exactement le type de recon qu'un outil comme
[Scryn](https://github.com/white-exee/Scryn) automatise avec crt.sh.)

## 4. DNS inverse

Passer de l'IP au nom peut révéler des schémas de nommage sur une plage :

```
dig -x 203.0.113.10
```

Balayer les PTR d'un netblock cartographie souvent tout un environnement.

## 5. Se défendre

- **Restreindre l'AXFR** aux seuls serveurs secondaires connus — jamais ouvert
  au monde.
- Garder les hôtes internes dans une zone **split-horizon** pour qu'ils ne
  soient pas dans la zone publique.
- Considérer les `TXT` comme publics : n'y mets rien de sensible.

---

Fait par **White** · [github.com/white-exee](https://github.com/white-exee)
