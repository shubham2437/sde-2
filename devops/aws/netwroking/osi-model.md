[⬅ Back to Aa](Aa.md)

# Networking Notes

> Tip: on GitHub, click the **☰ outline button** at the top right of this file to see all topics in a sidebar and jump to any one.

## 1. OSI Model: 7 Layers

A way to remember them from bottom to top: **"Please Do Not Throw Sausage Pizza Away"**

| No. | Layer | Main job | Data unit | Address / Example | Device |
|---|---|---|---|---|---|
| 7 | Application | Services for the user's apps | Data | HTTP, FTP, DNS, SMTP | — |
| 6 | Presentation | Format, encrypt, compress data | Data | SSL/TLS, JPEG, ASCII | — |
| 5 | Session | Start, manage and end connections | Data | NetBIOS, RPC | — |
| 4 | Transport | Reliable end-to-end delivery | Segment | Port numbers, TCP/UDP | — |
| 3 | Network | Routing between networks | Packet | IP address | Router |
| 2 | Data Link | Delivery within one LAN | Frame | MAC address | Switch, bridge |
| 1 | Physical | Sends raw bits over the medium | Bits | Cables, signals | Hub, repeater, cable |

## 2. LAN (Local Area Network)

A LAN is a small network of devices in one place, like a home, office, school or lab. Your laptop, phone, printer and smart TV at home all sit on the same LAN.

- It covers a small area (one building or floor).
- It is fast (100 Mbps to 10 Gbps) because distances are short.
- It is privately owned: you or your company controls it.
- Devices connect by cable (Ethernet) or wirelessly (Wi‑Fi, which is a wireless LAN, or WLAN).

## 3. Switch

A switch connects many devices *inside the same LAN* and lets them talk to each other.

### How a switch works

- Every network card has a unique hardware address called a **MAC address** (for example, `3C:5A:B4:12:9F:01`).
- The switch learns which MAC address is on which port and stores this in a **MAC address table**.
- When data (a *frame*) arrives, the switch reads the destination MAC and sends it **only to the correct port**, not to everyone.
- If it doesn't know the destination yet, it sends the frame to all ports (called *flooding*), then learns from the reply.

### Switch key points

- It works at **Layer 2 (Data Link layer)** of the OSI model.
- It uses **MAC addresses**.
- It is smarter than an old **hub**, which blindly sent every frame to every device.

### Switch example

PC‑A sends a file to the office printer. The switch sends it directly to the printer's port, and other PCs don't see it.

## 4. Router

A router connects **different networks** together, for example your home LAN and the internet.

### How a router works

- It uses **IP addresses** (for example, `192.168.1.10` or `142.250.183.14`).
- It keeps a **routing table**, a list of which path leads to which network.
- When a *packet* arrives, the router checks the destination IP and forwards it toward the right network, choosing the best path.

### Router key points

- It works at **Layer 3 (Network layer)**.
- It uses **IP addresses**.
- It does **NAT (Network Address Translation)**: all your home devices have private IPs (`192.168.x.x`), but the router shows the internet one public IP and keeps track of who asked for what.
- It often gives devices their IPs automatically using **DHCP**.
- It often includes a basic **firewall**.

Your home "Wi‑Fi box" is usually a router, switch and Wi‑Fi access point combined in one device.

## 5. Internet

The internet is a **huge network of networks**: millions of LANs and routers around the world connected together.

### How the internet works

- Your router connects to your **ISP** (Internet Service Provider, such as Jio, Airtel or BSNL).
- ISPs connect to bigger ISPs, and those connect worldwide through **undersea fiber cables**.
- Everything follows common rules called **TCP/IP**:
  - **IP** addresses and routes each packet.
  - **TCP** makes sure all packets arrive, in the right order.
- **DNS** converts names like `google.com` into IP addresses, like a phonebook.

## 6. Full Journey: What Happens When You Open google.com

1. Your laptop asks DNS for Google's IP address.
2. The laptop sends a packet to the **switch**, which forwards it to the **router** (using MAC addresses).
3. The **router** does NAT and sends the packet to your **ISP**.
4. Many **routers across the internet** pass it hop by hop to Google's server.
5. Google replies, and the reply comes back the same way: internet, then your router, then the switch, then your laptop.

---

[⬅ Back to Aa](Aa.md)
