# 3. Switch

A switch connects many devices *inside the same LAN* and lets them talk to each other.

**How it works:**

- Every network card has a unique hardware address called a **MAC address** (for example, `3C:5A:B4:12:9F:01`).
- The switch learns which MAC address is on which port and stores this in a **MAC address table**.
- When data (a *frame*) arrives, the switch reads the destination MAC and sends it **only to the correct port**, not to everyone.
- If it doesn't know the destination yet, it sends the frame to all ports (called *flooding*), then learns from the reply.

**Key points:**

- It works at **Layer 2 (Data Link layer)** of the OSI model.
- It uses **MAC addresses**.
- It is smarter than an old **hub**, which blindly sent every frame to every device.

**Example:** PC‑A sends a file to the office printer. The switch sends it directly to the printer's port, and other PCs don't see it.

---
[← 2. LAN](LAN) | **Next:** [4. Router →](Router)
