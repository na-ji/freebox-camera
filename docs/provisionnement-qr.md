<!-- generated-by: gsd-doc-writer -->
# Provisionnement par QR Code — Freebox F-HACAM01A

Ce guide explique comment connecter la caméra à un réseau WiFi sans passer par l'application
Freebox, en générant manuellement le QR Code de provisionnement.

---

## Principe

La caméra sort d'usine en mode « non appairée » (`paired=0`). Au démarrage, le binaire
`qr_code_action` capture des images en continu depuis `/dev/video0` et tente de décoder un
QR Code. Lorsqu'un QR valide est détecté, la caméra extrait les informations WiFi, se connecte
au réseau et bascule en mode opérationnel (`paired=1`).

Ce mécanisme est indépendant de l'application Freebox — n'importe quel QR Code au bon format
déclenche le provisionnement.

---

## Format du QR Code

Le QR Code contient une **chaîne de caractères ASCII**, construite selon ce schéma :

```
{ssid_len:02d}{ssid}{pwd_len:02d}{password}{hostname}
```

| Champ | Format | Description |
|-------|--------|-------------|
| `ssid_len` | 2 chiffres décimaux (zéro-padded) | Longueur du SSID |
| `ssid` | Chaîne ASCII | Nom du réseau WiFi |
| `pwd_len` | 2 chiffres décimaux (zéro-padded) | Longueur du mot de passe |
| `password` | Chaîne ASCII | Mot de passe WPA2 (PSK en clair) |
| `hostname` | Chaîne ASCII (5–10 chars) | Identifiant de la caméra |

### Exemple décomposé

Réseau : SSID = `MonRéseau` (10 caractères), mot de passe = `aBcD1234EfGh5678IjKl9012MnOp3456` (32 caractères), hostname = `cam01`.

```
10MonRéseau32aBcD1234EfGh5678IjKl9012MnOp3456cam01
│ │          │ │                                │
│ SSID       │ Mot de passe (32 chars)           Hostname
│            │
ssid_len=10  pwd_len=32
```

### Points importants

- **Aucun chiffrement.** Le mot de passe WiFi est transmis en clair dans le QR Code.
- La caméra utilise ce mot de passe directement comme PSK WPA2.
- Le `hostname` est enregistré dans la configuration système et annoncé via DHCP.
- La longueur du mot de passe peut aller jusqu'à 64 caractères (limite WPA2).

---

## Comment Free génère les QR Codes

Pour chaque caméra appairée, l'application Freebox :

1. Génère un PSK WPA2 **aléatoire de 32 caractères** spécifique à cette caméra.
2. Configure le routeur Freebox pour accepter ce PSK sur le SSID correspondant.
3. Génère un QR Code avec ce PSK et un hostname dérivé du numéro de série.
4. Affiche le QR Code à l'écran pour que l'utilisateur le présente à la caméra.

Ce fonctionnement explique pourquoi chaque caméra a un mot de passe WiFi différent dans le
contexte Freebox. **Sans Freebox**, il suffit d'utiliser le mot de passe WiFi de votre routeur.

---

## Flux de traitement (binaire qr_code_action)

1. Vérifie `paired=0` dans la configuration — si `paired=1`, le binaire quitte immédiatement.
2. Ouvre `/dev/video0` et capture des images en boucle.
3. Appelle `zbarimg` pour décoder chaque image.
4. Si un QR est détecté, parse la chaîne avec `sscanf "%2d"` pour les longueurs.
5. Écrit le mot de passe WiFi dans la configuration via `WLANWriteConfigData` (champ `wpa_ascii`).
6. Dérive le mot de passe d'administration HTTP : concatène `admin_name` (`"freeboxcam"`) et le PSK WiFi, calcule le MD5 des 16 octets résultants, l'encode en base64 et tronque à 8 caractères. Voir [Mot de passe HTTP après provisionnement QR](#mot-de-passe-http-après-provisionnement-par-qr-code).
7. Écrit le hostname dans la configuration système.
8. Positionne `paired=1` dans la configuration.
9. Lance le démon principal `hydra`.

---

## Génération du QR Code

### Via l'interface web (recommandé)

Ouvrez [`../index.html`](../index.html) dans votre navigateur. Renseignez :
- Le **SSID** de votre réseau WiFi
- Le **mot de passe WPA2** de votre routeur
- Le **nom de la caméra** (5–10 caractères alphanumériques)

Le QR Code est généré localement, sans envoi de données vers un serveur.

### Via un script Python

```python
import qrcode

def build_qr_payload(ssid: str, password: str, hostname: str) -> str:
    ssid_len = str(len(ssid)).zfill(2)
    pwd_len  = str(len(password)).zfill(2)
    return f"{ssid_len}{ssid}{pwd_len}{password}{hostname}"

ssid     = "MonRéseau"
password = "MonMotDePasse"
hostname = "cam01"

payload = build_qr_payload(ssid, password, hostname)
print(f"Payload : {payload}")

img = qrcode.make(payload, error_correction=qrcode.constants.ERROR_CORRECT_M)
img.save("freebox_cam_qr.png")
print("QR Code enregistré dans freebox_cam_qr.png")
```

Installation de la dépendance :

```bash
pip install qrcode[pil]
```

---

## Procédure de provisionnement sur un routeur non-Freebox

### Étapes

1. **Mettez la caméra sous tension.** Branchez le câble d'alimentation USB.

2. **Attendez la LED orange clignotante lente** — la caméra est en mode provisionnement.
   (Si la LED est verte, la caméra est déjà appairée. Effectuez un reset usine d'abord.)

3. **Générez le QR Code** avec votre SSID, mot de passe et un hostname court.

4. **Affichez le QR Code** sur un écran (smartphone, ordinateur) ou imprimez-le.

5. **Présentez le QR Code face à l'objectif**, à environ **20–40 cm**. Maintenez-le stable.
   La caméra scanne en continu — aucune action particulière n'est requise.

6. **Attendez la LED verte fixe** (généralement 10–30 secondes après détection du QR).

7. **Récupérez l'adresse IP** via votre routeur ou avec :
   ```bash
   nmap -sn 192.168.1.0/24 | grep cam01
   ```

8. **Testez la connexion** :
   ```bash
   curl http://<IP>/util/query.cgi
   ```

---

## Mot de passe HTTP après provisionnement par QR Code

Lors du provisionnement par QR Code, le binaire `qr_code_action` **dérive automatiquement** le
mot de passe d'administration HTTP à partir du PSK WiFi fourni. Ce mot de passe est différent de
celui qu'attribue l'application Freebox (qui génère un mot de passe aléatoire indépendant du PSK).

### Algorithme (reverse engineering du binaire qr_code_action)

```
admin_password = base64( MD5( "freeboxcam" + WiFi_PSK ) )[:8]
```

Étapes détaillées :

1. Concaténer la chaîne littérale `freeboxcam` (valeur fixe du champ `admin_name` dans le firmware)
   et les octets du PSK WiFi, tels qu'ils apparaissent dans le QR Code.
2. Calculer le condensat MD5 standard des 16 octets résultants.
3. Encoder ces 16 octets en base64 standard (RFC 4648) — produit une chaîne de ~24 caractères.
4. Ne conserver que les **8 premiers caractères** : c'est le mot de passe HTTP/RTSP.

### Calculer le mot de passe pour un PSK connu

**Python :**

```python
import hashlib, base64

def cam_password(wifi_psk: str) -> str:
    data = b"freeboxcam" + wifi_psk.encode()
    return base64.b64encode(hashlib.md5(data).digest()).decode()[:8]

print(cam_password("MonMotDePasse"))  # → '6zt2Rp8/'
```

**Shell (Linux) :**

```bash
PSK="MonMotDePasse"
printf "freeboxcam%s" "$PSK" | md5sum | cut -d' ' -f1 | xxd -r -p | base64 | cut -c1-8
```

**Shell (macOS) :**

```bash
PSK="MonMotDePasse"
printf "freeboxcam%s" "$PSK" | md5 | xxd -r -p | base64 | cut -c1-8
```

### Remarques

- Cette dérivation s'applique **uniquement au provisionnement QR** (sans application Freebox).
  Quand l'application Freebox est utilisée, elle envoie un mot de passe généré aléatoirement —
  indépendant du PSK WiFi.
- Le mot de passe peut contenir `/` ou `+` (caractères base64) — certains logiciels nécessitent
  d'encoder ces caractères dans les URLs.
- Pour changer le mot de passe après provisionnement :
  ```bash
  curl -u freeboxcam:<MOT_DE_PASSE_ACTUEL> \
    "http://<IP>/adm/set_group.cgi?group=USER&admin_password=NouveauMotDePasse"
  ```

---

## Comportement de la LED pendant le provisionnement

| Phase | Couleur / comportement |
|-------|------------------------|
| Démarrage | Rouge fixe (quelques secondes) |
| En attente QR Code | Orange clignotant lent (~1 Hz) |
| QR détecté, connexion WiFi | Orange clignotant rapide (~4 Hz) |
| WiFi connecté, démarrage services | Orange fixe |
| Opérationnel | Vert fixe |
| Erreur / échec WiFi | Rouge fixe |

---

## Re-provisionnement (changement de réseau)

Pour connecter la caméra à un nouveau réseau WiFi, il faut d'abord remettre `paired=0`.

### Via l'API (caméra accessible sur le réseau actuel)

```bash
# Reset complet aux paramètres d'usine
curl -u freeboxcam:<MOT_DE_PASSE> http://<IP>/adm/reset_to_default.cgi
```

### Via le bouton physique

Maintenez le bouton reset (généralement visible sur la base) enfoncé pendant **10 secondes**.
La caméra redémarre en mode provisionnement.

### Via le shell root

```bash
# Modifier directement la configuration
# (Dans le shell root, après exploit test.cgi ou UART)
/usr/sbin/cfgmgr set paired 0
reboot
```

---

## Dépannage

### La caméra ne détecte pas le QR Code

- **Distance** : essayez entre 15 et 50 cm. Trop proche ou trop loin, le décodage échoue.
- **Luminosité** : augmentez la luminosité de votre écran au maximum.
- **Résolution d'affichage** : si le QR est affiché en très petit, zoomez ou agrandissez.
- **Réflexions** : évitez les reflets sur l'écran (lumière de face).
- **Mode nuit** : si la caméra est dans un endroit sombre, ses LED IR s'activent — cela
  peut saturer le capteur. Éclairez légèrement la pièce.
- **QR trop complexe** : si le mot de passe ou le SSID contient des caractères non-ASCII,
  essayez d'abord avec un mot de passe simple pour diagnostiquer.

### La caméra détecte le QR mais ne se connecte pas au WiFi

- Vérifiez que le SSID et le mot de passe sont **exactement** corrects (sensible à la casse).
- Vérifiez que votre point d'accès émet bien en **2.4 GHz** — la caméra ne supporte pas le 5 GHz.
- Vérifiez que le réseau utilise le chiffrement **WPA2** (WPA3 non supporté).
- Vérifiez que le SSID n'est pas masqué (hidden SSID) — si c'est le cas, testez en le rendant visible temporairement.

### Les credentials HTTP ne fonctionnent pas après provisionnement

Le nom d'utilisateur est toujours `freeboxcam`. Le mot de passe dépend de la méthode de
provisionnement utilisée :

- **Provisionnement par QR Code** (ce projet) : le mot de passe est **calculable** à partir du
  PSK WiFi — voir [Mot de passe HTTP après provisionnement par QR Code](#mot-de-passe-http-après-provisionnement-par-qr-code).
  Utilisez le script Python ou la commande shell de cette section avec le mot de passe WiFi que
  vous avez saisi dans le générateur.

- **Provisionnement via l'application Freebox** : le mot de passe est généré aléatoirement par
  l'application, indépendamment du PSK WiFi. Retrouvez-le dans les paramètres de l'application
  Freebox (section caméra).

Si vous l'avez modifié via `/adm/set_group.cgi?group=USER`, utilisez le nouveau mot de passe.

### La caméra est déjà appairée (LED verte au démarrage)

Le binaire `qr_code_action` ne s'exécute que si `paired=0`. Effectuez un reset usine
(bouton physique 10 secondes ou `reset_to_default.cgi`).
