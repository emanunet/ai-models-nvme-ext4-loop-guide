# Storing Local AI Models on an External exFAT/NTFS NVMe Drive on Linux (Zorin OS / Ubuntu)

A practical, step-by-step guide to creating a virtual **ext4** disk image (`.img`) on an external **exFAT** or **NTFS** drive. This setup enables full Linux permissions and **symbolic link** support—required by **Hugging Face**, **LM Studio**, and **Unsloth Studio**—without repartitioning the drive or losing existing data.

---

## Table of Contents
1. [Why Use a Virtual ext4 Disk Image?](#1-why-use-a-virtual-ext4-disk-image)
2. [Prerequisites & Placeholders](#2-prerequisites--placeholders)
3. [Step 1: Create and Format the Virtual Disk Image](#3-step-1-create-and-format-the-virtual-disk-image)
4. [Step 2: Set Up Write Protection & Mount the Image](#4-step-2-set-up-write-protection--mount-the-image)
5. [Step 3: Migrate AI Models & Create Symlinks](#5-step-3-migrate-ai-models--create-symlinks)
6. [Step 4: Configure Unsloth Studio](#6-step-4-configure-unsloth-studio)
7. [Step 5: Auto-Mount at Boot (`/etc/fstab`)](#7-step-5-auto-mount-at-boot-etcfstab)
8. [Maintenance, Duplicate Check & Troubleshooting](#8-maintenance-duplicate-check--troubleshooting)

---

## 1. Why Use a Virtual ext4 Disk Image?
* **Cross-platform drives lack symlink support:** External NVMe enclosures are typically formatted in `exFAT` or `NTFS` for Windows/macOS compatibility. These filesystems do not support native Linux symbolic links (`ln -s`), which Hugging Face's `hub` cache relies on to deduplicate model weights.
* **No partition resizing required:** Resizing `exFAT` partitions directly on Linux is unsupported and risky. A loop-mounted `.img` container behaves like a native `ext4` partition while sitting safely as a single file on your existing drive.
* **High-speed performance:** Over USB4 / Thunderbolt (40 Gbps), model loading speeds match internal NVMe drives.

---

## 2. Prerequisites & Placeholders
Before running the commands below, mount your external drive and replace the following placeholders with your actual values:

| Placeholder | Description | Example |
| :--- | :--- | :--- |
| `<USER>` | Your Linux username (`whoami`) | `alex` |
| `<DRIVE_NAME>` | Mount folder name of your external drive | `ExternalNVMe` |
| `<SIZE_MB>` | Virtual disk size in Megabytes (`1024 * GB`) | `512000` (for 500 GB) |
| `<DRIVE_UUID>` | Partition UUID (`lsblk -f`) | `672B-A9FE` |

---

## 3. Step 1: Create and Format the Virtual Disk Image

1. **Allocate the container file** using `dd` (works reliably on `exFAT` where `fallocate` fails):
   ```bash
   dd if=/dev/zero of=/media/<USER>/<DRIVE_NAME>/ai_models.img bs=1M count=<SIZE_MB> status=progress
   ```

2. **Format the image file as `ext4`** (press `y` and `Enter` when prompted):
   ```bash
   mkfs.ext4 /media/<USER>/<DRIVE_NAME>/ai_models.img
   ```

> **Important:** Never double-click `ai_models.img` in your desktop file manager or GNOME Disks. Always access your files through the `/mnt/ai_models` mount point to avoid duplicate loop-device conflicts.

---

## 4. Step 2: Set Up Write Protection & Mount the Image

1. **Create the mount directory and lock its underlying permissions:**
   ```bash
   sudo mkdir -p /mnt/ai_models && sudo chmod 000 /mnt/ai_models
   ```
   > **Why `chmod 000`?** If you ever launch an AI tool while the external NVMe is unplugged, the system will block writes to `/mnt/ai_models` (`Permission denied`) instead of silently filling up your internal system drive. Once the `.img` is mounted on top of it, your normal user permissions take over automatically.

2. **Mount the virtual disk and claim ownership:**
   ```bash
   sudo mount -o loop /media/<USER>/<DRIVE_NAME>/ai_models.img /mnt/ai_models
   sudo chown -R <USER>:<USER> /mnt/ai_models
   ```

3. **Verify the mount** (confirm the output shows your virtual disk size rather than your root `/` partition):
   ```bash
   df -h /mnt/ai_models
   ```

---

## 5. Step 3: Migrate AI Models & Create Symlinks

### A. Hugging Face Cache (`~/.cache/huggingface`)
```bash
# 1. Copy existing cache preserving symbolic links
cp -rp ~/.cache/huggingface /mnt/ai_models/huggingface

# 2. Verify copied size
du -sh /mnt/ai_models/huggingface

# 3. Remove internal cache and link to the external virtual disk
rm -rf ~/.cache/huggingface && ln -s /mnt/ai_models/huggingface ~/.cache/huggingface
```

### B. LM Studio (Standard / AppImage Installation)
```bash
# 1. Create target folder and move existing models
mkdir -p /mnt/ai_models/lm-studio
mv ~/.lmstudio/models/* /mnt/ai_models/lm-studio/

# 2. Replace the default folder with a symlink
rmdir ~/.lmstudio/models && ln -s /mnt/ai_models/lm-studio ~/.lmstudio/models
```

### C. LM Studio (Flatpak Installation)
If you use the Flatpak version of LM Studio, link its sandboxed model directory and grant filesystem access:
```bash
cp -rn ~/.var/app/ai.lmstudio.lm-studio/.lmstudio/models/* /mnt/ai_models/lm-studio/
rm -rf ~/.var/app/ai.lmstudio.lm-studio/.lmstudio/models/*
rmdir ~/.var/app/ai.lmstudio.lm-studio/.lmstudio/models
ln -s /mnt/ai_models/lm-studio ~/.var/app/ai.lmstudio.lm-studio/.lmstudio/models

# Grant Flatpak permission to read/write /mnt/ai_models
flatpak override --user --filesystem=/mnt/ai_models ai.lmstudio.lm-studio
```

---

## 6. Step 4: Configure Unsloth Studio
1. Open **Unsloth Studio** in your browser and click the model selector dropdown.
2. Under **CUSTOM FOLDERS**, click **`+`** and add the direct mount path:
   ```text
   /mnt/ai_models/lm-studio
   ```
3. *(Optional)* Remove redundant paths (such as your entire `/home/<USER>` directory or broken links) by clicking **`X`** next to them to speed up local model scanning.

---

## 7. Step 5: Auto-Mount at Boot (`/etc/fstab`)

1. **Retrieve the UUID of your external drive:**
   ```bash
   lsblk -f | grep <DRIVE_NAME>
   ```

2. **Edit `/etc/fstab`:**
   ```bash
   sudo nano /etc/fstab
   ```

3. **Append the following two lines** at the bottom of the file:
   ```text
   UUID=<DRIVE_UUID> /media/<USER>/<DRIVE_NAME> exfat defaults,uid=1000,gid=1000,nofail,x-systemd.device-timeout=5 0 0
   /media/<USER>/<DRIVE_NAME>/ai_models.img /mnt/ai_models ext4 loop,nofail,x-systemd.requires=/media/<USER>/<DRIVE_NAME> 0 0
   ```
   * `nofail` & `x-systemd.device-timeout=5` ensure your PC boots normally without hanging if the external drive is disconnected.
   * `x-systemd.requires` guarantees the `.img` file is mounted only after the physical drive is ready.

4. **Reload systemd and test the configuration:**
   ```bash
   sudo systemctl daemon-reload && sudo umount /mnt/ai_models && sudo mount -a && df -h /mnt/ai_models
   ```

---

## 8. Maintenance, Duplicate Check & Troubleshooting

| Task | Command |
| :--- | :--- |
| **Mount after hot-plugging** (if plugged in after boot) | `sudo mount -a` |
| **Check used/free space inside the virtual disk** | `df -h /mnt/ai_models` |
| **List all model files larger than 500 MB** | `find /mnt/ai_models -type f -size +500M -exec du -sh {} + \| sort -hr` |
| **Find exact duplicate files (SHA-256)** | `find /mnt/ai_models -type f -size +500M -exec sha256sum {} + \| sort \| uniq -w64 -D` |

---

## License
This guide is released under the [MIT License](https://opensource.org/licenses/MIT). Feel free to use, modify, and share it.
