From mac os:
```
diskutil mount <probably disk0s1>
cp /Volumes/EFI/debian/grubx64.efi /Volumes/EFI/BOOT/bootx64.efi
bless --folder /Volumes/EFI/EFI/BOOT --label debian
```

Boot to linux:
```
# efibootmgr -v
...
Default order: 0080,0001,0000
...
0001 debian ...
...
0080 mac ...
...
```

then
```
# efibootmgr -o 0001,0080,0000
```

