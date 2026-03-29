# Cowrie Honeypot + Graylog Setup Documentation

### Ubuntu 24.04 + Raspberry Pi OS — SSH Honeypot with Centralized Logging

---

## Overview

The honeypot runs Cowrie on a Raspberry Pi, presenting a fake SSH server on port 22 that accepts attacker connections and records every credential attempt, command typed, and file download - without exposing any real system. A dedicated non-root user runs Cowrie, and `authbind` grants it permission to bind the privileged port 22 without root access. Real SSH admin access to the Pi is moved to port 2222. Logs are shipped in real time over GELF UDP to a Graylog instance running on an Ubuntu machine, where they are ingested, parsed, and displayed on a live dashboard. The setup can also be used as a classroom CTF exercise by planting flag files in Cowrie's fake filesystem.

---

## Table of Contents

1. [System Architecture](#system-architecture)

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
