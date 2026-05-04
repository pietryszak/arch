Arch Linux — LUKS2 + TPM2 PIN + Btrfs + Snapper + grub-btrfs + hibernacja + minimalne KDE

Szybki skok (sekcje 1–4): 1 — Start z Arch ISO (#1-start-z-arch-iso) · 2 — Partycjonowanie (#2-partycjonowanie) · 3 — LUKS2 (#3-luks2) · 4 — Btrfs i subvolumy (#4-btrfs-i-subvolumy)

1. Start z Arch ISO

Połączenie Wi-Fi z live ISO:

iwctl

W iwctl:

device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect NAZWA_WIFI
exit

Test:

ping -c 3 archlinux.org

loadkeys pl
timedatectl set-ntp true

Na live ISO ustaw hasło roota i uruchom SSH:

passwd
systemctl start sshd
ip -br a

Z drugiego komputera połącz się tak:

ssh root@ADRES_IP

Reszta komend przez ssh.

2. Partycjonowanie

Uwaga: to kasuje dysk.

DISK=/dev/nvme0n1

sgdisk --zap-all "$DISK"
wipefs -af "$DISK"

sgdisk -n 1:0:+1G -t 1:ef00 -c 1:"EFI" "$DISK"
sgdisk -n 2:0:0   -t 2:8309 -c 2:"CRYPTROOT" "$DISK"

partprobe "$DISK"

Format EFI:

mkfs.fat -F32 -n EFI ${DISK}p1

3. LUKS2

cryptsetup luksFormat --type luks2 ${DISK}p2
cryptsetup open ${DISK}p2 cryptroot

Sprawdzenie:

lsblk -f

4. Btrfs i subvolumy

mkfs.btrfs -f -L ARCH /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt

Tworzenie subvolumów:

Osobne subvolumy pod profile: @mozilla — Firefox (~/.mozilla), @vivaldi — stabilny Vivaldi (~/.config/vivaldi), @vivaldi-snapshot — Vivaldi Snapshot z AUR (~/.config/vivaldi-snapshot), @thunderbird — Thunderbird (~/.thunderbird). Nie montujesz ich w §5 — dopiero po instalacji, zalogowanym użytkowniku i pierwszym rozruchu dodajesz wpisy w /etc/fstab (albo robisz to w osobnym README / skrypcie); wtedy useradd i /etc/skel z §8 działają bez ostrzeżeń. @ssh i @gnupg — na późniejsze bind-mounty.

Snapper: jeśli zamontujesz któryś z tych subvolumów wewnątrz /home, snapshot snapper -c home nie obejmie ich plików (zagnieżdżone subvolumy Btrfs — patrz też zdanie pod §17).

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@home_snapshots
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@pkg
btrfs subvolume create /mnt/@tmp
btrfs subvolume create /mnt/@spool
btrfs subvolume create /mnt/@opt
btrfs subvolume create /mnt/@swap
btrfs subvolume create /mnt/@libvirt
btrfs subvolume create /mnt/@mozilla
btrfs subvolume create /mnt/@vivaldi
btrfs subvolume create /mnt/@vivaldi-snapshot
btrfs subvolume create /mnt/@thunderbird
btrfs subvolume create /mnt/@ssh
btrfs subvolume create /mnt/@gnupg

umount /mnt

5. Montowanie systemu

Opcje:

BTRFS_OPTS="noatime,compress=zstd:3,ssd,space_cache=v2,discard=async"
BTRFS_SWAP_OPTS="noatime,ssd,space_cache=v2,discard=async"

Root:

mount -o "$BTRFS_OPTS",subvol=@ /dev/mapper/cryptroot /mnt

Katalogi:

mkdir -p /mnt/{boot,home,.snapshots,var/log,var/cache/pacman/pkg,var/tmp,var/spool,opt,swap,var/lib/libvirt}

Mounty:

mount ${DISK}p1 /mnt/boot

mount -o "$BTRFS_OPTS",subvol=@home /dev/mapper/cryptroot /mnt/home
mkdir -p /mnt/home/.snapshots
mount -o "$BTRFS_OPTS",subvol=@home_snapshots /dev/mapper/cryptroot /mnt/home/.snapshots

mount -o "$BTRFS_OPTS",subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
mount -o "$BTRFS_OPTS",subvol=@log /dev/mapper/cryptroot /mnt/var/log

mount -o "$BTRFS_OPTS",subvol=@cache /dev/mapper/cryptroot /mnt/var/cache
mkdir -p /mnt/var/cache/pacman/pkg
mount -o "$BTRFS_OPTS",subvol=@pkg /dev/mapper/cryptroot /mnt/var/cache/pacman/pkg

mount -o "$BTRFS_OPTS",subvol=@tmp /dev/mapper/cryptroot /mnt/var/tmp
mount -o "$BTRFS_OPTS",subvol=@spool /dev/mapper/cryptroot /mnt/var/spool
mount -o "$BTRFS_OPTS",subvol=@opt /dev/mapper/cryptroot /mnt/opt
mount -o "$BTRFS_SWAP_OPTS",subvol=@swap /dev/mapper/cryptroot /mnt/swap
mount -o "$BTRFS_OPTS",subvol=@libvirt /dev/mapper/cryptroot /mnt/var/lib/libvirt

Sprawdzenie:

findmnt -R /mnt -o TARGET,SOURCE,FSTYPE,OPTIONS

Przed pacstrap utwórz /mnt/etc/vconsole.conf, bo instalacja kernela odpala mkinitcpio, a hook sd-vconsole szuka tego pliku w instalowanym systemie:

mkdir -p /mnt/etc

cat > /mnt/etc/vconsole.conf <<'EOF'
KEYMAP=pl
EOF

6. Minimalny pacstrap

pacstrap -K /mnt \
  base linux linux-headers linux-firmware \
  intel-ucode \
  btrfs-progs cryptsetup tpm2-tools \
  grub efibootmgr \
  networkmanager wireless-regdb \
  sudo neovim git curl wget man-db man-pages \
  pipewire wireplumber pipewire-pulse pipewire-alsa \
  bluez bluez-utils \
  sof-firmware alsa-utils \
  mesa vulkan-intel intel-media-driver \
  snapper grub-btrfs \
  plasma-desktop plasma-login-manager \
  plasma-nm plasma-pa powerdevil bluedevil kscreen \
  xdg-desktop-portal-kde \
  breeze-gtk kde-gtk-config \
  noto-fonts noto-fonts-emoji ttf-dejavu \
  konsole dolphin bash-completion micro

Dla NVIDIA:

pacstrap -K /mnt \
  base linux linux-headers linux-firmware \
  amd-ucode \
  btrfs-progs cryptsetup tpm2-tools \
  grub efibootmgr \
  networkmanager wireless-regdb \
  sudo neovim git curl wget man-db man-pages \
  pipewire wireplumber pipewire-pulse pipewire-alsa \
  bluez bluez-utils \
  alsa-utils \
  mesa \
  nvidia-open nvidia-utils nvidia-settings opencl-nvidia \
  libva-nvidia-driver \
  snapper grub-btrfs \
  plasma-desktop plasma-login-manager \
  plasma-nm plasma-pa powerdevil bluedevil kscreen \
  xdg-desktop-portal-kde \
  breeze-gtk kde-gtk-config \
  noto-fonts noto-fonts-emoji ttf-dejavu \
  konsole dolphin bash-completion micro

7. fstab i chroot

genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt

Hostname:

echo arch > /etc/hostname

cat > /etc/hosts <<'EOF'
127.0.0.1 localhost
::1       localhost
127.0.1.1 arch.localdomain arch
EOF

Dla NVIDIA:

echo gigant > /etc/hostname

cat > /etc/hosts <<'EOF'
127.0.0.1 localhost
::1       localhost
127.0.1.1 gigant.localdomain gigant
EOF

Locale i konsola:

cat > /etc/vconsole.conf <<'EOF'
KEYMAP=pl
EOF

ln -sf /usr/share/zoneinfo/Europe/Warsaw /etc/localtime
hwclock --systohc

sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
sed -i 's/^#pl_PL.UTF-8 UTF-8/pl_PL.UTF-8 UTF-8/' /etc/locale.gen
locale-gen

cat > /etc/locale.conf <<'EOF'
LANG=en_US.UTF-8
LC_TIME=pl_PL.UTF-8
LC_MONETARY=pl_PL.UTF-8
LC_MEASUREMENT=pl_PL.UTF-8
LC_PAPER=pl_PL.UTF-8
EOF

8. Użytkownik i sudo

passwd

useradd -m -G wheel,audio,video,input,storage,lp,network,power -s /bin/bash pietryszak
passwd pietryszak

grep -q '^%wheel ALL=(ALL:ALL) ALL' /etc/sudoers || sed -i 's/^# %wheel ALL=(ALL:ALL) ALL/%wheel ALL=(ALL:ALL) ALL/' /etc/sudoers

Sprawdzenie:

id pietryszak
passwd -S root
passwd -S pietryszak

9. Poprawka /swap w fstab

Po genfstab trzeba usunąć compress=zstd:3 z wpisu /swap.

sed -i '\|[[:space:]]/swap[[:space:]]| s/,compress=zstd:3//' /etc/fstab
grep -E '[[:space:]]/swap[[:space:]]' /etc/fstab

Poprawny wpis /swap ma wyglądać mniej więcej tak:

UUID=... /swap btrfs rw,noatime,ssd,discard=async,space_cache=v2,subvol=/@swap 0 0

Bez compress=zstd:3.

10. Swapfile pod hibernację — ważna poprawiona metoda

swapoff -a 2>/dev/null || true
rm -f /swap/swapfile

# usuń problematyczny atrybut m/no-compress, jeśli wcześniej został ustawiony
chattr -m /swap 2>/dev/null || true
btrfs property set /swap compression "" 2>/dev/null || true

touch /swap/swapfile
chattr +C /swap/swapfile
lsattr /swap/swapfile

Tworzenie pliku 40 GiB:

dd if=/dev/zero of=/swap/swapfile bs=1M count=40968 status=progress
chmod 600 /swap/swapfile
mkswap /swap/swapfile
swapon /swap/swapfile

grep -q '^/swap/swapfile ' /etc/fstab || echo "/swap/swapfile none swap defaults 0 0" >> /etc/fstab

swapon --show

Dla NVIDIA:

dd if=/dev/zero of=/swap/swapfile bs=1M count=81936 status=progress
chmod 600 /swap/swapfile
mkswap /swap/swapfile
swapon /swap/swapfile

grep -q '^/swap/swapfile ' /etc/fstab || echo "/swap/swapfile none swap defaults 0 0" >> /etc/fstab

swapon --show

btrfs inspect-internal map-swapfile -r /swap/swapfile

11. mkinitcpio pod LUKS + systemd initramfs

Ustawienie hooków bez ręcznej edycji:

sed -i 's/^HOOKS=.*/HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)/' /etc/mkinitcpio.conf

grep '^HOOKS=' /etc/mkinitcpio.conf
mkinitcpio -P

Dla NVIDIA:

sed -i 's/^MODULES=.*/MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)/' /etc/mkinitcpio.conf
sed -i 's/^HOOKS=.*/HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)/' /etc/mkinitcpio.conf

mkdir -p /etc/pacman.d/hooks

cat > /etc/pacman.d/hooks/nvidia.hook <<'EOF'
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia-open
Target=nvidia-utils
Target=linux

[Action]
Description=Update NVIDIA module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/usr/bin/mkinitcpio -P
EOF

grep '^MODULES=' /etc/mkinitcpio.conf
grep '^HOOKS=' /etc/mkinitcpio.conf
mkinitcpio -P
grep '^HOOKS=' /etc/mkinitcpio.conf

Oczekiwane:

HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)

12. UUID-y, resume offset, crypttab.initramfs

CRYPT_UUID=$(blkid -s UUID -o value /dev/nvme0n1p2)
ROOT_UUID=$(findmnt -no UUID /)
RESUME_OFFSET=$(btrfs inspect-internal map-swapfile -r /swap/swapfile)

echo "CRYPT_UUID=$CRYPT_UUID"
echo "ROOT_UUID=$ROOT_UUID"
echo "RESUME_OFFSET=$RESUME_OFFSET"

cat > /etc/crypttab.initramfs <<EOF
cryptroot UUID=${CRYPT_UUID} none tpm2-device=auto
EOF

cat /etc/crypttab.initramfs

13. GRUB kernel params

sed -i "s|^GRUB_CMDLINE_LINUX_DEFAULT=.*|GRUB_CMDLINE_LINUX_DEFAULT=\"quiet loglevel=3 rd.luks.name=${CRYPT_UUID}=cryptroot root=/dev/mapper/cryptroot rootflags=subvol=@ resume=UUID=${ROOT_UUID} resume_offset=${RESUME_OFFSET}\"|" /etc/default/grub

grep '^GRUB_CMDLINE_LINUX_DEFAULT' /etc/default/grub

Dla NVIDIA:

sed -i "s|^GRUB_CMDLINE_LINUX_DEFAULT=.*|GRUB_CMDLINE_LINUX_DEFAULT=\"quiet loglevel=3 nvidia_drm.modeset=1 rd.luks.name=${CRYPT_UUID}=cryptroot root=/dev/mapper/cryptroot rootflags=subvol=@ resume=UUID=${ROOT_UUID} resume_offset=${RESUME_OFFSET}\"|" /etc/default/grub

grep '^GRUB_CMDLINE_LINUX_DEFAULT' /etc/default/grub

Instalacja GRUB:

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ARCH
grub-mkconfig -o /boot/grub/grub.cfg

Ostrzeżenie o os-prober można zignorować, jeśli nie ma dual-boota.

14. TPM2 + PIN

Sprawdzenie TPM:

systemd-cryptenroll --tpm2-device=list

Dodanie TPM2 + PIN:

systemd-cryptenroll /dev/nvme0n1p2 --tpm2-device=auto --tpm2-with-pin=yes

Sprawdzenie:

systemd-cryptenroll /dev/nvme0n1p2

Po poprawnym dodaniu TPM2 tokena, czyli gdy systemd-cryptenroll /dev/nvme0n1p2 pokazuje slot tpm2, przebuduj initramfs i GRUB:

mkinitcpio -P
grub-mkconfig -o /boot/grub/grub.cfg

15. Usługi

Włączenie podstaw:

systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable plasmalogin.service

Sprawdzenie login managera:

systemctl list-unit-files | grep -Ei 'plasma|login'

Oczekiwane:

plasmalogin.service

16. Snapper root — poprawiona metoda z --no-dbus

Root:

umount /.snapshots 2>/dev/null || true
rm -rf /.snapshots

snapper --no-dbus -c root create-config /

# Snapper tworzy własny subvolume /.snapshots, więc go usuwamy
btrfs subvolume delete /.snapshots

# Przywracamy mountpoint dla przygotowanego @snapshots
mkdir /.snapshots
mount /.snapshots

chmod 750 /.snapshots

snapper --no-dbus -c root create --description "fresh encrypted Arch install"
snapper --no-dbus -c root list

17. Snapper home — poprawiona metoda z --no-dbus

umount /home/.snapshots 2>/dev/null || true
rm -rf /home/.snapshots

snapper --no-dbus -c home create-config /home

btrfs subvolume delete /home/.snapshots

mkdir /home/.snapshots
mount /home/.snapshots

chmod 750 /home/.snapshots

snapper --no-dbus -c home create --description "fresh encrypted home snapshot"
snapper --no-dbus -c home list

Snapshoty snapper -c home nie obejmują zagnieżdżonych subvolumów profili z §4 — jeśli zamontujesz je pod /home, patrz krótka uwaga przy §4.

17.1 Instalacja snap-pac po konfiguracji Snappera

snap-pac instalujemy dopiero po utworzeniu konfiguracji Snappera dla / i /home.
Nie instalować snap-pac w pacstrap, bo jego hooki pacmana mogą uruchomić się zanim Snapper będzie gotowy.
Uwaga: w chroot po instalacji snap-pac może pojawić się komunikat fatal library error, lookup self. Jeśli pacman -Q snap-pac potwierdza instalację, a snapper --no-dbus -c root list działa, można to zignorować. Po normalnym bootowaniu systemu hooki snap-pac działają poprawnie.

pacman -S --needed snap-pac

18. Timery Snappera i grub-btrfs

systemctl enable snapper-timeline.timer
systemctl enable snapper-cleanup.timer
systemctl enable grub-btrfsd.service

Po pierwszych snapshotach:

grub-mkconfig -o /boot/grub/grub.cfg

Oczekiwane:

Detecting snapshots ...
Found snapshot: ... @snapshots/1/snapshot ...

19. Finalny check przed rebootem

cat /etc/fstab | grep -E ' / |/boot|/home|/swap|swapfile'
grep -E '[[:space:]]/[[:space:]]+btrfs' /etc/fstab

cat /etc/crypttab.initramfs
systemd-cryptenroll /dev/nvme0n1p2
systemctl is-enabled NetworkManager bluetooth plasmalogin.service
swapon --show

grep '^HOOKS=' /etc/mkinitcpio.conf
grep '^GRUB_CMDLINE_LINUX_DEFAULT' /etc/default/grub

snapper --no-dbus -c root list
snapper --no-dbus -c home list

Ostatnie generowanie:

mkinitcpio -P
grub-mkconfig -o /boot/grub/grub.cfg

Wyjście:

exit

Jeśli /mnt/swap jest busy:

swapoff /mnt/swap/swapfile 2>/dev/null || swapoff -a
umount -R /mnt
reboot

20. Po pierwszym bootowaniu

Po zalogowaniu:

cat /proc/cmdline
swapon --show
findmnt -R /
sudo snapper -c root list
sudo snapper -c home list
cat /sys/power/state

Jeśli jest:

freeze mem disk

test hibernacji:

systemctl hibernate

Dla NVIDIA:

lsmod | grep nvidia
nvidia-smi

21. Baseline snapshot po pierwszych zmianach

Po zainstalowaniu i drobnych zmianach wykonano baseline:

sudo snapper -c root create --description "baseline before system tweaks"
sudo snapper -c home create --description "baseline home before tweaks"

sudo snapper -c root list
sudo snapper -c home list

Oznaczenie jako important:

sudo snapper -c root modify --cleanup-algorithm important numer
sudo snapper -c home modify --cleanup-algorithm important numer

Odświeżenie GRUB:

sudo grub-mkconfig -o /boot/grub/grub.cfg

Oczekiwane:

Found snapshot: ... @snapshots/8/snapshot | single | baseline before system tweaks |

Uwaga: GRUB pokazuje snapshoty root. Snapshoty /home nie są bootowalne i nie będą w menu GRUB.
