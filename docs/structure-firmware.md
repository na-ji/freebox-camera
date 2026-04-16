<!-- generated-by: gsd-doc-writer -->
# Structure du firmware — Freebox F-HACAM01A

Ce document décrit l'organisation de la mémoire flash de la caméra : partitions MTD, layout
de la flash 16 Mo, contenu du système de fichiers SquashFS, ainsi que les procédures de dump
et de re-flash.

---

## Flash et SoC

| Paramètre | Valeur |
|-----------|--------|
| Capacité flash | 16 Mo (SPI NOR) |
| SoC | ARM (armv5tejl) |
| Noyau | Linux 3.2.7 |
| Espace utilisateur | BusyBox + uClibc |
| Interface | MTD (Memory Technology Device) |

---

## Partitions MTD

Les partitions sont accessibles depuis `/proc/mtd` sur la caméra (après obtention d'un shell root).

| Partition | Nom | Taille | Description |
|-----------|-----|--------|-------------|
| mtd0 | `boot` | 576 Ko | Bootloader (U-Boot ou équivalent) |
| mtd1 | `MAC` | 64 Ko | Adresse MAC, numéro de série |
| mtd2 | `calibration` | 64 Ko | Données de calibration RF WiFi |
| mtd3 | `configuration` | 64 Ko | Configuration chiffrée active |
| mtd4 | `bak_configuration` | 64 Ko | Sauvegarde de la configuration |
| mtd5 | `reserve` | 64 Ko | Réservé |
| mtd6 | `kernel` | 3 Mo | Image noyau Linux (uImage ARM) |
| mtd7 | `root_fs` | 12 Mo | Système de fichiers SquashFS 4.0 (XZ) |
| mtd8 | `product_info` | 64 Ko | Informations produit OEM |
| mtd9 | `All` | 16 Mo | Flash complète (dump/flash entier) |
| mtd10 | `protect` | 1 Mo | Zone protégée |
| mtd11 | `boot_image` | 15 Mo | Image de boot complète |
| mtd12 | `upgrade_image` | 15 Mo | Image de mise à jour |

### Accès aux partitions (shell root)

```bash
# Lire la partition de configuration
dd if=/dev/mtd3 of=/tmp/config.bin

# Lire l'image noyau
dd if=/dev/mtd6 of=/tmp/kernel.bin

# Lire le système de fichiers root
dd if=/dev/mtd7 of=/tmp/rootfs.bin

# Dump complet de la flash (équivalent à flash_dumper.cgi)
dd if=/dev/mtd9 of=/tmp/flash_full.bin
```

---

## Layout de la flash (image fw_backup.bin, 16 Mo)

| Offset | Contenu |
|--------|---------|
| `0x000000` | En-tête MCR2 + bootloader |
| `0x100000` | Image noyau Linux (uImage ARM, format U-Boot) |
| `0x400000` | SquashFS v4.0 (compression XZ) |
| `0xFFFF8F` | Signature Sercomm : hardware ID `CVP`, version FW `0x4` (70 octets) |
| `0xFFFFE0` | Checksum MD5 n°2 = MD5(`image[0x100000..0xFEFFFF]`) |

### Structure de l'en-tête MCR2

L'en-tête débute à l'offset `0x000000` et identifie le modèle matériel. Le champ `hw_id` contient
la chaîne `CVP` qui est vérifiée lors de l'upload via `fileupload.cgi`.

### Signature Sercomm (offset 0xFFFF8F)

Les 70 octets à cet offset contiennent :
- Le `hw_id` (hardware ID) : `CVP`
- La version firmware : `0x4`
- Des flags OEM qui permettent au vérificateur de valider la compatibilité

Ces valeurs doivent être **conservées à l'identique** lors d'une modification du firmware.

### Checksum MD5 (offset 0xFFFFE0)

Le checksum couvre la plage `[0x100000 .. 0xFEFFFF]` (noyau + SquashFS, hors bootloader et hors
les 16 derniers secteurs contenant la signature et le checksum lui-même).

```python
import hashlib

with open('fw_backup.bin', 'rb') as f:
    f.seek(0x100000)
    data = f.read(0xFEFFFF - 0x100000)

md5 = hashlib.md5(data).digest()
print(md5.hex())
```

---

## Système de fichiers SquashFS

| Paramètre | Valeur |
|-----------|--------|
| Format | SquashFS 4.0 |
| Compression | XZ |
| Offset dans la flash | `0x400000` |
| Fichiers | 490 |
| Répertoires | 99 |
| Inodes | 984 |

### Extraction

```bash
# Extraire le SquashFS depuis le dump flash
dd if=fw_backup.bin bs=1 skip=$((0x400000)) of=rootfs.sqfs

# Monter ou extraire (nécessite unsquashfs)
unsquashfs rootfs.sqfs
```

### Binaires et scripts clés

| Chemin | Description |
|--------|-------------|
| `/bin/busybox` | BusyBox (shell, utilitaires système) |
| `/usr/sbin/hydra` | Démon principal de la caméra (non lié au craquage de mots de passe) |
| `/usr/sbin/telnetd` | Serveur Telnet (utilisé par l'exploit root) |
| `/usr/bin/zbarimg` | Décodeur de QR Codes |
| `/usr/sbin/qr_code_action` | Binaire de provisionnement QR |
| `/usr/lib/libcgicomm.so` | Bibliothèque CGI partagée (contient la table id2pwd à offset `0x26518`) |
| `/etc/init.d/rcS` | Script d'init principal |
| `/etc/stunnel/stunnel.pem` | Clé privée TLS partagée entre tous les appareils Sercomm |
| `/usr/sbin/fileupload.cgi` | Vérificateur de firmware (42 Ko) |
| `/usr/sbin/auto_upgrade` | Mise à jour automatique firmware |

### Répertoires principaux

```
/bin/          Utilitaires BusyBox
/usr/sbin/     Démons et CGIs
/usr/lib/      Bibliothèques partagées (.so)
/etc/          Configuration système
/etc/init.d/   Scripts d'initialisation
/etc/stunnel/  Certificats et clé TLS
/var/          Données variables (lien symbolique vers /tmp en RAM)
/tmp/          Système de fichiers tmpfs (données volatiles)
/proc/         Système de fichiers proc (info noyau)
/dev/          Périphériques
/dev/video0    Périphérique capture vidéo
```

### Version noyau et date de compilation

```
Linux version 3.2.7
Architecture : ARM (armv5tejl)
Libc : uClibc
```

---

## Dump du firmware

### Via l'API HTTP (recommandé)

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  http://<IP>/adm/flash_dumper.cgi \
  -o fw_backup.bin \
  --progress-bar
```

L'opération prend plusieurs minutes (transfert de 16 Mo via WiFi). Le fichier résultant est
l'image brute de la flash entière.

### Via le shell root (plus fiable)

Après avoir obtenu un shell root (voir [Accès root](acces-root.md)) :

```bash
# Dans le shell root de la caméra
dd if=/dev/mtd9 bs=4096 | nc <IP_HOTE> 9999

# Sur votre machine
nc -l 9999 > fw_backup.bin
```

---

## Modification et re-flash

### Procédure de modification du SquashFS

1. **Faire un dump** du firmware actuel :

```bash
curl -u freeboxcam:<MOT_DE_PASSE> http://<IP>/adm/flash_dumper.cgi -o fw_backup.bin
```

2. **Extraire le SquashFS** :

```bash
dd if=fw_backup.bin bs=1 skip=$((0x400000)) of=rootfs.sqfs
unsquashfs rootfs.sqfs
# Le contenu est dans ./squashfs-root/
```

3. **Modifier les fichiers** dans `squashfs-root/`.

4. **Recréer le SquashFS** (en préservant le format exact) :

```bash
mksquashfs squashfs-root rootfs_new.sqfs \
  -comp xz -noappend -b 131072
```

5. **Réinjecter dans l'image flash** :

```python
import shutil

shutil.copy('fw_backup.bin', 'fw_patched.bin')

with open('rootfs_new.sqfs', 'rb') as f:
    sqfs_data = f.read()

with open('fw_patched.bin', 'r+b') as f:
    f.seek(0x400000)
    f.write(sqfs_data)
```

6. **Recalculer le checksum MD5** :

```python
import hashlib, struct

with open('fw_patched.bin', 'r+b') as f:
    f.seek(0x100000)
    data = f.read(0xFEFFFF - 0x100000)
    md5 = hashlib.md5(data).digest()
    # Écrire le MD5 à l'offset 0xFFFFE0
    f.seek(0xFFFFE0)
    f.write(md5)

print(f"Nouveau MD5 : {md5.hex()}")
```

7. **Uploader le firmware patché** :

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  -F "file=@fw_patched.bin" \
  http://<IP>/adm/upgrade.cgi
```

> **Attention :** La caméra redémarre automatiquement après l'upload. Ne coupez pas l'alimentation
> pendant l'opération. Un firmware invalide (mauvais hw_id, MD5 incorrect) sera rejeté par
> `fileupload.cgi` avant flashage.

### Vérifications effectuées par fileupload.cgi

Le binaire `fileupload.cgi` (42 Ko) effectue les contrôles suivants avant d'accepter un firmware :

1. **Vérification de la signature matérielle** : lit 70 octets à l'offset `0xFFFF8F`, vérifie que
   `hw_id == "CVP"` et que les flags OEM correspondent.
2. **Détection du type de firmware** : analyse les blocs de 32 et 128 octets en début de fichier
   pour déterminer le format (MCR2, raw, etc.).
3. **Vérification MD5** : calcule MD5(`image[0x100000..0xFEFFFF]`) et compare avec la valeur
   stockée à l'offset `0xFFFFE0`.

Si l'une de ces vérifications échoue, l'upload est rejeté et la caméra reste sur le firmware actuel.
