I dualbooted debian and arch, but I ran into some issues:

# Point EFI back to primary boot loader
Debian registered its grub in the efi boot loader list, so I had to change it
back to arch's grub.

# Adjust to new disk UUIDs by fixing fstab
Once arch booted, it was stuch at the fsck screen for a while

# Regen grub.cfg
Debian obviously wont be in the old boot loader, so its config with os-probing
needs to be rerun.

