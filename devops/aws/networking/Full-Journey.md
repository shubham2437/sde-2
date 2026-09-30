# 6. Full Journey: What Happens When You Open google.com

1. Your laptop asks DNS for Google's IP address.
2. The laptop sends a packet to the **switch**, which forwards it to the **router** (using MAC addresses).
3. The **router** does NAT and sends the packet to your **ISP**.
4. Many **routers across the internet** pass it hop by hop to Google's server.
5. Google replies, and the reply comes back the same way: internet, then your router, then the switch, then your laptop.

---
[← 5. Internet](Internet) | [Back to Home](Home)
