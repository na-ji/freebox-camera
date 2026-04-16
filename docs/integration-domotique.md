<!-- generated-by: gsd-doc-writer -->
# Intégration domotique — Freebox F-HACAM01A

Ce guide couvre l'utilisation de la caméra avec les principaux logiciels et systèmes domotiques :
VLC, Home Assistant, Synology Surveillance Station, Blue Iris, Jeedom, ainsi que la configuration
des alertes de mouvement et l'interface ONVIF.

---

## URLs des flux vidéo

Remplacez `<IP>` par l'adresse IP de votre caméra dans toutes les URLs ci-dessous.

| Type | URL | Description |
|------|-----|-------------|
| RTSP HD | `rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/<sp_uri>` | 720p (1280×720), H.264 + AAC |
| RTSP SD | `rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/<sp_uri2>` | 360p (640×360), H.264 |
| HLS | `http://freeboxcam:<MOT_DE_PASSE>@<IP>/img/stream.m3u8` | Flux HLS (segments TS) |
| Snapshot JPEG | `http://freeboxcam:<MOT_DE_PASSE>@<IP>/img/snapshot.cgi` | Image fixe à la demande |

### Configuration préalable des URIs RTSP

Les chemins RTSP (`sp_uri`, `sp_uri2`) sont des paramètres configurables dans le groupe `H264`.
Leur valeur par défaut est **vide** après une remise à zéro. Si VLC retourne une erreur 404,
vérifiez et définissez les URIs :

```bash
# Lire la configuration actuelle
curl -u freeboxcam:<MOT_DE_PASSE> "http://<IP>/adm/get_group.cgi?group=H264"

# Définir les URIs (exemple avec live.sdp / live2.sdp)
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/set_group.cgi?group=H264&sp_uri=live.sdp&sp_uri2=live2.sdp"
```

Vous pouvez choisir n'importe quel nom (ex. `cam.sdp`, `video.sdp`).
Une fois défini, l'URL devient `rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/live.sdp`.

### Paramètres du flux RTSP

- **Codec vidéo :** H.264 (Baseline Profile)
- **Résolution HD :** 1280×720 @ 15–25 fps (configurable)
- **Codec audio :** AAC
- **Authentification RTSP :** Basic (dans l'URL ou en-tête)
- **Texte en incrustation (OSD) :** configurable — voir [Configurer le texte affiché sur le flux vidéo](endpoints-api.md#exemple--configurer-le-texte-affiché-sur-le-flux-vidéo-osd)

---

## VLC

### Ouvrir un flux RTSP

```bash
vlc "rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/live.sdp"
```

Ou depuis l'interface graphique : **Média → Ouvrir un flux réseau** → coller l'URL.

### Enregistrement continu avec ffmpeg

```bash
ffmpeg -i "rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/live.sdp" \
  -c copy \
  -f segment \
  -segment_time 3600 \
  -segment_format mp4 \
  -strftime 1 \
  "enregistrement_%Y%m%d_%H%M%S.mp4"
```

### Snapshot périodique avec ffmpeg

```bash
# Un snapshot toutes les 5 minutes
while true; do
  ffmpeg -i "rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/live.sdp" \
    -frames:v 1 \
    "snap_$(date +%Y%m%d_%H%M%S).jpg" -y
  sleep 300
done
```

---

## Home Assistant

### Caméra générique (flux MJPEG / RTSP via go2rtc ou ffmpeg)

Ajoutez dans `configuration.yaml` :

```yaml
camera:
  - platform: generic
    name: "Freebox Cam"
    still_image_url: "http://freeboxcam:<MOT_DE_PASSE>@<IP>/img/snapshot.cgi"
    stream_source: "rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/live.sdp"
    username: freeboxcam
    password: <MOT_DE_PASSE>
    verify_ssl: false
```

### Via go2rtc (recommandé pour Home Assistant OS/Supervised)

go2rtc est intégré dans Home Assistant depuis la version 2023.x. Ajoutez dans
`/homeassistant/go2rtc.yaml` :

```yaml
streams:
  freebox_cam:
    - rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/live.sdp
  freebox_cam_sd:
    - rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/live2.sdp
```

Puis dans `configuration.yaml` :

```yaml
camera:
  - platform: generic
    name: "Freebox Cam"
    still_image_url: "http://freeboxcam:<MOT_DE_PASSE>@<IP>/img/snapshot.cgi"
    stream_source: "rtsp://localhost:8554/freebox_cam"
```

### Intégration ONVIF

Home Assistant dispose d'une intégration ONVIF native. Depuis l'interface :

1. **Paramètres → Appareils et services → Ajouter une intégration → ONVIF**
2. Renseigner :
   - **Hôte :** adresse IP de la caméra
   - **Port :** `80`
   - **Nom d'utilisateur :** `freeboxcam`
   - **Mot de passe :** `<MOT_DE_PASSE>`

> **Note :** L'intégration ONVIF de Home Assistant utilise le profil ONVIF S. La caméra expose
> `GetCapabilities`, `GetProfiles`, `GetStreamUri` et les méthodes de détection de mouvement
> (si activée). La méthode `GetSystemDateAndTime` ne requiert pas d'authentification.

### Détection de mouvement via webhook

Configurez la caméra pour envoyer une requête HTTP à Home Assistant lors d'une détection de
mouvement :

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/set_group.cgi?group=HTTP_EVENT\
&enable=1\
&url=http://<IP_HA>:8123/api/webhook/freebox_cam_mouvement\
&http_event_type=motion"
```

Dans Home Assistant, créez une automatisation déclenchée par le webhook :

```yaml
automation:
  - alias: "Mouvement caméra Freebox"
    trigger:
      - platform: webhook
        webhook_id: freebox_cam_mouvement
    action:
      - service: notify.mobile_app_mon_telephone
        data:
          message: "Mouvement détecté par la caméra !"
```

---

## Synology Surveillance Station

1. Ouvrez Surveillance Station → **Caméra IP → Ajouter**.
2. Choisissez **Sercomm** dans la liste des fabricants, ou utilisez **Configuration manuelle**.
3. Renseignez :
   - **Adresse IP :** adresse de la caméra
   - **Port :** `80`
   - **Protocole :** `HTTP`
   - **Nom d'utilisateur :** `freeboxcam`
   - **Mot de passe :** `<MOT_DE_PASSE>`
4. En configuration manuelle, entrez les URLs des flux :
   - **Flux principal :** `rtsp://<IP>:554/live.sdp`
   - **Flux secondaire :** `rtsp://<IP>:554/live2.sdp`
   - **Snapshot :** `http://<IP>/img/snapshot.cgi`
5. Authentification snapshot : cochez **Utiliser des informations d'identification différentes** si nécessaire.

---

## Blue Iris

1. Créez une nouvelle caméra : **Caméra → Ajouter/Modifier**.
2. Onglet **Vidéo** :
   - **Type :** `Network IP camera`
   - **Adresse :** adresse IP de la caméra
   - **Port :** `554` (RTSP)
3. Onglet **Général** :
   - **Profil de caméra :** sélectionnez `Sercomm` ou configurez manuellement
   - **Nom d'utilisateur :** `freeboxcam`
   - **Mot de passe :** `<MOT_DE_PASSE>`
4. Configuration manuelle des URLs :
   - **Flux principal :** `rtsp://<IP>:554/live.sdp`
   - **Flux sous-flux :** `rtsp://<IP>:554/live2.sdp`
   - **Snapshot :** `http://freeboxcam:<MOT_DE_PASSE>@<IP>/img/snapshot.cgi`

---

## Jeedom

### Plugin Caméra (plugin officiel)

1. Installez le plugin **Caméra** depuis le market Jeedom.
2. Créez un nouvel équipement caméra.
3. Configurez les URLs :
   - **URL du flux :** `rtsp://freeboxcam:<MOT_DE_PASSE>@<IP>:554/live.sdp`
   - **URL snapshot :** `http://freeboxcam:<MOT_DE_PASSE>@<IP>/img/snapshot.cgi`
4. Configurez la commande de snapshot avec authentification Basic si nécessaire.

### Plugin Sercomm

Si vous disposez d'un plugin Sercomm dans le community Jeedom, renseignez :
- **IP :** adresse de la caméra
- **Login :** `freeboxcam`
- **Mot de passe :** `<MOT_DE_PASSE>`
- **Type :** RC8310A ou équivalent

---

## Détection de mouvement — Configuration via API

La détection de mouvement est configurable directement via l'API.

### Activer la détection

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/set_group.cgi?group=MOTION\
&enable=1\
&sensitivity=5\
&threshold=20"
```

| Paramètre | Valeur | Description |
|-----------|--------|-------------|
| `enable` | `0` ou `1` | Active/désactive la détection |
| `sensitivity` | `1`–`10` | Sensibilité (10 = très sensible) |
| `threshold` | `0`–`100` | Seuil de déclenchement |

### Notifications HTTP sur mouvement

```bash
# Configurer une URL de notification
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/set_group.cgi?group=HTTP_EVENT\
&enable=1\
&url=http://192.168.1.10:8123/api/webhook/cam_motion\
&http_event_type=motion"
```

### Notification par e-mail

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/set_group.cgi?group=EMAIL\
&enable=1\
&smtp_server=smtp.example.com\
&smtp_port=587\
&from=cam@example.com\
&to=vous@example.com\
&tls_enable=1"
```

### Envoi FTP des snapshots sur mouvement

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  "http://<IP>/adm/set_group.cgi?group=FTP\
&ftp_server=192.168.1.10\
&ftp_port=21\
&ftp_account=ftpuser\
&ftp_passwd=ftppass\
&ftp_path=/snapshots"
```

---

## Interface ONVIF

La caméra implémente **ONVIF Profile S** sur le port 80.

| Méthode ONVIF | Authentification | Description |
|---------------|-----------------|-------------|
| `GetSystemDateAndTime` | Non requise | Date et heure système |
| `GetCapabilities` | WS-Security PasswordDigest | Capacités de l'appareil |
| `GetProfiles` | WS-Security PasswordDigest | Profils vidéo disponibles |
| `GetStreamUri` | WS-Security PasswordDigest | URL du flux RTSP |
| `GetSnapshotUri` | WS-Security PasswordDigest | URL du snapshot |

### Exemple : GetStreamUri via ONVIF

```bash
curl -s -X POST http://<IP>/onvif/device_service \
  -H "Content-Type: application/soap+xml; charset=utf-8" \
  -u freeboxcam:<MOT_DE_PASSE> \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope
  xmlns:soap="http://www.w3.org/2003/05/soap-envelope"
  xmlns:wsdl="http://www.onvif.org/ver10/media/wsdl"
  xmlns:sch="http://www.onvif.org/ver10/schema">
  <soap:Body>
    <wsdl:GetStreamUri>
      <wsdl:StreamSetup>
        <sch:Stream>RTP-Unicast</sch:Stream>
        <sch:Transport><sch:Protocol>RTSP</sch:Protocol></sch:Transport>
      </wsdl:StreamSetup>
      <wsdl:ProfileToken>Profile_1</wsdl:ProfileToken>
    </wsdl:GetStreamUri>
  </soap:Body>
</soap:Envelope>'
```

---

## Automatisation — Snapshot programmé

Exemple de script bash pour capturer un snapshot toutes les heures et l'archiver :

```bash
#!/bin/bash
CAM_IP="192.168.1.42"
DEST_DIR="/var/archives/cam"
mkdir -p "$DEST_DIR"

DATE=$(date +%Y%m%d_%H%M%S)
curl -s -u freeboxcam:<MOT_DE_PASSE> \
  "http://${CAM_IP}/img/snapshot.cgi" \
  -o "${DEST_DIR}/snap_${DATE}.jpg"
```

Ajout dans crontab (`crontab -e`) :

```
0 * * * * /usr/local/bin/snap_cam.sh
```
