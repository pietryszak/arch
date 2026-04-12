# Arch Linux na Dell Latitude 5421

## Btrfs + Snapper + hibernacja + GRUB snapshoty + pełne Plasma + Plasma Login Manager

Ten przewodnik opisuje czystą instalację Arch Linuksa na **Dell Latitude 5421** z:

* UEFI
* Btrfs
* snapshotami przez Snapper
* osobnymi snapshotami dla `/` i `/home`
* działającą hibernacją
* GRUB + grub-btrfs do wybierania snapshotów z menu startowego
* pełnym KDE Plasma przez `plasma-meta`
* Plasma Login Manager zamiast SDDM
* językiem systemu ustawionym na `en_US.UTF-8`
* polską klawiaturą
* strefą czasową `Europe/Warsaw`

Docelowa konfiguracja:

* dysk docelowy: `/dev/nvme0n1`
* pełne wyczyszczenie dysku
* brak szyfrowania
* użytkownik: `pietryszak`
* hostname: `arch`

Ten układ zostawia **`/boot` na Btrfs**, a partycję EFI montuje jako **`/boot/efi`**. Dzięki temu kernel i initramfs pozostają na Btrfs i lepiej pasują do rollbacku snapshotów.

---

# 1. Układ partycji

* `nvme0n1p1` — EFI — `1 GiB`
* `nvme0n1p2` — swap — `40 GiB`
* `nvme0n1p3` — Btrfs — reszta dysku

---

# 2. Układ subvolume

## Subvolume systemowe

* `@` -> `/`
* `@home` -> `/home`
* `@snapshots` -> `/.snapshots`
* `@home_snapshots` -> `/home/.snapshots`
* `@log` -> `/var/log`
* `@cache` -> `/var/cache`
* `@pkg` -> `/var/cache/pacman/pkg`
* `@tmp` -> `/var/tmp`
* `@spool` -> `/var/spool`
* `@opt` -> `/opt`
* `@libvirt` -> `/var/lib/libvirt`

## Subvolume użytkownika wyłączone z rollbacku `/home`

* `@mozilla` -> `/home/pietryszak/.mozilla`
* `@brave` -> `/home/pietryszak/.config/BraveSoftware`
* `@thunderbird` -> `/home/pietryszak/.thunderbird`
* `@gnupg` -> `/home/pietryszak/.gnupg`
* `@ssh` -> `/home/pietryszak/.ssh`

---

# 3. Ustawienia BIOS / UEFI

Przed uruchomieniem instalatora z pendrive ustaw:

* tryb bootowania: **UEFI**
* **Secure Boot: Off**
* jeśli Linux nie widzi dysku: tryb storage ustaw na **AHCI**

---

# 4. Start Arch ISO i połączenie z internetem

Po uruchomieniu Arch ISO:

```
timedatectl set-ntp true
```

Jeśli masz Ethernet:

```
ping -c 3 archlinux.org
```

Jeśli potrzebujesz Wi-Fi:

```
iwctl
```

W `iwctl`:

```
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "NAZWA_SIECI"
exit
```

Potem sprawdź połączenie:

```
ping -c 3 archlinux.org
```

---

# 5. Włączenie SSH na live ISO

Na live ISO ustaw hasło roota i uruchom SSH:

```
passwd
systemctl start sshd
ip -br a
```

Z drugiego komputera połącz się tak:

```
ssh root@ADRES_IP
```

---

# 6. Sprawdzenie trybu bootu i dysków

```
ls /sys/firmware/efi/efivars >/dev/null && echo UEFI_OK || echo NIE_UEFI
lsblk -e7 -o NAME,SIZE,TYPE,MODEL
free -h
lspci | grep -E "VGA|3D|Display"
```

Ten przewodnik zakłada, że:

* pendrive instalacyjny to `sda`
* dysk docelowy to `/dev/nvme0n1`

---

# 7. Partycjonowanie dysku

> **Uwaga:** ten krok kasuje cały `/dev/nvme0n1`.

```
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

# 8. Formatowanie i tworzenie subvolume Btrfs

```
mkfs.fat -F32 /dev/nvme0n1p1
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2
mkfs.btrfs -f /dev/nvme0n1p3
```

Tworzenie wszystkich subvolume na top-level Btrfs:

```
mount /dev/nvme0n1p3 /mnt

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
btrfs subvolume create /mnt/@libvirt
btrfs subvolume create /mnt/@mozilla
btrfs subvolume create /mnt/@brave
btrfs subvolume create /mnt/@thunderbird
btrfs subvolume create /mnt/@gnupg
btrfs subvolume create /mnt/@ssh

umount /mnt
```

---

# 9. Montowanie docelowego układu systemowego

Na tym etapie montujemy wszystko oprócz subvolume specyficznych dla użytkownika. Te zamontujemy dopiero po utworzeniu użytkownika.

Ważna kolejność:

* najpierw montujesz główne subvolume, takie jak `@home` i `@cache`
* dopiero potem tworzysz katalogi wewnątrz nich, takie jak `/home/.snapshots` i `/var/cache/pacman/pkg`
* na końcu montujesz `@home_snapshots` i `@pkg`

```
mount -o subvol=@,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt

mkdir -p /mnt/{boot/efi,home,.snapshots,opt}
mkdir -p /mnt/var/log
mkdir -p /mnt/var/cache
mkdir -p /mnt/var/tmp
mkdir -p /mnt/var/spool
mkdir -p /mnt/var/lib/libvirt

mount -o subvol=@home,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/home
mount -o subvol=@snapshots,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/.snapshots
mount -o subvol=@log,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/var/log
mount -o subvol=@cache,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/var/cache
mount -o subvol=@tmp,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/var/tmp
mount -o subvol=@spool,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/var/spool
mount -o subvol=@opt,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/opt
mount -o subvol=@libvirt,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/var/lib/libvirt

mkdir -p /mnt/home/.snapshots
mkdir -p /mnt/var/cache/pacman/pkg

mount -o subvol=@home_snapshots,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/home/.snapshots
mount -o subvol=@pkg,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/var/cache/pacman/pkg
mount /dev/nvme0n1p1 /mnt/boot/efi

findmnt -R /mnt
swapon --show
```

---

# 10. Instalacja pakietów

Instalujemy:

* bazę systemu
* GRUB
* sieć
* PipeWire
* pełne KDE Plasma przez `plasma-meta`
* Snapper
* grub-btrfs
* dodatkowe narzędzia

```
pacstrap -K /mnt \
  base linux linux-firmware intel-ucode btrfs-progs \
  grub efibootmgr \
  networkmanager openssh sudo neovim bash-completion wget git curl btop fastfetch man-db man-pages \
  pipewire-audio pipewire-alsa pipewire-pulse wireplumber sof-firmware \
  plasma-meta \
  dolphin konsole \
  xdg-user-dirs xdg-desktop-portal power-profiles-daemon \
  speech-dispatcher \
  snapper grub-btrfs inotify-tools bash-completion
```

> **Uwaga:** `plasma-meta` dociąga pełny zestaw komponentów Plasma, więc instalacja będzie wyraźnie większa niż przy ręcznie skrojonym, minimalnym zestawie.  
> **Uwaga:** podczas `pacstrap` może pojawić się ostrzeżenie z `mkinitcpio` o brakującym `/etc/vconsole.conf`.  
> To jest normalne na tym etapie, bo plik zostanie utworzony dopiero w późniejszej konfiguracji systemu.  
> Właściwe `mkinitcpio -P` i tak wykonujesz później, już po ustawieniu locale, keymap i hibernacji.

---

# 11. Podstawowa konfiguracja systemu i utworzenie użytkownika

Wejdź do chroota:

```
arch-chroot /mnt /bin/bash
```

W chroocie wykonaj:

```
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
```

Wyjdź z chroota:

```
exit
```

---

# 12. Montowanie subvolume specyficznych dla użytkownika

Teraz, gdy użytkownik już istnieje i ma poprawne `/home`, można zamontować subvolume wyłączone ze snapshotów `/home`.

```
mkdir -p /mnt/home/pietryszak/.config/BraveSoftware
mkdir -p /mnt/home/pietryszak/.mozilla
mkdir -p /mnt/home/pietryszak/.thunderbird
mkdir -p /mnt/home/pietryszak/.gnupg
mkdir -p /mnt/home/pietryszak/.ssh

mount -o subvol=@mozilla,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/home/pietryszak/.mozilla
mount -o subvol=@brave,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/home/pietryszak/.config/BraveSoftware
mount -o subvol=@thunderbird,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/home/pietryszak/.thunderbird
mount -o subvol=@gnupg,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/home/pietryszak/.gnupg
mount -o subvol=@ssh,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/home/pietryszak/.ssh
```

Nadaj właściciela i poprawne uprawnienia:

```
arch-chroot /mnt chown -R pietryszak:pietryszak /home/pietryszak
arch-chroot /mnt chmod 700 /home/pietryszak/.gnupg
arch-chroot /mnt chmod 700 /home/pietryszak/.ssh
```

---

# 13. Generowanie fstab

Dopiero teraz, gdy wszystkie docelowe mountpointy są już zamontowane, generujemy finalny `fstab`.

```
genfstab -U /mnt > /mnt/etc/fstab
cat /mnt/etc/fstab
```

Powinny pojawić się wpisy dla:

* `/`
* `/home`
* `/.snapshots`
* `/home/.snapshots`
* `/var/log`
* `/var/cache`
* `/var/cache/pacman/pkg`
* `/var/tmp`
* `/var/spool`
* `/opt`
* `/var/lib/libvirt`
* `/home/pietryszak/.mozilla`
* `/home/pietryszak/.config/BraveSoftware`
* `/home/pietryszak/.thunderbird`
* `/home/pietryszak/.gnupg`
* `/home/pietryszak/.ssh`
* `/boot/efi`
* swap

---

# 14. Konfiguracja hibernacji

Ustawienie `resume=UUID=...` dla GRUB i hooka `resume` w `mkinitcpio`:

```
arch-chroot /mnt /bin/bash <<'EOF'
set -e

SWAP_UUID=$(blkid -s UUID -o value /dev/nvme0n1p2)

sed -i "s/^GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT=\"loglevel=3 quiet resume=UUID=${SWAP_UUID}\"/" /etc/default/grub

grep -q '\<resume\>' /etc/mkinitcpio.conf || sed -i 's/\<fsck\>/resume fsck/' /etc/mkinitcpio.conf

mkinitcpio -P
EOF
```

---

# 15. Konfiguracja Snappera dla `/`

## 15.1 Utworzenie konfiguracji root bez D-Bus

> W chroocie trzeba użyć `--no-dbus`, bo bez tego Snapper może wywalić błąd `org.freedesktop.DBus.Error.ServiceUnknown`.

```
umount /mnt/.snapshots
rmdir /mnt/.snapshots

arch-chroot /mnt snapper --no-dbus -c root create-config /
```

## 15.2 Usunięcie `/.snapshots` utworzonego w `@` i ponowne podpięcie dedykowanego `@snapshots`

```
mkdir -p /mnt/.btrfs-root
mount -o subvolid=5 /dev/nvme0n1p3 /mnt/.btrfs-root

btrfs subvolume delete /mnt/.btrfs-root/@/.snapshots

umount /mnt/.btrfs-root
rmdir /mnt/.btrfs-root

mkdir -p /mnt/.snapshots
mount -o subvol=@snapshots,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/.snapshots
```

---

# 16. Konfiguracja Snappera dla `/home`

## 16.1 Utworzenie konfiguracji home bez D-Bus

```
umount /mnt/home/.snapshots
rmdir /mnt/home/.snapshots

arch-chroot /mnt snapper --no-dbus -c home create-config /home
```

## 16.2 Usunięcie `/home/.snapshots` utworzonego w `@home` i ponowne podpięcie dedykowanego `@home_snapshots`

```
mkdir -p /mnt/.btrfs-root
mount -o subvolid=5 /dev/nvme0n1p3 /mnt/.btrfs-root

btrfs subvolume delete /mnt/.btrfs-root/@home/.snapshots

umount /mnt/.btrfs-root
rmdir /mnt/.btrfs-root

mkdir -p /mnt/home/.snapshots
mount -o subvol=@home_snapshots,compress=zstd:1,noatime /dev/nvme0n1p3 /mnt/home/.snapshots
```

---

# 17. Włączenie timeline i cleanup dla Snappera

`snap-pac` instalujemy dopiero po skonfigurowaniu Snappera dla `/` i `/home`, żeby hooki pacmana nie próbowały robić snapshotów zbyt wcześnie.

Ustawienia dla `root`:

```
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
```

Ustawienia dla `home`:

```
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
  /etc/snapper/configs/home
```

Włączenie timerów i utworzenie pierwszych snapshotów:

```
arch-chroot /mnt systemctl enable snapper-timeline.timer
arch-chroot /mnt systemctl enable snapper-cleanup.timer

arch-chroot /mnt snapper --no-dbus -c root create -d "fresh-install-root"
arch-chroot /mnt snapper --no-dbus -c home create -d "fresh-install-home"

arch-chroot /mnt pacman -S --noconfirm snap-pac

arch-chroot /mnt snapper --no-dbus -c root list
arch-chroot /mnt snapper --no-dbus -c home list
```

---

# 18. Instalacja GRUB i integracja snapshotów

```
arch-chroot /mnt /bin/bash <<'EOF'
set -e

grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg

systemctl enable grub-btrfsd.service
systemctl enable NetworkManager
systemctl enable sshd
systemctl enable power-profiles-daemon
systemctl enable plasmalogin.service
EOF
```

---

# 19. Ostatnia kontrola przed restartem

```
arch-chroot /mnt cat /etc/locale.conf
arch-chroot /mnt cat /etc/vconsole.conf
arch-chroot /mnt cat /etc/hostname
arch-chroot /mnt grep '^GRUB_CMDLINE_LINUX_DEFAULT' /etc/default/grub
arch-chroot /mnt grep HOOKS /etc/mkinitcpio.conf
arch-chroot /mnt systemctl is-enabled plasmalogin.service
arch-chroot /mnt systemctl is-enabled grub-btrfsd.service
arch-chroot /mnt snapper --no-dbus -c root list
arch-chroot /mnt snapper --no-dbus -c home list
```

Powinno wyjść mniej więcej:

* `LANG=en_US.UTF-8`
* `KEYMAP=pl`
* hostname `arch`
* `resume=UUID=...`
* `resume` w HOOKS
* `plasmalogin.service` = `enabled`
* `grub-btrfsd.service` = `enabled`
* snapshot `fresh-install-root`
* snapshot `fresh-install-home`

---

# 20. Restart

```
umount -R /mnt
swapoff /dev/nvme0n1p2
reboot
```

Wyjmij pendrive instalacyjny.

---

# 21. Kontrola po pierwszym starcie

Po zalogowaniu do zainstalowanego systemu:

```
sudo cat /proc/cmdline
sudo swapon --show
sudo snapper -c root list
sudo snapper -c home list
sudo systemctl status grub-btrfsd.service --no-pager
sudo grep -n "submenu 'Arch Linux snapshots'" /boot/grub/grub.cfg
```

To potwierdza:

* aktywne `resume=UUID=...`
* aktywny swap
* działającego Snappera dla `/`
* działającego Snappera dla `/home`
* działające submenu snapshotów w GRUB-ie

---

# 22. Test hibernacji

Najprostszy test:

```
sudo systemctl hibernate
```

---

# 23. Snapshot bazowy po udanej instalacji

Po pierwszym poprawnym starcie dobrze od razu zrobić snapshot bazowy:

```
sudo snapper -c root create -d "working-base-root"
sudo snapper -c home create -d "working-base-home"

sudo snapper -c root list
sudo snapper -c home list
```

---

# 24. Wymuszenie polskiego układu klawiatury dla GUI

Jeśli chcesz dodatkowo ustawić układ X11/GUI na polski:

```
sudo localectl --no-convert set-x11-keymap pl
```

---

# 25. Po instalacji: narzędzia deweloperskie i `yay`

`git`, `wget` i `curl` są już instalowane w bazowym `pacstrap`, ale do budowania pakietów z AUR potrzebujesz jeszcze `base-devel`.

Do jednorazowej instalacji `yay` najprościej użyć `/tmp`:

```
sudo pacman -S --needed base-devel git wget curl
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

To wystarczy do poprawnej instalacji `yay`.

Opcjonalnie możesz później utworzyć własny katalog, na przykład `~/.gc`, jeśli chcesz trzymać tam:

* klony z GitHuba
* PKGBUILD-y z AUR
* własne skrypty i repozytoria

Przykład:

```
mkdir -p ~/.gc
```

---

# 26. Po instalacji: szybka zbiorcza instalacja pakietów

Jeśli wolisz zainstalować większość dodatkowych pakietów jednym strzałem po pierwszym starcie systemu, użyj tego wariantu.

## Pakiety z oficjalnych repozytoriów

```
sudo pacman -S --needed \
  cups system-config-printer avahi nss-mdns sane simple-scan \
  firefox thunderbird \
  bluez bluez-utils bluez-obex \
  linux-headers dkms \
  plymouth plymouth-kcm breeze-grub \
  vlc ffmpeg x264 x265 \
  gst-plugins-base gst-plugins-good gst-plugins-bad gst-plugins-ugly gst-libav \
  libdvdcss libdvdread libdvdnav libbluray \
  a52dec faac faad2 flac jasper lame libdca libmad libmpeg2 libtheora libvorbis libxv wavpack xvidcore aom dav1d libvpx opus \
  mjpegtools speex twolame zvbi libdvbpsi projectm chromaprint libnfs libmicrodns srt
```

## Usługi systemowe

```
sudo systemctl enable --now cups.service avahi-daemon.service bluetooth.service
```

## Pakiety z AUR

```
yay -S brother-dcp-b7520dw brscan4 brscan-skey brave-bin xpadneo-dkms plymouth-theme-arch-breeze-git spotify
```

`--needed` jest ważne, bo nie próbuje reinstalować pakietów, które już masz.

> **Uwaga:** `spotify` z AUR instaluje pełnego klienta Spotify. Nie używaj `spotify-launcher` — ten pakiet jest tylko launcherem który dopiero pobiera Spotify i ma problemy z widocznością w menu KDE.

---

# 27. Po instalacji: drukarka i skaner Brother DCP-B7520DW

Adres IP drukarki/skanera w sieci lokalnej jest stały: `192.168.1.100`.

Dodaj urządzenie do konfiguracji `brscan4`:

```
sudo brsaneconfig4 -a name=Brother model=DCP-B7520DW ip=192.168.1.100
scanimage -L
```

Jeśli `scanimage -L` pokaże urządzenie, skanowanie jest gotowe.

---

# 28. Po instalacji: przeglądarka i poczta

## Brave na KDE: trwałe obejście wolnego startu

Jeśli Brave uruchamia się bardzo długo na KDE Plasma, ustaw go na stałe z flagą `--password-store=basic`.

Najpierw utwórz lokalny wpis `.desktop`:

```
mkdir -p ~/.local/share/applications
cp /usr/share/applications/brave-browser.desktop ~/.local/share/applications/
```

Potem podmień linię `Exec=`:

```
sed -i 's|^Exec=.*|Exec=brave --password-store=basic %U|' ~/.local/share/applications/brave-browser.desktop
update-desktop-database ~/.local/share/applications 2>/dev/null || true
```

Od tej chwili Brave uruchamiany z menu aplikacji będzie używał `--password-store=basic`.

---

# 29. Po instalacji: Bluetooth w KDE

## AirPods Pro / sterowanie mediami

Dla AirPods Pro obecny stos z tego README jest wystarczający. Nie trzeba dodawać osobnego sterownika.

Żeby działało sterowanie mediami z przycisków słuchawek Bluetooth, uruchom usługę użytkownika:

```
systemctl --user enable --now mpris-proxy.service
```

Jeśli po tym sterowanie dalej nie działa, doinstaluj `playerctl` i sprawdź MPRIS ręcznie:

```
sudo pacman -S playerctl
playerctl -l
playerctl play-pause
```

## Xbox pad po Bluetooth

Po instalacji `xpadneo-dkms` najlepiej zrestartować system.

---

# 30. Ładny splash screen Arch + motyw GRUB

Ustawienie motywu Plymouth:

```
sudo plymouth-set-default-theme -R arch-breeze
```

Konfiguracja GRUB-a:

```
sudo nvim /etc/default/grub
```

Upewnij się, że linia wygląda mniej więcej tak:

```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet splash resume=UUID=TWOJ_SWAP_UUID"
GRUB_THEME="/usr/share/grub/themes/breeze/theme.txt"
```

Jeśli masz już ustawione `resume=UUID=...`, po prostu dopisz `splash` i dodaj linię `GRUB_THEME=...`.

Konfiguracja `mkinitcpio`:

```
sudo nvim /etc/mkinitcpio.conf
```

Dodaj hook `plymouth` przed `filesystems`.

Przykład:

```
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block plymouth filesystems resume fsck)
```

Potem przebuduj initramfs i GRUB:

```
sudo mkinitcpio -P
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Gdybyś kiedyś chciał wrócić do zwykłego Breeze, użyj:

```
sudo pacman -S breeze-plymouth
sudo plymouth-set-default-theme -R breeze
```

---

# 31. Stan końcowy systemu

Po wykonaniu wszystkich kroków system ma:

* Arch Linux
* Btrfs
* snapshoty przez Snapper dla `/` i `/home`
* automatyczne snapshoty pacmana przez `snap-pac`
* GRUB z menu snapshotów dzięki `grub-btrfs`
* działającą hibernację
* pełne KDE Plasma przez `plasma-meta`
* Plasma Login Manager
* hostname `arch`
* `en_US.UTF-8`
* polską klawiaturę
* strefę `Europe/Warsaw`
* `neovim`
* `bash-completion`
* `wget`, `git`, `curl`, `btop`, `fastfetch`
* `yay`
* obsługę drukarki i skanera Brother DCP-B7520DW
* Firefox
* Thunderbird
* Brave
* Spotify
* VLC z pełnym zestawem kodeków audio/wideo
* Bluetooth w KDE
* wsparcie dla AirPods Pro
* `mpris-proxy.service` dla sterowania mediami po Bluetooth
* `xpadneo-dkms` dla pada Xbox po Bluetooth
* ładny motyw GRUB-a
* splash screen Plymouth
* motyw startowy Arch + Breeze pasujący do ciemnego KDE Plasma
* `speech-dispatcher`

Dodatkowo rollback `/home` nie będzie ruszał:

* Firefoksa
* Brave
* Thunderbirda
* kluczy GPG
* kluczy SSH

bo te katalogi mają własne subvolume poza snapshotami `@home`.
