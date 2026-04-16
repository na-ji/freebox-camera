<!-- generated-by: gsd-doc-writer -->
# Identification matérielle — Freebox F-HACAM01A

Ce document décrit l'identité du matériel, le dossier FCC, la description du PCB, la localisation
des pads UART et les spécifications techniques complètes.

---

## Identité du modèle

| Champ | Valeur |
|-------|--------|
| Nom commercial (Free) | Freebox F-HACAM01A 0-WN |
| Modèle OEM | **Sercomm RC8310A** |
| Identifiant FCC | P27RC8510A |
| Hardware ID (firmware) | `CVP` |
| Vendor ID OEM | `0x0062` (version Freebox) |

La caméra distribuée par Free est une version **OEM (white-label)** du modèle Sercomm RC8310A.
Le firmware a été légèrement modifié par Free : certains endpoints ont été désactivés et les
credentials par défaut ont été modifiés, mais l'architecture logicielle est identique.

---

## Dossier FCC (P27RC8510A)

Le dossier FCC est publiquement accessible via la base de données FCC (fcc.gov).

| Champ | Valeur |
|-------|--------|
| FCC ID | P27RC8510A |
| Demandeur | Sercomm Corporation |
| Type d'appareil | Caméra IP WiFi |
| Fréquences | 2.4 GHz (802.11b/g/n) |

Le dossier inclut les photos internes du PCB, les schémas d'antenne et les rapports de test SAR/RF.
Ces documents sont utiles pour identifier les pads UART sur le PCB.

---

## Spécifications techniques complètes

### Vidéo

| Paramètre | Valeur |
|-----------|--------|
| Résolution maximale | 1280×720 (720p HD) |
| Résolution secondaire | 640×360 |
| Codec vidéo | H.264 (Baseline Profile) |
| Fréquence d'images | 1–25 fps (configurable) |
| Objectif | Fixe (focale fixe, grand angle) |
| Vision nocturne | LED IR intégrées |
| Champ de vision | ~100° (horizontal) |

### Audio

| Paramètre | Valeur |
|-----------|--------|
| Microphone | Intégré (omnidirectionnel) |
| Codec audio | AAC |
| Transmission | Incluse dans le flux RTSP |
| Haut-parleur | Non |

### Réseau

| Paramètre | Valeur |
|-----------|--------|
| WiFi | 802.11 b/g/n, 2.4 GHz uniquement |
| Bande 5 GHz | Non supportée |
| Chiffrement WiFi | WPA2-PSK (TKIP/AES) |
| Ethernet filaire | Non (WiFi uniquement) |
| Port HTTP | 80 |
| Port HTTPS | 443 |
| Port RTSP | 554 |
| UPnP/SSDP | Oui |
| Bonjour/mDNS | Oui |
| ONVIF | Profile S |

### Système

| Paramètre | Valeur |
|-----------|--------|
| SoC | ARM (armv5tejl) |
| Architecture | ARMv5TEJ |
| Noyau | Linux 3.2.7 |
| Espace utilisateur | BusyBox + uClibc |
| Flash | 16 Mo (SPI NOR) |
| RAM | ~64 Mo (SDRAM) |

### Alimentation et boîtier

| Paramètre | Valeur |
|-----------|--------|
| Alimentation | USB 5V (câble micro-USB ou USB-A) |
| Consommation | ~3–5W |
| Boîtier | Plastique blanc, dôme circulaire |
| Diamètre PCB | ~55 mm |
| Support | Fixation magnétique + socle orientable |
| LED d'état | LED bicolore (rouge/vert/orange) |
| Bouton reset | Oui (accessible sur la base) |
| Utilisation | Intérieur uniquement |

---

## Description du PCB

Le PCB de la caméra est de forme **circulaire**, d'environ 55 mm de diamètre. Il est logé dans
le boîtier dôme blanc.

Composants principaux visibles :

- **SoC ARM** : puce principale au centre du PCB
- **Module WiFi** : puce ou module intégré (souvent marqué avec le modèle 802.11n)
- **Capteur d'image** : sur la face avant, derrière l'objectif
- **LED IR** : rangée de LED infrarouges autour de l'objectif pour la vision nocturne
- **LED d'état** : LED bicolore visible depuis l'extérieur du boîtier
- **Connecteur multi-broches blanc** : alimentation USB et potentiellement d'autres signaux
- **Bouton reset** : micro-bouton tactile
- **Pads UART** : ligne de 4 pads non soudés, situés près du connecteur multi-broches

---

## Ouverture du boîtier

> Opérez sur surface douce pour ne pas rayer le boîtier.

Le boîtier est composé de deux parties assemblées par **encliquetage** (sans vis) :

1. **Repérez la rainure** entre le dôme blanc supérieur et la bague de base.
2. Insérez un **pry tool en plastique** (ou un tournevis plat fin enveloppé d'adhésif pour
   éviter les marques) dans la rainure à n'importe quel point.
3. Faites **progressivement le tour du boîtier** en dégageant les clips un à un.
4. Soulevez le dôme — le PCB reste fixé sur la base.
5. Le PCB est accessible pour identifier et souder les pads UART.

> **Note :** L'ouverture n'affecte pas le fonctionnement de la caméra et ne laisse pas de traces
> visibles si réalisée avec précaution.

---

## UART — Localisation et connexion

### Localisation des pads

Les pads UART se trouvent sur le PCB, à proximité du **connecteur blanc multi-broches**. Ils
se présentent généralement sous forme d'une rangée de 4 pastilles (throughhole ou SMD non soudées).

Pour identifier les pads avec certitude, comparez avec les photos du dossier FCC (P27RC8510A)
disponibles sur le site fcc.gov — les photos internes montrent le PCB et les marquages.

### Identification des pads (sans schéma)

| Pad | Méthode d'identification |
|-----|--------------------------|
| GND | Continuité avec la masse du PCB (multimètre en mode continuité) |
| VCC | Tension ~3.3V par rapport à GND au démarrage — **ne pas connecter** |
| TX | Oscille pendant le boot (observable à l'oscilloscope, ou brancher l'adaptateur UART en écoute et vérifier si du texte apparaît) |
| RX | Pad restant (entre VCC et TX dans la rangée standard) |

Ordre typique sur ce type de PCB : `GND – VCC – TX – RX` ou `GND – TX – RX – VCC`.

### Câblage

| Adaptateur UART (PC) | Caméra (PCB) |
|----------------------|--------------|
| GND | GND |
| RX | TX |
| TX | RX |
| 3.3V | **Non connecté** |

> Alimentation de la caméra via son câble USB habituel. Ne jamais alimenter via VCC UART.

### Paramètres de la liaison

```
Vitesse    : 115200 baud
Bits data  : 8
Parité     : aucune
Bits stop  : 1 (8N1)
Niveau     : 3.3V CMOS
```

### Connexion depuis un PC Linux/macOS

```bash
# Identifier l'adaptateur
ls /dev/ttyUSB* /dev/tty.usbserial-*

# Ouvrir la console
screen /dev/ttyUSB0 115200
# ou
picocom -b 115200 /dev/ttyUSB0
# ou
minicom -D /dev/ttyUSB0 -b 115200
```

### Sortie console au démarrage

La console UART affiche le boot Linux complet, y compris :
- L'initialisation U-Boot (ou bootloader équivalent)
- Le décompression du noyau
- La détection des périphériques MTD
- Le montage du SquashFS
- Le lancement des scripts d'init

À la fin du boot, une invite de connexion apparaît :

```
cam01 login: root
Password: Aq6#0062
```

Voir [Accès root](acces-root.md) pour plus de détails sur le mot de passe root et l'algorithme
qui le génère.

---

## Variantes matérielles connues

| Modèle | Distributeur | Identifiant |
|--------|-------------|-------------|
| F-HACAM01A 0-WN | Free (France) | Vendor ID `0x0062` |
| RC8310A | Sercomm (OEM direct) | Vendor ID variable |
| RC8221 | Divers opérateurs | Firmware similaire |
| RC8230 | Divers opérateurs | Firmware similaire |

Les modèles RC8221 et RC8230 partagent une architecture logicielle proche. La vulnérabilité
d'injection dans `test.cgi` et la clé privée Stunnel partagée sont documentées sur les variantes
Sercomm RC de cette génération.

---

## Identifiants d'exemple

Ces identifiants sont ceux d'un appareil spécifique — ils varient d'un exemplaire à l'autre.

| Champ | Valeur d'exemple |
|-------|-----------------|
| Adresse MAC | `XX:XX:XX:XX:XX:XX` |
| Numéro de série | `XXXXXXXXXXXXXXXXXX` |
| Hostname par défaut | Défini lors du provisionnement |

L'adresse MAC et le numéro de série sont stockés dans la partition `mtd1` (MAC, 64 Ko).
