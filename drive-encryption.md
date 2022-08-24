# Quick Start
```
cryptsetup luksFormat --type luks1 /dev/sda1
cryptsetup -v luksOpen /dev/sda1 sda1-open
mkfs.ext4 /dev/mapper/sda1-open
mount /dev/mapper/sda1-open /mnt
```

# `/etc/fstab`
use fstab if you have multiple encrypted drives so that they always mount in
the right spot.

Get UUID of drives
```
lsblk -f
```

Then add line to /etc/fstab
```
UUID=99ea80c4-e748-47eb-835c-64025de53e26 /data ext4 noauto,rw 0 2
```
The `noauto` is important since it will be locked at boot.

# Source
- [good guide](https://www.baeldung.com/linux/encrypt-partition)
