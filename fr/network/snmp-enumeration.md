# Énumération SNMP

SNMP sert à gérer les équipements réseau et les serveurs, et il adore parler. En
v1/v2c, la seule « authentification » est une **community string** — et elle est
très souvent laissée par défaut. Trouve-la et l'équipement te sort interfaces,
processus, logiciels, routes, parfois même les utilisateurs.

Tous les exemples sont génériques. Rien ici n'est la solution d'un challenge
précis.

## 1. Les bases

- UDP **161** (requêtes), 162 (traps).
- **v1 / v2c** : community string en clair. `public` = lecture seule par défaut,
  `private` = lecture/écriture.
- **v3** : vraie auth + chiffrement (donc ces astuces ne s'appliquent pas).

Comme c'est de l'UDP, un scan TCP rapide le rate — scanne spécifiquement
UDP 161.

## 2. Trouver la community string

Les chaînes par défaut sont partout. Brute une petite liste vite avec
`onesixtyone` :

```
onesixtyone -c community.txt 192.168.1.10
# ou nmap
nmap -sU -p161 --script snmp-brute 192.168.1.10
```

À tester en premier : `public`, `private`, `manager`, `community`, le nom du
constructeur.

## 3. Parcourir l'arbre

Avec une community string valide, `snmpwalk` déverse toute la MIB :

```
snmpwalk -v2c -c public 192.168.1.10
snmpbulkwalk -v2c -c public 192.168.1.10          # plus rapide
snmp-check 192.168.1.10 -c public                 # résumé bien formaté
```

OID utiles quand tu ne veux pas tout :

```
1.3.6.1.2.1.1.5     # hostname
1.3.6.1.2.1.25.4.2.1.2   # processus en cours
1.3.6.1.2.1.25.6.3.1.2   # logiciels installés
1.3.6.1.2.1.2.2.1.2      # interfaces réseau
1.3.6.1.4.1.77.1.2.25    # utilisateurs Windows (LanMan)
```

Ce qui ressort : noms d'hôtes, tables d'interfaces et de routage complètes,
cache ARP, liste des processus, paquets installés, et sous Windows les comptes
utilisateurs — une image complète de l'hôte sans toucher à autre chose.

## 4. Accès en écriture

Si une chaîne type `private` autorise l'écriture (`snmpset`), tu peux changer la
config de l'équipement — rerouter, désactiver des interfaces, etc. Rare mais
très puissant sur du matériel réseau.

## 5. Se défendre

- **Désactiver SNMP** si inutilisé. Si tu l'utilises, **passe en v3** avec auth
  + chiffrement.
- Changer les community strings par défaut pour quelque chose de non devinable,
  et les mettre en **lecture seule** sauf besoin réel d'écriture.
- **Filtrer UDP 161** au pare-feu vers les seuls hôtes de supervision.

---

Fait par **White** · [github.com/white-exee](https://github.com/white-exee)
