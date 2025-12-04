# Phase 1 - Semaine 2 : RAID & LVM (Storage)

## Ce qu'on va faire cette semaine

Construire un système de stockage ROBUSTE.

**Analogie** : La semaine passée, tu as sécurisé la porte (SSH). Cette semaine, tu construis une chambre-forte pour tes données.

---

## Pourquoi RAID + LVM ?

### Sans RAID + LVM (Dangereux en production)

```
Un disque
    ↓
Données dedans
    ↓
Disque meurt
    ↓
TOUT EST PERDU ❌
```

### Avec RAID + LVM (Sécurisé)

```
2 disques en miroir
    ↓
RAID (données dupliquées)
    ↓
Un disque meurt
    ↓
Autres disques continuent
    ↓
Zéro perte de données ✅
```

**Analogie** : C'est comme faire une copie d'un contrat important :
- Sans copie : le contrat est perdu si le fichier brûle
- Avec copie : tu as toujours une sauvegarde

---

## Jour 1-2 : Préparer les Disques

### Étape 1 : Voir l'état des disques

```bash
# Affiche tous les disques et partitions
lsblk
```

**Ce que tu vas voir** :

```
sda  : 931 GB (disque complet)
sdb  : 279 GB (OS actuel, ne pas toucher)
sdc  : 279 GB (VIDE, on va l'utiliser)
sdd  : 279 GB (VIDE, on va l'utiliser)
```

**Explication** :
- `sda`, `sdb`, `sdc`, `sdd` = noms des disques
- Chiffres après (1,2,3) = partitions sur le disque
- Chaque disque peut avoir plusieurs partitions

### Étape 2 : Nettoyer les disques

**⚠️ ATTENTION : Cette étape EFFACE les données !**

```bash
# Efface complètement sdc et sdd
sudo wipefs -a /dev/sdc
sudo wipefs -a /dev/sdd

# Vérifie qu'ils sont vides
lsblk | grep -E "sdc|sdd"
```

**Explication** :
- `wipefs` = "wipe file system" = efface le système de fichiers
- `-a` = "all" = toutes les signatures

**Résultat** : Les disques reviennent complètement vides.

**❌ Erreur production classique**

```bash
sudo wipefs -a /dev/sdb  # NOOOOON ! C'est là que l'OS est !
```

Tu viens d'effacer ton système d'exploitation. LOCKED OUT.

**Processus sûr** :
```bash
lsblk                    # Vérifie QUEL disque tu vas effacer
sudo wipefs -a /dev/sdc  # Seulement après VÉRIFICATION
```

### Étape 3 : Créer les partitions

```bash
# Crée une partition sur sdc (tout le disque)
sudo parted -s /dev/sdc mklabel gpt
sudo parted -s /dev/sdc mkpart primary 0% 100%

# Même chose pour sdd
sudo parted -s /dev/sdd mklabel gpt
sudo parted -s /dev/sdd mkpart primary 0% 100%

# Vérifie
lsblk
```

**Explication** :
- `parted` = outil pour partitionner
- `-s` = mode silencieux (pas de questions)
- `mklabel gpt` = crée une table de partitions GPT (moderne)
- `mkpart` = crée une partition
- `primary 0% 100%` = partition primaire utilisant 100% du disque

**Résultat** : Tu as maintenant :
```
sdc1 : 279 GB (partition)
sdd1 : 279 GB (partition)
```

**Prêt pour RAID !**

---

## Jour 3 : Créer RAID 1 (Miroir)

### Qu'est-ce que RAID 1 ?

RAID = Redundant Array of Independent Disks

**RAID 1 = Miroir** : Même donnée sur 2 disques, en temps réel.

```
Données originales
    ↓
Disque 1 : [données]
Disque 2 : [données]  ← Copie exacte en temps réel
```

Si disque 1 meurt → disque 2 continue.
Si disque 2 meurt → disque 1 continue.

**Analogie** : C'est comme avoir un journaliste qui recopie le journal EXACT en temps réel. Si l'original brûle, tu as la copie.

### Les Commandes

```bash
# Crée le RAID 1
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdc1 /dev/sdd1

# Il va te demander "Continue creating array?" → Réponds : y

# Vérifie l'état du RAID
cat /proc/mdstat
```

**Explication** :
- `mdadm` = outil RAID
- `--create /dev/md0` = crée un device RAID appelé md0
- `--level=1` = RAID niveau 1 (miroir)
- `--raid-devices=2` = utilise 2 disques
- `/dev/sdc1 /dev/sdd1` = les 2 partitions à utiliser

**Ce que tu vas voir** :

```
md0 : active raid1 sdd1[1] sdc1[0]
292287488 blocks super 1.2 [2/2] [UU]
[=>.......................] resync = 5.0% (14696576/292287488)
```

- `[UU]` = les 2 disques sont "up" (actifs)
- `resync = 5%` = le miroir se copie (5% complété)
- Ça prend 20-60 minutes selon la vitesse des disques

**⚠️ Important** :

Pendant la synchronisation, le RAID fonctionne normalement. Tu peux continuer.

**❌ Erreur débutant**

```bash
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdc /dev/sdd
# (sans les 1)
```

Ça crée pas les partitions correctement.

**Bon** :
```bash
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdc1 /dev/sdd1
# (avec les 1)
```

---

## Jour 4 : LVM (Logical Volume Manager)

### Qu'est-ce que LVM ?

LVM = découper un gros disque en plus petits disques logiques.

**Analogie** : C'est comme diviser un terrain en lots :
- Terrain physique = RAID (279 GB)
- Lots = Volumes logiques (/data, /backups, /docker)

**Avantage** : Tu peux redimensionner sans reboot !

```
RAID /dev/md0 (279 GB)
    ↓
LVM Physical Volume (PV)
    ↓
LVM Volume Group (VG)
    ↓
LVM Logical Volumes (LV)
    ├─ lv_data (200 GB)
    ├─ lv_backups (50 GB)
    └─ lv_docker (29 GB)
```

### Les Commandes

```bash
# 1. Crée une physical volume sur le RAID
sudo pvcreate /dev/md0

# 2. Crée un volume group
sudo vgcreate vg_data /dev/md0

# 3. Crée les logical volumes
sudo lvcreate -L 200G -n lv_data vg_data
sudo lvcreate -L 50G -n lv_backups vg_data
sudo lvcreate -L 28G -n lv_docker vg_data

# 4. Vérifie
sudo lvs
```

**Explication** :

```bash
sudo pvcreate /dev/md0
```
- `pvcreate` = crée une "physical volume"
- C'est comme dire "ce RAID est du stockage LVM"

```bash
sudo vgcreate vg_data /dev/md0
```
- `vgcreate` = crée un "volume group"
- `vg_data` = le nom du groupe
- Rassemble les physical volumes

```bash
sudo lvcreate -L 200G -n lv_data vg_data
```
- `lvcreate` = crée un logical volume
- `-L 200G` = taille 200 GB
- `-n lv_data` = nom du volume
- `vg_data` = dans quel groupe

**Résultat** :

```
lv_data     : 200 GB
lv_backups  : 50 GB
lv_docker   : 28 GB
Total       : 278 GB (tout le RAID)
```

**❌ Erreur débutant**

```bash
sudo lvcreate -L 400G -n big_volume vg_data
# ERROR: Insufficient free space (300 GB total)
```

Tu demandes plus que ce qui existe !

**Bon** :
```bash
sudo lvcreate -L 200G -n lv_data vg_data    # OK
```

---

## Jour 5 : Format et Mount

### Étape 1 : Formater les volumes

```bash
# Crée des filesystems ext4
sudo mkfs.ext4 /dev/vg_data/lv_data
sudo mkfs.ext4 /dev/vg_data/lv_backups
sudo mkfs.ext4 /dev/vg_data/lv_docker
```

**Explication** :
- `mkfs.ext4` = "make filesystem ext4"
- ext4 = format de fichier (comme NTFS sur Windows)

**Analogie** : C'est comme formater une clé USB. Avant, c'est vide. Après, c'est prêt pour stocker des fichiers.

### Étape 2 : Créer les points de montage

```bash
# Crée les dossiers où tu vas monter les volumes
sudo mkdir -p /data
sudo mkdir -p /backups
sudo mkdir -p /docker
```

### Étape 3 : Monter les volumes

```bash
# Monte les volumes
sudo mount /dev/vg_data/lv_data /data
sudo mount /dev/vg_data/lv_backups /backups
sudo mount /dev/vg_data/lv_docker /docker

# Vérifie
df -h | grep -E "/data|/backups|/docker"
```

**Explication** :
- `mount` = "attache" le volume logique au système de fichiers
- Après, les fichiers dans `/data` sont stockés sur le LV

**Résultat** :

```
/dev/vg_data-lv_data     196G   28K  186G   1% /data
/dev/vg_data-lv_backups   49G   24K   47G   1% /backups
/dev/vg_data-lv_docker    28G   24K   26G   1% /docker
```

### Étape 4 : Ajouter au fstab (permanence au reboot)

```bash
# Édite /etc/fstab
sudo nano /etc/fstab

# Ajoute à la fin :
/dev/vg_data/lv_data     /data       ext4 defaults,nofail 0 2
/dev/vg_data/lv_backups  /backups    ext4 defaults,nofail 0 2
/dev/vg_data/lv_docker   /docker     ext4 defaults,nofail 0 2
```

**Explication** :
- `fstab` = "file system table"
- Liste les filesystems à monter automatiquement
- Sans ça, après reboot les volumes ne sont plus montés

**`nofail`** = si le volume a un problème, continue quand même (ne bloque pas le boot)

**❌ Erreur production**

```bash
# Tu ajoutes au fstab avec des erreurs
sudo reboot
# Le serveur ne redémarre pas (stuck sur erreur fstab)
```

**Toujours vérifier** :
```bash
sudo mount -a  # Test tous les entrées fstab avant reboot
```

---

## Jour 6 : Permissions

### Pourquoi les permissions ?

Chaque volume a un propriétaire et des permissions.

**Analogie** : C'est comme donner les clés de différentes pièces à différentes personnes :
- Chef cuisine = clés pour `/data`
- Comptable = clés pour `/backups`
- Informaticien = clés pour `/docker`

### Les Commandes

```bash
# Propriétaires
sudo chown -R root:root /data
sudo chown -R backup_user:backup_user /backups
sudo chown -R docker_user:docker_user /docker

# Permissions (755 = rwx r-x r-x)
sudo chmod 755 /data
sudo chmod 755 /backups
sudo chmod 755 /docker
```

**Explication** :

```
755 = rwxr-xr-x

7 = owner     (root) : read + write + execute
5 = group            : read + execute (pas write)
5 = others           : read + execute (pas write)
```

**Analogie** :
- 7 = tu peux ouvrir la porte, lire, modifier
- 5 = ta famille peut ouvrir et lire, pas modifier
- 5 = les voisins peuvent ouvrir et lire, pas modifier

**❌ Erreur production**

```bash
sudo chmod 777 /data  # N'IMPORTE QUI peut tout faire !
```

N'importe quelle app bugguée peut effacer tout.

**Bon** :
```bash
sudo chmod 755 /data  # Contrôle strict
```

---

## ✅ Checkpoint Semaine 2

**Vérifie tout fonctionne** :

```bash
# RAID est 100% synced ?
cat /proc/mdstat

# LVM volumes créés ?
sudo lvs

# Volumes montés ?
df -h | grep -E "/data|/backups|/docker"

# Permissions correctes ?
ls -ld /data /backups /docker
```

Si tout OK → **SEMAINE 2 COMPLÉTÉE** ✅

---

## 💡 Résumé : Ce que tu as appris

1. **RAID 1** : Miroir pour protection disques
2. **LVM** : Découper le stockage en volumes flexibles
3. **Filesystems** : Formater ext4
4. **Montage** : Attacher volumes au système
5. **Permissions** : Qui accède à quoi

---

**Prochaine semaine** : Docker & Applications
