---
name: qbittorrent-frp-setup
description: qBittorrent + frp intranet penetration setup details for future blog/diary
metadata:
  type: project
---

User runs qBittorrent behind campus network. Campus network: no IPv6 public, IPv4 TCP NAT-3, UDP NAT-4 — can't do hole-punching. Solution: frp intranet penetration.

Problems encountered:
1. **frpc users show as 127.0.0.1** — peerbanhelper can't ban leeching clients because they all appear as localhost
2. **Outbound connections work fine** — user can actively connect out to peers with public IPs, those show correct IPs. This gives "semi-public" seeding capability.
3. **Tracker via proxy** — records VPS public IP, so NAT'd peers can find user through tracker → VPS IP → frp tunnel
4. **DHT/PEX issues** — peers from DHT/PEX can't discover user's public IP. DHT count stayed at 0, possibly container-related. Switched to host network mode which worked temporarily, but later went back to 0. Ended up disabling DHT.

Related: [[pending-diary-tray-host]]
