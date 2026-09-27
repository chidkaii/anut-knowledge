> **TL;DR** — Ubuntu dual-path disk mount via bind mount — partition accessible at /mnt (CLI) and /media/$USER (GUI/Nautilus) with fstab persistence.
> Tags: {#linux} {#mount} {#fstab} | Updated: 2026-09-28

# Skill: Persistent & Dual-Path Disk Mounting in Ubuntu

# 1. Create target folder under /media
sudo mkdir -p /media/$USER/Share-Drive

# 2. Add Bind Mount entry to /etc/fstab
echo '/mnt/Share-Drive  /media/anut-ubuntu/Share-Drive  none  bind  0  0' | sudo tee -a /etc/fstab

# 3. Reload & Apply
sudo systemctl daemon-reload
sudo mount -a


## 📌 Problem Overview
An existing partition (`/dev/nvme0n1p8` formatted as `ext4`, labeled `Share-Drive`) was intended to be accessible via both:
1. System/CLI scripts targeting `/mnt/Share-Drive` (e.g., Ollama configs, custom AI agents, automated workflows).
2. The Ubuntu/GNOME File Manager (Nautilus) left sidebar for easy GUI access.

Standard `/mnt` mount points do not natively appear in the Nautilus sidebar, whereas paths under `/media/$USER/` are treated as interactive storage drives and displayed automatically.

---

## 🛠️ Solution Architecture
To maintain **100% compatibility with legacy paths** without editing application configurations while enabling **GUI visibility**, a two-layer mount configuration in `/etc/fstab` is established:

1. **Primary Block Mount**: Mounts the raw UUID partition directly to `/mnt/Share-Drive`.
2. **Bind Mount**: Mirror `/mnt/Share-Drive` to `/media/$USER/Share-Drive`.

---

## 📋 Step-by-Step Implementation

### Step 1: Prepare Mount Target Directory
Create the target directory under the active user's `/media` path:

```bash
sudo mkdir -p /media/$USER/Share-Drive
```

### Step 2: Configure `/etc/fstab`
Open `/etc/fstab` with administrative privileges:

```bash
sudo nano /etc/fstab
```

Ensure the following two lines exist in sequence (replace `0843be3c-195b-40da-b49b-ad515e44e693` with your partition UUID and `anut-ubuntu` with your username):

```text
# Primary Partition Mount (System/Legacy Path)
UUID=0843be3c-195b-40da-b49b-ad515e44e693  /mnt/Share-Drive                 ext4  defaults,noatime  0  2

# Bind Mount for File Manager Sidebar Integration
/mnt/Share-Drive                          /media/anut-ubuntu/Share-Drive  none  bind              0  0
```

> **Note on Syntax:** Ensure spaces or tabs separate the fields cleanly. Avoid unescaped special characters.

### Step 3: Reload System Daemon & Apply Mounts
Notify systemd of the updated table and apply all mounts:

```bash
sudo systemctl daemon-reload
sudo mount -a
```

---

## 🔍 Verification & Troubleshooting

### Check Active Mounts
Verify that both paths point to the same filesystem:

```bash
df -h /mnt/Share-Drive /media/anut-ubuntu/Share-Drive
```

Expected Output:
```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p8  245G   52G  181G  22% /mnt/Share-Drive
/dev/nvme0n1p8  245G   52G  181G  22% /media/anut-ubuntu/Share-Drive
```

### Common Errors & Fixes
* **`mount: /etc/fstab: parse error at line X`**: Indicates a syntax error or stray control character on the specified line. Inspect line spacing or rewrite the entry.
* **`mount: (hint) your fstab has been modified...`**: Systemd unit cache is stale. Run `sudo systemctl daemon-reload` before executing `sudo mount -a`.
* **Sidebar item missing**: Ensure the second path is explicitly under `/media/<username>/` (not `/mnt/` or raw `/media/`).

---

## 🎯 Verification Checklist
- [x] Filesystem mounted without data loss or reformatting.
- [x] Legacy application path (`/mnt/Share-Drive/ollama`) completely intact.
- [x] Drive icon visible under **Drives / Devices** in Nautilus / GNOME File Manager.
- [x] Auto-mount persists across system reboots.
