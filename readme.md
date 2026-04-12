# Arch Linux na Dell Latitude 5421
## Btrfs + Snapper + hibernacja + GRUB snapshoty + minimalne KDE + Plasma Login Manager

Ten przewodnik opisuje czystą instalację Arch Linuksa na **Dell Latitude 5421** z:

- UEFI
- Btrfs
- snapshotami przez Snapper
- działającą hibernacją
- GRUB + grub-btrfs do wybierania snapshotów z menu startowego
- minimalistycznym KDE Plasma
- Plasma Login Manager zamiast SDDM
- językiem systemu ustawionym na `en_US.UTF-8`
- polską klawiaturą
- strefą czasową `Europe/Warsaw`

Docelowa konfiguracja:

- dysk docelowy: `/dev/nvme0n1`
- pełne wyczyszczenie dysku
- brak szyfrowania
- użytkownik: `pietryszak`
- hostname: `arch`

---

# Układ partycji

- `nvme0n1p1` — EFI — `1 GiB`
- `nvme0n1p2` — swap — `40 GiB`
- `nvme0n1p3` — Btrfs — reszta dysku

Subvolume Btrfs:

- `@`
- `@home`
- `@snapshots`

---

# 0. Ustawienia BIOS / UEFI

Przed uruchomieniem instalatora z pendrive ustaw:

- tryb bootowania: **UEFI**
- **Secure Boot: Off**
- jeśli Linux nie widzi dysku: tryb storage ustaw na **AHCI**

---

# 1. Start Arch ISO i połączenie z internetem

Po uruchomieniu Arch ISO:

```bash
timedatectl set-ntp true
```

Jeśli masz Ethernet:

```bash
ping -c 3 archlinux.org
```

Jeśli potrzebujesz Wi-Fi:

```bash
iwctl
```

W `iwctl`:

```text
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "NAZWA_SIECI"
exit
```

Potem sprawdź połączenie:

```bash
ping -c 3 archlinux.org
```

---

# 2. Włączenie SSH na live ISO

Na live ISO ustaw hasło roota i uruchom SSH:

```bash
passwd
systemctl start sshd
ip -br a
```

Z drugiego komputera połącz się tak:

```bash
ssh root@ADRES_IP
```

---

# 3. Zabezpieczenie sesji SSH przed zerwaniem przez tmux

Od razu po zalogowaniu po SSH zainstaluj i uruchom `tmux`:

```bash
pacman -Sy --noconfirm tmux
tmux new -A -s arch
```

Od tej chwili wykonuj całą instalację **wewnątrz `tmux`**.

Jeśli SSH się zerwie:

```bash
ssh root@ADRES_IP
tmux attach -t arch
```

Przydatne skróty:

```text
Ctrl+b d   odłączenie sesji bez jej zabijania
Ctrl+b c   nowe okno
Ctrl+b n   następne okno
Ctrl+b p   poprzednie okno
```

---

# 4. Sprawdzenie trybu bootu i dysków

```bash
ls /sys/firmware/efi/efivars >/dev/null && echo UEFI_OK || echo NIE_UEFI
lsblk -e7 -o NAME,SIZE,TYPE,MODEL
free -h
lspci | grep -E "VGA|3D|Display"
```

Ten przewodnik zakłada, że:

- pendrive instalacyjny to `sda`
- dysk docelowy to `/dev/nvme0n1`

---

# 5. Partycjonowanie dysku

> **Uwaga:** ten krok kasuje cały `/dev/nvme0n1`.

```bash
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
```

---

# 6. Formatowanie i tworzenie subvolume Btrfs

```bash
mkfs.fat -F32 /dev/nvme0n1p1
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2
mkfs.btrfs -f /dev/nvme0n1p3
```

Tworzenie subvolume:

```bash
mount /dev/nvme0n1p3 /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots

umount /mnt
```

---

# 7. Montowanie docelowego układu

```bash
mount -o subvol=@,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt
mkdir -p /mnt/{boot,home,.snapshots}

mount -o subvol=@home,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/home
mount -o subvol=@snapshots,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/.snapshots
mount /dev/nvme0n1p1 /mnt/boot

findmnt -R /mnt
swapon --show
```

---

# 8. Instalacja pakietów

Instalujemy:

- bazę systemu
- GRUB
- sieć
- PipeWire
- minimalne KDE
- Plasma Login Manager
- Snapper
- grub-btrfs
- dodatkowe narzędzia

```bash
pacstrap -K /mnt \
  base linux linux-firmware intel-ucode btrfs-progs \
  grub efibootmgr \
  networkmanager openssh sudo neovim wget git curl btop fastfetch man-db man-pages \
  pipewire-audio pipewire-alsa pipewire-pulse wireplumber sof-firmware \
  plasma-desktop plasma-login-manager plasma-nm polkit-kde-agent \
  dolphin konsole \
  xdg-user-dirs xdg-desktop-portal xdg-desktop-portal-kde power-profiles-daemon \
  snapper snap-pac grub-btrfs inotify-tools
```

---

# 9. Generowanie fstab

```bash
genfstab -U /mnt > /mnt/etc/fstab
cat /mnt/etc/fstab
```

Powinny pojawić się wpisy dla:

- `/`
- `/home`
- `/.snapshots`
- `/boot`
- swap

---

# 10. Podstawowa konfiguracja systemu

Wejdź do chroota:

```bash
arch-chroot /mnt /bin/bash
```

W chroocie wykonaj:

```bash
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
```

Wyjdź z chroota:

```bash
exit
```

---

# 11. Konfiguracja hibernacji

Ustawienie `resume=UUID=...` dla GRUB i hooka `resume` w `mkinitcpio`:

```bash
arch-chroot /mnt /bin/bash <<'EOF'
set -e

SWAP_UUID=$(blkid -s UUID -o value /dev/nvme0n1p2)

sed -i "s/^GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT=\"loglevel=3 quiet resume=UUID=${SWAP_UUID}\"/" /etc/default/grub

grep -q '\<resume\>' /etc/mkinitcpio.conf || sed -i 's/\<fsck\>/resume fsck/' /etc/mkinitcpio.conf

mkinitcpio -P
EOF
```

---

# 12. Konfiguracja Snappera

## 12.1 Utworzenie konfiguracji Snappera bez D-Bus

> W chroocie trzeba użyć `--no-dbus`, bo bez tego Snapper może wywalić błąd `org.freedesktop.DBus.Error.ServiceUnknown`.

```bash
umount /mnt/.snapshots
rmdir /mnt/.snapshots

arch-chroot /mnt snapper --no-dbus -c root create-config /
```

## 12.2 Usunięcie `/.snapshots` utworzonego w `@` i podpięcie własnego `@snapshots`

```bash
mkdir -p /mnt/.btrfs-root
mount -o subvolid=5 /dev/nvme0n1p3 /mnt/.btrfs-root

btrfs subvolume delete /mnt/.btrfs-root/@/.snapshots

umount /mnt/.btrfs-root
rmdir /mnt/.btrfs-root

mkdir -p /mnt/.snapshots
mount -o subvol=@snapshots,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/.snapshots
```

## 12.3 Włączenie timeline i cleanup

```bash
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
```

---

# 13. Instalacja GRUB i integracja snapshotów

```bash
arch-chroot /mnt /bin/bash <<'EOF'
set -e

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg

systemctl enable grub-btrfsd.service
EOF
```

---

# 14. Ostatnia kontrola przed restartem

```bash
arch-chroot /mnt cat /etc/locale.conf
arch-chroot /mnt cat /etc/vconsole.conf
arch-chroot /mnt cat /etc/hostname
arch-chroot /mnt grep '^GRUB_CMDLINE_LINUX_DEFAULT' /etc/default/grub
arch-chroot /mnt grep HOOKS /etc/mkinitcpio.conf
arch-chroot /mnt systemctl is-enabled plasmalogin.service
arch-chroot /mnt systemctl is-enabled grub-btrfsd.service
arch-chroot /mnt snapper --no-dbus -c root list
```

Powinno wyjść mniej więcej:

- `LANG=en_US.UTF-8`
- `KEYMAP=pl`
- hostname `arch`
- `resume=UUID=...`
- `resume` w HOOKS
- `plasmalogin.service` = `enabled`
- `grub-btrfsd.service` = `enabled`
- snapshot `fresh-install`

---

# 15. Restart

```bash
umount -R /mnt
swapoff /dev/nvme0n1p2
reboot
```

Wyjmij pendrive instalacyjny.

---

# 16. Kontrola po pierwszym starcie

Po zalogowaniu do zainstalowanego systemu:

```bash
cat /proc/cmdline
swapon --show
snapper -c root list
systemctl status grub-btrfsd.service --no-pager
grep -n "submenu 'Arch Linux snapshots'" /boot/grub/grub.cfg
```

To potwierdza:

- aktywne `resume=UUID=...`
- aktywny swap
- działającego Snappera
- działające submenu snapshotów w GRUB-ie

---

# 17. Test hibernacji

Najprostszy test:

```bash
sudo systemctl hibernate
```

---

# 18. Snapshot po udanej instalacji

Po pierwszym poprawnym starcie dobrze od razu zrobić snapshot bazowy:

```bash
sudo snapper -c root create -d "working-base"
sudo snapper -c root list
```

---

# 19. Opcjonalnie: wymuszenie polskiego układu klawiatury dla GUI

Jeśli chcesz dodatkowo ustawić układ X11/GUI na polski:

```bash
sudo localectl --no-convert set-x11-keymap pl
```

---

# 20. Stan końcowy systemu

Po wykonaniu wszystkich kroków system ma:

- Arch Linux
- Btrfs
- snapshoty przez Snapper
- automatyczne snapshoty pacmana przez `snap-pac`
- GRUB z menu snapshotów dzięki `grub-btrfs`
- działającą hibernację
- minimalne KDE Plasma
- Plasma Login Manager
- hostname `arch`
- `en_US.UTF-8`
- polską klawiaturę
- strefę `Europe/Warsaw`
- `neovim`
- `wget`, `git`, `curl`, `btop`, `fastfetch`
