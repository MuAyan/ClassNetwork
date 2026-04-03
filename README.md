# Class Network

## Overview
This project is a local network consisting of routers, managed and unmanaged switches with configured VLANs, machines running services including Graylog, pfSense, Nextcloud, and IDS, and dedicated Raspberry Pis serving as honeypots.

This repository serves the purpose of organizing and explaining parts of a local network built to educate students at PTC. It provides a learning opportunity with CTFs planted in workstations/honeypots for students to find. As students are working through different challenges, the data can be logged and analyzed for different attack patterns relating back to the real world.


## Components

[Hardware Documentation](./Hardware_docs) including Edgerouter, Mainrouter, and Access points.

[NAS](./Services/NAS_Documentation.md) setup and configuration  &mdash;  Complete
 > Nextcloud configured on Ubuntu.

[Honeypot](./Services/Honeypot_setup.md) setup and configuration  &mdash;  Setup in-progress
 > Graylog configured on Ubuntu, Cowrie setup on Raspberry pi 4.

## Hardware
5900x workstation, Raspberry Pi 4, consumer edge router, TP-Link EAP245 WAP.
