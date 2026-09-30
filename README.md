# TELUS Mediaroom IPTV on UniFi Cloud Gateway (UCG-Fiber)

Resolve the **4-minute, 55-second (~295-second) multicast drop** on TELUS Mediaroom (Optik TV) set-top boxes (STBs) operating downstream of a UniFi Cloud Gateway (UCG-Fiber) and smart switches.

---

## The Problem: The 4m 55s Timeout

When tuning into a live channel on a TELUS Mediaroom STB:
1. **0–15 Seconds (Instant Channel Change):** The stream starts immediately via an initial unicast UDP/RTP burst from the TELUS servers.
2. **15 Seconds+ (Multicast Transition):** The STB issues an `IGMPv3 Join` request to join the multicast group (`239.x.x.x` / `224.0.0.0/4`).
3. **4 Minutes, 55 Seconds (~295 Seconds):** The video stream freezes completely.

### Root Cause
Under RFC 3376 (IGMPv3), multicast group state timeouts follow:
$$\text{Group Membership Interval} = (\text{Robustness Variable} \times \text{Query Interval}) + \text{Query Response Interval}$$
$$(2 \times 125\text{s}) + 10\text{s} + \text{buffer} \approx \mathbf{295\text{ seconds (4m 55s)}}.$$

The UCG-Fiber's embedded `igmpproxy` service successfully handles the initial Join upstream to TELUS, but **fails to inject periodic IGMP General Queries (`224.0.0.1`) down into the local LAN**. Because the STB is never polled:
* The STB remains silent and does not send periodic IGMP Membership Reports.
* Switch ASICs and upstream multicast tables expire the group membership.
* At exactly 295 seconds, the multicast feed is severed.

---

## Architecture Overview

```text
[TELUS NAH / ONT]
       │
       │ (10G WAN / Bridge)
       ▼
[UniFi UCG-Fiber]
       │
       │ (LAN Trunk)
       ▼
[Managed / Smart Switch] (e.g., TL-SG108E, UniFi Pro Max)
       │
       ├──── Port X: Mediaroom STB
       └──── Port Y: Local LAN Clients

[Local NAS / Docker Host] (Synology / QNAP)
       │
       └── Runs: `igmp-querier` container (Emits periodic queries to 224.0.0.1)
```

---

## 1. UniFi Cloud Gateway Configuration

### A. Enable IGMP Proxy (UniFi Network 10.x)
1. Go to **Settings > Internet > WAN1**.
2. Toggle on **IGMP Proxy**.
3. Set your target LAN / Default network as the **Viewing Network**.

### B. Firewall Rules
Allow inbound multicast streams and IGMP signaling across the gateway:

1. **Firewall Groups:**
   * **TELUS Subnets:** `207.0.0.0/8`, `209.0.0.0/8`, `216.0.0.0/8`
   * **Multicast Range:** `224.0.0.0/4`
2. **Internet In (WAN IN):**
   * **Action:** Accept
   * **Protocol:** UDP
   * **Source:** Address Group `TELUS Subnets`
   * **Destination:** Address Group `Multicast Range`
3. **Internet Local (WAN LOCAL):**
   * **Action:** Accept
   * **Protocol:** IGMP (IPv4 Protocol 2)
   * **Source / Destination:** Any

---

## 2. Downstream Switch Configuration

### IGMP Snooping Considerations
* If routing Mediaroom traffic through the gateway's IGMP proxy without an active querier, **disable IGMP Snooping** on intermediate switches (e.g., TP-Link Easy Smart switches). Otherwise, switch hardware will silently prune the port after 260–300 seconds.
* With the Docker IGMP Querier running, IGMP Snooping can safely remain enabled.

### Dedicated VLAN 50 Bypass (Alternative to Routing)
To bypass the UCG-Fiber entirely for TV traffic, map the STB directly to an isolated VLAN bridged to the TELUS NAH:
* **Switch Uplink:** Tagged on VLAN 50.
* **STB Port:** Untagged on VLAN 50, with **PVID 50**.
* **TELUS NAH Port 1:** Patched into an untagged VLAN 50 port in the electrical room.

---

## 3. The Fix: Docker IGMP Querier

Because the gateway fails to emit downstream queries, this containerized service sends raw IGMPv2 General Queries (`224.0.0.1`) every 90 seconds, forcing the STB to transmit renewal reports and keeping the stream alive indefinitely.

### Key Requirements
* **`network_mode: "host"`**: Interacts directly with the physical network adapter, bypassing Docker NAT to send raw multicast packets to the subnet.
* **`cap_add: [ NET_RAW ]`**: Grants raw socket privileges (`socket.SOCK_RAW`) for Layer 3 IGMP protocol construction.

### `docker-compose.yml`

```yaml
version: "3.8"

services:
  igmp-querier:
    image: python:3-alpine
    container_name: igmp-querier
    restart: unless-stopped
    network_mode: "host"
    cap_add:
      - NET_RAW
    command:
      - python3
      - -u
      - -c
      - |
        import socket, time
        # IGMPv2 General Query: Type 0x11, Max Resp 10s (0x64), Checksum 0xee9b, Group 0.0.0.0
        query_pkt = b'\x11\x64\xee\x9b\x00\x00\x00\x00'
        dest = '224.0.0.1'
        print('Starting IGMP Querier on LAN...', flush=True)
        sock = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_IGMP)
        sock.setsockopt(socket.IPPROTO_IP, socket.IP_MULTICAST_TTL, 1)
        while True:
            try:
                sock.sendto(query_pkt, (dest, 0))
                print(f"[{time.strftime('%X')}] IGMP General Query sent to 224.0.0.1", flush=True)
            except Exception as e:
                print(f"Error sending IGMP packet: {e}", flush=True)
            time.sleep(90)
```

---

## 4. Deployment Instructions

### Synology DSM (Container Manager)
1. Open **Container Manager > Project > Create**.
2. Name the project `igmp-querier` and set a directory path.
3. Select **Create docker-compose.yml**, paste the configuration above, and finish the setup wizard.

### QNAP QTS / QuTS hero (Container Station)
1. Open **Container Station > Applications > Create**.
2. Name the application `igmp-querier`.
3. Paste the Compose YAML into the editor.
4. Click **Validate YAML**, then click **Create**.

### Standard Docker CLI
```bash
docker run -d \
  --name igmp-querier \
  --restart unless-stopped \
  --network host \
  --cap-add NET_RAW \
  python:3-alpine \
  python3 -u -c "import socket, time; p=b'\x11\x64\xee\x9b\x00\x00\x00\x00'; s=socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_IGMP); s.setsockopt(socket.IPPROTO_IP, socket.IP_MULTICAST_TTL, 1); [s.sendto(p, ('224.0.0.1', 0)) or time.sleep(90) for _ in iter(int, 1)]"
```

---

## 5. Verification

1. Check container output:
   ```text
   Starting IGMP Querier on LAN...
   IGMP General Query sent to 224.0.0.1
   IGMP General Query sent to 224.0.0.1
   ```
2. Turn on the Mediaroom STB and tune to any live channel.
3. Confirm playback continues seamlessly past the **4:55 / 5:00** mark.
