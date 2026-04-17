<!-- generated-by: gsd-doc-writer -->
# Référence API — Sercomm RC8310A / Freebox F-HACAM01A

Ce document décrit l'ensemble des endpoints HTTP exposés par la caméra. L'API est accessible sur
le port 80 (HTTP) et le port 443 (HTTPS, certificat auto-signé). L'authentification est de type
**HTTP Basic**.

---

## Identifiants par défaut

| Champ | Valeur |
|-------|--------|
| Utilisateur | `freeboxcam` |
| Mot de passe | *(propre à chaque caméra, défini par l'application Freebox lors de l'appairage)* |

Le nom d'utilisateur est identique sur toutes les caméras F-HACAM01A. Le mot de passe est généré
par l'application Freebox et est **unique par appareil** — il n'est pas inscrit dans le firmware.
Retrouvez-le dans les réglages de l'application Freebox (section Caméra). Dans tous les exemples
`curl` ci-dessous, remplacez `<MOT_DE_PASSE>` par votre mot de passe réel.

Ces identifiants peuvent être modifiés via `set_group.cgi?group=USER`.

---

## Endpoints publics (sans authentification)

### `GET /util/query.cgi`

Retourne les informations publiques de la caméra au format texte (paires `clé=valeur`).

```bash
curl http://<IP>/util/query.cgi
```

Exemple de réponse :

```
hostname=cam01
mac=XX:XX:XX:XX:XX:XX
model=F-HACAM01A
resolution=1280x720
resolution2=640x360
```

### `GET /util/query.cgi?extension=yes`

Retourne les informations étendues : firmware, adresse IP, ports.

```bash
curl http://<IP>/util/query.cgi?extension=yes
```

Exemple de réponse (champs supplémentaires) :

```
fw_version=1.0.16.4
ip_addr=192.168.1.42
http_port=80
https_port=443
rtsp_port=554
```

---

## Endpoints d'information système (authentification requise)

### `GET /adm/sysinfo.cgi`

Retourne la version firmware et le numéro de série.

```bash
curl -u freeboxcam:<MOT_DE_PASSE> http://<IP>/adm/sysinfo.cgi
```

Exemple de réponse :

```
fw_version=1.0.16.4
serial=XXXXXXXXXXXXXXXXXX
hw_id=CVP
```

### `GET /adm/ping.cgi`

Test de disponibilité. Répond `OK` si la caméra fonctionne normalement.

```bash
curl -u freeboxcam:<MOT_DE_PASSE> http://<IP>/adm/ping.cgi
```

### `GET /adm/log.cgi`

Retourne les journaux système. La réponse est souvent vide sur les firmwares Freebox.

```bash
curl -u freeboxcam:<MOT_DE_PASSE> http://<IP>/adm/log.cgi
```

### `GET /adm/admdbg_status.cgi`

Retourne la liste des modules de débogage sous forme XML.

```bash
curl -u freeboxcam:<MOT_DE_PASSE> http://<IP>/adm/admdbg_status.cgi
```

---

## Gestion de la configuration

### `GET /adm/get_group.cgi?group=<GROUPE>`

Lit tous les paramètres d'un groupe de configuration. Voir le tableau des groupes ci-dessous.

```bash
curl -u freeboxcam:<MOT_DE_PASSE> "http://<IP>/adm/get_group.cgi?group=SYSTEM"
```

Exemple de réponse (groupe SYSTEM) :

```
hostname=cam01
ntp_server=pool.ntp.org
timezone=Europe/Paris
```

### `GET /adm/set_group.cgi?group=<GROUPE>&<clé>=<valeur>`

### `POST /adm/set_group.cgi`

Écrit un ou plusieurs paramètres dans un groupe. Peut s'utiliser en GET (paramètres dans l'URL)
ou en POST (corps `application/x-www-form-urlencoded`).

```bash
# Exemple GET
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/set_group.cgi?group=SYSTEM&hostname=macam"

# Exemple POST
curl -u freeboxcam:<MOT_DE_PASSE> \
  -X POST \
  -d "group=NETWORK&ip_addr=192.168.1.50&ip_type=static" \
  http://<IP>/adm/set_group.cgi
```

### `GET /adm/admcfg.cfg`

Télécharge la configuration complète de la caméra, encodée en base64 et chiffrée.

```bash
curl -u freeboxcam:<MOT_DE_PASSE> http://<IP>/adm/admcfg.cfg -o config.cfg
```

### `POST /adm/upload.cgi`

Upload d'une configuration précédemment sauvegardée.

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  -F "file=@config.cfg" \
  http://<IP>/adm/upload.cgi
```

---

## Flux vidéo et capture

### `GET /img/snapshot.cgi`

Retourne une image JPEG de la vue courante.

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  http://<IP>/img/snapshot.cgi -o snapshot.jpg
```

### `GET /img/stream.m3u8`

Retourne le manifest HLS. Peut être ouvert directement dans VLC ou un lecteur compatible.

```
http://freeboxcam:<MOT_DE_PASSE>@<IP>/img/stream.m3u8
```

### RTSP (port 554)

Les flux RTSP ne sont pas servis via CGI mais directement par le démon RTSP.

| Flux | URL |
|------|-----|
| HD 720p (H.264 + AAC) | `rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/<sp_uri>` |
| SD 360p | `rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/<sp_uri2>` |

> **Important :** `sp_uri` et `sp_uri2` sont des paramètres configurables du groupe `H264`.
> Leur valeur par défaut est **vide** après une remise à zéro. Définissez-les avant utilisation :
> ```bash
> curl -u freeboxcam:<MOT_DE_PASSE> \
>   "http://<IP>/adm/set_group.cgi?group=H264&sp_uri=live.sdp&sp_uri2=live2.sdp"
> ```
> Voir [Intégration domotique](integration-domotique.md#configuration-préalable-des-uris-rtsp) pour le détail.

---

## Firmware

### `GET /adm/flash_dumper.cgi`

Retourne le dump complet de la flash (16 Mo). Peut prendre plusieurs minutes.

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  http://<IP>/adm/flash_dumper.cgi -o fw_backup.bin
```

> **Note :** Ce endpoint est l'un des plus sensibles. Il expose l'intégralité du contenu de la
> flash, y compris les partitions de configuration et de calibration.

### `POST /adm/upgrade.cgi`

Upload et installation d'un firmware. La caméra redémarre automatiquement après l'opération.

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  -F "file=@firmware_patche.bin" \
  http://<IP>/adm/upgrade.cgi
```

> **Attention :** Un firmware invalide peut rendre la caméra inutilisable (brick). Consultez
> [Structure du firmware](structure-firmware.md) avant toute tentative.

---

## Actions système

### `GET /adm/reboot.cgi`

**Redémarre immédiatement la caméra sans demande de confirmation.**

```bash
curl -u freeboxcam:<MOT_DE_PASSE> http://<IP>/adm/reboot.cgi
```

### `GET /adm/reset_to_default.cgi`

**Réinitialise la caméra aux paramètres d'usine sans demande de confirmation.** La caméra
repassera en mode provisionnement (LED orange clignotante).

```bash
curl -u freeboxcam:<MOT_DE_PASSE> http://<IP>/adm/reset_to_default.cgi
```

---

## Endpoints désactivés par Free

Ces endpoints existent dans le firmware Sercomm d'origine mais ont été bloqués dans la version
distribuée par Free :

| Endpoint | Comportement |
|----------|-------------|
| `/adm/file.cgi?todo=inject_telnetd` | Retourne HTTP 403 |
| `/adm/enable_ui.cgi` | Retourne `ERROR` |

---

## Test et injection de commandes

### `GET /adm/test.cgi?action=ftp_test&...`

Prévu pour tester une connexion FTP. Le paramètre `ftp_path` est passé directement à `system()`
sans validation, ce qui permet une exécution de commandes arbitraires.

Voir [Accès root](acces-root.md) pour le détail de l'exploitation.

---

## ONVIF

### `POST /onvif/device_service`

Interface ONVIF Profile S. La méthode `GetSystemDateAndTime` répond sans authentification.
Les autres méthodes requièrent l'authentification WS-Security PasswordDigest.

```bash
# Exemple : récupérer la date/heure sans auth
curl -s -X POST http://<IP>/onvif/device_service \
  -H "Content-Type: application/soap+xml" \
  -d '<?xml version="1.0"?>
<Envelope xmlns="http://www.w3.org/2003/05/soap-envelope">
  <Body>
    <GetSystemDateAndTime xmlns="http://www.onvif.org/ver10/device/wsdl"/>
  </Body>
</Envelope>'
```

---

## Groupes de configuration

Le tableau suivant liste tous les groupes accessibles via `get_group.cgi` et `set_group.cgi`,
avec leurs principaux paramètres.

| Groupe | Description | Paramètres principaux |
|--------|-------------|----------------------|
| `SYSTEM` | Paramètres système généraux | `hostname`, `ntp_server`, `timezone` |
| `VIDEO` | Configuration vidéo globale | `flip`, `mirror`, `color`, `exposure`, `sharpness`, `saturation`, `contrast`, `hue`, `night_mode`, `text_overlay`, `text` |
| `H264` | Encodage H.264 | `bitrate`, `quality`, `gop`, `profile` |
| `JPEG` | Capture JPEG | `quality`, `resolution` |
| `NETWORK` | Configuration réseau filaire | `ip_type` (dhcp/static), `ip_addr`, `netmask`, `gateway`, `dns1`, `dns2` |
| `WIRELESS` | Configuration WiFi | `ssid`, `wpa_ascii`, `channel`, `mode`, `encrypt` |
| `RTSP_RTP` | Serveur RTSP/RTP | `rtsp_port`, `rtp_port_start`, `multicast_enable` |
| `HTTP` | Serveur HTTP | `http_port`, `https_port`, `auth_type` |
| `HTTP_NOTIFY` | Notifications HTTP GET/POST sur événement | `http_notify`, `http_url`, `http_method` (0=GET, 1=POST), `http_user`, `http_password`, `event_data_flag` |
| `HTTP_EVENT` | Envoi de snapshots JPEG vers serveur HTTP/FTP | `http_event_en`, `http_post_en`, `http_post_url`, `http_post_user`, `http_post_pass` |
| `EMAIL` | Alertes par e-mail | `smtp_server`, `smtp_port`, `from`, `to`, `tls_enable` |
| `FTP` | Transfert FTP (captures) | `ftp_server`, `ftp_port`, `ftp_account`, `ftp_passwd`, `ftp_path` |
| `MOTION` | Détection de mouvement | `md_mode`, `md_sensitivity1`–`4`, `md_threshold1`–`4`, `md_switch1`–`4`, `md_window1`–`4` |
| `AUDIO` | Configuration audio | `audio_in`, `in_volume`, `au_trigger_en`, `au_trigger_volume`, `au_trigger_method` |
| `EVENT` | Planification des événements | `schedule_enable`, `event_trigger` |
| `UPNP` | UPnP / IGD | `enable`, `friendly_name` |
| `BONJOUR` | mDNS/Bonjour | `enable`, `service_name` |
| `USER` | Gestion des utilisateurs | `username`, `password` (modifie les credentials HTTP/RTSP) |

### Exemple : lire la configuration WiFi

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/get_group.cgi?group=WIRELESS"
```

Réponse typique :

```
ssid=MonRéseau
encrypt=WPA2PSK
wpa_ascii=MonMotDePasse
channel=0
mode=11bgn
```

### Exemple : activer la détection de mouvement

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/set_group.cgi?group=MOTION&enable=1&sensitivity=5"
```

### Exemple : configurer le texte affiché sur le flux vidéo (OSD)

La caméra peut superposer un texte sur le flux RTSP/HLS. Ce comportement est contrôlé par deux
paramètres du groupe `VIDEO` :

| Paramètre | Type | Description |
|-----------|------|-------------|
| `text_overlay` | `0` / `1` | Active (`1`) ou désactive (`0`) l'affichage du texte |
| `text` | chaîne | Texte affiché en incrustation sur l'image |

```bash
# Afficher "Salon" sur le flux
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/set_group.cgi?group=VIDEO&text_overlay=1&text=Salon"

# Désactiver l'incrustation
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/set_group.cgi?group=VIDEO&text_overlay=0"

# Lire la configuration actuelle
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/get_group.cgi?group=VIDEO"
```

> **Note :** La valeur par défaut de `text` après remise à zéro est une chaîne vide.
> Si un texte inattendu apparaît sur le flux, vérifiez ce champ.

---

### Exemple : configurer une notification HTTP sur événement mouvement

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/set_group.cgi?group=HTTP_EVENT&enable=1&url=http://192.168.1.10:8123/api/webhook/cam_motion&http_event_type=motion"
```

---

## Architecture CGI

La caméra expose 34+ fichiers CGI, dont la plupart sont des **liens symboliques** pointant vers
5 binaires principaux. Le binaire est déterminé par `argv[0]` (le nom du lien symbolique).

Binaires principaux :

| Binaire | Liens symboliques associés |
|---------|---------------------------|
| `fileupload.cgi` | `upgrade.cgi`, `upload.cgi` |
| `sysinfo.cgi` | `query.cgi`, `ping.cgi`, `log.cgi`, `reboot.cgi`, `reset_to_default.cgi` |
| `admcfg.cgi` | `get_group.cgi`, `set_group.cgi`, `admcfg.cfg`, `admdbg_status.cgi` |
| `test.cgi` | `test.cgi` |
| `snapshot.cgi` | `snapshot.cgi`, `stream.m3u8` |
