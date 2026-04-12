Finalną instrukcję od zera, już z wszystkimi zmianami:

Btrfs
Snapper
hibernacja
GRUB + grub-btrfs do wybierania snapshotów z menu
minimalne KDE
Plasma Login Manager zamiast SDDM
hostname: arch
locale: en_US.UTF-8
klawiatura: polska
strefa: Europe/Warsaw
user: pietryszak
neovim zamiast vim
dodatkowo: wget git curl btop fastfetch
instalacja po SSH z zabezpieczeniem sesji przez tmux

To wszystko opiera się o aktualne ArchWiki i oficjalne pakiety Archa: instalację przez SSH, tmux, grub-btrfs, snapper, snap-pac, hibernację z resume, minimalne KDE przez plasma-desktop oraz plasma-login-manager z usługą plasmalogin.service.

Założenia

Ta instrukcja:

kasuje cały dysk /dev/nvme0n1
zakłada boot w UEFI
zakłada brak szyfrowania
robi układ:
EFI — 1 GiB
swap — 40 GiB
Btrfs — reszta
tworzy subvolume:
@
@home
@snapshots
0. BIOS / UEFI

Przed startem z pendrive:

ustaw UEFI
wyłącz Secure Boot
jeśli Linux nie widzi dysku, ustaw storage na AHCI
1. Start Arch ISO i sieć

Po uruchomieniu ISO:

timedatectl set-ntp true

Jeśli masz kabel:

ping -c 3 archlinux.org

Jeśli Wi-Fi:

iwctl

W iwctl:

device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "NAZWA_SIECI"
exit

Potem:

ping -c 3 archlinux.org
2. SSH do live ISO

Na live ISO:

passwd
systemctl start sshd
ip -br a

Z drugiego komputera:

ssh root@ADRES_IP

Instalacja przez SSH jest normalnie opisana w ArchWiki.

3. Zabezpieczenie sesji przed zerwaniem SSH

Od razu po zalogowaniu po SSH zainstaluj i uruchom tmux:

pacman -Sy --noconfirm tmux
tmux new -A -s arch

Od tej chwili wszystko rób wewnątrz tmux.

Jeśli SSH się zerwie:

ssh root@ADRES_IP
tmux attach -t arch

Odłączenie sesji bez jej zabijania:

Ctrl+b d

To właśnie jest ten trik, który uratowałby instalację przy zerwanym SSH.

4. Sprawdzenie dysków i trybu bootu
ls /sys/firmware/efi/efivars >/dev/null && echo UEFI_OK || echo NIE_UEFI
lsblk -e7 -o NAME,SIZE,TYPE,MODEL
free -h
lspci | grep -E "VGA|3D|Display"

Dalej zakładam, że:

pendrive to sda
dysk systemowy to nvme0n1
5. Partycjonowanie dysku

To skasuje cały nvme0n1.

set -e

umount -R /mnt 2>/dev/null || true
swapoff -a 2>/dev/null || true

sgdisk --zap-all /dev/nvme0n1
sgdisk -n 1:0:+1G   -t 1:ef00 -c 1:EFI  /dev/nvme0n1
sgdisk -n 2:0:+40G  -t 2:8200 -c 2:swap /dev/nvme0n1
sgdisk -n 3:0:0     -t 3:8300 -c 3:arch /dev/nvme0n1

partprobe /dev/nvme0n1
sleep 2

lsblk -f /dev/nvme0n1
6. Formatowanie i tworzenie Btrfs
mkfs.fat -F32 /dev/nvme0n1p1
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2
mkfs.btrfs -f /dev/nvme0n1p3

Tworzenie subvolume:

mount /dev/nvme0n1p3 /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots

umount /mnt

Btrfs obsługuje snapshoty na poziomie subvolume, a snapper służy do ich zarządzania.

7. Montowanie docelowego układu
mount -o subvol=@,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt
mkdir -p /mnt/{boot,home,.snapshots}

mount -o subvol=@home,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/home
mount -o subvol=@snapshots,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/.snapshots
mount /dev/nvme0n1p1 /mnt/boot

findmnt -R /mnt
swapon --show
8. Instalacja pakietów

Tu już masz:

neovim zamiast vim
wget git curl btop fastfetch
minimalne KDE
Plasma Login Manager
Snapper
GRUB + grub-btrfs
inotify-tools pod grub-btrfsd
pacstrap -K /mnt \
  base linux linux-firmware intel-ucode btrfs-progs \
  grub efibootmgr \
  networkmanager openssh sudo neovim wget git curl btop fastfetch man-db man-pages \
  pipewire-audio pipewire-alsa pipewire-pulse wireplumber sof-firmware \
  plasma-desktop plasma-login-manager plasma-nm polkit-kde-agent \
  dolphin konsole \
  xdg-user-dirs xdg-desktop-portal xdg-desktop-portal-kde power-profiles-daemon \
  snapper snap-pac grub-btrfs inotify-tools

plasma-login-manager, grub-btrfs, snap-pac, snapper, neovim, btop, fastfetch i inotify-tools są dostępne w oficjalnych repo Arch.

9. fstab
genfstab -U /mnt > /mnt/etc/fstab
cat /mnt/etc/fstab

Powinieneś widzieć wpisy dla:

/
/home
/.snapshots
/boot
swap
10. Podstawowa konfiguracja systemu

Wejdź do chroota:

arch-chroot /mnt /bin/bash

W środku wklej:

ln -sf /usr/share/zoneinfo/Europe/Warsaw /etc/localtime
hwclock --systohc

sed -i 's/^#\(en_US.UTF-8 UTF-8\)/\1/' /etc/locale.gen
locale-gen

printf 'LANG=en_US.UTF-8\n' > /etc/locale.conf
printf 'KEYMAP=pl\n' > /etc/vconsole.conf

printf 'arch\n' > /etc/hostname

cat > /etc/hosts <<'EOF'
127.0.0.1 localhost
::1 localhost
127.0.1.1 arch.localdomain arch
EOF

sed -i '/^# %wheel ALL=(ALL:ALL) ALL/s/^# //' /etc/sudoers

useradd -m -G wheel -s /bin/bash pietryszak
passwd
passwd pietryszak

systemctl enable NetworkManager
systemctl enable sshd
systemctl enable power-profiles-daemon
systemctl enable plasmalogin.service

Wyjdź z chroota:

exit

plasmalogin.service jest usługą dostarczaną przez plasma-login-manager.

11. Hibernacja

Dla hibernacji ustawiasz:

parametr kernela resume=UUID=...
hook resume w mkinitcpio
arch-chroot /mnt /bin/bash <<'EOF'
set -e

SWAP_UUID=$(blkid -s UUID -o value /dev/nvme0n1p2)

sed -i "s/^GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT=\"loglevel=3 quiet resume=UUID=${SWAP_UUID}\"/" /etc/default/grub

grep -q '\<resume\>' /etc/mkinitcpio.conf || sed -i 's/\<fsck\>/resume fsck/' /etc/mkinitcpio.conf

mkinitcpio -P
EOF

ArchWiki dla hibernacji wprost wskazuje resume= i hook resume przy busyboxowym mkinitcpio; ArchWiki dla GRUB-a pokazuje też użycie UUID swapu do resume.

12. Snapper

Najpierw trzeba zrobić konfigurację Snappera bez D-Bus, bo w chroocie normalnie wywala błąd:

umount /mnt/.snapshots
rmdir /mnt/.snapshots

arch-chroot /mnt snapper --no-dbus -c root create-config /

Teraz usuń /.snapshots, które Snapper utworzył w @, i podepnij swoje @snapshots:

mkdir -p /mnt/.btrfs-root
mount -o subvolid=5 /dev/nvme0n1p3 /mnt/.btrfs-root

btrfs subvolume delete /mnt/.btrfs-root/@/.snapshots

umount /mnt/.btrfs-root
rmdir /mnt/.btrfs-root

mkdir -p /mnt/.snapshots
mount -o subvol=@snapshots,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/.snapshots

Włącz timeline i cleanup:

arch-chroot /mnt sed -i \
  -e 's/^TIMELINE_CREATE=.*/TIMELINE_CREATE="yes"/' \
  -e 's/^TIMELINE_CLEANUP=.*/TIMELINE_CLEANUP="yes"/' \
  -e 's/^NUMBER_CLEANUP=.*/NUMBER_CLEANUP="yes"/' \
  -e 's/^NUMBER_LIMIT=.*/NUMBER_LIMIT="10"/' \
  -e 's/^NUMBER_LIMIT_IMPORTANT=.*/NUMBER_LIMIT_IMPORTANT="5"/' \
  -e 's/^TIMELINE_LIMIT_HOURLY=.*/TIMELINE_LIMIT_HOURLY="10"/' \
  -e 's/^TIMELINE_LIMIT_DAILY=.*/TIMELINE_LIMIT_DAILY="7"/' \
  -e 's/^TIMELINE_LIMIT_WEEKLY=.*/TIMELINE_LIMIT_WEEKLY="4"/' \
  -e 's/^TIMELINE_LIMIT_MONTHLY=.*/TIMELINE_LIMIT_MONTHLY="3"/' \
  -e 's/^TIMELINE_LIMIT_YEARLY=.*/TIMELINE_LIMIT_YEARLY="0"/' \
  -e 's/^EMPTY_PRE_POST_CLEANUP=.*/EMPTY_PRE_POST_CLEANUP="yes"/' \
  /etc/snapper/configs/root

arch-chroot /mnt systemctl enable snapper-timeline.timer
arch-chroot /mnt systemctl enable snapper-cleanup.timer

arch-chroot /mnt snapper --no-dbus -c root create -d "fresh-install"
arch-chroot /mnt snapper --no-dbus -c root list

snap-pac dodaje snapshoty pre/post dla pacman, a snapper zarządza snapshotami Btrfs.

13. GRUB + snapshoty w menu

Teraz instalacja GRUB-a i generacja menu:

arch-chroot /mnt /bin/bash <<'EOF'
set -e

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg

systemctl enable grub-btrfsd.service
EOF

grub-btrfs właśnie po to istnieje: dodaje snapshoty Btrfs do opcji bootowania GRUB-a, a grub-btrfsd pilnuje zmian w katalogu snapshotów.

14. Ostatnia kontrola przed restartem
arch-chroot /mnt cat /etc/locale.conf
arch-chroot /mnt cat /etc/vconsole.conf
arch-chroot /mnt cat /etc/hostname
arch-chroot /mnt grep '^GRUB_CMDLINE_LINUX_DEFAULT' /etc/default/grub
arch-chroot /mnt grep HOOKS /etc/mkinitcpio.conf
arch-chroot /mnt systemctl is-enabled plasmalogin.service
arch-chroot /mnt systemctl is-enabled grub-btrfsd.service
arch-chroot /mnt snapper --no-dbus -c root list

Powinno być:

LANG=en_US.UTF-8
KEYMAP=pl
hostname arch
resume=UUID=...
hook resume
plasmalogin.service = enabled
grub-btrfsd.service = enabled
snapshot fresh-install
15. Restart
umount -R /mnt
swapoff /dev/nvme0n1p2
reboot

Wyjmij pendrive.

16. Po pierwszym starcie

Po zalogowaniu do Plasma:

cat /proc/cmdline
swapon --show
snapper -c root list
systemctl status grub-btrfsd.service --no-pager
grep -n "submenu 'Arch Linux snapshots'" /boot/grub/grub.cfg

To potwierdzi:

resume=UUID=...
aktywny swap
działającego Snappera
działające submenu snapshotów w GRUB-ie
17. Test hibernacji

Najprościej po prostu:

sudo systemctl hibernate

U Ciebie ten test już realnie przeszedł, więc to jest najbardziej wiarygodna próba.

18. Dodatkowy snapshot po udanej instalacji

Po pierwszym poprawnym starcie zrób od razu snapshot bazowy:

sudo snapper -c root create -d "working-base"
sudo snapper -c root list
