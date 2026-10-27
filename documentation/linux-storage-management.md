# Linux storage management

## Overview

Use [Phase 3B](../README.md#phase-3b--expand-lvm-storage) for the complete backup, inspection, expansion, and verification sequence. This page explains the storage layers and example outcome.

## Commands

```bash
lsblk
sudo vgs
sudo lvs
df -h /
```

## Observations

- The root logical volume/filesystem initially used only part of the provisioned 20 GB virtual disk; this does not mean the virtual disk itself was smaller.
- LVM reported free extents in the volume group.
- The root filesystem was expanded using `lvextend` with `-r` to resize the filesystem as well.

## Example expansion

Confirm the root logical volume, available volume-group extents, and a completed backup before any change. If the filesystem already uses the available extents, skip resizing and verify its current size.

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

## Example result

The original lab expanded the root filesystem from approximately 10 GiB to about 17 GiB. This used free extents already in the volume group; it did not enlarge the virtual disk. Compare your own `lsblk`, `vgs`, `lvs`, and `df -h /` results before/after and note how the filesystem size and volume-group free space changed.

## Key takeaway

Always verify the exact logical volume and volume group before modifying storage, and keep a backup before making significant changes.
