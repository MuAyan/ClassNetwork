# Cowrie Honeypot + Graylog Setup Documentation

### Ubuntu 24.04 + Raspberry Pi OS — SSH Honeypot with Centralized Logging

---

## Overview

The honeypot runs Cowrie on a Raspberry Pi, presenting a fake SSH server on port 22 that accepts attacker connections and records every credential attempt, command typed, and file download - without exposing any real system. A dedicated non-root user runs Cowrie, and `authbind` grants it permission to bind the privileged port 22 without root access. Real SSH admin access to the Pi is moved to port 2222. Logs are shipped in real time over GELF UDP to a Graylog instance running on an Ubuntu machine, where they are ingested, parsed, and displayed on a live dashboard. The setup can also be used as a classroom CTF exercise by planting flag files in Cowrie's fake filesystem.

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Prerequisites](#prerequisites)
3. [Pi Preperation](#pi-preparation)
4. [Cowrie Installation](#cowrie-installation)

---

## System Architecture

```
Internet / LAN
      ↓
Raspberry Pi — 192.168.0.xxx                                     Attacker
 ├── Port 22   → Cowrie (fake SSH, attacker-facing)     ←    SSH into port 22
 └── Port 2222 → Real SSH (admin access)
      ↓
      │  GELF HTTP — port 12201 
      ↓
Ubuntu Machine — 192.168.0.xxx
 └── Graylog
      ├── OpenSearch (log indexing)
      ├── MongoDB (metadata)
      └── Web dashboard — port 9000
```

*Both machines must be on the same subnet with IPs provided by a router configured with DHCP.* <br>
> *Click [here](./Hardware_docs/MainRouter.md) to set up a router*

---

**Components:**

| Component    | Role                                                                  |
|--------------|-----------------------------------------------------------------------|
| Raspberry Pi | Host running Cowrie and exposing the fake SSH service                 |
| Cowrie       | Medium-interaction SSH honeypot — logs credentials and commands       |
| authbind     | Allows a non-root process to bind to privileged ports below 1024      |
| Ubuntu       | Host operating system running the Graylog stack                       |
| Graylog      | Log management platform — ingests, parses, and visualizes Cowrie logs |
| OpenSearch   | Search and indexing backend used by Graylog                           |
| MongoDB      | Database storing Graylog configuration and metadata                   |

---

## Prerequisites

- Raspberry Pi with a fresh Raspberry Pi OS install and SSH enabled
- Ubuntu 24.04 machine on the same LAN subnet
- Both machines reachable from each other (verify with `ping`)
- Router configured with DHCP enabled


---

## Pi Preparation

### What This Section Does

Before Cowrie is installed, the Pi requires three things: its real ssh daemon moved off of port 22 for Cowrie to claim, the Python build tools and libraries that Cowrie depends on, and a dedicated unprivileged user account to run Cowrie under. These steps ensure a clean and secure foundation before any honeypot software is configured.

### Move Real SSH to Port 2222

SSH daemons listen on port 22 by default, but Cowrie also needs port 22. that is the port attackers expect to find SSH on, and presenting anything on a non-standard port defeats the purpose of the honeypot. The solution is to move the Pi's real SSH daemon to a different port before Cowrie is installed, freeing port 22 for Cowrie to claim.


Open the SSH daemon configuration file:

```bash
sudo nano /etc/ssh/sshd_config
```

`sshd_config` is the main configuration file for the OpenSSH server daemon. It controls what port SSH listens on, which authentication methods are allowed, and which users can connect.

Find the line `#Port 22` and change it to:

```
Port 2222
```
> `#` means this line is commented, which causes it this line to be ignored and default to port 22. remove it so the file reads the line.

Restart the SSH service to apply the change:

```bash
sudo systemctl restart ssh
```

**Do not close your current SSH session.** Open a second terminal and verify the new port works before closing anything:

```bash
ssh -p 2222 pi@<PI_IP>
```
The `-p 2222` flag tells the SSH client to connect on port 2222 instead of the default 22. If the connection succeeds, real ssh is confirmed on port 2222 and it is safe to close the original session.
> All future admin connections to the Pi must use 'p 2222' as port 22 is dedicated to Cowrie.


### Install Dependencies

Cowrie is written in Python and relies on several external libraries for cryptography, SSL, and network handling. These must be installed before Cowrie itself can be set up.

```bash
sudo apt install -y python3-virtualenv libssl-dev libffi-dev build-essential \
  libpython3-dev python3-minimal authbind python3-pip git
```

What each package does:

| Package | Purpose |
|---|---|
| `python3-virtualenv` | Creates isolated Python environments so Cowrie's dependencies don't conflict with system packages |
| `libssl-dev` | SSL/TLS development headers required to compile Python cryptography libraries |
| `libffi-dev` | Foreign Function Interface headers required by several Cowrie dependencies |
| `build-essential` | C compiler and build tools needed to compile Python extensions from source |
| `libpython3-dev` | Python development headers required for compiling Python extension modules |
| `python3-minimal` | Minimal Python 3 runtime |
| `authbind` | Utility that allows non-root users to bind to privileged ports below 1024 |
| `python3-pip` | Python package installer used to install Cowrie's Python dependencies |
| `git` | Version control tool used to clone the Cowrie source code from GitHub |

> `-y` automatically answers yes to any confirmation prompts.

### Create a Dedicated Cowrie User

Cowrie must never run as root or as your personal user account. If Cowrie were running as root and an attacker found a way to escape the fake shell, (a known attack class against honeypots) they would land in a real root shell on the Pi. Running Cowrie under a dedicated unprivileged account limits what an escaped attacker can access.

```bash
sudo adduser --disabled-password cowrie
```
> The `--disabled-password` flag creates the account without a password so no one can log into this account interactively with a password. It can only be accessed through `sudo su`. This was done as this account exists only to run Cowrie.

---

## Cowrie Installation

### What Is Cowrie?

Cowrie is a medium-interaction SSH and Telnet honeypot. When an attacker attempts to connect to port 22, Cowrie displays a fully functional fake login prompt that accepts any credential. Once authenticated, the attackers appears inside a simulated linux shell backed by a fake filesystem consisting of fake directory listings, fake configuration files, fake system information, etc. Every command typed, every file read, every download attempted is logged in detail. The attacker believes they are on a real compromised server when in fact they are inside a sandbox.

The term "medium-interaction" makes Cowrie a solid choice. A low-Interaction honeypot only simulates a port by responding to connection attempts but does not let the attacker do much, making it easily detectable. A high-interaction honeypot runs a real OS inside a VM or container and lets attackers operate in a real environment, which provides better data but carries a higher risk. Cowrie sits in the middle by providing a convincing interactive shell experience without exposing any real system resources.

### Installation Steps

Switch to the cowrie user:

```bash
sudo su - cowrie
```

> `sudo su - cowrie` switches the current session to the cowrie user account. The `-` flag loads a full login environment, including the cowrie user's home directory and shell settings.

Clone the Cowrie repository from GitHub:

```bash
git clone https://github.com/cowrie/cowrie
cd cowrie
```

#### The following commands in this section run as the cowrie user, not as your personal account.


Create an isolated Python virtual environment for Cowrie.

```bash
python3 -m venv cowrie-env
```

Active the virtual environment

```bash
source cowrie-env/bin/activate
```

> A virtual environment is an isolated copy of Python with its own separate set of installed packages. Without this, installing Cowrie's dependencies would modify Python system-wide, which can break other software. The `(cowrie-env)` prefix that appears in the terminal prompt confirms the environment is active.

Install Cowrie's Python dependencies:

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

> `pip install --upgrade pip` updates pip itself to the latest version first, which prevents compatibility warnings during the next step. `pip install -r requirements.txt` reads the `requirements.txt` file in the Cowrie directory which is a list of every Python package Cowrie depends on, and installs all of them.

Install Cowrie itself as a package:

```bash
pip install -e .
```

> The previous step installed Cowrie's dependencies but not Cowrie itself. `pip install -e .` installs Cowrie from the local directory in editable mode. Without it, Cowrie commands such as `Cowrie start` would return `Command not found`.

---

## Key Concepts & Terminology
 
| Term | Definition |
|---|---|
| **Honeypot** | A deliberately exposed decoy system designed to attract and log attacker activity without exposing any real resources |
| **Medium-interaction honeypot** | A honeypot that provides a convincing interactive shell experience using emulation, without running a real OS |
| **Cowrie** | A medium-interaction SSH honeypot that simulates a real Linux shell using a fake filesystem |
| **Fake filesystem** | Cowrie's simulated directory tree (`honeyfs/`) containing realistic fake files that attackers navigate after logging in |
| **authbind** | A Linux utility that allows non-root processes to bind to privileged ports below 1024 via per-port permission files |
| **Privileged port** | Any port below 1024 on Linux — only root can bind these by default |
| **Virtual environment** | An isolated Python environment (`venv`) that keeps Cowrie's dependencies separate from the system |
| **Editable install** | A `pip install -e .` that registers a local package's entry point commands into the active virtual environment |
 
---
