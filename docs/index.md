<!-- generated-by: gsd-doc-writer -->
# Documentation — Freebox F-HACAM01A

Bienvenue dans la documentation technique de la **Freebox F-HACAM01A** (Sercomm RC8310A).
Ces guides couvrent l'ensemble des aspects nécessaires à une utilisation autonome de la caméra,
du provisionnement initial jusqu'à l'intégration dans un système domotique.

---

## Index des guides

### Utilisation

| Guide | Description |
|-------|-------------|
| [Provisionnement QR Code](provisionnement-qr.md) | Comment connecter la caméra à un réseau WiFi sans Freebox Delta, format du QR Code, dépannage |
| [Intégration domotique](integration-domotique.md) | Flux RTSP/HLS, Home Assistant, Synology, Blue Iris, Jeedom, détection de mouvement |

### Référence technique

| Guide | Description |
|-------|-------------|
| [Endpoints API](endpoints-api.md) | Référence complète de l'API HTTP d'administration : tous les endpoints, paramètres, exemples de réponses, groupes de configuration |
| [Structure du firmware](structure-firmware.md) | Partitions MTD, layout de la flash 16 Mo, contenu SquashFS, dump et re-flash |
| [Modification du firmware](modification-firmware.md) | Contournement des vérifications de `fileupload.cgi`, script de reconstruction, procédure complète |

### Sécurité et accès avancé

| Guide | Description |
|-------|-------------|
| [Accès root](acces-root.md) | Injection de commande via `test.cgi`, algorithme de mot de passe root (`id2pwd`), console UART série |
| [Identification matérielle](identification-materiel.md) | Modèle OEM, dossier FCC, description du PCB, localisation UART, ouverture du boîtier, spécifications complètes |

---

## Prérequis

Pour suivre ces guides, vous aurez besoin :

- D'une caméra **Freebox F-HACAM01A** (ou Sercomm RC8310A)
- D'un réseau WiFi WPA2 2.4 GHz
- D'un navigateur web récent (pour le générateur de QR Code)
- De `curl` pour les exemples d'API
- Optionnellement : `nmap`, `vlc`, `ffmpeg`, un câble UART 3.3V

---

## Accès rapide

**Identifiants par défaut (après provisionnement) :**

| Champ | Valeur |
|-------|--------|
| Utilisateur | `freeboxcam` |
| Mot de passe | *(unique par caméra, défini par l'app Freebox — voir [endpoints-api.md](endpoints-api.md))* |
| Port HTTP | 80 |
| Port HTTPS | 443 |
| Port RTSP | 554 |

**Flux vidéo (remplacer `<IP>` par l'adresse de votre caméra) :**

```
rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/<sp_uri>      # HD 720p + audio
rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/<sp_uri2>     # SD
http://freeboxcam:<MOT_DE_PASSE>@<IP>/img/stream.m3u8   # HLS
http://freeboxcam:<MOT_DE_PASSE>@<IP>/img/snapshot.cgi  # Snapshot JPEG
```

> `sp_uri` / `sp_uri2` se configurent via `set_group.cgi?group=H264`. Voir [Intégration domotique](integration-domotique.md).

---

## À propos de ce projet

Analyse réalisée sur matériel personnel. Aucune affiliation avec Iliad/Free SA ou Sercomm Corporation.
Retour vers le [README principal](../README.md).
