# Arch Linux na Dell Latitude 5421 — LUKS2 + TPM2 PIN + Btrfs + Snapper + grub-btrfs + hibernacja + minimalne KDE

## 1. Start z Arch ISO

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

```bash
loadkeys pl
timedatectl set-ntp true
```

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
Reszta komend przez ssh.

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

Sprawdzenie:

```bash
findmnt -R /mnt -o TARGET,SOURCE,FSTYPE,OPTIONS
```

---

## 6. Minimalny pacstrap

# Wymagane przed pacstrap, bo instalacja kernela odpala mkinitcpio,
# a hook sd-vconsole/keymap szuka /etc/vconsole.conf w instalowanym systemie.

```bash
mkdir -p /mnt/etc

cat > /mnt/etc/vconsole.conf <<'EOF'
KEYMAP=pl
EOF
```

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
  snapper grub-btrfs \
  plasma-desktop plasma-login-manager \
  plasma-nm plasma-pa powerdevil bluedevil kscreen \
  xdg-desktop-portal-kde \
  breeze-gtk kde-gtk-config \
  noto-fonts noto-fonts-emoji ttf-dejavu \
  konsole dolphin
```
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

## 17.1 Instalacja snap-pac po konfiguracji Snappera

`snap-pac` instalujemy dopiero po utworzeniu konfiguracji Snappera dla `/` i `/home`.  
Nie instalować `snap-pac` w `pacstrap`, bo jego hooki pacmana mogą uruchomić się zanim Snapper będzie gotowy.

```bash
pacman -S --needed snap-pac
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

# 25. Po instalacji — dodatkowa konfiguracja z poprzedniego README, poprawiona pod aktualny minimalny system

Poprzednia instrukcja miała dużo kroków „po czystym systemie”: Plymouth, AUR, Brave, Brother, iPhone, AirPods, Xbox pad, DPTF throttling, firmware, firewall, narzędzia itd.  
Nie wrzucamy tego wszystkiego do `pacstrap`, bo obecny cel to minimalny system. Te rzeczy instalujesz dopiero po pierwszym poprawnym uruchomieniu, najlepiej z ręcznymi snapshotami przed większymi zmianami.

Przed większym blokiem zmian:

```bash
sudo snapper -c root create --description "before post-install tweaks"
sudo snapper -c home create --description "home before post-install tweaks"
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 25.1 Pakiety bazowe po instalacji, opcjonalne

Jeśli chcesz wrócić do wygody ze starego README, ale bez pełnego `plasma-meta`, możesz doinstalować tylko potrzebne rzeczy:

```bash
sudo pacman -S --needed \
  bash-completion btop fastfetch openssh playerctl \
  zip unzip unrar p7zip \
  exfatprogs dosfstools mtools \
  usbutils lsof net-tools smartmontools traceroute \
  wireguard-tools networkmanager-openvpn \
  firewalld
```

Włączenie usług, jeśli ich używasz:

```bash
sudo systemctl enable --now sshd
sudo systemctl enable --now firewalld
sudo systemctl enable --now fstrim.timer
```

`fstrim.timer` warto mieć na SSD/NVMe.

---

## 25.2 Kodeki multimedialne

Minimalny system ma `ffmpeg` jako zależność części KDE/multimediów, ale jeśli chcesz pełniejsze kodeki:

```bash
sudo pacman -S --needed \
  ffmpeg gst-plugins-base gst-plugins-good gst-plugins-bad gst-plugins-ugly gst-libav
```

`libdvdcss` instaluj tylko jeśli faktycznie potrzebujesz odtwarzania szyfrowanych DVD:

```bash
sudo pacman -S --needed libdvdcss
```

---

## 25.3 Firefox, Thunderbird, Brave

Firefox i Thunderbird z repo:

```bash
sudo pacman -S --needed firefox thunderbird
```

Brave z AUR, więc najpierw `yay`.

---

## 25.4 `base-devel` i `yay`

Do AUR potrzebujesz `base-devel`.

```bash
sudo pacman -S --needed base-devel
```

Instalacja `yay`:

```bash
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

Opcjonalny katalog na klony/AUR/własne rzeczy:

```bash
mkdir -p ~/.gc
```

Pakiety AUR ze starego README, jeśli nadal ich chcesz:

```bash
yay -S brave-bin brother-dcp-b7520dw brscan4 brscan-skey xpadneo-dkms plymouth-theme-arch-breeze-git
```

Uwaga: `xpadneo-dkms` wymaga `dkms` i nagłówków kernela. `linux-headers` już był w bazowej instalacji, ale jeśli go nie masz:

```bash
sudo pacman -S --needed linux-headers dkms
```

Po instalacji `xpadneo-dkms` najlepiej zrobić restart.

---

## 25.5 Plymouth i splash screen

W bazowej instalacji celowo nie było Plymouth, bo minimalny system i hibernacja/LUKS/TPM powinny najpierw działać bez upiększeń.

Instalacja Plymouth:

```bash
sudo pacman -S --needed plymouth plymouth-kcm
```

Jeśli używasz motywu z AUR:

```bash
yay -S plymouth-theme-arch-breeze-git
```

Ustawienie motywu:

```bash
sudo plymouth-set-default-theme -R arch-breeze
```

Powrót do zwykłego Breeze:

```bash
sudo pacman -S --needed breeze-plymouth
sudo plymouth-set-default-theme -R breeze
```

Dodanie `splash` do GRUB:

```bash
sudo sed -i 's/\(GRUB_CMDLINE_LINUX_DEFAULT="[^"]*\)"/\1 splash"/' /etc/default/grub
sudo sed -i 's/  */ /g' /etc/default/grub
```

Dodanie hooka `plymouth` do `mkinitcpio` przy obecnym układzie `systemd + sd-encrypt`:

```bash
sudo sed -i 's/\<sd-vconsole\>/sd-vconsole plymouth/' /etc/mkinitcpio.conf
sudo sed -i 's/\(plymouth \)\+/plymouth /g' /etc/mkinitcpio.conf
grep '^HOOKS=' /etc/mkinitcpio.conf
```

Oczekiwany układ z Plymouth:

```text
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole plymouth block sd-encrypt filesystems fsck)
```

Przebudowa:

```bash
sudo mkinitcpio -P
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Uwaga: przy LUKS + TPM2 PIN najpierw upewnij się, że zwykły boot i hibernacja działają. Plymouth dodawaj dopiero później.

---

## 25.6 Motyw GRUB Breeze

Jeśli chcesz motyw GRUB Breeze:

```bash
sudo pacman -S --needed breeze-grub
```

Ustawienie:

```bash
grep -q '^GRUB_THEME=' /etc/default/grub \
  && sudo sed -i 's|^GRUB_THEME=.*|GRUB_THEME="/usr/share/grub/themes/breeze/theme.txt"|' /etc/default/grub \
  || echo 'GRUB_THEME="/usr/share/grub/themes/breeze/theme.txt"' | sudo tee -a /etc/default/grub

sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 25.7 Wymuszenie polskiego układu klawiatury dla GUI

Systemowo masz `KEYMAP=pl`, ale dla GUI/Plasma możesz dodatkowo ustawić:

```bash
sudo localectl --no-convert set-x11-keymap pl
```

Sprawdzenie:

```bash
localectl status
```

---

## 25.8 Dell Latitude 5421 — fix DPTF throttling po hibernacji / monitorach USB-C

Stare README miało fix na przypadek, gdy Dell Latitude 5421 po zewnętrznych monitorach USB-C/Thunderbolt lub po hibernacji dławi CPU do około 200 MHz.

Utwórz usługę:

```bash
sudo tee /etc/systemd/system/fix-dptf-throttle.service << 'EOF'
[Unit]
Description=Unbind DPTF proc_thermal after resume
After=hibernate.target suspend.target hybrid-sleep.target suspend-then-hibernate.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'sleep 2 && echo 0000:00:04.0 > /sys/bus/pci/drivers/proc_thermal/unbind 2>/dev/null || true'

[Install]
WantedBy=hibernate.target suspend.target hybrid-sleep.target suspend-then-hibernate.target
EOF

sudo systemctl enable fix-dptf-throttle.service
```

Diagnostyka, gdy CPU znowu spada do bardzo niskich taktowań:

```bash
cat /proc/cpuinfo | grep "MHz" | head -4
```

Ręczny fix:

```bash
sudo sh -c 'echo 0000:00:04.0 > /sys/bus/pci/drivers/proc_thermal/unbind'
```

Uwaga: po odpięciu DPTF kernel nadal ma własne zabezpieczenia termiczne, ale po tej zmianie warto obserwować temperatury przez kilka dni, np. w `btop`.

---

## 25.9 Drukarka i skaner Brother DCP-B7520DW

Jeżeli chcesz obsługę drukarki/skanera Brother ze starego README, doinstaluj:

```bash
sudo pacman -S --needed cups system-config-printer sane simple-scan avahi nss-mdns
sudo systemctl enable --now cups.service
sudo systemctl enable --now avahi-daemon.service
```

Pakiety Brother z AUR:

```bash
yay -S brother-dcp-b7520dw brscan4 brscan-skey
```

Konfiguracja skanera po IP:

```bash
sudo brsaneconfig4 -a name=Brother model=DCP-B7520DW ip=192.168.1.100
scanimage -L
```

Jeśli `scanimage -L` pokazuje urządzenie, skanowanie jest gotowe.

---

## 25.10 Brave — obejście wolnego startu na KDE

Jeśli Brave uruchamia się długo przez integrację z portfelem/secret service, możesz ustawić `--password-store=basic`.

```bash
mkdir -p ~/.local/share/applications
cp /usr/share/applications/brave-browser.desktop ~/.local/share/applications/

sed -i 's|^Exec=.*|Exec=brave --password-store=basic %U|' ~/.local/share/applications/brave-browser.desktop
update-desktop-database ~/.local/share/applications 2>/dev/null || true
kbuildsycoca6
```

Od tej chwili Brave z menu aplikacji użyje `--password-store=basic`.

---

## 25.11 iPhone — parowanie i dostęp do plików

Pakiety:

```bash
sudo pacman -S --needed gvfs gvfs-afc gvfs-gphoto2 libimobiledevice usbmuxd ifuse
```

Podłącz iPhone'a kablem, odblokuj ekran i wykonaj:

```bash
idevicepair pair
idevicepair validate
```

Na iPhonie potwierdź „Ufaj temu komputerowi”.

Jeśli Dolphin pokazuje błąd `Unhandled lockdownd code '-5'`:

```bash
sudo rm -rf /var/lib/lockdown/*
sudo systemctl restart usbmuxd
```

Odłącz/podłącz iPhone'a, odblokuj ekran i ponów:

```bash
idevicepair pair
idevicepair validate
```

Montaż ręczny:

```bash
mkdir -p ~/iPhone
ifuse ~/iPhone
```

Zdjęcia:

```text
~/iPhone/DCIM
```

Odmontowanie:

```bash
fusermount -u ~/iPhone
```

---

## 25.12 Bluetooth — AirPods Pro i sterowanie mediami

Do sterowania mediami z przycisków słuchawek Bluetooth:

```bash
systemctl --user enable --now mpris-proxy.service
```

Test MPRIS:

```bash
playerctl -l
playerctl play-pause
```

Jeśli nie masz `playerctl`:

```bash
sudo pacman -S --needed playerctl
```

---

## 25.13 Bluetooth — Xbox pad

Pakiet z AUR:

```bash
yay -S xpadneo-dkms
```

Po instalacji:

```bash
sudo reboot
```

---

## 25.14 Aktualizacje firmware Dell / LVFS

W bazowym minimalnym systemie `fwupd` nie był instalowany. Jeśli chcesz aktualizacje BIOS/UEFI/Thunderbolt/NVMe przez LVFS:

```bash
sudo pacman -S --needed fwupd
sudo systemctl enable --now fwupd-refresh.timer
```

Sprawdzenie:

```bash
fwupdmgr refresh
fwupdmgr get-updates
```

Aktualizacja:

```bash
fwupdmgr update
```

---

## 25.15 Ukrywanie niechcianych wpisów w menu KDE

Nie usuwać pakietów typu `avahi`, `qt6-tools`, `v4l-utils`, jeśli są zależnościami. Lepiej ukryć wpisy `.desktop`.

Sprawdzenie właścicieli:

```bash
pacman -Qo /usr/share/applications/*avahi* 2>/dev/null
pacman -Qo /usr/share/applications/*qt* 2>/dev/null
pacman -Qo /usr/share/applications/*qv4l* 2>/dev/null
pacman -Qo /usr/share/applications/*emoji* 2>/dev/null
```

Sprawdzenie zależności:

```bash
pacman -Qi avahi qt6-tools v4l-utils plasma-workspace | grep -E 'Name|Required By|Optional For'
```

Ukrycie Avahi, qv4l2 i Emoji Selector:

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


---

## 25.17 Bezpieczne SSH tylko w domu

Ten wariant został ustawiony po instalacji. Cel:

```text
sshd działa
root przez SSH zablokowany
logowanie hasłem wyłączone
logowanie tylko kluczem SSH
firewall wpuszcza SSH tylko z domowej podsieci LAN
publiczna strefa firewalld nie ma otwartego SSH
```

### Instalacja narzędzi i firewalla

Jeśli nie były jeszcze zainstalowane:

```bash
sudo pacman -S --needed \
  bash-completion btop fastfetch openssh playerctl \
  zip unzip p7zip \
  exfatprogs dosfstools \
  usbutils lsof smartmontools traceroute \
  wireguard-tools \
  firewalld
```

Włączenie `sshd` i `firewalld`:

```bash
sudo systemctl enable --now sshd
sudo systemctl enable --now firewalld
```

Sprawdzenie:

```bash
systemctl is-enabled sshd
systemctl is-active sshd

sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all
```

### Firewall: SSH tylko z domowej podsieci

Najpierw sprawdź adres IP laptopa i podsieć:

```bash
ip -4 addr show wlp0s20f3
ip route
```

Przykład:

```text
192.168.1.123/24
```

Wtedy domowa podsieć to:

```text
192.168.1.0/24
```

W poniższych komendach podmień `192.168.1.0/24`, jeśli Twoja sieć ma inny zakres.

Usuń SSH ze strefy `public`:

```bash
sudo firewall-cmd --permanent --zone=public --remove-service=ssh
```

Utwórz osobną strefę dla domowego SSH:

```bash
sudo firewall-cmd --permanent --new-zone=home-ssh
```

Dodaj domową podsieć:

```bash
sudo firewall-cmd --permanent --zone=home-ssh --add-source=192.168.1.0/24
```

Pozwól na SSH tylko w tej strefie:

```bash
sudo firewall-cmd --permanent --zone=home-ssh --add-service=ssh
```

Przeładuj firewalla:

```bash
sudo firewall-cmd --reload
```

Sprawdź:

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
sudo firewall-cmd --zone=home-ssh --list-all
```

Oczekiwany wynik logiczny:

```text
public:
  interfaces: wlp0s20f3
  services: dhcpv6-client
  brak ssh

home-ssh:
  sources: 192.168.1.0/24
  services: ssh
```

Przykład poprawnego stanu:

```text
home-ssh
  sources: 192.168.1.0/24
public (default)
  interfaces: wlp0s20f3

public (default, active)
  services: dhcpv6-client

home-ssh (active)
  sources: 192.168.1.0/24
  services: ssh
```

### Hardening SSH — etap bezpieczny, jeszcze z hasłem

Najpierw ustaw konfigurację tak, żeby nie odciąć się przed dodaniem klucza:

```bash
sudo mkdir -p /etc/ssh/sshd_config.d

sudo tee /etc/ssh/sshd_config.d/99-hardening.conf >/dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication yes
KbdInteractiveAuthentication no
PubkeyAuthentication yes

X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
PermitTunnel no

MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2

AllowUsers pietryszak
EOF

sudo sshd -t
sudo systemctl restart sshd
```

Sprawdzenie realnej konfiguracji:

```bash
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|x11forwarding|allowtcpforwarding|allowagentforwarding|permittunnel|allowusers'
```

Na tym etapie oczekiwane:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication yes
kbdinteractiveauthentication no
x11forwarding no
allowtcpforwarding no
allowagentforwarding no
allowusers pietryszak
permittunnel no
```

### Dodanie klucza SSH

Na komputerze, z którego będziesz się łączyć:

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519_arch
```

Skopiuj klucz na laptopa z Archem:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_arch.pub pietryszak@IP_ARCHA
```

Test:

```bash
ssh -i ~/.ssh/id_ed25519_arch pietryszak@IP_ARCHA
```

Jeżeli klucz działa, można wyłączyć logowanie hasłem.

Jeśli nie używasz `ssh-copy-id`, dodaj klucz lokalnie na Archu do:

```text
/home/pietryszak/.ssh/authorized_keys
```

i ustaw prawa:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Finalnie: SSH tylko kluczem, bez hasła

Dopiero po potwierdzeniu, że logowanie kluczem działa:

```bash
sudo sed -i 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config.d/99-hardening.conf
sudo sshd -t
sudo systemctl restart sshd
```

Sprawdzenie:

```bash
sudo sshd -T | grep -Ei 'passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|permitrootlogin|allowusers'
```

Oczekiwane:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
allowusers pietryszak
```

Finalny stan bezpieczeństwa:

```text
SSH działa tylko dla użytkownika pietryszak
root login przez SSH jest zablokowany
hasła przez SSH są zablokowane
działają tylko klucze SSH
firewalld wpuszcza SSH tylko z 192.168.1.0/24
publiczna strefa nie ma otwartego SSH
```

### Snapshot po zabezpieczeniu SSH

Po potwierdzeniu, że logowanie kluczem działa:

```bash
sudo snapper -c root create --description "secure ssh key-only home LAN"
sudo snapper -c home create --description "home after secure ssh key-only"
sudo grub-mkconfig -o /boot/grub/grub.cfg
```


## 25.18 Snapshot po post-install

Po większych zmianach po instalacji:

```bash
sudo snapper -c root create --description "after post-install tweaks"
sudo snapper -c home create --description "home after post-install tweaks"
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Jeśli snapshot ma zostać na dłużej, oznacz go jako important:

```bash
sudo snapper -c root list
sudo snapper -c root modify --cleanup-algorithm important NUMER

sudo snapper -c home list
sudo snapper -c home modify --cleanup-algorithm important NUMER
```

---

# 26. Stan końcowy systemu — wariant aktualny

Po wykonaniu bazowej instalacji system ma:

- Arch Linux
- LUKS2 na `/dev/nvme0n1p2`
- TPM2 + PIN
- Btrfs na `/dev/mapper/cryptroot`
- swapfile 40 GiB na `/swap/swapfile`
- działające `resume_offset`
- Snapper dla `/`
- Snapper dla `/home`
- `snap-pac`
- GRUB z snapshotami przez `grub-btrfs`
- hibernację
- minimalne KDE Plasma
- Plasma Login Manager przez `plasmalogin.service`
- NetworkManager
- Bluetooth
- PipeWire + WirePlumber
- hostname `arch`
- `en_US.UTF-8`
- polską klawiaturę
- strefę `Europe/Warsaw`
- `neovim`
- `konsole`
- `dolphin`
- fonty Noto/Noto Emoji/DejaVu

Po wykonaniu opcjonalnych kroków post-install możesz dodatkowo mieć:

- Plymouth splash
- motyw GRUB Breeze
- `yay`
- Brave
- Firefox
- Thunderbird
- Brother DCP-B7520DW
- obsługę iPhone'a
- AirPods Pro MPRIS
- Xbox pad przez `xpadneo`
- aktualizacje firmware przez `fwupd`
- firewall
- bezpieczne SSH tylko z domowej sieci LAN, tylko kluczem
- kodeki
- narzędzia diagnostyczne
- fix DPTF throttling na Dell Latitude 5421


