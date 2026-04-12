# Arch Linux na Dell Latitude 5421
## Btrfs + Snapper + hibernacja + GRUB snapshoty + minimalne KDE + Plasma Login Manager

Przewodnik opisuje instalację Arch Linuksa od zera na laptopie **Dell Latitude 5421** z:

- UEFI
- Btrfs
- snapshotami przez Snapper
- hibernacją
- GRUB + grub-btrfs do wybierania snapshotów z menu startowego
- minimalnym KDE
- Plasma Login Manager zamiast SDDM
- językiem systemu `en_US.UTF-8`
- polską klawiaturą
- strefą czasową `Europe/Warsaw`

Założenia:

- dysk docelowy to `/dev/nvme0n1`
- instalacja kasuje cały dysk
- brak szyfrowania
- użytkownik docelowy to `pietryszak`
- hostname to `arch`

---

## Układ partycji

- `nvme0n1p1` — EFI — `1 GiB`
- `nvme0n1p2` — swap — `40 GiB`
- `nvme0n1p3` — Btrfs — reszta dysku

Subvolume Btrfs:

- `@`
- `@home`
- `@snapshots`

---

## 0. Ustawienia BIOS / UEFI

Przed startem z pendrive:

- tryb bootowania: **UEFI**
- **Secure Boot: Off**
- jeśli Linux nie widzi dysku: tryb storage ustaw na **AHCI**

---

## 1. Start Arch ISO i sieć

Po uruchomieniu Arch ISO:

```bash
timedatectl set-ntp true
