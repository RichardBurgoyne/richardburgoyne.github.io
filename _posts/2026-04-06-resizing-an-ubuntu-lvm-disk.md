---
title: "Resizing an Ubuntu LVM Disk"
date: 2026-04-06T22:48:00+11:00
publishDate: 2026-04-06T22:48:00+11:00
draft: false
tags: ["lvm", "ubuntu", "disk"]
---

When you expand a virtual disk, Ubuntu doesn’t automatically take advantage of the new space. The disk may be bigger, but the partition, LVM layers, and filesystem are all still living in their old footprint.

Here’s the full process to stretch everything out so Ubuntu can actually use the extra room.

---

## 1. Resize the Partition

Your virtual disk is larger now, but the partition sitting on it hasn’t changed. Time to fix that.

You can use either **fdisk** or **parted**.

### Using `fdisk`

```bash
sudo fdisk /dev/sda
```

Inside `fdisk`, delete partition **3** (yes, delete it’s safe). You’re not wiping data; you’re just redefining where the partition ends. Recreate it with the same starting sector, but let it run all the way to the end of the disk. Once that’s done, write the changes and exit.

### Using `parted`

```bash
sudo parted /dev/sda
(parted) resizepart 3
```

Parted will ask how large you want the partition to be. Tell it to use the whole disk `100%` works nicely. Quit when you’re done.

At this point, the partition finally matches the size of the disk. Ubuntu still doesn’t know what to do with the extra space, but we’re getting there.

---

## 2. Resize the LVM Physical Volume

Now that the partition has grown, LVM needs to be nudged so it notices the extra room.

```bash
sudo pvresize /dev/sda3
```

This updates LVM’s metadata so it recognises the additional space. No data moves it’s just bookkeeping.

---

## 3. Extend the Logical Volume

With the physical volume updated, the volume group suddenly has more space to hand out. Time to give it to the logical volume that actually matters.

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

This tells LVM to take every remaining free extent and hand it over to your logical volume. No need to calculate sizes LVM handles it.

---

## 4. Resize the Filesystem

The logical volume is now bigger, but the filesystem sitting on top of it hasn’t stretched yet. Let’s fix that.

```bash
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

`resize2fs` grows the filesystem to match the logical volume. It automatically uses the full available size.

---

## Done

Everything should now be in sync — disk → partition → LVM → filesystem.

A quick check:

```bash
df -h
```

You should now see the expanded space reflected in your filesystem. Ubuntu is finally using the full size of your virtual disk.
