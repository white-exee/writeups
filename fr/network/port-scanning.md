# Scan de ports & énumération de services

Le scan te dit ce qui écoute ; l'énumération te dit ce que *c'est* et où c'est
faible. Le but n'est pas un mur de ports ouverts — c'est une courte liste de
services avec des versions sur lesquelles agir. On va du rapide-et-large vers le
lent-et-profond.

Tous les exemples sont génériques. Rien ici n'est la solution d'un challenge précis.

## 1. Trouver les hôtes d'abord

Avant les ports, sache ce qui est vivant sur le sous-réseau :

```
nmap -sn 10.10.10.0/24          # balayage ping, sans scan de ports
```

Sur un LAN où l'ICMP est filtré, l'ARP est plus fiable :

```
arp-scan --localnet
```

## 2. Balaye tous les ports vite, puis creuse

Deux étapes, c'est plus rapide qu'un seul gros scan lent. D'abord un balayage
TCP complet, puis un scan détaillé des seuls ports qui ont répondu :

```
nmap -p- --min-rate 2000 -T4 10.10.10.5           # les 65535 ports TCP
nmap -p 22,80,445 -sC -sV 10.10.10.5              # scripts + versions
```

- `-sS` scan SYN (défaut en root, semi-ouvert, assez discret)
- `-sV` détection de service/version
- `-sC` scripts NSE par défaut
- `-O` empreinte d'OS
- `-sU` UDP (lent — scanne une courte liste top-ports, pas `-p-`)

N'oublie pas l'UDP : DNS, SNMP, TFTP, NTP s'y cachent tous.

```
nmap -sU --top-ports 50 10.10.10.5
```

## 3. Lire les versions, les relier aux faiblesses

Chaque chaîne de version est une piste. Une sortie `-sV` comme `OpenSSH 7.2`,
`vsftpd 2.3.4`, `Apache 2.4.49` invite à chercher des failles connues
(`searchsploit <produit version>`). Grab de bannière manuel quand NSE est vague :

```
nc -nv 10.10.10.5 22
curl -sI http://10.10.10.5
```

## 4. Énumérer par service

Le scan n'est que le début. Chaque service ouvert a sa propre énumération :

- **80/443 HTTP** — répertoires (`feroxbuster`/`ffuf`), vhosts, stack technique,
  pages par défaut, robots.txt.
- **445 SMB** — partages et sessions null (`smbclient -L //host -N`,
  `enum4linux-ng`).
- **21 FTP** — login anonyme, répertoires inscriptibles.
- **161 UDP SNMP** — community strings (`snmpwalk -v2c -c public`).
- **53 DNS** — transfert de zone, brute de sous-domaines (voir le writeup DNS).
- **3306/5432 BDD** — identifiants par défaut, version.

## 5. Rester efficace

- Scanne tout le range TCP une fois, tôt — un service sur le port 54321 se rate
  facilement.
- Sauvegarde une sortie exploitable (`-oA nom`) pour grep les ports plus tard.
- Limite le débit sur les cibles fragiles (`--max-rate`) — matraquer peut faire
  planter de vieux services ou déclencher la détection.

## 6. Se défendre

- N'expose que les ports dont un service a vraiment besoin ; firewall le reste
  (deny par défaut en entrée).
- Ne laisse pas fuiter les versions — ajuste bannières et server tokens quand tu
  peux.
- Patche les versions qui s'affichent ; une bannière exacte sur une build à jour
  est acceptable, une version périmée est une invitation.
- Surveille les balayages : un hôte unique qui touche beaucoup de ports vite est
  une signature.

---

Fait par **White** · [github.com/white-exee](https://github.com/white-exee)
