# MrGamer Lab Cheat Sheet

> Reusable Linux forensic artifacts and commands encountered during the MrGamer lab. This is not a write-up and does not contain challenge answers.

## System information

```bash
cat /mnt/Linux/etc/os-release   # OS and version
cat /mnt/Linux/etc/hostname     # hostname
cat /mnt/Linux/etc/passwd       # local users
```

## Important user artifacts

```text
~/.bash_history                 Shell commands
~/.mozilla/firefox/             Firefox data
~/.thunderbird/                 Email
~/.minecraft/                   Minecraft data
~/.config/                      Application configuration
~/.local/share/keyrings/        GNOME Keyring
~/.cache/thumbnails/            File previews
~/Downloads/                    Downloads
~/Pictures/                     Screenshots
```

### Firefox

```text
places.sqlite                   Visited pages
formhistory.sqlite              Search/form history
sessionstore-backups/           Recent tabs
cookies.sqlite                  Cookies
```

### Thunderbird

```text
Mail/ and ImapMail/             Messages
Inbox, Sent, Trash              mbox message files
*.msf                           Indexes only
global-messages-db.sqlite       Message index
```

### Installed software

```bash
grep -iE 'install|remove|upgrade' \
  /mnt/Linux/var/log/apt/history.log
```

```bash
grep -F 'status installed' /mnt/Linux/var/log/dpkg.log |
sed 's/^.*status installed //' |
sort -u
```

## Searching evidence

Search file contents:

```bash
grep -rslFi 'search text' /mnt/Linux
```

- `-r` — recursive
- `-s` — hide errors
- `-l` — filenames only
- `-F` — literal string
- `-i` — ignore case

Find images:

```bash
find /mnt/Linux/home -type f \
  \( -iname '*.png' -o -iname '*.jpg' -o -iname '*.jpeg' \)
```

Find files from a particular day:

```bash
find /mnt/Linux/home -type f \
  -newermt '2022-02-09' ! -newermt '2022-02-10'
```

Inspect thumbnail source:

```bash
exiftool thumbnail.png
```

Look for `Thumb URI`.

## Useful investigation principles

```text
Browser visit       ≠ complete video watched
Thumbnail           ≠ file deliberately opened
Saved Wi-Fi profile ≠ confirmed connection
Base64              = encoding, not encryption
```

Correlate multiple artifacts instead of relying on one.

---

# EWF Tools Cheat Sheet

EWF/E01 is a forensic disk-image format. Segments such as `.E01`, `.E02` and `.E03` form one image and must remain together.

## Installation

```bash
sudo apt install ewf-tools
```

## Main tools

```text
ewfinfo      Show image metadata
ewfverify    Verify image integrity
ewfmount     Expose the image as a raw disk
ewfexport    Convert E01 to RAW
ewfacquire   Create an E01 image
```

## Standard workflow

```bash
ewfinfo evidence.E01
ewfverify evidence.E01

sudo mkdir -p /mnt/ewf
sudo ewfmount evidence.E01 /mnt/ewf
```

The raw disk becomes:

```text
/mnt/ewf/ewf1
```

Inspect partitions:

```bash
sudo fdisk -l /mnt/ewf/ewf1
mmls /mnt/ewf/ewf1
```

Mount a Linux partition read-only using its starting sector:

```bash
sudo mkdir -p /mnt/Linux

sudo mount -o ro,noload,loop,offset=$((START_SECTOR * 512)) \
  /mnt/ewf/ewf1 /mnt/Linux
```

Unmount in reverse order:

```bash
sudo umount /mnt/Linux
sudo fusermount -u /mnt/ewf
```

## Essential workflow

```text
ewfinfo → ewfverify → ewfmount → mmls → mount read-only
```

