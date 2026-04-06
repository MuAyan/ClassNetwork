# Edge Router

---

## Overview
This router acts as an edge router to provide an internet connection to the entire network via its WAN (Wide Area Network) port. 
From LAN (Local Area Network) port 1, a cable connects directly to the WAN port on the network's main router.
Seperating WAN and LAN to 2 routers allows the LAN router to focus on internal traffic while WAN router handles WAN-facing complexities.

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Configuration Settings](#configuration-settings)

## 1. System Architecture

```
         ISP 
          ↓ (WAN port)
 Edge Router - VPN Gateway
          ↓ (LAN port)
     Main Router 
```
## Configuration Settings

1. Navigate to quick setup in left-hand bar.
2. Select WAN port that is connected physically to ISP.
3. Set connection type to Dynamic IP.
4. Click next.
