# pacman guide

# update
pacman -Syu
# force refresh, update
pacman -Syyu
# remove pkg
pacman -Rns <pkg>
# list non dependents
pacman -Qdt
# list non dependents without version
pacman -Qdtq
pacman -Rns $(pacman Qdtq)
# list manually installed
pacman -Qe

# AUR
paru -Syu

