# Complete LTFS + LTO9 Guide on Linux

## 📦 1️⃣ Prerequisites and Services

1. Check FUSE module:

```bash
lsmod | grep fuse
```

* If not loaded:

```bash
modprobe fuse
```

2. Install essential dependencies:

```bash
dnf install -y lsof tree git wget curl
```

* `lsof` → check if tape is in use
* `tree` → view directory structure
* `git` → clone GitHub repository
* `curl` / `wget` → download files

---

## 💾 2️⃣ Install HPE StoreOpen LTFS

1. Download HPE LTFS package:

```bash
curl -fL -o "HPE_StoreOpen_Software_3.5.0_RHELx64.tar.gz" https://downloads.hpe.com/pub/softlib2/software1/pubsw-generic/p854080462/v238180/HPE_StoreOpen_Software_3.5.0_RHELx64.tar.gz
```

2. Extract:

```bash
tar -xzf HPE_StoreOpen_Software_3.5.0_RHELx64.tar.gz
```

3. Install the RPM:

```bash
rpm -ivh HPE-SOS-3.5.0-62.x86_64.rpm
```

* Resolve `libicu50` dependencies if needed
* LTFS commands available via `ltfs`

---

## 📼 3️⃣ Prepare LTO9 Drive and LTFS

1. Check devices:

```bash
lsscsi -g
```

* Identify tape device: `/dev/st0`

2. Mount LTFS tape:

```bash
mkdir -p /mnt/ltfs
ltfs -o devname=/dev/st0 /mnt/ltfs
```

* Verify mount:

```bash
mount | grep ltfs
```

---

## 🔍 4️⃣ Verification and Maintenance

1. Check processes using the tape:

```bash
lsof /dev/st0
```

2. Check LTFS filesystem:

```bash
ltfsck /dev/st0
```

3. Properly unmount the tape:

```bash
umount /mnt/ltfs
```

---

## 📂 5️⃣ Copy Files / GitHub Repository

1. Clone repository:

```bash
git clone --branch master_8.1.x https://github.com/ProgrammeVitam/vitam.git /tmp/vitam
```

2. Create a tar (optional):

```bash
cd /tmp
tar -czf vitam-master_8.1.x.tar.gz vitam/
```

3. Copy to tape:

```bash
cp /tmp/vitam-master_8.1.x.tar.gz /mnt/ltfs/
# or entire repo: cp -r /tmp/vitam /mnt/ltfs/
```

---

## 📝 6️⃣ Generate a Complete CSV Register with Metadata and Hashes

### Script: `/root/ltfs_register.sh`

```bash
#!/bin/bash
MOUNTPOINT="/mnt/ltfs"
REGISTER="/root/ltfs_register_$(date +%F).csv"

echo "Path;Size;Modification_Date;SHA256" > $REGISTER
cd $MOUNTPOINT

find . -type f | while read file; do
    SIZE=$(stat -c %s "$file")
    DATE=$(stat -c %y "$file")
    HASH=$(sha256sum "$file" | awk '{print $1}')
    echo "$file;$SIZE;$DATE;$HASH" >> $REGISTER
 done

echo "Register generated: $REGISTER"
```

* Execute:

```bash
chmod +x /root/ltfs_register.sh
/root/ltfs_register.sh
```

* Output: `/root/ltfs_register_YYYY-MM-DD.csv`

---

## ✅ 7️⃣ Integrity Verification (Hash Comparison)

### Script: `/root/ltfs_integrity_check.sh`

```bash
#!/bin/bash
MOUNTPOINT="/mnt/ltfs"
REGISTER="/root/ltfs_register_2025-09-30.csv"
REPORT="/root/ltfs_integrity_report_$(date +%F).csv"

echo "Path;Status;Expected_SHA256;Current_SHA256" > $REPORT

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

echo "Integrity report generated: $REPORT"
```

* Possible statuses: **OK / MODIFIED / MISSING**

---

## 📊 8️⃣ Explore and Monitor Tape

* List files:

```bash
ls -lhR /mnt/ltfs
tree /mnt/ltfs
```

* Check used/free space:

```bash
df -h /mnt/ltfs
```

* Check hash of a specific file:

```bash
sha256sum /mnt/ltfs/vitam/packaging/README.md
```

* Read CSV register or report:

```bash
less /root/ltfs_register_2025-09-30.csv
less /root/ltfs_integrity_report_2025-09-30.csv
```

* Transfer to local machine:

```bash
scp root@server:/root/ltfs_register_2025-09-30.csv .
scp root@server:/root/ltfs_integrity_report_2025-09-30.csv .
```

---

## 🧾 Architecture Summary

| Step              | Commands / Script                    | Purpose                          |
| ----------------- | ------------------------------------ | -------------------------------- |
| Install LTFS      | `rpm -ivh HPE-SOS-*.rpm`             | Install HPE LTFS software        |
| Mount tape        | `ltfs -o devname=/dev/st0 /mnt/ltfs` | Make tape navigable              |
| Copy files        | `cp /tmp/vitam ...`                  | Store files / GitHub repo        |
| Generate register | `ltfs_register.sh`                   | Inventory with SHA256 hash       |
| Verify integrity  | `ltfs_integrity_check.sh`            | Compare files and detect changes |
| Explore           | `ls -lhR`, `tree`, `df -h`           | View content and space           |

---

# ICU Dependencies

During RPM installation, system may report missing ICU libraries (`libicudata.so.50`, `libicui18n.so.50`, `libicuuc.so.50`).

### Recommended Option

1️⃣ Enable EPEL repository:

```bash
dnf install -y epel-release
```

2️⃣ Search for a compat package:

```bash
dnf search compat | grep icu
```

* Some versions may have `compat-libicu50` or similar for backward compatibility.
  3️⃣ If available, install:

```bash
dnf install -y compat-libicu50
```

* Note: On Rocky Linux 9, `compat-libicu50` may not be available. RPM may still install successfully, and LTFS works with the newer ICU version.

---

# Tar Archive Guide on Linux

## 📦 1️⃣ Create a tar archive

### 1. Simple tar (uncompressed)

```bash
tar -cvf archive_name.tar /path/to/folder_or_files
```

* `c` → create archive
* `v` → verbose mode
* `f` → specify archive name

### 2. Tar compressed with gzip (.tar.gz)

```bash
mkdir -p /tmp/vitam_extract

tar -czvf archive_name.tar.gz /path/to/folder_or_files
```

* `z` → gzip compression
* `v` → verbose
* `f` → archive name

### 3. Tar compressed with bzip2 (.tar.bz2)

```bash
tar -cjvf archive_name.tar.bz2 /path/to/folder_or_files
```

* `j` → bzip2

### 4. Tar compressed with xz (.tar.xz)

```bash
tar -cJvf archive_name.tar.xz /path/to/folder_or_files
```

* `J` → xz

---

## 📂 2️⃣ Extract a tar archive

### 1. Tar.gz (gzip)

```bash
mkdir -p /tmp/extract_dir

tar -xzvf archive_name.tar.gz -C /tmp/extract_dir
```

* `x` → extract
* `z` → gzip
* `v` → verbose
* `f` → archive file
* `-C /tmp/extract_dir` → destination folder

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

💡 Tip: After extraction, check folder structure:

```bash
ls -lh /tmp/extract_dir
tree /tmp/extract_dir
```
# Integrity Verification Before and After Archiving with SHA256

This procedure ensures that files (videos, documents, sensitive data) are not altered during the creation of a `.tar` archive.

---

## 1️⃣ Generate SHA256 hashes before archiving

1. Open a terminal and navigate to the root of the folder to archive:

```bash
cd /path/to/folder_to_archive
```

2. Compute the SHA256 hash of all files and save it to `hashes_before.txt`:

```bash
find . -type f -exec sha256sum {} \; > ~/hashes_before.txt
```

* `find . -type f` → recursively lists all files from the root folder.
* `sha256sum` → generates the SHA256 hash.
* `>` → redirects the output to a file.

---

## 2️⃣ Create the tar archive

### Gzip option (.tar.gz)

```bash
tar -czvf archive_name.tar.gz .
```

### Bzip2 option (.tar.bz2)

```bash
tar -cjvf archive_name.tar.bz2 .
```

### XZ option (.tar.xz)

```bash
tar -cJvf archive_name.tar.xz .
```

> ⚠️ Make sure the `tar` process completes successfully before proceeding.

---

## 3️⃣ Extract the archive (for verification)

1. Create a temporary folder for extraction:

```bash
mkdir -p /tmp/archive_extract
```

2. Extract the archive:

```bash
tar -xzvf archive_name.tar.gz -C /tmp/archive_extract
# or for bzip2: tar -xjvf archive_name.tar.bz2 -C /tmp/archive_extract
# or for xz:    tar -xJvf archive_name.tar.xz -C /tmp/archive_extract
```

---

## 4️⃣ Generate SHA256 hashes after extraction

```bash
cd /tmp/archive_extract
find . -type f -exec sha256sum {} \; > ~/hashes_after.txt
```

---

## 5️⃣ Compare the hashes

```bash
diff ~/hashes_before.txt ~/hashes_after.txt
```

* If no differences appear → all files are intact.
* If differences are detected → some files have been modified or corrupted.

---

## ✅ Tips

* Always keep `hashes_before.txt` for future audits.
* For very large datasets, use `sha256sum` with `parallel` to speed up the computation.
* This procedure can be combined with an automated script for full archival and integrity verification.
