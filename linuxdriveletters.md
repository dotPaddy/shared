# USB Drive Letters for Wine / SAM Broadcaster on Linux Mint

Persistent, Windows-style drive letters for USB disks under Linux Mint, automatically exposed to a Wine prefix running SAM Broadcaster.

## Background

This setup was built for a specific scenario:

- **Host:** Linux Mint 22 on an old Dell Optiplex 780
- **App:** SAM Broadcaster 4.9.1 running under Wine (emulating Windows 7)
- **Users:** Elderly people accustomed to Windows 7 drive letters
- **Problem:** SAM Broadcaster expects USB music libraries at a fixed drive letter (e.g. `D:\Music`). On Linux, USB drives get mounted to unpredictable paths like `/media/user/<LABEL>`, which changes per disk and confuses both the software and the user.

## What this does

1. Identifies each USB disk by its **filesystem UUID** (which never changes for the life of the filesystem).
2. Keeps a small registry file mapping `UUID → letter`.
3. Mounts registered disks at a **fixed path** like `/mnt/usb/USB_D`.
4. Creates a symlink in `~/.wine/dosdevices/` so SAM Broadcaster sees it as `D:\`, `E:\`, etc.
5. A **udev rule** triggers this automatically on plug-in — known drives re-appear with their original letter; unknown drives are ignored.
6. Unknown drives are adopted with a single click on a desktop launcher (password prompt once per session).

Drive letters start at `D` because Wine uses `A:`/`B:` for floppies and `C:` for the system drive.

## Behavior summary

| Event | What happens |
|---|---|
| Plug in a **known** drive | Auto-mounted at same path, same letter. No user action. |
| Plug in an **unknown** drive | Nothing happens automatically. User clicks launcher to register. |
| Unplug | Mount disappears. Wine symlink stays (points at empty folder). |
| Reboot | udev replays "add" events, known drives come back automatically. |
| Reformat a USB | New UUID — treated as brand new drive. Use `usb-register forget USB_X` to clean up old entry. |

---

## 1. The main script

Save as `/usr/local/bin/usb-register` and `chmod +x` it.

```bash
#!/bin/bash
# usb-register — assign persistent Windows-style drive letters to USB disks
set -u

REGISTRY="/etc/usb-drives/registry.conf"
MOUNT_BASE="/mnt/usb"
LETTERS=(D E F G H I J K L M N O P Q R S T U V W X Y Z)

# Figure out the desktop user (the one whose ~/.wine we touch)
if   [[ -n "${PKEXEC_UID:-}" ]]; then WINE_USER=$(getent passwd "$PKEXEC_UID" | cut -d: -f1)
elif [[ -n "${SUDO_USER:-}"   ]]; then WINE_USER="$SUDO_USER"
else WINE_USER="$USER"; fi
WINE_HOME=$(getent passwd "$WINE_USER" | cut -d: -f6)
WINE_DOSDEVICES="$WINE_HOME/.wine/dosdevices"

need_root() {
    [[ $EUID -eq 0 ]] && return
    exec pkexec --disable-internal-agent \
        env DISPLAY="${DISPLAY:-}" XAUTHORITY="${XAUTHORITY:-}" "$0" "$@"
}

notify() {
    echo "$*"
    if command -v zenity >/dev/null && [[ -n "${DISPLAY:-}" ]]; then
        sudo -u "$WINE_USER" DISPLAY="$DISPLAY" XAUTHORITY="${XAUTHORITY:-}" \
            zenity --info --no-wrap --title="USB Drive" --text="$*" 2>/dev/null &
    fi
}

init() {
    mkdir -p /etc/usb-drives "$MOUNT_BASE"
    [[ -f "$REGISTRY" ]] || printf '# UUID|LETTER|LABEL|FSTYPE\n' > "$REGISTRY"
}

get_letter_for_uuid() {
    while IFS='|' read -r u l _ _; do
        [[ -z "$u" || "$u" =~ ^# ]] && continue
        [[ "$u" == "$1" ]] && { echo "$l"; return 0; }
    done < "$REGISTRY"
    return 1
}

next_free_letter() {
    local used=" "
    while IFS='|' read -r u l _ _; do
        [[ -z "$u" || "$u" =~ ^# ]] && continue
        used+="$l "
    done < "$REGISTRY"
    for l in "${LETTERS[@]}"; do
        [[ "$used" != *" $l "* ]] && { echo "$l"; return 0; }
    done
    return 1
}

# Emit: DEVICE|UUID|LABEL|FSTYPE for each USB partition with a filesystem
find_usb_partitions() {
    lsblk -rpno NAME,TYPE,TRAN,HOTPLUG,UUID,LABEL,FSTYPE,PKNAME \
    | while read -r name type tran hotplug uuid label fstype pkname; do
        [[ "$type"   == "part" ]] || continue
        [[ -n "$uuid" && -n "$fstype" ]] || continue
        parent_tran=$(lsblk -rpno TRAN "$pkname" 2>/dev/null | head -1)
        if [[ "$parent_tran" == "usb" || "$tran" == "usb" || "$hotplug" == "1" ]]; then
            echo "$name|$uuid|$label|$fstype"
        fi
    done
}

mount_drive() {
    local dev="$1" letter="$2" fstype="$3"
    local mp="$MOUNT_BASE/USB_$letter"
    mkdir -p "$mp"

    if mountpoint -q "$mp"; then
        [[ "$(findmnt -n -o SOURCE "$mp")" == "$dev" ]] && return 0
        umount "$mp" 2>/dev/null || true
    fi

    local uid gid opts
    uid=$(id -u "$WINE_USER"); gid=$(id -g "$WINE_USER")
    case "$fstype" in
        vfat|exfat|ntfs|ntfs3|fuseblk)
            opts="uid=$uid,gid=$gid,umask=022,rw,nofail" ;;
        ext2|ext3|ext4|btrfs|xfs)
            opts="rw,nofail" ;;
        *)  opts="rw,nofail" ;;
    esac
    mount -o "$opts" "$dev" "$mp"
}

make_wine_link() {
    local letter="$1"
    local mp="$MOUNT_BASE/USB_$letter"
    local link="$WINE_DOSDEVICES/${letter,,}:"
    [[ -d "$WINE_DOSDEVICES" ]] || { echo "Wine prefix missing at $WINE_HOME/.wine"; return 0; }
    [[ -e "$link" || -L "$link" ]] && rm -f "$link"
    sudo -u "$WINE_USER" ln -s "$mp" "$link"
    # Also expose a drive type hint so Wine treats it as a fixed disk
    sudo -u "$WINE_USER" bash -c "echo 'hd' > '$WINE_DOSDEVICES/${letter,,}::'"
}

cmd_register() {
    need_root "$@"; init
    local total=0 new=0 known=0 report=""
    while IFS='|' read -r dev uuid label fstype; do
        total=$((total+1))
        if letter=$(get_letter_for_uuid "$uuid"); then
            known=$((known+1))
            mount_drive "$dev" "$letter" "$fstype" && make_wine_link "$letter"
            report+=$'\n'"• ${label:-(no label)}  →  USB_$letter  (${letter}:\\)"
        else
            if letter=$(next_free_letter); then
                new=$((new+1))
                echo "$uuid|$letter|$label|$fstype" >> "$REGISTRY"
                mount_drive "$dev" "$letter" "$fstype" && make_wine_link "$letter"
                report+=$'\n'"• ${label:-(no label)}  →  NEW: USB_$letter  (${letter}:\\)"
            else
                report+=$'\n'"• ${label:-(no label)}  →  no free letters!"
            fi
        fi
    done < <(find_usb_partitions)

    if [[ $total -eq 0 ]]; then
        notify "No USB drives detected.\nIs the drive plugged in?"
    else
        notify "Done — $new new, $known already known.${report}"
    fi
}

cmd_auto() {
    # Called by udev on plug-in. Silent. Only acts on known UUIDs.
    init
    local dev="${1:-}"
    [[ -b "$dev" ]] || exit 0

    local uuid fstype
    uuid=$(blkid -s UUID  -o value "$dev" 2>/dev/null)
    fstype=$(blkid -s TYPE -o value "$dev" 2>/dev/null)
    [[ -n "$uuid" && -n "$fstype" ]] || exit 0

    local letter
    letter=$(get_letter_for_uuid "$uuid") || exit 0   # unknown → do nothing

    mount_drive "$dev" "$letter" "$fstype" && make_wine_link "$letter"
}

cmd_list() {
    init
    printf "%-38s %-7s %-20s %-8s\n" UUID LETTER LABEL FSTYPE
    while IFS='|' read -r u l lab f; do
        [[ -z "$u" || "$u" =~ ^# ]] && continue
        printf "%-38s USB_%-3s %-20s %-8s\n" "$u" "$l" "${lab:-<none>}" "$f"
    done < "$REGISTRY"
}

cmd_forget() {
    need_root "$@"; init
    local target="${1:-}"; [[ -z "$target" ]] && { echo "Usage: $0 forget USB_X"; exit 1; }
    local letter="${target#USB_}"
    umount "$MOUNT_BASE/USB_$letter" 2>/dev/null || true
    rm -f "$WINE_DOSDEVICES/${letter,,}:" "$WINE_DOSDEVICES/${letter,,}::"
    rmdir "$MOUNT_BASE/USB_$letter" 2>/dev/null || true
    grep -v "|$letter|" "$REGISTRY" > "$REGISTRY.tmp" && mv "$REGISTRY.tmp" "$REGISTRY"
    echo "Forgot USB_$letter"
}

case "${1:-register}" in
    register|"") cmd_register ;;
    auto)        shift; cmd_auto "$@" ;;
    list)        cmd_list ;;
    forget)      shift; cmd_forget "$@" ;;
    *) echo "Usage: $0 [register|auto DEV|list|forget USB_X]"; exit 1 ;;
esac
```

Install:

```bash
sudo cp usb-register /usr/local/bin/
sudo chmod +x /usr/local/bin/usb-register
```

### Commands

| Command | Purpose |
|---|---|
| `usb-register` (or `register`) | Scan plugged-in USBs, assign letters to new ones, mount known ones. Interactive; prompts for root. |
| `usb-register auto <device>` | Silent mode for udev. Does nothing if UUID is unknown. |
| `usb-register list` | Show all registered drives. |
| `usb-register forget USB_X` | Remove a registration and clean up. |

---

## 2. Desktop launcher (for manual registration of new drives)

Save as `~/Desktop/Register-USB.desktop` (as the desktop user, not root):

```ini
[Desktop Entry]
Type=Application
Name=Register USB Drive
Comment=Plug in your USB, then click this to give it a drive letter
Exec=/usr/local/bin/usb-register
Icon=drive-removable-media-usb
Terminal=false
Categories=Utility;
```

Then mark it trusted:

```bash
chmod +x ~/Desktop/Register-USB.desktop
gio set ~/Desktop/Register-USB.desktop metadata::trusted true
```

Clicking it triggers a `pkexec` password prompt (once per session), mounts any new USBs under `/mnt/usb/USB_<letter>`, and creates matching `<letter>:\` entries inside the Wine prefix.

---

## 3. udev rule for auto-mount on plug-in

Save as `/etc/udev/rules.d/99-usb-register.rules`:

```
ACTION=="add", SUBSYSTEMS=="usb", KERNEL=="sd[a-z][0-9]", ENV{ID_FS_UUID}=="?*", \
    RUN+="/usr/bin/systemd-run --no-block --collect /usr/local/bin/usb-register auto $env{DEVNAME}"
```

Reload rules:

```bash
sudo udevadm control --reload-rules
```

### Why `systemd-run`?

udev kills child processes quickly and runs them in a restrictive sandbox. `systemd-run --no-block` launches the script as an independent transient service so it can mount things and write to the Wine prefix without udev interfering mid-run.

### Why `ENV{ID_FS_UUID}=="?*"`?

This filter ensures the rule only fires on partitions that actually have a filesystem UUID — it skips the raw disk device (`sda` itself) and any empty/unformatted slots.

---

## 4. (Optional) Hide managed drives from Nemo / udisks

By default, Cinnamon's `udisks2` service **also** auto-mounts USB drives to `/media/<user>/<LABEL>` and shows them in Nemo's sidebar. This is harmless but redundant — the same drive appears in two places. For elderly Windows-era users, that's confusing.

To suppress this only for **registered** drives (unregistered drives still show normally, so users can copy music onto a fresh USB before registering it):

### 4a. Replace the udev rule

Replace `/etc/udev/rules.d/99-usb-register.rules` with this version:

```
# Auto-mount known USB drives to their assigned letter, and hide them from Nemo
ACTION=="add", SUBSYSTEMS=="usb", KERNEL=="sd[a-z][0-9]", ENV{ID_FS_UUID}=="?*", \
    PROGRAM="/usr/local/bin/usb-register-known $env{ID_FS_UUID}", \
    ENV{UDISKS_IGNORE}="1", \
    RUN+="/usr/bin/systemd-run --no-block --collect /usr/local/bin/usb-register auto $env{DEVNAME}"
```

### 4b. Add the helper

Save as `/usr/local/bin/usb-register-known`:

```bash
#!/bin/bash
# Exit 0 if UUID is in the registry, 1 otherwise.
grep -q "^$1|" /etc/usb-drives/registry.conf 2>/dev/null
```

```bash
sudo chmod +x /usr/local/bin/usb-register-known
sudo udevadm control --reload-rules
```

With this setup:

- **Registered** drives → only appear as `D:\`, `E:\`, etc. in Wine. Nothing in Nemo.
- **Unregistered** drives → show up in Nemo normally, user can prep/copy files, then click the launcher to register.

---

## Workflow

### First time with a new music USB

1. Plug it in.
2. Double-click the **Register USB Drive** desktop launcher.
3. Enter password when prompted.
4. It picks the next free letter (e.g. `D`), stores the UUID in `/etc/usb-drives/registry.conf`, mounts it at `/mnt/usb/USB_D`, and creates `~/.wine/dosdevices/d:` → that path.
5. Point SAM Broadcaster at `D:\Music` (or wherever on the stick).

### Every subsequent plug-in (weeks later, different USB port, whatever)

- Just plug it in. udev fires, script recognizes the UUID, re-mounts to `/mnt/usb/USB_D`. SAM Broadcaster sees `D:\` as always. No clicks, no prompts.

### Adding a second music drive

- Plug it in, click launcher, password. It gets `USB_E` / `E:\`. From then on, auto-mounts.

---

## Troubleshooting

### The drive doesn't appear in Wine after plug-in

Check the Wine prefix path the script is using:

```bash
usb-register list
ls -la ~/.wine/dosdevices/
```

If `~/.wine` is somewhere non-standard, you'll need to edit `WINE_HOME` logic in the script.

### Check that udev fired

```bash
journalctl -f
```
Then plug in a known drive. You should see a short-lived transient unit start.

### A drive got reformatted and won't auto-mount

Reformatting changes the UUID. Clean up the old registration:

```bash
usb-register list
sudo usb-register forget USB_D
```

Then plug the drive back in and re-register with the launcher.

### Specific letter required (e.g. SAM library already references `E:`)

Edit `/etc/usb-drives/registry.conf` directly — it's a plain text file with `UUID|LETTER|LABEL|FSTYPE` per line. Either add the entry manually before first plug-in, or register, then edit the letter and rename the `/mnt/usb/USB_X` folder.

---

## Files reference

| Path | Purpose |
|---|---|
| `/usr/local/bin/usb-register` | Main script |
| `/usr/local/bin/usb-register-known` | UUID lookup helper (udev) |
| `/etc/udev/rules.d/99-usb-register.rules` | Auto-mount trigger |
| `/etc/usb-drives/registry.conf` | UUID → letter registry (persistent) |
| `/mnt/usb/USB_<letter>/` | Mount point for each drive |
| `~/.wine/dosdevices/<letter>:` | Symlink Wine uses for drive letter |
| `~/Desktop/Register-USB.desktop` | Clickable launcher for registration |

---

## License

Use freely. No warranty. Tested on Linux Mint 22 with Wine + SAM Broadcaster 4.9.1.
