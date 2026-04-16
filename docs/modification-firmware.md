# Modification du firmware — Contournement des vérifications

Ce document explique les vérifications effectuées par `fileupload.cgi` lors d'un upload de
firmware, et comment les satisfaire pour flasher un firmware modifié via l'interface web.

---

## Contexte

Le firmware de la F-HACAM01A est une image brute de 16 Mo correspondant au contenu exact de la
flash SPI. L'upload se fait via `POST /adm/upgrade.cgi`, qui délègue la vérification au binaire
`fileupload.cgi` (42 Ko, Thumb-2 ARM).

**Bonne nouvelle** : les vérifications sont entièrement contournables sans matériel physique,
à condition de partir d'un dump de votre propre caméra.

---

## Les trois vérifications (rétro-ingénierie de fileupload.cgi)

### Vérification 1 — Signature matérielle (`hw_id`)

**Localisation dans le firmware :** offset `0xFFFF8F` (−113 octets depuis la fin).

`fileupload.cgi` lit **70 octets** à cet offset dans le firmware uploadé, puis lit les mêmes
70 octets depuis la flash en cours d'utilisation, et compare :

- **Octets `[0x0b:0x2b]`** (32 octets) : identifiant matériel `"CVP"` + padding — doit correspondre
- **Octets `[0x2f:0x31]`** (2 octets) : flags OEM — doivent correspondre

**Comment passer cette vérification :** ne pas modifier cette zone. Puisque vous construisez
votre firmware à partir d'un dump de la même caméra, la signature est identique par définition.

```python
# Vérification préalable : s'assurer que l'octet 0x0b dans la zone de signature contient "CVP"
with open('fw_backup.bin', 'rb') as f:
    f.seek(0xFFFF8F + 0x0b)
    hw_id = f.read(3)
    assert hw_id == b'CVP', f"hw_id inattendu : {hw_id}"
    print(f"hw_id OK : {hw_id}")
```

> **Important :** ne jamais copier la signature d'un dump d'une autre caméra — les flags OEM
> pourraient différer et la vérification échouerait.

---

### Vérification 2 — Type de firmware

`fileupload.cgi` lit un bloc de **32 octets** puis un bloc de **128 octets** en début de fichier
pour identifier le format (MCR2, raw image, etc.) et dispatcher vers la fonction de vérification
correspondante.

L'image brute de 16 Mo issue du dump (`/adm/flash_dumper.cgi` ou `dd if=/dev/mtd9`) est déjà au
bon format. Cette vérification passe automatiquement si l'on part du dump original.

> Ne pas envelopper le firmware dans un conteneur MCR2 ou tout autre format — uploader l'image
> brute directement.

---

### Vérification 3 — Checksum MD5

C'est la seule vérification qui nécessite une action active après modification du firmware.

**Algorithme :**
```
md5#2 = MD5( image[0x100000 .. 0xFEFFFF] )
```

- **Région couverte :** de l'offset `0x100000` (début du noyau) à `0xFEFFFF` inclus
  — soit **15 728 640 octets** (exactement `0xEF0000` octets, ~15 Mo)
- **Emplacement du hash :** les 16 octets à l'offset `0xFFFFE0`

`fileupload.cgi` recalcule ce MD5 sur le fichier uploadé et le compare à la valeur stockée à
`0xFFFFE0`. Si les deux ne correspondent pas → rejet avec `"Upgrade file checksum error"`.

---

## Script de construction du firmware modifié

Le script suivant prend un dump original, remplace le SquashFS, et recalcule le MD5 correctement.

```python
#!/usr/bin/env python3
"""
build_firmware.py — Reconstruit une image flash 16 Mo avec un SquashFS modifié.

Usage :
    python3 build_firmware.py fw_backup.bin rootfs_new.sqfs fw_output.bin
"""
import sys
import hashlib

FLASH_SIZE        = 0x1000000   # 16 Mo
KERNEL_OFFSET     = 0x100000    # début du noyau (et de la région MD5)
ROOTFS_OFFSET     = 0x400000    # début du SquashFS
HASH_START        = 0x100000    # début région couverte par md5#2
HASH_END          = 0xFF0000    # fin exclusive (0xFEFFFF inclus)
MD5_OFFSET        = 0xFFFFE0    # où écrire le hash

def build(backup_path, sqfs_path, output_path):
    print(f"Lecture du dump original : {backup_path}")
    with open(backup_path, 'rb') as f:
        image = bytearray(f.read())

    assert len(image) == FLASH_SIZE, \
        f"Taille inattendue : {len(image)} octets (attendu {FLASH_SIZE})"

    print(f"Injection du SquashFS depuis : {sqfs_path}")
    with open(sqfs_path, 'rb') as f:
        sqfs = f.read()

    max_sqfs_size = HASH_END - ROOTFS_OFFSET   # ~12,5 Mo
    if len(sqfs) > max_sqfs_size:
        raise ValueError(
            f"SquashFS trop grand : {len(sqfs)} octets "
            f"(max {max_sqfs_size} = {max_sqfs_size/1024/1024:.1f} Mo)"
        )

    image[ROOTFS_OFFSET : ROOTFS_OFFSET + len(sqfs)] = sqfs

    # Remplir le reste de la zone SquashFS avec 0xFF (état flash vierge)
    end = ROOTFS_OFFSET + len(sqfs)
    image[end : HASH_END] = b'\xff' * (HASH_END - end)

    print("Recalcul du checksum MD5...")
    region = bytes(image[HASH_START:HASH_END])
    md5 = hashlib.md5(region).digest()
    print(f"  md5#2 = {md5.hex()}")

    image[MD5_OFFSET : MD5_OFFSET + 16] = md5

    print(f"Écriture de l'image finale : {output_path}")
    with open(output_path, 'wb') as f:
        f.write(image)

    # Vérification croisée
    with open(output_path, 'rb') as f:
        f.seek(MD5_OFFSET)
        stored = f.read(16)
    assert stored == md5, "ERREUR : le hash écrit ne correspond pas !"
    print("Vérification OK — firmware prêt à uploader.")

if __name__ == '__main__':
    if len(sys.argv) != 4:
        print("Usage : python3 build_firmware.py <dump> <squashfs> <output>")
        sys.exit(1)
    build(sys.argv[1], sys.argv[2], sys.argv[3])
```

---

## Procédure complète pas à pas

### Étape 1 — Dump du firmware original

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  http://<IP>/adm/flash_dumper.cgi \
  -o fw_backup.bin \
  --progress-bar

# Vérification de taille
wc -c fw_backup.bin   # doit afficher 16777216
```

### Étape 2 — Extraction du SquashFS

```bash
dd if=fw_backup.bin bs=1 skip=$((0x400000)) of=rootfs.sqfs
file rootfs.sqfs
# → Squash filesystem, little endian, version 4.0, xz compressed...

unsquashfs rootfs.sqfs
# Contenu dans ./squashfs-root/
```

### Étape 3 — Modifications

Exemple : activer telnetd au démarrage (dans `squashfs-root/etc/init.d/Snormal`)

```bash
# Ajouter avant la dernière ligne "echo normal boot up down..."
echo '/usr/sbin/telnetd -l /bin/sh &' >> squashfs-root/etc/init.d/Snormal
```

Autres modifications possibles :
- Remplacer `/etc/passwd` pour fixer le mot de passe root
- Ajouter des binaires dans `/usr/local/bin/`
- Modifier les scripts d'initialisation dans `/etc/init.d/`
- Patcher les binaires (`hydra`, `fileupload.cgi`, etc.)

> La partition SquashFS fait au maximum **12 534 784 octets** (~12,5 Mo).
> `mksquashfs` échoue si le contenu dépasse cette limite.

### Étape 4 — Recompilation du SquashFS

```bash
mksquashfs squashfs-root rootfs_new.sqfs \
  -comp xz \
  -noappend \
  -b 131072

ls -la rootfs_new.sqfs   # vérifier la taille
```

> Utiliser **obligatoirement** `-comp xz` et `-b 131072` pour reproduire le format original.
> Une compression différente serait reconnue par le noyau mais pourrait déclencher des
> vérifications supplémentaires dans `fileupload.cgi`.

### Étape 5 — Reconstruction de l'image

```bash
python3 build_firmware.py fw_backup.bin rootfs_new.sqfs fw_patched.bin
# → md5#2 = <hash calculé>
# → Vérification OK — firmware prêt à uploader.
```

### Étape 6 — Upload

```bash
curl -u freeboxcam:<MOT_DE_PASSE> \
  -F "file=@fw_patched.bin;type=application/octet-stream" \
  http://<IP>/adm/upgrade.cgi \
  --progress-bar
```

La caméra vérifie le firmware (quelques secondes), puis flashe et redémarre automatiquement.
Le processus de flash dure environ 2 minutes. **Ne pas couper l'alimentation pendant cette phase.**

---

## Ce qui peut et ne peut pas être modifié

| Zone | Offset | Modifiable ? | Notes |
|------|--------|-------------|-------|
| Bootloader | `0x000000` | ⚠️ Risqué | Un bootloader corrompu = brick physique |
| Partition MAC (`mtd1`) | `0x090000` | Non recommandé | Contient le numéro de série |
| Noyau Linux | `0x100000` | Oui (avec soin) | Couvert par le MD5 — recalculer |
| **SquashFS (rootfs)** | **`0x400000`** | **Oui ✓** | Zone principale à modifier |
| Signature Sercomm | `0xFFFF8F` | **Non** | Doit rester identique à la flash actuelle |
| Checksum MD5 | `0xFFFFE0` | **Oui (obligatoire)** | À recalculer après chaque modification |

---

## Messages d'erreur courants

| Message | Cause | Solution |
|---------|-------|----------|
| `Upgrade file checksum error` | MD5 à `0xFFFFE0` incorrect | Recalculer avec `build_firmware.py` |
| `hw_id mismatch` | Signature `0xFFFF8F` ne correspond pas | Partir du dump de **votre** caméra |
| `File too large` | Image > 16 Mo | Réduire le contenu du SquashFS |
| Pas de réponse après upload | Flash en cours | Attendre 2-3 min avant de reconnecter |
| Caméra ne démarre plus | Firmware invalide | Accès UART pour reflash via U-Boot |

---

## Récupération en cas de brick

Si la caméra ne démarre plus après un flash raté, la seule solution est l'**accès UART**.

Voir [Accès UART](identification-materiel.md#uart) pour le câblage, puis dans U-Boot :

```
# Interrompre U-Boot (touche Entrée au boot)
setenv serverip <IP_TFTP>
setenv ipaddr 192.168.1.100
tftpboot 0x80000000 fw_backup.bin
sf probe 0
sf erase 0 0x1000000
sf write 0x80000000 0 0x1000000
reset
```

> Avoir un serveur TFTP (`dnsmasq --enable-tftp` ou `atftpd`) prêt avec `fw_backup.bin`
> avant toute intervention.
