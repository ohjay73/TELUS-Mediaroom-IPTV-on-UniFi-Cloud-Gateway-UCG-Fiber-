# TELUS-Mediaroom-IPTV-on-UniFi-Cloud-Gateway-UCG-Fiber-
TELUS Mediaroom IPTV on UniFi Cloud Gateway (UCG-Fiber)
TELUS Mediaroom IPTV on UniFi Cloud Gateway (UCG-Fiber)A comprehensive guide to resolving the 4-minute, 55-second (295-second) multicast drop on TELUS Mediaroom (Optik TV) set-top boxes (STBs) when running downstream of a UniFi Cloud Gateway (UCG-Fiber) and managed/smart switches.1. Problem Root CauseWhen tuning to a live channel on a TELUS Mediaroom STB:0–15 Seconds (Unicast Instant Channel Change): The server delivers an initial unicast burst via UDP/RTP so playback begins immediately without lag.15 Seconds+ (Multicast Transition): The STB issues an IGMPv3 Join to switch to the live multicast group (239.x.x.x / 224.0.0.0/4).4 Minutes, 55 Seconds (~295 Seconds): The stream freezes dead.Why the 4m 55s Timeout OccursUnder standard IGMPv3 specifications:Query Interval: 125 secondsGroup Membership Interval: $(2 \times 125\text{s}) + \text{Query Response Buffer} = \mathbf{295\text{ seconds}}$.The UCG-Fiber's embedded igmpproxy implementation accepts the initial join from the STB and subscribes upstream to TELUS. However, it fails to inject periodic IGMP General Queries (224.0.0.1) down into the LAN.Because the STB is never actively queried:It never issues recurring IGMP Membership Reports.Switches with IGMP Snooping enabled and upstream routing tables age out the group membership.Exactly at the 295-second mark, the multicast state expires, and the stream terminates.2. UniFi Cloud Gateway ConfigurationA. Enable IGMP Proxy (UniFi Network 10.x)Navigate to Settings > Internet > WAN1 (connected to the TELUS NAH 10G bridge).Enable IGMP Proxy.Select your primary LAN / Default network as the Viewing Network.B. Firewall Rules (Policy Engine / Security)Create the necessary firewall openings for incoming TELUS IPTV traffic:Firewall Groups:TELUS Subnets: 207.0.0.0/8, 209.0.0.0/8, 216.0.0.0/8Multicast Range: 224.0.0.0/4Internet In (WAN IN):Action: AcceptProtocol: UDPSource: Address Group TELUS SubnetsDestination: Address Group Multicast RangeInternet Local (WAN LOCAL):Action: AcceptProtocol: IGMP (IPv4 Protocol 2)Source / Destination: Any3. Switch Configuration (TP-Link Easy Smart / TL-SG108E)If an intermediate managed switch (e.g., TP-Link TL-SG108E) is placed between the UCG-Fiber and the STB:Step 1: 802.1Q VLAN SetupPort 8 (Trunk Uplink back to Gateway / Main Switch): Member of Default VLAN 1 (Untagged) and VLAN 50 (Tagged).Port 7 (Connected to Mediaroom STB): Untagged Member of VLAN 50.Ports 1–6 (Local LAN Clients): Untagged Members of Default VLAN 1.Step 2: 802.1Q PVID SettingSet Port 7 PVID to 50.Keep Ports 1–6 and Port 8 set to PVID 1.Step 3: IGMP Snooping BehaviorNavigate to Switching > IGMP Snooping.If utilizing pure routing through the UCG-Fiber without a dedicated VLAN 50 hardware bypass to the NAH, disable IGMP Snooping on intermediate smart switches to prevent the local switch ASIC from independently pruning the unqueried port.Commit changes under System Tools > Save Config.4. The Keep-Alive Solution: Docker IGMP QuerierBecause the UCG-Fiber does not emit periodic queries, running a lightweight container on a local Synology NAS, QNAP NAS, or Linux server forces downstream devices (and the gateway) to continuously refresh multicast subscriptions.Technical Requirementsnetwork_mode: host: Uses the physical host network adapter directly, bypassing Docker NAT to transmit raw Layer 2/3 multicast frames across the subnet.cap_add: [ NET_RAW ]: Grants the raw socket capability (socket.SOCK_RAW) required to construct protocol 2 (IGMP) packets without granting full --privileged access.Query Frequency: Broadcasts an IGMPv2 General Query (224.0.0.1) every 90 seconds (comfortably under the 295-second timeout window).5. Docker Compose DeploymentStandalone docker-compose.yml (Inline Python)This Compose file embeds the querier directly into the execution command, requiring no auxiliary script files:YAMLversion: "3.8"

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
6. NAS Deployment InstructionsSynology DSM (Container Manager)Open Container Manager > Project > Create.Set Project Name to igmp-querier.Choose a path (e.g., /docker/igmp-querier).Select Create docker-compose.yml, paste the YAML above, and complete the wizard.QNAP QTS / QuTS hero (Container Station)Open Container Station > Applications > Create.Set Application Name to igmp-querier.Paste the YAML into the configuration field.Click Validate YAML, then click Create.7. Verification & LogsOpen container logs in Container Manager / Station:PlaintextStarting IGMP Querier on LAN...
[18:30:00] IGMP General Query sent to 224.0.0.1
[18:31:30] IGMP General Query sent to 224.0.0.1
Power on the TELUS Mediaroom STB and tune to any live HD channel.Validate that playback sustains past the 4:55 / 5:00 mark. The periodic general query solicits membership reports from the box, maintaining the multicast pipeline indefinitely.
