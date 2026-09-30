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
