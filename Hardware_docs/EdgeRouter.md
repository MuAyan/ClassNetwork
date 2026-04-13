# Edge Router

---

## Overview
This router acts as an edge router to provide an internet connection to the entire network via its WAN (Wide Area Network) port. 
From LAN (Local Area Network) port 1, a cable connects directly to the WAN port on the network's main router.
Seperating WAN and LAN to 2 routers allows the LAN router to focus on internal traffic while WAN router handles WAN-facing complexities.

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [WAN Configuration](#wan-configuration)
3. [LAN Configuration](#lan-configuration)

## 1. System Architecture

```
         ISP 
          ↓ (WAN port)
 Edge Router - VPN Gateway
          ↓ (LAN port)
     Main Router 
```
## WAN Configuration

1. Navigate to quick setup in left-hand bar.
2. Select WAN port that is connected physically to ISP.
3. Set connection type to Dynamic IP.
4. Click next.

---
## LAN Configuration

1. Navigate to Network -> LAN in left-hand bar.
2. Scroll down to network list and click edit under Operation.
3. Set IP address to something different than main router, same subnet. (192.168.0.xxx)
4. Scroll down to DHCP and select DHCP relay
5. Set Status to Enabled
6. Change Server Address to the default gateway set on main router
