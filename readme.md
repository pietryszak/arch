# Arch Linux na Dell Latitude 5421 — LUKS2 + TPM2 PIN + Btrfs + Snapper + grub-btrfs + hibernacja + minimalne KDE

Instrukcja powstała po faktycznej instalacji i zawiera poprawki błędów, które wyszły w trakcie.  
Docelowy system:

- Arch Linux
- UEFI
- LUKS2 na root
- TPM2 + PIN do odblokowania LUKS
- Btrfs z subvolumami
- Snapper dla `/` i `/home`
- `grub-btrfs` z bootowalnymi snapshotami w GRUB
- hibernacja na Btrfs swapfile
- minimalne KDE Plasma
- Plasma Login Manager, czyli `plasmalogin`, bez SDDM
- NetworkManager, Bluetooth, PipeWire/WirePlumber
- bez `nano`, bez `vim`, używany `neovim`
- bez `fwupd`, bez Flatpak, bez pełnego KDE Gear

---

## 0. Założenia

Dysk:

```bash
/dev/nvme0n1
```

Partycje:

```text
/dev/nvme0n1p1    EFI        1G      FAT32
/dev/nvme0n1p2    cryptroot  reszta  LUKS2
```

W środku LUKS:

```text
/dev/mapper/cryptroot  Btrfs
```

Hostname:

```text
arch
```

Użytkownik:

```text
pietryszak
```

Subvolumy Btrfs:

```text
@                  /
@home              /home
@snapshots         /.snapshots
@home_snapshots    /home/.snapshots
@log               /var/log
@cache             /var/cache
@pkg               /var/cache/pacman/pkg
@tmp               /var/tmp
@spool             /var/spool
@opt               /opt
@swap              /swap
@libvirt           /var/lib/libvirt
@mozilla           /home/pietryszak/.mozilla
@brave             /home/pietryszak/.config/BraveSoftware
@thunderbird       /home/pietryszak/.thunderbird
@ssh               /home/pietryszak/.ssh
@gnupg             /home/pietryszak/.gnupg
```

---

## 1. Start z Arch ISO

```bash
loadkeys pl
timedatectl set-ntp true
```

Sprawdzenie UEFI:

```bash
ls /sys/firmware/efi/efivars
```

Połączenie Wi-Fi z live ISO:

```bash
iwctl
```

W `iwctl`:

```text
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect NAZWA_WIFI
exit
```

Test:

```bash
ping -c 3 archlinux.org
```

---

## 2. Partycjonowanie

Uwaga: to kasuje dysk.

```bash
DISK=/dev/nvme0n1

sgdisk --zap-all "$DISK"
wipefs -af "$DISK"

sgdisk -n 1:0:+1G -t 1:ef00 -c 1:"EFI" "$DISK"
sgdisk -n 2:0:0   -t 2:8309 -c 2:"CRYPTROOT" "$DISK"

partprobe "$DISK"
```

Format EFI:

```bash
mkfs.fat -F32 -n EFI ${DISK}p1
```

---

## 3. LUKS2

```bash
cryptsetup luksFormat --type luks2 ${DISK}p2
cryptsetup open ${DISK}p2 cryptroot
```

Sprawdzenie:

```bash
lsblk -f
```

---

## 4. Btrfs i subvolumy

```bash
mkfs.btrfs -f -L ARCH /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt
```

Tworzenie subvolumów:

```bash
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
btrfs subvolume create /mnt/@brave
btrfs subvolume create /mnt/@thunderbird
btrfs subvolume create /mnt/@ssh
btrfs subvolume create /mnt/@gnupg
```

```bash
umount /mnt
```

---

## 5. Montowanie systemu

Opcje:

```bash
BTRFS_OPTS="noatime,compress=zstd:3,ssd,space_cache=v2,discard=async"
BTRFS_SWAP_OPTS="noatime,ssd,space_cache=v2,discard=async"
```

Root:

```bash
mount -o "$BTRFS_OPTS",subvol=@ /dev/mapper/cryptroot /mnt
```

Katalogi:

```bash
mkdir -p /mnt/{boot,home,.snapshots,var/log,var/cache/pacman/pkg,var/tmp,var/spool,opt,swap,var/lib/libvirt}
```

Mounty:

```bash
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
```

Ważna poprawka z instalacji: po zamontowaniu `/mnt/home` trzeba dopiero utworzyć `/mnt/home/.snapshots`, bo wcześniejszy katalog może zostać przykryty mountem. To samo dotyczy `/mnt/var/cache/pacman/pkg` po zamontowaniu `/mnt/var/cache`.

Sprawdzenie:

```bash
findmnt -R /mnt -o TARGET,SOURCE,FSTYPE,OPTIONS
```

---

## 6. Minimalny pacstrap

Finalna użyta idea: minimum KDE, ale używalne — z terminalem i menedżerem plików.  
Bez `sddm`, bez `nano`, bez `vim`, bez `fwupd`, bez Flatpak.

```bash
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
  snapper snap-pac grub-btrfs \
  plasma-desktop plasma-login-manager \
  plasma-nm plasma-pa powerdevil bluedevil kscreen \
  xdg-desktop-portal-kde \
  breeze-gtk kde-gtk-config \
  noto-fonts noto-fonts-emoji ttf-dejavu \
  konsole dolphin
```

Uwagi:

- `plasma-login-manager` daje usługę `plasmalogin.service`.
- Nie instalować `sddm`.
- `qt6-tools`, `avahi`, `v4l-utils`, `emoji selector` mogą potem pojawić się w menu jako „śmieci”, ale są zależnościami ważnych komponentów:
  - `avahi` wymagane m.in. przez `pipewire-pulse`
  - `qt6-tools` wymagane przez `kwin` i `plasma-workspace`
  - `v4l-utils` wymagane przez `ffmpeg`
  - emoji selector jest częścią `plasma-desktop`
- Tych pakietów nie usuwać. Ewentualnie ukryć wpisy `.desktop`.

---

## 7. fstab i chroot

```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```

Hostname:

```bash
echo arch > /etc/hostname

cat > /etc/hosts <<'EOF'
127.0.0.1 localhost
::1       localhost
127.0.1.1 arch.localdomain arch
EOF
```

Locale i konsola:

```bash
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
```

Ważna poprawka z instalacji: jeśli `mkinitcpio` przy `pacstrap` krzyczy o braku `/etc/vconsole.conf`, to po utworzeniu powyższego pliku trzeba ponownie wykonać:

```bash
mkinitcpio -P
```

Warning o `qat_6xxx` można zignorować na Dell Latitude 5421.

---

## 8. Użytkownik i sudo

```bash
passwd

useradd -m -G wheel,audio,video,input,storage,lp,network,power -s /bin/bash pietryszak
passwd pietryszak

grep -q '^%wheel ALL=(ALL:ALL) ALL' /etc/sudoers || sed -i 's/^# %wheel ALL=(ALL:ALL) ALL/%wheel ALL=(ALL:ALL) ALL/' /etc/sudoers
```

Sprawdzenie:

```bash
id pietryszak
passwd -S root
passwd -S pietryszak
```

---

## 9. Poprawka `/swap` w fstab

Po `genfstab` trzeba usunąć `compress=zstd:3` z wpisu `/swap`.

```bash
sed -i '\|[[:space:]]/swap[[:space:]]| s/,compress=zstd:3//' /etc/fstab
grep -E '[[:space:]]/swap[[:space:]]' /etc/fstab
```

Poprawny wpis `/swap` ma wyglądać mniej więcej tak:

```text
UUID=... /swap btrfs rw,noatime,ssd,discard=async,space_cache=v2,subvol=/@swap 0 0
```

Bez `compress=zstd:3`.

---

## 10. Swapfile pod hibernację — ważna poprawiona metoda

W trakcie instalacji wyszło, że `btrfs filesystem mkswapfile` może wywalić:

```text
ERROR: cannot set NOCOW flag: Invalid argument
```

U nas dodatkowo problem pojawił się po ustawieniu:

```bash
btrfs property set /swap compression none
```

To ustawiło atrybut `m` i blokowało `chattr +C`.

Finalna poprawiona metoda:

```bash
swapoff -a 2>/dev/null || true
rm -f /swap/swapfile

# usuń problematyczny atrybut m/no-compress, jeśli wcześniej został ustawiony
chattr -m /swap 2>/dev/null || true
btrfs property set /swap compression "" 2>/dev/null || true

touch /swap/swapfile
chattr +C /swap/swapfile
lsattr /swap/swapfile
```

Oczekiwane:

```text
---------------C------ /swap/swapfile
```

Tworzenie pliku 40 GiB:

```bash
dd if=/dev/zero of=/swap/swapfile bs=1M count=40960 status=progress
chmod 600 /swap/swapfile
mkswap /swap/swapfile
swapon /swap/swapfile

grep -q '^/swap/swapfile ' /etc/fstab || echo "/swap/swapfile none swap defaults 0 0" >> /etc/fstab

swapon --show
```

Oczekiwane:

```text
NAME           TYPE SIZE USED PRIO
/swap/swapfile file  40G   0B   -1
```

Offset do hibernacji:

```bash
btrfs inspect-internal map-swapfile -r /swap/swapfile
```

W tej instalacji wyszło:

```text
1549568
```

Nie wpisywać na ślepo — na innej instalacji policzyć ponownie.

---

## 11. mkinitcpio pod LUKS + systemd initramfs

Ustawienie hooków bez ręcznej edycji:

```bash
sed -i 's/^HOOKS=.*/HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)/' /etc/mkinitcpio.conf

grep '^HOOKS=' /etc/mkinitcpio.conf
mkinitcpio -P
```

Oczekiwane:

```text
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)
```

---

## 12. UUID-y, resume offset, crypttab.initramfs

```bash
CRYPT_UUID=$(blkid -s UUID -o value /dev/nvme0n1p2)
ROOT_UUID=$(findmnt -no UUID /)
RESUME_OFFSET=$(btrfs inspect-internal map-swapfile -r /swap/swapfile)

echo "CRYPT_UUID=$CRYPT_UUID"
echo "ROOT_UUID=$ROOT_UUID"
echo "RESUME_OFFSET=$RESUME_OFFSET"
```

W tej instalacji było:

```text
CRYPT_UUID=bc4080e7-6014-4803-ae83-b86f8377bced
ROOT_UUID=572457ed-c62c-4a3f-8d14-891559bd3ea2
RESUME_OFFSET=1549568
```

`/etc/crypttab.initramfs`:

```bash
cat > /etc/crypttab.initramfs <<EOF
cryptroot UUID=${CRYPT_UUID} none tpm2-device=auto
EOF

cat /etc/crypttab.initramfs
```

---

## 13. GRUB kernel params

```bash
sed -i "s|^GRUB_CMDLINE_LINUX_DEFAULT=.*|GRUB_CMDLINE_LINUX_DEFAULT=\"quiet loglevel=3 rd.luks.name=${CRYPT_UUID}=cryptroot root=/dev/mapper/cryptroot rootflags=subvol=@ resume=UUID=${ROOT_UUID} resume_offset=${RESUME_OFFSET}\"|" /etc/default/grub

grep '^GRUB_CMDLINE_LINUX_DEFAULT' /etc/default/grub
```

Instalacja GRUB:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ARCH
grub-mkconfig -o /boot/grub/grub.cfg
```

Ostrzeżenie o `os-prober` można zignorować, jeśli nie ma dual-boota.

---

## 14. TPM2 + PIN

Sprawdzenie TPM:

```bash
systemd-cryptenroll --tpm2-device=list
```

W tej instalacji:

```text
/dev/tpmrm0 STM0125:00 tpm_tis
```

Dodanie TPM2 + PIN:

```bash
systemd-cryptenroll /dev/nvme0n1p2 --tpm2-device=auto --tpm2-with-pin=yes
```

Sprawdzenie:

```bash
systemd-cryptenroll /dev/nvme0n1p2
```

Oczekiwane:

```text
SLOT TYPE
   0 password
   1 tpm2
```

Po enrolowaniu:

```bash
mkinitcpio -P
grub-mkconfig -o /boot/grub/grub.cfg
```

### Opcjonalnie: TPM bez PIN

Da się, ale jest mniej bezpieczne.  
Zmiana po instalacji:

```bash
sudo systemd-cryptenroll /dev/nvme0n1p2 --wipe-slot=tpm2
sudo systemd-cryptenroll /dev/nvme0n1p2 --tpm2-device=auto
sudo systemd-cryptenroll /dev/nvme0n1p2
sudo mkinitcpio -P
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Rekomendacja: na laptopie zostawić TPM2 + PIN.

---

## 15. Usługi

Włączenie podstaw:

```bash
systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable plasmalogin.service
```

Sprawdzenie login managera:

```bash
systemctl list-unit-files | grep -Ei 'plasma|login'
```

Oczekiwane:

```text
plasmalogin.service
```

---

## 16. Snapper root — poprawiona metoda z `--no-dbus`

W chroot Snapper bez `--no-dbus` wywalał:

```text
Failure (org.freedesktop.DBus.Error.ServiceUnknown)
```

Dlatego w chroot używać `snapper --no-dbus`.

Root:

```bash
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
```

---

## 17. Snapper home — poprawiona metoda z `--no-dbus`

```bash
umount /home/.snapshots 2>/dev/null || true
rm -rf /home/.snapshots

snapper --no-dbus -c home create-config /home

btrfs subvolume delete /home/.snapshots

mkdir /home/.snapshots
mount /home/.snapshots

chmod 750 /home/.snapshots

snapper --no-dbus -c home create --description "fresh encrypted home snapshot"
snapper --no-dbus -c home list
```

---

## 18. Timery Snappera i grub-btrfs

```bash
systemctl enable snapper-timeline.timer
systemctl enable snapper-cleanup.timer
systemctl enable grub-btrfsd.service
```

Po pierwszych snapshotach:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

Oczekiwane:

```text
Detecting snapshots ...
Found snapshot: ... @snapshots/1/snapshot ...
```

---

## 19. Finalny check przed rebootem

```bash
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
```

Ostatnie generowanie:

```bash
mkinitcpio -P
grub-mkconfig -o /boot/grub/grub.cfg
```

Wyjście:

```bash
exit
```

Jeśli `/mnt/swap` jest busy:

```bash
swapoff /mnt/swap/swapfile 2>/dev/null || swapoff -a
umount -R /mnt
reboot
```

---

## 20. Po pierwszym bootowaniu

Po zalogowaniu:

```bash
cat /proc/cmdline
swapon --show
findmnt -R /
sudo snapper -c root list
sudo snapper -c home list
cat /sys/power/state
```

Jeśli jest:

```text
freeze mem disk
```

test hibernacji:

```bash
systemctl hibernate
```

---

## 21. Baseline snapshot po pierwszych zmianach

Po zainstalowaniu i drobnych zmianach wykonano baseline:

```bash
sudo snapper -c root create --description "baseline before system tweaks"
sudo snapper -c home create --description "baseline home before tweaks"
```

W tej instalacji:

```text
root snapshot: 8
home snapshot: 2
```

Oznaczenie jako important:

```bash
sudo snapper -c root modify --cleanup-algorithm important 8
sudo snapper -c home modify --cleanup-algorithm important 2
```

Odświeżenie GRUB:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Oczekiwane:

```text
Found snapshot: ... @snapshots/8/snapshot | single | baseline before system tweaks |
```

Uwaga: GRUB pokazuje snapshoty root. Snapshoty `/home` nie są bootowalne i nie będą w menu GRUB.

---

## 22. Ukrywanie niechcianych wpisów w menu KDE

Po instalacji w menu mogą pojawić się m.in.:

- Emoji Selector
- Avahi Zeroconf Browser
- Qt / V4L2 tools

Nie usuwać pakietów, jeśli są zależnościami. Sprawdzenie:

```bash
pacman -Qo /usr/share/applications/*avahi* 2>/dev/null
pacman -Qo /usr/share/applications/*qt* 2>/dev/null
pacman -Qo /usr/share/applications/*qv4l* 2>/dev/null
pacman -Qo /usr/share/applications/*emoji* 2>/dev/null

pacman -Qi avahi qt6-tools v4l-utils plasma-workspace | grep -E 'Name|Required By|Optional For'
```

W tej instalacji:

```text
avahi       -> Required By: libcups pipewire-pulse tinysparql
qt6-tools   -> Required By: aurorae kwin plasma-workspace
v4l-utils   -> Required By: ffmpeg
emoji       -> plasma-desktop
```

Czyli nie usuwać. Można ukryć wpisy:

```bash
mkdir -p ~/.local/share/applications

for f in \
  /usr/share/applications/avahi-discover.desktop \
  /usr/share/applications/qv4l2.desktop \
  /usr/share/applications/org.kde.plasma.emojier.desktop
do
  cp "$f" ~/.local/share/applications/
  grep -q '^NoDisplay=true' ~/.local/share/applications/"$(basename "$f")" || \
    echo 'NoDisplay=true' >> ~/.local/share/applications/"$(basename "$f")"
done

kbuildsycoca6
systemctl --user restart plasma-plasmashell.service
```

Przywrócenie:

```bash
rm ~/.local/share/applications/avahi-discover.desktop
rm ~/.local/share/applications/qv4l2.desktop
rm ~/.local/share/applications/org.kde.plasma.emojier.desktop
kbuildsycoca6
```

---

## 23. Typowe komendy po instalacji

Ręczny snapshot przed zmianami:

```bash
sudo snapper -c root create --description "before XYZ"
sudo snapper -c home create --description "home before XYZ"
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Lista snapshotów:

```bash
sudo snapper -c root list
sudo snapper -c home list
```

Oznaczenie ważnego snapshotu:

```bash
sudo snapper -c root modify --cleanup-algorithm important NUMER
sudo snapper -c home modify --cleanup-algorithm important NUMER
```

Aktualizacja GRUB po snapshotach:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Sprawdzenie hibernacji:

```bash
cat /sys/power/state
systemctl hibernate
```

Sprawdzenie TPM/LUKS:

```bash
sudo systemd-cryptenroll /dev/nvme0n1p2
```

Sprawdzenie kernel params:

```bash
cat /proc/cmdline
```

---

## 24. Rzeczy, które wyszły w trakcie i są ważne

1. `plasma-login-manager` nie daje usługi `plasma-login-manager.service`, tylko:

```text
plasmalogin.service
```

2. Nie instalować `sddm`, jeśli celem jest Plasma Login Manager.

3. Nie instalować `nano` ani `vim`, jeśli używany ma być tylko `neovim`.

4. `fwupd` nie jest wymagane do działania firmware. Do firmware wystarczy `linux-firmware`.

5. `linux-firmware` jest właściwym pakietem firmware w Archu. Nie trzeba osobno dodawać `linux-firmware-intel` i `linux-firmware-realtek`, jeśli używany jest meta-pakiet `linux-firmware`.

6. `avahi`, `qt6-tools`, `v4l-utils` i emoji selector wyglądają jak śmieci w menu, ale są zależnościami. Nie usuwać na ślepo.

7. W chroot Snapper robi błędy DBus. Używać:

```bash
snapper --no-dbus ...
```

8. Jeśli `/mnt/swap` jest busy przy odmontowaniu, to aktywny jest swapfile. Najpierw:

```bash
swapoff /mnt/swap/swapfile 2>/dev/null || swapoff -a
```

9. `grub-btrfs` wykrywa tylko snapshoty root. Snapshoty home nie będą w GRUB.

10. `os-prober` warning w GRUB jest normalny bez dual-boota.

11. Warning `qat_6xxx` przy `mkinitcpio` można zignorować na tym laptopie.

12. Swapfile na Btrfs wymaga `NOCOW`. Jeśli `btrfs filesystem mkswapfile` nie działa, użyć manualnej metody z `touch`, `chattr +C`, `dd`, `mkswap`.

---

## 25. Aktualizacja istniejącego repo

Obecna instrukcja w repo:

```text
https://github.com/pietryszak/arch
```

jest nieaktualna. Ten plik może zastąpić obecny `README.md` albo wejść jako:

```text
README.md
docs/arch-btrfs-luks-tpm-snapper.md
```

Proponowana nazwa:

```text
README.md
```
