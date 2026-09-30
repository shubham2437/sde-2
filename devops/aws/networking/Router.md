# 4. Router

A router connects **different networks** together, for example your home LAN and the internet.

**How it works:**

- It uses **IP addresses** (for example, `192.168.1.10` or `142.250.183.14`).
- It keeps a **routing table**, a list of which path leads to which network.
- When a *packet* arrives, the router checks the destination IP and forwards it toward the right network, choosing the best path.

**Key points:**

- It works at **Layer 3 (Network layer)**.
- It uses **IP addresses**.
- It does **NAT (Network Address Translation)**: all your home devices have private IPs (`192.168.x.x`), but the router shows the internet one public IP and keeps track of who asked for what.
- It often gives devices their IPs automatically using **DHCP**.
- It often includes a basic **firewall**.

Your home "Wi‑Fi box" is usually a router, switch and Wi‑Fi access point combined in one device.

---
[← 3. Switch](Switch) | **Next:** [5. Internet →](Internet)
