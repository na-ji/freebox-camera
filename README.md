<!-- generated-by: gsd-doc-writer -->
# Caméra Freebox — Utilisation sans Freebox Delta

Ce projet documente l'utilisation de la caméra IP **Freebox F-HACAM01A** (matériel Sercomm RC8310A), distribuée dans le pack sécurité de la Freebox Delta,
de manière totalement autonome, sans abonnement Free ni Freebox Delta. Il regroupe l'ensemble des
informations techniques obtenues par analyse du firmware et du protocole de provisionnement.

---

## Fonctionnalités

- **Flux vidéo RTSP et HLS** — 720p avec audio, accessibles depuis tout lecteur compatible
- **Accès root** — via injection de commande dans `test.cgi`, sans modification matérielle
- **Provisionnement QR Code** — connectez la caméra à n'importe quel routeur WPA2 sans application Free
- **API d'administration complète** — lecture/écriture de la configuration, capture, reboot
- **Intégration domotique** — Home Assistant, Synology Surveillance Station, Blue Iris, Jeedom
- **Analyse firmware** — dump 16 Mo, structure SquashFS, procédure de re-flash

---

## Démarrage rapide

### 1. Provisionner la caméra sur votre réseau WiFi

Utilisez le **[générateur de QR Code](https://na-ji.github.io/freebox-camera/)**.
Renseignez votre SSID et mot de passe WiFi. Présentez le QR Code généré face à l'objectif.
Attendez la LED verte fixe.

### 2. Trouver l'adresse IP

```bash
nmap -sn 192.168.1.0/24
# Cherchez le hostname "cam01" (ou celui que vous avez choisi)
```

Ou via l'interface DHCP de votre routeur.

### 3. Accéder aux flux

```bash
# Flux RTSP (VLC, ffmpeg, Home Assistant…)
# Configurer d'abord sp_uri :
curl -u freeboxcam:<MOT_DE_PASSE> "http://<IP>/adm/set_group.cgi?group=H264&sp_uri=live.sdp&sp_uri2=live2.sdp"
# Puis ouvrir :
rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/live.sdp

# Snapshot JPEG
curl -u freeboxcam:<MOT_DE_PASSE> http://<IP>/img/snapshot.cgi -o capture.jpg

# Infos publiques (sans authentification)
curl http://<IP>/util/query.cgi
```

---

## Générateur de QR Code

Ouvrez [`index.html`](index.html) dans un navigateur. Aucune installation requise, aucun serveur
nécessaire. Le QR Code est généré localement dans votre navigateur.

---

## Documentation

| Document | Description |
|----------|-------------|
| [Index de la documentation](docs/index.md) | Vue d'ensemble de tous les guides |
| [Endpoints API](docs/endpoints-api.md) | Référence complète de l'API d'administration Sercomm |
| [Structure du firmware](docs/structure-firmware.md) | Partitions MTD, layout flash, SquashFS, re-flash |
| [Accès root](docs/acces-root.md) | Injection de commande, algorithme id2pwd, console UART |
| [Provisionnement QR](docs/provisionnement-qr.md) | Format QR, provisionnement sans Freebox, dépannage |
| [Intégration domotique](docs/integration-domotique.md) | Home Assistant, Synology, Blue Iris, Jeedom |
| [Identification matérielle](docs/identification-materiel.md) | PCB, UART, specs complètes, FCC |

---

## Compatibilité matérielle

Ce projet s'applique principalement à la caméra **Freebox F-HACAM01A 0-WN**, qui est une version
OEM de la **Sercomm RC8310A**. Les caméras Sercomm partageant le même firmware (RC8221, RC8230…)
présentent des vulnérabilités similaires, notamment la clé privée Stunnel partagée entre tous les
appareils.

Caractéristiques principales :
- SoC ARM (armv5tejl), Linux 3.2.7, BusyBox, uClibc
- Flash 16 Mo (SPI NOR), WiFi 802.11b/g/n 2.4 GHz
- Résolution 720p (1280×720), objectif fixe, LED IR
- Audio intégré (microphone)

---

## Avertissements et responsabilités

Ce projet est publié à des fins **éducatives et de recherche en sécurité**. L'utilisation de ces
informations sur des équipements qui ne vous appartiennent pas est illégale. Les vulnérabilités
documentées (injection de commandes, clé TLS partagée) ont été identifiées dans le cadre d'une
analyse sur matériel personnel.

**Aucune affiliation avec Iliad/Free SA, Sercomm Corporation ou tout opérateur télécom.**

---

## Licence

Contenu de documentation sous licence [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
Code (scripts, générateur HTML) sous licence MIT.
