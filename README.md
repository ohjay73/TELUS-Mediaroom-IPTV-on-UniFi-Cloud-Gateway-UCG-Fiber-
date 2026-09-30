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
