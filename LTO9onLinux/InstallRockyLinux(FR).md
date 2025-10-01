# Guide complet LTFS + LTO9 sous Linux

## 📦 1️⃣ Pré-requis et services

1. Vérification du module FUSE :

```bash
lsmod | grep fuse
```

* Si non chargé :

```bash
modprobe fuse
```

2. Installation des dépendances essentielles :

```bash
dnf install -y lsof tree git wget curl
```

* `lsof` → vérifier si la bande est utilisée
* `tree` → visualiser l'arborescence
* `git` → cloner dépôt GitHub
* `curl` / `wget` → télécharger des fichiers

---

## 💾 2️⃣ Installation HPE StoreOpen LTFS

1. Télécharger le package HPE LTFS :

```bash
curl -fL -o "HPE_StoreOpen_Software_3.5.0_RHELx64.tar.gz" https://downloads.hpe.com/pub/softlib2/software1/pubsw-generic/p854080462/v238180/HPE_StoreOpen_Software_3.5.0_RHELx64.tar.gz
```

2. Décompresser :

```bash
tar -xzf HPE_StoreOpen_Software_3.5.0_RHELx64.tar.gz
```

3. Installer le RPM HPE StoreOpen Software :

```bash
rpm -ivh HPE-SOS-3.5.0-62.x86_64.rpm
```

* Résolution des dépendances `libicu50` si nécessaire
* LTFS est commandé à la demande via `ltfs`

---

## 📼 3️⃣ Préparer le lecteur LTO9 et LTFS

1. Vérifier les périphériques :

```bash
lsscsi -g
```

* Identifier la bande : `/dev/st0`

2. Monter la bande LTFS :

```bash
mkdir -p /mnt/ltfs
ltfs -o devname=/dev/st0 /mnt/ltfs
```

* Vérifier le montage :

```bash
mount | grep ltfs
```

---

## 🔍 4️⃣ Vérification et maintenance

1. Vérifier les processus utilisant la bande :

```bash
lsof /dev/st0
```

2. Vérifier la structure LTFS :

```bash
ltfsck /dev/st0
```

3. Démonter la bande proprement :

```bash
umount /mnt/ltfs
```

---

## 📂 5️⃣ Copier des fichiers / dépôts GitHub

1. Cloner le dépôt :

```bash
git clone --branch master_8.1.x https://github.com/ProgrammeVitam/vitam.git /tmp/vitam
```

2. Créer un tar (optionnel) :

```bash
cd /tmp
tar -czf vitam-master_8.1.x.tar.gz vitam/
```

3. Copier sur la bande :

```bash
cp /tmp/vitam-master_8.1.x.tar.gz /mnt/ltfs/
# ou le dépôt complet : cp -r /tmp/vitam /mnt/ltfs/
```

---

## 📝 6️⃣ Générer un registre complet (CSV) avec métadonnées et hash

### Script : `/root/ltfs_register.sh`

```bash
#!/bin/bash
MOUNTPOINT="/mnt/ltfs"
REGISTER="/root/ltfs_register_$(date +%F).csv"

echo "Chemin;Taille;Date_modif;SHA256" > $REGISTER
cd $MOUNTPOINT

find . -type f | while read file; do
    SIZE=$(stat -c %s "$file")
    DATE=$(stat -c %y "$file")
    HASH=$(sha256sum "$file" | awk '{print $1}')
    echo "$file;$SIZE;$DATE;$HASH" >> $REGISTER
 done

echo "Registre généré : $REGISTER"
```

* Exécuter :

```bash
chmod +x /root/ltfs_register.sh
/root/ltfs_register.sh
```

* Résultat : `/root/ltfs_register_YYYY-MM-DD.csv`

---

## ✅ 7️⃣ Vérification de l’intégrité (comparaison hashes)

### Script : `/root/ltfs_integrity_check.sh`

```bash
#!/bin/bash
MOUNTPOINT="/mnt/ltfs"
REGISTER="/root/ltfs_register_2025-09-30.csv"
REPORT="/root/ltfs_integrity_report_$(date +%F).csv"

echo "Chemin;Status;Expected_SHA256;Current_SHA256" > $REPORT

tail -n +2 "$REGISTER" | while IFS=';' read -r file size date expected_hash; do
    FULLPATH="$MOUNTPOINT/${file#./}"
    if [[ -f "$FULLPATH" ]]; then
        current_hash=$(sha256sum "$FULLPATH" | awk '{print $1}')
        if [[ "$current_hash" == "$expected_hash" ]]; then
            status="OK"
        else
            status="MODIFIED"
        fi
    else
        status="MISSING"
        current_hash=""
    fi
    echo "$file;$status;$expected_hash;$current_hash" >> $REPORT
 done

echo "Rapport d'intégrité généré : $REPORT"
```

* Statuts possibles : **OK / MODIFIED / MISSING**

---

## 📊 8️⃣ Explorer et contrôler la bande

* Liste fichiers :

```bash
ls -lhR /mnt/ltfs
tree /mnt/ltfs
```

* Vérifier espace utilisé / libre :

```bash
df -h /mnt/ltfs
```

* Vérifier hash d’un fichier spécifique :

```bash
sha256sum /mnt/ltfs/vitam/packaging/README.md
```

* Lire registre ou rapport CSV sur le serveur :

```bash
less /root/ltfs_register_2025-09-30.csv
less /root/ltfs_integrity_report_2025-09-30.csv
```

* Transférer sur machine locale :

```bash
scp root@serveur:/root/ltfs_register_2025-09-30.csv .
scp root@serveur:/root/ltfs_integrity_report_2025-09-30.csv .
```

---

## 🧾 Résumé de l’architecture

| Étape                  | Commandes / Script                   | Objectif                                    |
| ---------------------- | ------------------------------------ | ------------------------------------------- |
| Installation LTFS      | `rpm -ivh HPE-SOS-*.rpm`             | Installer le logiciel HPE LTFS              |
| Montage bande          | `ltfs -o devname=/dev/st0 /mnt/ltfs` | Rendre la bande navigable                   |
| Copie fichiers         | `cp /tmp/vitam ...`                  | Stocker fichiers / dépôt GitHub             |
| Génération registre    | `ltfs_register.sh`                   | Inventaire avec hash SHA256                 |
| Vérification intégrité | `ltfs_integrity_check.sh`            | Comparer fichiers et détecter modifications |
| Exploration            | `ls -lhR`, `tree`, `df -h`           | Visualiser contenu et espace                |

---

# Option 1 

## ⚠️ Dépendances ICU

Lors de l'installation du RPM HPE-SOS, le système peut signaler que les bibliothèques ICU (libicudata.so.50, libicui18n.so.50, libicuuc.so.50) sont manquantes.

### Option recommandée (propre)

1️⃣ Active le dépôt EPEL (si pas encore activé) :

```bash
dnf install -y epel-release
```

2️⃣ Chercher s’il existe un paquet de compatibilité :

```bash
dnf search compat | grep icu
```

* Sur certaines versions, tu peux trouver : `compat-libicu50` ou un paquet similaire pour rétrocompatibilité.

3️⃣ Si disponible, installer :

```bash
dnf install -y compat-libicu50
```

### Note pratique

* Sur Rocky Linux 9, le paquet `compat-libicu50` peut ne pas être disponible.
* Dans ce cas, le RPM HPE-SOS peut quand même s'installer correctement et LTFS fonctionne avec la version plus récente de ICU déjà présente sur le système.

# Option 2 

# Guide : Création et extraction d’archives tar sous Linux

## 📦 1️⃣ Créer un fichier tar (archive)

### 1. Tar simple (non compressé)

```bash
tar -cvf archive_name.tar /chemin/vers/dossier_ou_fichiers
```

* `c` → créer une archive
* `v` → mode verbeux (affiche les fichiers ajoutés)
* `f` → spécifie le nom du fichier archive

### 2. Tar compressé en gzip (.tar.gz)

```bash
mkdir -p /tmp/vitam_extract

tar -czvf archive_name.tar.gz /chemin/vers/dossier_ou_fichiers
```

* `z` → compresser avec gzip
* `v` → mode verbeux
* `f` → spécifie le nom du fichier archive

### 3. Tar compressé en bzip2 (.tar.bz2)

```bash
tar -cjvf archive_name.tar.bz2 /chemin/vers/dossier_ou_fichiers
```

* `j` → compresser avec bzip2

### 4. Tar compressé en xz (.tar.xz)

```bash
tar -cJvf archive_name.tar.xz /chemin/vers/dossier_ou_fichiers
```

* `J` → compresser avec xz

---

## 📂 2️⃣ Décompresser un fichier tar

### 1. Tar.gz (gzip)

```bash
mkdir -p /tmp/extract_dir

tar -xzvf archive_name.tar.gz -C /tmp/extract_dir
```

* `x` → extraire
* `z` → gzip
* `v` → mode verbeux
* `f` → fichier archive
* `-C /tmp/extract_dir` → chemin de destination

### 2. Tar.bz2 (bzip2)

```bash
mkdir -p /tmp/extract_dir
tar -xjvf archive_name.tar.bz2 -C /tmp/extract_dir
```

* `j` → bzip2

### 3. Tar.xz (xz)

```bash
mkdir -p /tmp/extract_dir
tar -xJvf archive_name.tar.xz -C /tmp/extract_dir
```

* `J` → xz

💡 Astuce : après extraction, vérifier l’arborescence avec :

```bash
ls -lh /tmp/extract_dir
tree /tmp/extract_dir
```

# Option 3 

# Vérification d’intégrité avant et après archivage avec SHA256

Cette procédure permet de s’assurer que les fichiers (vidéos, documents, données sensibles) ne sont pas modifiés pendant la création d’une archive `.tar`.

---

## 1️⃣ Générer les hash SHA256 avant l’archivage

1. Ouvrir un terminal et se placer à la racine du dossier à archiver :

```bash
cd /chemin/vers/dossier_a_archiver
```

2. Calculer le hash SHA256 de tous les fichiers et enregistrer dans un fichier `hashes_before.txt` :

```bash
find . -type f -exec sha256sum {} \; > ~/hashes_before.txt
```

* `find . -type f` → liste tous les fichiers récursivement depuis la racine.
* `sha256sum` → génère le hash SHA256.
* `>` → redirige le résultat vers un fichier.

---

## 2️⃣ Créer l’archive tar

### Option gzip (.tar.gz)

```bash
tar -czvf archive_name.tar.gz .
```

### Option bzip2 (.tar.bz2)

```bash
tar -cjvf archive_name.tar.bz2 .
```

### Option xz (.tar.xz)

```bash
tar -cJvf archive_name.tar.xz .
```

> ⚠️ Assurez-vous que le processus `tar` se termine correctement avant de continuer.

---

## 3️⃣ Extraire l’archive (pour vérification)

1. Créer un dossier temporaire pour l’extraction :

```bash
mkdir -p /tmp/archive_extract
```

2. Extraire l’archive :

```bash
tar -xzvf archive_name.tar.gz -C /tmp/archive_extract
# ou pour bzip2 : tar -xjvf archive_name.tar.bz2 -C /tmp/archive_extract
# ou pour xz    : tar -xJvf archive_name.tar.xz -C /tmp/archive_extract
```

---

## 4️⃣ Générer les hash SHA256 après extraction

```bash
cd /tmp/archive_extract
find . -type f -exec sha256sum {} \; > ~/hashes_after.txt
```

---

## 5️⃣ Comparer les hash

```bash
diff ~/hashes_before.txt ~/hashes_after.txt
```

* Si aucune différence n’apparaît → tous les fichiers sont intacts.
* Si des différences sont détectées → certains fichiers ont été modifiés ou corrompus.

---

## ✅ Astuces

* Toujours stocker le fichier `hashes_before.txt` pour audit futur.
* Pour de très gros volumes, utiliser `sha256sum` avec `parallel` pour accélérer le calcul.
* Peut être combiné avec un script automatique pour archivage et vérification intégrale.

# Option 4

## Commande d'eject de bande
```
mt -f /dev/nst0 eject
```
