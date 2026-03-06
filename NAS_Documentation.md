# NAS Setup Documentation

### Ubuntu 24.04 — Nextcloud-Based Private Cloud

---

## Overview

The NAS runs on an Ubuntu machine using an application called Nextcloud, which manages file storage, access permissions, and hosts the server. The files are stored on an ext4 disk mounted at `/mnt/nas/`, with the shared folder named `lab118`. Nextcloud accesses this disk through Snap, which ensures proper read and write permissions. Nginx acts as a reverse proxy, redirecting traffic to the Nextcloud server for convenience, since the default Nextcloud share link is tedious. The NAS can be accessed via a wired connection or through a wireless network provided by a WAP.

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Storage Setup](#storage-setup)
3. [Nextcloud Setup](#nextcloud-setup)
4. [Connecting Storage to Nextcloud](#connecting-storage-to-nextcloud)
5. [Access Control & User Management](#access-control--user-management)
6. [Public Sharing](#public-sharing)
7. [Nginx Reverse Proxy](#nginx-reverse-proxy)
8. [Wireless Access](#wireless-access)
9. [Upload Configuration](#upload-configuration)
10. [Important Notes](#important-notes)
11. [Key Concepts & Terminology](#key-concepts--terminology)

---

## System Architecture

```
Client Device (Wired or Wireless)
          ↓
  Router (DHCP / NAT)
          ↓
 Ubuntu Server — 192.168.0.100
          ↓
    Nginx (Port 80)
          ↓
 Snap Nextcloud (Port 8080)
          ↓
  /mnt/nas/lab118 (ext4 Disk)
```

**Components:**

| Component  | Role                                                         |
|------------|--------------------------------------------------------------|
| Ubuntu     | Host operating system running all services                   |
| Nextcloud  | Application managing files, users, access, and sharing       |
| Snap       | Package manager providing a sandboxed Nextcloud environment  |
| Nginx      | Reverse proxy handling clean URL routing                     |
| ext4 Disk  | Actual storage location (`/mnt/nas/lab118`)                  |
| EAP245 WAP | TP-Link access point providing wireless LAN access           |

---

## Storage Setup

### Drive Preparation

The dedicated storage drive was formatted using the **ext4** filesystem — the standard Linux filesystem that supports proper file ownership and UNIX permissions.

The drive was partitioned, formatted, and mounted at `/mnt/nas/`. Within that mount, `lab118` is the folder used by Nextcloud as the primary storage location.

### Persistent Mounting

To ensure the drive automatically mounts when the server reboots, an entry was added to `/etc/fstab`. This file tells Ubuntu what to mount at boot time. The drive's UUID (a unique identifier) is used instead of a device name like `/dev/sdb`, because device names can change between reboots.

### Permissions

Two key permission commands were used:

- **`chown`** — Changes *who owns* a file or folder. The NAS folder was assigned to `root` so that the Snap-confined Nextcloud process could access it.
- **`chmod`** — Sets *who can read, write, or enter* a file or folder. The permissions were set to `750`:
  - `7` → Owner (root): full access
  - `5` → Group (root): read and enter
  - `0` → Others: no access

---

## Nextcloud Setup

Nextcloud was installed via **Snap**, Ubuntu's sandboxed package manager. The Snap version bundles everything needed — Apache (web server), PHP, and the database — into one isolated environment.

After installation, Nextcloud is accessible through the browser at:

```
http://192.168.0.100
```

An admin account was created during initial setup, which controls all settings, users, and storage configurations.

---

## Connecting Storage to Nextcloud

### The Snap Confinement Problem

By default, Snap sandboxes applications and **prevents them from accessing external drives** mounted under `/mnt`. This is a security feature, but it also means Nextcloud cannot see the NAS drive without explicit permission.

### Solution: `removable-media` Interface

The Snap `removable-media` interface was connected, granting Nextcloud permission to access drives under `/mnt`:

```bash
sudo snap connect nextcloud:removable-media
```

> Without this step, external storage always fails — even if permissions and paths are configured correctly. This is one of the most critical steps of the entire setup.

### Adding External Storage in Nextcloud

Once Snap had access, the NAS folder was registered inside Nextcloud:

1. Navigate to **Admin → Settings → External Storage**
2. Enable the **External Storage Support** app
3. Add a new storage entry:
   - **Type:** Local
   - **Path:** `/mnt/nas/lab118`
   - **Available for:** Selected users/groups
4. A **green checkmark** confirms Nextcloud can read and write to the folder

The folder then appears in the Nextcloud file browser for assigned users.

---

## Access Control & User Management

### Groups

Nextcloud uses **groups** to control who sees what. A `guests` group was created to separate limited-access users from administrators.

### Storage Visibility

In the External Storage settings, each storage entry can be assigned to specific users or groups. By assigning `lab118` only to selected groups, other users cannot see or access it.

### Folder-Level Sharing

For more granular control, specific subfolders (e.g., `GuestShare/`) were shared with the guest group rather than exposing the entire NAS root. This ensures guests only see what they are explicitly given.

---

## Public Sharing

A public share link was created for the `lab118` folder, accessible without a Nextcloud account.

**Key properties of public links:**

- Stored in Nextcloud's database — survive reboots and restarts
- Can be password-protected
- Can be set to expire after a specified date
- Are read-only or allow uploads depending on settings
- Randomly generated token ensures security (e.g., `/index.php/s/ZzQcTZeJRtafaKn`)

The full share URL format:

```
http://192.168.0.100/index.php/s/ZzQcTZeJRtafaKn
```

---

## Nginx Reverse Proxy

### What Is Nginx?

Nginx is a **web traffic manager** (reverse proxy) that sits in front of Nextcloud. It listens on standard port 80 and forwards requests to Nextcloud's internal port. Think of it as a receptionist at the front door: it directs visitors to the right place without storing any files itself.

### Why Nginx Was Used

The default Nextcloud public share link is tedious. Nginx was configured to redirect a clean, human-readable URL to the full share link:

```
http://192.168.0.100/lab118  →  http://192.168.0.100:8080/index.php/s/ZzQcTZeJRtafaKn
```

### Setup Summary

1. Nextcloud was moved from port 80 to port 8080 (to free up port 80 for Nginx)
2. Nginx was installed and configured with two behaviors:
   - **`/lab118`** → redirect to the public share link
   - **`/`** → forward all other traffic to Nextcloud on port 8080
3. Nginx was enabled and restarted

### Nginx's Role in This Stack

| Task                         | Handled by |
|------------------------------|------------|
| Storing files                | Nextcloud  |
| Managing users/permissions   | Nextcloud  |
| Hosting web interface        | Nextcloud  |
| Clean URL redirect           | **Nginx**  |
| Forwarding port 80 → 8080    | **Nginx**  |
| Upload size limits (proxy)   | **Nginx**  |

---

## Wireless Access

The NAS is accessible both via wired connection and wirelessly through a **TP-Link EAP245** access point (WAP).

### Configuration Requirements

| Setting            | Required Value               |
|--------------------|------------------------------|
| AP Mode            | Access Point (not Router)    |
| Client Isolation   | **Disabled**                 |
| DHCP               | Provided by main router      |
| Subnet             | Same as wired LAN (e.g., `192.168.0.x`) |

**Client Isolation**, if enabled, prevents wireless devices from communicating with other devices on the LAN — including the NAS. Disabling it allows wireless clients to reach the server the same way wired clients do.

The EAP245 bridges wireless traffic directly to the main router's LAN, placing all devices on the same subnet.

---

## Upload Configuration

### Browser Upload Limitations

Large file uploads via the browser can fail mid-transfer due to:

- PHP upload size limits (set inside Snap)
- Nginx proxy body size limits
- Chunk timeouts

For uploads consistently larger than a few hundred MB, the **Nextcloud Desktop Client** is recommended — it splits files into chunks, handles retries, and resumes interrupted transfers automatically.

### Relevant Limits (Snap Nextcloud)

| Limit                  | Adjusted via                              |
|------------------------|-------------------------------------------|
| PHP upload size        | `snap set nextcloud php.upload-max-filesize` |
| PHP post size          | `snap set nextcloud php.post-max-size`    |
| Nginx body size        | `client_max_body_size` in Nginx config    |

---

## Important Notes

- **Snap can't access externally mounted drives unless explicitly granted permission.** The `removable-media` interface must be connected; otherwise, External Storage will always appear as "temporarily unavailable."

- **Nginx is responsible for only redirecting traffic; it must be configured separately for it to operate properly.** It does not store files or handle authentication — those remain Nextcloud's responsibility.

- **Wireless devices must be on the same subnet as the NAS.** If client isolation is enabled on the access point, wireless devices cannot reach any LAN resource, including the NAS.

- **Public share links are permanent across reboots** but will break if the associated folder is deleted, the share is manually removed, or Nextcloud is reinstalled with a fresh database.

- **Browser uploads are less reliable than the Nextcloud Desktop Client**, especially for large files. For consistent results, use the desktop sync client.

---

## Key Concepts & Terminology

| Term            | Definition                                                                                                |
|-----------------|-----------------------------------------------------------------------------------------------------------|
| **NAS**         | Network Attached Storage — a server that shares storage over a local network                              |
| **Nextcloud**   | The application managing files, users, access control, and the web interface                              |
| **Snap**        | Ubuntu's sandboxed package manager; bundles Nextcloud with all its dependencies                           |
| **ext4**        | The standard Linux filesystem; supports proper UNIX permissions required by Nextcloud                     |
| **Nginx**       | A reverse proxy / web server used here to create clean URLs and forward traffic to Nextcloud              |
| **`chown`**     | Linux command to change ownership of a file or folder (who "owns" it)                                    |
| **`chmod`**     | Linux command to set read/write/execute permissions on a file or folder                                   |
| **WAP**         | Wireless Access Point — a device that extends wireless LAN access (e.g., TP-Link EAP245)                 |
| **Reverse Proxy** | A server that sits in front of another server, forwarding requests and optionally transforming URLs     |
| **Client Isolation** | An AP setting that prevents wireless devices from communicating with other LAN devices            |
| **External Storage** | Nextcloud feature that connects local or network folders as visible storage inside the web UI     |
| **`removable-media`** | Snap permission interface that allows Snap apps to access drives mounted under `/mnt`            |
| **fstab**       | A Linux configuration file (`/etc/fstab`) that defines what drives to mount automatically at boot         |
| **UUID**        | Universally Unique Identifier — used in `fstab` to reliably identify a disk regardless of device naming  |

---

*Documentation compiled from NAS setup session — Ubuntu 24.04, Snap Nextcloud, ext4, Nginx, TP-Link EAP245.*
