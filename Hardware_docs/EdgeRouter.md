# Edge Router

---

## 1. Overview
This router acts as an edge router to provide an internet connection to the entire network via its WAN (Wide Area Network) port. 
From LAN (Local Area Network) port 1, a cable connects directly to the WAN port on the network's main router.
Seperating WAN and LAN to 2 routers allows the LAN router to focus on internal traffic while WAN router handles WAN-facing complexities.

---

## System Architecture

```
         ISP 
          ↓ (WAN port)
 Edge Router - VPN Gateway
          ↓ (LAN port)
     Main Router 
```
