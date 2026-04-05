# Cowrie Honeypot + Graylog Setup Documentation

### Ubuntu 24.04 + Raspberry Pi OS - SSH Honeypot with Centralized Logging

---

## Overview

The honeypot runs Cowrie on a Raspberry Pi, presenting a fake SSH server on port 22 that accepts attacker connections and records every credential attempt, command typed, and file download - without exposing any real system. A dedicated non-root user runs Cowrie, and `authbind` grants it permission to bind the privileged port 22 without root access. Real SSH admin access to the Pi is moved to port 2222. Logs are shipped in real time over GELF HTTP to a Graylog instance running on an Ubuntu machine, where they are ingested, parsed, and displayed on a live dashboard. The setup can also be used as a classroom CTF exercise by planting flag files in Cowrie's fake filesystem.

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Prerequisites](#prerequisites)
3. [Pi Preparation](#pi-preparation)
4. [Cowrie Installation](#cowrie-installation)
5. [Cowrie Configuration](#cowrie-configuration)
6. [Binding Port 22 with Authbind](#binding-port-22-with-authbind)
7. [Graylog Installation](#graylog-installation)
8. [Connecting Cowrie to Graylog](#connecting-cowrie-to-graylog)
9. [Cowrie systemd Service](#cowrie-systemd-service)
10. [Important Notes](#important-notes)
11. [Key Concepts & Terminology](#key-concepts--terminology)

---

## System Architecture

```
Internet / LAN
      ↓
Raspberry Pi - 192.168.0.xxx                                     Attacker
 ├── Port 22   → Cowrie (fake SSH, attacker-facing)     ←    SSH into port 22
 └── Port 2222 → Real SSH (admin access)
      ↓
      │  GELF HTTP - port 12201 
      ↓
Ubuntu Machine - 192.168.0.xxx
 └── Graylog
      ├── OpenSearch (log indexing)
      ├── MongoDB (metadata)
      └── Web dashboard - port 9000
```

*Both machines must be on the same subnet with IPs provided by a router configured with DHCP.* <br>
> *Click [here](../Hardware_docs/MainRouter.md) to set up a router*

---

**Components:**

| Component    | Role                                                                  |
|--------------|-----------------------------------------------------------------------|
| Raspberry Pi | Host running Cowrie and exposing the fake SSH service                 |
| Cowrie       | Medium-interaction SSH honeypot - logs credentials and commands       |
| authbind     | Allows a non-root process to bind to privileged ports below 1024      |
| Ubuntu       | Host operating system running the Graylog stack                       |
| Graylog      | Log management platform - ingests, parses, and visualizes Cowrie logs |
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
> `#` means this line is commented, which causes this line to be ignored and default to port 22. remove it so the file reads the line.

Restart the SSH service to apply the change:

```bash
sudo systemctl restart ssh
```

**Do not close your current SSH session.** Open a second terminal and verify the new port works before closing anything:

```bash
ssh -p 2222 pi@<PI_IP>
```
The `-p 2222` flag tells the SSH client to connect on port 2222 instead of the default 22. If the connection succeeds, real ssh is confirmed on port 2222 and it is safe to close the original session.
> All future admin connections to the Pi must use '-p 2222' as port 22 is dedicated to Cowrie.


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

Activate the virtual environment

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
## Cowrie Configuration

### What the Configuration File Controls

Everything that Cowrie does is controlled by `cowrie.cfg`. This includes what hostnames attackers see when they log in, what port Cowrie listens on, which output plugins are enabled, and where logs are sent. The repository ships a template called `cowrie.cfg.dist`. This must be copied to `cowrie.cfg` before any changes are made because Cowrie only reads `cowrie.cfg`.

Copy the file:

```bash
cp etc/cowrie.cfg.dist etc/cowrie.cfg
```

Open it:

```bash
nano etc/cowrie.cfg
```

Set the following values:

```ini
[honeypot]
# The hostname attackers see when they connect
hostname = ubuntu-server

[ssh]
# Port Cowrie listens on, must match the port authbind is configured for
listen_endpoints = tcp:22:interface=0.0.0.0

[output_jsonlog]
enabled = true

[output_graylog]
enabled = true
url = http://<GRAYLOG_MACHINE_IP>:12201/gelf
```

### What Each Setting Does

**`hostname`** sets the fake hostname that attackers see in the shell prompt when they log in. Setting it to something simple like `ubuntu-server` makes the honeypot look like a real machine rather than a trap.

**`listen_endpoints`** tells Cowrie which port and interface to listen on. `tcp:22:interface=0.0.0.0` means listen on TCP port 22 on ALL network interfaces. Port 22 is the standard SSH port - attackers will hit this port first. `0.0.0.0` means accept connections from any IP address.
> authbind grants permission to bind a specific port number. If listen_endpoints says tcp:22 but authbind was configured for a different port, Cowrie will fail to bind with a permission error.

**`output_jsonlog`** enables Cowrie's built-in JSON log file at `var/log/cowrie/cowrie.json`. This is the local log, every session is written here whether Graylog is running or not. It serves as a backup and is useful for direct inspection with `tail` or `cat`

**`output_graylog`** enables the Graylog output plugin. When enabled, a copy of every log event is sent to Graylog in real time over HTTP. The `url` field must point to the `/gelf` endpoint on the Graylog machine at port 12201. Cowrie's graylog plugin sends logs as HTTP POST requests formatted as GELF. This is important as Cowrie's built-in Graylog plugin only supports HTTP post with a url field with protocol, ip, port, and path (`/gelf`) clearly defined.
> Using a different format such as `host` and `port` fields will silently fail.

---

## Binding Port 22 with Authbind

### Why Authbind Is Needed

On Linux, any port below 1024 is considered a privileged port. Only processes running as root are allowed to bind to these ports by default. This is a security measure as it prevents unprivileged processes from impersonating well-known services like SSH (22), HTTP (80), or HTTPS (443).

Cowrie must run as the unprivileged `cowrie` user, not root. But it also needs to bind port 22. These two requirements conflict. `authbind` resolves this by granting a specific user permission to bind a specific port without running as root. It works through a file in `/etc/authbind/byport/`. If a file named after the port number exists and is owned by a given user, that user is allowed to bind that port.

### Setup

Exit back to your regular user first as the following commands require sudo, then create the permission file for port 22:

```bash
exit
sudo touch /etc/authbind/byport/22
sudo chown cowrie /etc/authbind/byport/22
sudo chmod 770 /etc/authbind/byport/22
```

`sudo touch /etc/authbind/byport/22` creates an empty file named `22` in the authbind configuration directory. The filename is the port number, authbind checks for the existence of this file when a process tries to bind that port.

`sudo chown cowrie /etc/authbind/byport/22` transfers ownership of the file to the cowrie user. Authbind only grants the bind permission to the user who owns the file.

`sudo chmod 770` sets the file permissions to allow the owner (cowrie) and group to read, write, and execute, while blocking all others. The execute bit on the file is what authbind checks to determine if the bind is permitted.

### Starting Cowrie

Switch back to the cowrie user, activate the virtual environment, and start Cowrie through authbind:

```bash
sudo su - cowrie
cd cowrie
source cowrie-env/bin/activate
authbind --deep cowrie start
```

`authbind --deep` wraps the following command so that any process it spawns also inherits the port binding permission. The `--deep` flag is required because Cowrie uses Twisted, which spawns child processes. 
> Without `--deep`, only the parent process gets the permission and Twisted's worker processes fail to bind the port.

`cowrie start` launches Cowrie as a background daemon. It reads `etc/cowrie.cfg`, starts the fake SSH service on the configured port, and begins accepting connections.

Verify Cowrie is running:

```bash
cowrie status
```

To stop Cowrie:

```bash
cowrie stop
```

### Test the Honeypot

From another machine on the network, connect to port 22 on the Pi:

```bash
ssh root@<PI_IP>
```

Cowrie displays a fake login prompt. Enter any credential such as `root` / `123456`, Cowrie accepts it. Once inside the fake shell, commands like `ls`, `whoami`, and `cat /etc/passwd` return realistic fake output. The session is being fully logged.

Confirm the session was captured in the JSON log:

```bash
tail -f ~/cowrie/var/log/cowrie/cowrie.json
```

`tail -f` follows the file in real time, printing new lines as they are appended. Each SSH event - connection, login attempt, command, disconnect - produces a separate JSON entry.

Confirm the graylog output engine loaded correctly:

```bash
tail -20 ~/cowrie/var/log/cowrie/cowrie.log
```

The line `Loaded output engine: graylog` must be present. If it is missing, the `[output_graylog]` section in `cowrie.cfg` is not being read correctly. Check that `enabled = true` and the `url` field are set under `[output_graylog]`.

---

## Graylog Installation

### What Is Graylog?

Graylog is a centralized log management platform. Its role in this setup is to take in every Cowrie log - logins, commands, file downloads, session durations, feed it into a searchable database, and display them on a live dashboard. It transforms raw JSON log files into a visual interface showing different attacker patterns. Which credentials are most commonly tried, which commands are run post-login, and which IPs are most active are just some of Graylog's indexing features.

Graylog is not a single application, it is a stack of three components that operate and work together:

- **MongoDB** stores Graylog's own configuration data: user accounts, dashboard layouts, stream definitions, and alert rules. It does not store the actual log messages.
- **OpenSearch** is the search and indexing engine that stores and indexes the log data. Every log message that Graylog receives is written to OpenSearch which makes it quickly searchable.
- **Graylog server** is the application layer that sits between the two. It is responsible for receiving incoming logs, parsing and sorting them, writes them to OpenSearch, reads Graylog's own config from MongoDB, and presents a web interface.


All three must be running for the stack to function. If any one stops, Graylog loses its backend.

### Install Java

Graylog and OpenSearch are both Java applications. Java 17 is required:

```bash
sudo apt install -y apt-transport-https openjdk-17-jre-headless uuid-runtime pwgen curl dirmngr
```

| Package | Purpose |
|---|---|
| `apt-transport-https` | Allows `apt` to download packages from HTTPS repositories, required for the MongoDB and OpenSearch repos |
| `openjdk-17-jre-headless` | Java 17 runtime - required by both Graylog and OpenSearch |
| `uuid-runtime` | Generates UUIDs used internally by Graylog |
| `pwgen` | Generates random strings used for Graylog's required secrets |
| `curl` | Used to download GPG keys for the MongoDB and OpenSearch repositories |
| `dirmngr` | Manages GPG keys used to verify package signatures |

### Install MongoDB

MongoDB stores Graylog's configuration, user data, and stream metadata. 
> The version available in Ubuntu's default repositories is too old for Graylog 6.0, so the official MongoDB 6.0 repository must be added manually.

Add the MongoDB GPG key so Ubuntu can verify packages from the MongoDB repository:

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-6.0.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-6.0.gpg
```

`curl -fsSL` downloads the GPG key silently, failing cleanly if the download fails. The key is piped to `gpg --dearmor`, which converts it from ASCII armored format to binary and saves it to `/usr/share/keyrings/`.
> This file is referenced in the next step so `apt` knows which key to use when verifying MongoDB packages.

Add the MongoDB 6.0 repository to apt's sources:

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg ] \
  https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
```

This writes a new repository definition file to `/etc/apt/sources.list.d/`. The `signed-by` field references the GPG key downloaded above.
`jammy` refers to Ubuntu 22.04's codename, which Ubuntu 24.04 is compatible with for this repository.
> apt uses this to verify that packages from this repository were signed by MongoDB and have not been tampered with.

Install MongoDB and enable it as a system service:

```bash
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl enable mongod
sudo systemctl start mongod
```
> `enable` configures MongoDB to start automatically on boot, `start` starts it immediately without requiring a reboot.

Verify it is running:

```bash
sudo systemctl status mongod
```

The output should show **active (running)**. 
> If it shows failed or inactive, check `sudo journalctl -u mongod` for error details.

### Install OpenSearch

Opensearch is the search and indexing engine that is responsible for storing Cowrie's actual log data. It is a fork of Elasticsearch maintained by AWS as an open-source project. Graylog uses it as its log storage backend.

Add the OpenSearch GPG key:

```bash
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | \
  sudo gpg --dearmor -o /usr/share/keyrings/opensearch-keyring.gpg
```

Add the OpenSearch 2.x repository:

```bash
echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring.gpg] \
  https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/opensearch-2.x.list
```

Install OpenSearch. The environment variable sets a temporary admin password required by the installer:

```bash
sudo apt update
sudo OPENSEARCH_INITIAL_ADMIN_PASSWORD=$(tr -dc A-Z-a-z-0-9_@#%^-_=+ < /dev/urandom | head -c${1:-32}) \
  apt install opensearch -y
```

`tr -dc A-Z-a-z-0-9_@#%^-_=+` filters random bytes from `/dev/urandom` to produce a string containing only the specified characters. `head -c 32` takes the first 32 characters. The result is assigned as an environment variable that the OpenSearch installer requires to set an initial admin password. 
> This password is only used during installation and is not needed afterward since OpenSearch security is disabled in the next step.

Configure OpenSearch for a single-node Graylog deployment:

```bash
sudo nano /etc/opensearch/opensearch.yml
```

Set the following values:

```yaml
cluster.name: graylog
node.name: node-1
network.host: 127.0.0.1
http.port: 9200
discovery.type: single-node
plugins.security.disabled: true
action.auto_create_index: false
```

What each setting does:

| Setting | Purpose |
|---|---|
| `cluster.name: graylog` | Names the OpenSearch cluster. Graylog uses this to identify which cluster to connect to |
| `node.name: node-1` | Names this node within the cluster. Required for single-node setups |
| `network.host: 127.0.0.1` | Restricts OpenSearch to only accept connections from localhost. It should never be exposed to the network |
| `http.port: 9200` | The port OpenSearch listens on for HTTP requests from Graylog |
| `discovery.type: single-node` | Tells OpenSearch not to attempt to discover or join other nodes. Required for single-node deployments |
| `plugins.security.disabled: true` | Disables OpenSearch's built-in TLS and authentication layer. Required so Graylog can connect over plain HTTP on localhost |
| `action.auto_create_index: false` | Prevents OpenSearch from automatically creating new indexes. Graylog manages its own index creation |

Enable and start OpenSearch:

```bash
sudo systemctl enable opensearch
sudo systemctl start opensearch
```

Verify it is responding:

```bash
curl -s http://localhost:9200
```

`curl` sends an HTTP GET request to OpenSearch's REST API. A JSON response containing `"cluster_name" : "graylog"` confirms OpenSearch is running and configured correctly. 
> If nothing is returned, OpenSearch is still starting. Wait 30 seconds and try again.


### Install Graylog

Download the Graylog repository package:

```bash
wget https://packages.graylog2.org/repo/packages/graylog-6.0-repository_latest.deb
```

Install the package, adding it to the Graylog repository to apt's sources and installs its GPG key:

```bash
sudo dpkg -i graylog-6.0-repository_latest.deb
```

Refresh the package lists and install Graylog server:

```bash
sudo apt update
sudo apt install -y graylog-server
```


### Configure Graylog

Graylog requires two cryptographic secrets to be set before it will start. These are not optional as Graylog will not launch if either is missing or too short.

Generate the secrets:

```bash
# password_secret - must be at least 96 characters
pwgen -N 1 -s 96

# root_password_sha2 - SHA256 hash of your chosen admin password
echo -n "yourpassword" | sha256sum
```

> `pwgen -N 1 -s 96` generates one completely random 96-character string. This becomes the `password_secret`: a server-side secret used internally by Graylog to encrypt session tokens and other sensitive data. It is never entered by a user.

> `echo -n "yourpassword" | sha256sum` hashes the chosen admin password using SHA256. Replace `yourpassword` with your actual password. The `-n` flag prevents `echo` from appending a newline character. Without it, the hash would include the newline and not match when you type the password in the browser. The output is a 64-character hex string followed by `  -`. Only the hex string goes into the config file.

Open the Graylog configuration file:

```bash
sudo nano /etc/graylog/server/server.conf
```

Set these four values:

```ini
password_secret = <output of pwgen>
root_password_sha2 = <output of sha256sum, without the trailing ' -'>
http_bind_address = 0.0.0.0:9000
elasticsearch_hosts = http://127.0.0.1:9200
```

| Setting | Purpose |
|---|---|
| `password_secret` | Server-side encryption secret. Must be at least 96 characters, randomly generated |
| `root_password_sha2` | SHA256 hash of the admin password. Used to verify the password entered at the login page |
| `http_bind_address` | The address and port the Graylog web interface binds to. `0.0.0.0` means accept connections from any IP |
| `elasticsearch_hosts` | The URL of the OpenSearch backend, Must be set explicitly or Graylog enters preflight mode and blocks the UI |

> `elasticsearch_hosts` must be set explicitly even though Graylog is connecting to OpenSearch. Without this line, Graylog enters preflight mode on startup and blocks all web UI access with a basic auth popup instead of showing the login page.

Start Graylog:

```bash
sudo systemctl enable graylog-server
sudo systemctl start graylog-server
```

Monitor the startup log:

```bash
sudo journalctl -fu graylog-server
```

> `journalctl -fu graylog-server` follows the systemd journal for the Graylog service in real time. Wait until the line `Graylog server up and running` appears before attempting to log in.

Open a browser and navigate to `http://<UBUNTU_IP>:9000`. Log in with username `admin` and the plain text password used when generating the SHA256 hash, not the hash itself. The hash is how Graylog stores the password internally; the plain text is what you type in the browser.

---

## Connecting Cowrie to Graylog

### How the Pipeline Works

Cowrie does not directly write to Graylog's database. Instead, it sends each log event formatted as a GELF JSON message over http to a listener called an input that runs inside Graylog. The input receives the message, Graylog parses it, and OpenSearch indexes it. From there it is then searchable in the Graylog web UI.

GELF (Graylog Extended Log Format) is a structured JSON format designed specifically for log shipping. Unlike plain syslog, GELF preserves arbitrary key-value fields — so Cowrie can include fields like `username`, `password`, `src_ip`, and `input` (the command typed) as separate searchable fields rather than embedding them in a flat string.

Cowrie's built-in graylog output plugin uses GELF HTTP, it POSTs each log entry as JSON to an HTTP endpoint. The Graylog input must be configured to match this transport method exactly.

### Create a GELF HTTP Input

In the Graylog web UI:

1. Navigate to **System → Inputs**
2. Select **GELF HTTP** from the dropdown
   > this must be GELF HTTP, not GELF UDP
4. Click **Launch new input**
5. Set the title to `Cowrie`, bind address to `0.0.0.0`, port to `12201`
6. Click **Save**, the input should show as **Running**

The input is now listening on port 12201 for incoming GELF HTTP POST requests. When Cowrie logs a session event, it sends an HTTP POST to `http://<GRAYLOG_IP>:12201/gelf` and this input receives it.

### Verify the Pipeline

From any machine on the network, SSH into the honeypot on port 22 and run a few commands. In Graylog, navigate to **Search**, set the time range to **Last 5 minutes**, and search. Log entries from Cowrie should appear with fields including `src_ip`, `username`, `password`, `input`, and `eventid`.

The `eventid` field identifies the type of event:

| eventid | Meaning |
|---|---|
| `cowrie.session.connect` | A new connection was made to the honeypot |
| `cowrie.login.failed` | A login attempt was made with incorrect credentials |
| `cowrie.login.success` | A login attempt succeeded |
| `cowrie.command.input` | A command was typed in the fake shell |
| `cowrie.session.closed` | The session ended |

---

## Cowrie systemd Service

### Why a systemd Service Is Needed

Right now Cowrie only runs if someone manually switches to the cowrie user, activates the virtual environment, and runs `authbind --deep cowrie start`. If the Pi loses power or reboots, the honeypot goes down and stays down until someone SSHs in and starts it again manually. That is not acceptable for a permanent classroom asset.

systemd is the Linux init system that manages all services and background processes. It starts services at boot, restarts them if they crash, and provides a consistent interface for checking their status. Registering Cowrie as a systemd service means it starts automatically on every boot and recovers on its own if it crashes, with no manual intervention required.

### Create the Startup Script

systemd does not activate Python virtual environments on its own. Calling the cowrie binary directly from the unit file leaves the environment incomplete, causing the service to fail. The fix is a shell script that sets up the environment correctly before handing off to Cowrie. systemd calls this script instead of calling Cowrie directly.

Create the script:

```bash
sudo nano /home/cowrie/start-cowrie.sh
```

Paste the following:

```bash
#!/bin/bash
cd /home/cowrie/cowrie
export PATH=/home/cowrie/cowrie/cowrie-env/bin:/usr/bin:/bin
source cowrie-env/bin/activate
authbind --deep cowrie-env/bin/cowrie start
```

`cd /home/cowrie/cowrie` moves into the Cowrie directory first. Cowrie resolves paths like `honeyfs/` and `var/run/` relative to its working directory - without this, it cannot find its own files and fails immediately.

`export PATH=...` prepends the virtual environment's bin directory to PATH so any subprocess Cowrie spawns can find the right binaries.

`source cowrie-env/bin/activate` activates the virtual environment for the current shell session.

`authbind --deep cowrie-env/bin/cowrie start` launches Cowrie through authbind using the full path to the cowrie binary inside the venv. `--deep` ensures all child processes inherit the port 22 binding permission, which is required because Cowrie uses Twisted and spawns child processes internally.

Make the script executable and set the correct owner:

```bash
sudo chmod +x /home/cowrie/start-cowrie.sh
sudo chown cowrie /home/cowrie/start-cowrie.sh
```

### Create the Unit File

A unit file is a configuration file that tells systemd how to run a service - what user to run it as, what command to execute, when to start it, and how to handle failures. Create the Cowrie unit file at `/etc/systemd/system/cowrie.service`:

```bash
sudo nano /etc/systemd/system/cowrie.service
```

Paste the following:

```ini
[Unit]
Description=Cowrie SSH Honeypot
After=network.target

[Service]
User=cowrie
WorkingDirectory=/home/cowrie/cowrie
ExecStart=/bin/bash /home/cowrie/start-cowrie.sh
Restart=on-failure
RestartSec=10
Type=forking
PIDFile=/home/cowrie/cowrie/var/run/cowrie.pid

[Install]
WantedBy=multi-user.target
```

What each setting does:

| Setting | Purpose |
|---|---|
| `After=network.target` | Delays Cowrie startup until the network interface is up, preventing a bind failure on port 22 at boot |
| `User=cowrie` | Runs the service as the cowrie user, not root |
| `WorkingDirectory` | Sets the working directory to the Cowrie installation folder before starting |
| `ExecStart` | Calls bash to run the startup script, which sets up the environment and launches Cowrie |
| `Restart=on-failure` | Automatically restarts Cowrie if it crashes |
| `RestartSec=10` | Waits 10 seconds before attempting a restart |
| `Type=forking` | Tells systemd that the start command will fork a background daemon and exit, which is how `cowrie start` works |
| `PIDFile` | The path to Cowrie's PID file, which systemd uses to track the background process after the start command exits |
| `WantedBy=multi-user.target` | Registers Cowrie to start during normal multi-user boot |

> `Type=forking` is required because `cowrie start` launches a background daemon - it forks and exits. Without this, systemd assumes the service died when the start command returns and immediately attempts a restart loop.

### Enable and Start the Service

Tell systemd to load the new unit file, then enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable cowrie
sudo systemctl start cowrie
```

`sudo systemctl daemon-reload` tells systemd to re-read all unit files. This must be run any time a unit file is created or modified, otherwise systemd will not see the changes.

`sudo systemctl enable cowrie` registers Cowrie to start automatically on every boot.

`sudo systemctl start cowrie` starts Cowrie immediately without requiring a reboot.

Verify the service is running:

```bash
sudo systemctl status cowrie
```

The output should show **active (running)**. From this point forward, Cowrie starts on boot and restarts automatically on failure. Manual `sudo su - cowrie` and `authbind --deep cowrie start` are no longer needed.

---


## Important Notes

- **`pip install -e .` is required after `pip install -r requirements.txt`.** The requirements install only Cowrie's dependencies, not Cowrie itself. Without the editable install step, the `cowrie` command is not registered in the virtual environment and the honeypot cannot be started.

- **The virtual environment must be active when running Cowrie commands.** If `cowrie start` returns command not found after a reboot or new session, run `source ~/cowrie/cowrie-env/bin/activate` first before running any `cowrie` commands.

- **authbind ownership is critical.** The file `/etc/authbind/byport/22` must be owned by the `cowrie` user with permissions `770`. Any deviation prevents Cowrie from binding port 22 and it will silently fail to start on that port.

- **Cowrie's graylog output plugin uses GELF HTTP, not GELF UDP.** The `[output_graylog]` section requires a `url` field pointing to `http://<IP>:12201/gelf`. The Graylog input must be set to GELF HTTP to match. Using `host` and `port` fields or creating a GELF UDP input will result in no logs appearing with no error message.

- **`elasticsearch_hosts` must be set in `server.conf`** even when using OpenSearch. Without it, Graylog starts in preflight mode and the web UI shows a basic auth popup instead of the login page.

- **OpenSearch security must be explicitly disabled for local single-node deployments.** The `plugins.security.disabled: true` setting in `opensearch.yml` is required. Without it, Graylog cannot connect to OpenSearch over plain HTTP on localhost.

- **Graylog requires both secrets set before it will start.** A missing or incorrectly formatted `password_secret` or `root_password_sha2` causes Graylog to exit immediately with no useful error in the UI. Check `journalctl` if the web interface never loads.

- **MongoDB, OpenSearch, and Graylog must all be running simultaneously** for the stack to function. If any one service is stopped, Graylog loses its backend and stops accepting or displaying logs.
  
---


## Key Concepts & Terminology
 
| Term | Definition |
|---|---|
| **Honeypot** | A deliberately exposed decoy system designed to attract and log attacker activity without exposing any real resources |
| **Medium-interaction honeypot** | A honeypot that provides a convincing interactive shell experience using emulation, without running a real OS |
| **Cowrie** | A medium-interaction SSH honeypot that simulates a real Linux shell using a fake filesystem |
| **Fake filesystem** | Cowrie's simulated directory tree (`honeyfs/`) containing realistic fake files that attackers navigate after logging in |
| **authbind** | A Linux utility that allows non-root processes to bind to privileged ports below 1024 via per-port permission files |
| **Privileged port** | Any port below 1024 on Linux - only root can bind these by default |
| **Virtual environment** | An isolated Python environment (`venv`) that keeps Cowrie's dependencies separate from the system |
| **Editable install** | A `pip install -e .` that registers a local package's entry point commands into the active virtual environment |
| **GELF** | Graylog Extended Log Format: a structured JSON log format that preserves arbitrary key-value fields for rich log shipping |
| **GELF HTTP** | The transport method Cowrie uses to POST GELF-formatted log entries to Graylog over HTTP |
| **Graylog** | A log management platform that ingests, indexes, searches, and visualizes log data through a web interface |
| **Input** | A Graylog listener configured to receive log messages over a specific protocol and port |
| **OpenSearch** | A search and analytics engine that acts as Graylog's log storage and indexing backend |
| **MongoDB** | A NoSQL database storing Graylog's own configuration, users, streams, and dashboard metadata - not the log data itself |
| **Preflight mode** | A Graylog startup state triggered when backend connectivity is not configured - blocks the web UI with a basic auth popup |
| **`systemctl`** | The Linux command used to manage systemd services - start, stop, restart, enable, and check status |
| **`journalctl`** | The Linux command used to read systemd service logs - useful for diagnosing startup failures |
| **`pwgen`** | A command-line tool that generates cryptographically random strings, used here for Graylog's `password_secret` |
| **eventid** | A Cowrie log field identifying the type of event logged - e.g., `cowrie.login.failed`, `cowrie.command.input` |
| **`sshd_config`** | The OpenSSH server daemon configuration file - controls port, authentication methods, and allowed users |
---
