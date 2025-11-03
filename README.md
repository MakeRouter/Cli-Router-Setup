# Cli-Router-Setup

## 라즈베리파이 NAT 공유기 아키텍처

                 [ Internet / 상위 공유기 ]
                           │
                     (WAN, DHCP)
                     192.168.3.x/24
                           │
                   ┌────────────────┐
                   │  eth0 (WAN)    │
                   │  IP: 192.168.3.8│
                   │  DHCP Client   │
                   │  ───────────── │
                   │ Raspberry Pi 3B+│
                   │ Debian 12       │
                   │────────────────│
                   │  wlan0 (LAN, AP)│
                   │  IP: 192.168.50.1│
                   │  SSID: PiRouter_AP│
                   │  WPA2-PSK        │
                   │────────────────│
                   │  NAT + DHCP + DNS│
                   │  (iptables + dnsmasq)│
                   └────────────────┘
                           │
              192.168.50.0/24 내부망
               │         │          │
          [Smartphone] [Laptop] [IoT Device]
             DHCP로 IP 자동할당
             ex) 192.168.50.101~200
