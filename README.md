# Class Network

## Overview
This project is a local network consisting of configured routers, switches - dumb and smart with configured VLANS, machines running services including Graylog, pfSense, Nextcloud, IDS, and dedicated raspberry pi's serving as honeypots.

This repository aims serves the purpose of organizing and explaining parts of a local network built to educate students at PTC. It provides a learning oppurtunity with CTF's planted in workstations/honeypots for students to find. As students are working through different challenges, the data can be logged and analyzed for different attack patterns relating back to the real world.


## Components

[Hardware Documentation](./Hardware_docs) including Edgerouter, Mainrouter, and Access points.

[NAS](./Services/NAS_Documentation.md) setup and configuration  &mdash;  Complete
 > Nextcloud configured on Ubuntu.

[Honeypot](./Services/Honeypot_setup.md) setup and configuration  &mdash;  Complete
 > Graylog configured on Ubuntu, Cowrie setup on Raspberry pi 4.
