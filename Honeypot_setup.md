# Cowrie Honeypot + Graylog Setup Documentation

### Ubuntu 24.04 + Raspberry Pi OS — SSH Honeypot with Centralized Logging

---

## Overview

The honeypot runs Cowrie on a Raspberry Pi, presenting a fake SSH server on port 22 that accepts attacker connections and records every credential attempt, command typed, and file download - without exposing any real system. A dedicated non-root user runs Cowrie, and `authbind` grants it permission to bind the privileged port 22 without root access. Real SSH admin access to the Pi is moved to port 2222. Logs are shipped in real time over GELF UDP to a Graylog instance running on an Ubuntu machine, where they are ingested, parsed, and displayed on a live dashboard. The setup can also be used as a classroom CTF exercise by planting flag files in Cowrie's fake filesystem.

---
