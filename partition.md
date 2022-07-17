# How to create iso files

# create iso
lsblk -f
e2fsck -f /dev/sdc3
resize2fs -M /dev/sdc3
dumpe2fs -h /dev/sdc3
fs_sector_size = fsblock_size * block_length / sector_size
fdisk -l
new_end_sector = start_sector + fs_sector_size - 1
parted /dev/sdc
  print free
  resizepart 3 {new_end_sector}s
e2fsck -f /dev/sdc3
dd if=/dev/sdc3 of=my_img.iso bs=1M

# apply iso
dd if=my_img.iso of=/dev/sdb2 bs=1M
e2fsck -f /dev/sdb2
resize2fs /dev/sdb2
