---
date: '2025-07-21T21:16:44+02:00'
draft: false
title: 'FEX - LVM'
summary: "FEX - LVM"
tags: ["fex","linux"]
categories: ["fex"]
---

> Ce document est une fiche d'exploitation pour l'administration des disques via LVM (Logical Volum Manager).

---

Ci-dessous, les commandes utiles pour `lvm`

```bash
# Installation des outils LVM
sudo apt install -y lvm2
sudo yum install -y lvm2

# Lister les devices
sudo lsblk

# Lister les Physical Volumes
sudo pvs
# Plus de détails
sudo pvdisplay

# Lister les Volumes Groups
sudo vgs
# Plus de détails
sudo vgdisplay

# Lister les Logical Volumes
sudo lvs
# Plus de détails
sudo lvdisplay
```

Initialisation d'un disque en tant que PV puis extend du VG et LV avec resize FS.

```bash
# Initialisation d'un disque en tant que PV 
sudo pvcreate <device_pv>
sudo pvcreate /dev/sdb

# Extend d'un VG
sudo vgextend <vg> /dev/<device_pv>
sudo vgextend debian-vg /dev/sdb

# Extend d'un LV avec resize fs
sudo lvextend -r -L +<capacity> <lv_path>
sudo lvextend -r -L +5G /dev/debian-vg/lv-var
sudo lvextend -r -l +100%FREE /dev/debian-vg/lv-var
```

Initialisation d'un disque en tant que PV puis création d'un VG et d'un LV avec formatage du FS.

```bash
# Initialisation d'un disque en tant que PV 
sudo pvcreate <device_pv>
sudo pvcreate /dev/sdb

# Création d'un VG
sudo vgcreate <vg_name> <device_pv>
sudo vgcreate vg-1 /dev/sdb

# Création d'un LV
sudo lvcreate -L <capacity> -n <lv_name> <vg_name>
sudo lvcreate -L 1G -n lv-1 vg-1

# Formatage du LV
sudo mkf.<fs_type> <lv_path>
sudo mkfs.ext4 /dev/vg1/lv-1

# Montage du FS
sudo mount <lv_path> <fs_path>
sudo mount /dev/vg-1/lv-1 /mnt/lv1/
```

Réduction d'un LV ou d'un VG. Le FS doit être démonté.

```bash
# Démontage du FS
sudo umount <fs_path>
sudo umount /mnt/lv1

# Healthcheck du FS
sudo e2fsck -f <lv_path>
sudo e2fsck -f /dev/vg-1/lv-1

# Réduction du FS
sudo resize2fs <lv_path> <capacity>
sudo resize2fs /dev/vg-1/lv-1 500M

# Réduction du LV
sudo lvreduce -L <capacity> <lv_path>
sudo lvreduce -L 500M /dev/vg-1/lv-1

# Remontage du FS
sudo mount /mnt/lv1
```

Suppression d'un LV.

```bash
# Démontage du FS
sudo umount <fs_path>
sudo umount /mnt/lv1

# Suppression du LV
sudo lvremove <lv_path>
sudo lvremove /dev/vg-1/lv-1

# Suppression du VG
sudo vgremove <vg_name>
sudo vgremove vg-1

# Suppresison du PV (suppression du label)
sudo pvremove <device>
sudo pvremove /dev/sdb
```

