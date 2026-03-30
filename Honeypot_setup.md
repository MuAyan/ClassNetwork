# Cowrie Honeypot + Graylog Setup Documentation

### Ubuntu 24.04 + Raspberry Pi OS — SSH Honeypot with Centralized Logging

---

## Overview

The honeypot runs Cowrie on a Raspberry Pi, presenting a fake SSH server on port 22 that accepts attacker connections and records every credential attempt, command typed, and file download - without exposing any real system. A dedicated non-root user runs Cowrie, and `authbind` grants it permission to bind the privileged port 22 without root access. Real SSH admin access to the Pi is moved to port 2222. Logs are shipped in real time over GELF UDP to a Graylog instance running on an Ubuntu machine, where they are ingested, parsed, and displayed on a live dashboard. The setup can also be used as a classroom CTF exercise by planting flag files in Cowrie's fake filesystem.

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Prerequisites](#prerequisites)

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

> *If setting up on virtual machines, both VMs must be on the same virtual network. NAT mode in VMware Workstation satisfies this requirement — host-only does not, as it blocks internet access needed for package installation.*

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
 
---
