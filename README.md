# Cli-Router-Setup

## 라즈베리파이 NAT 공유기 아키텍처

```
[ Internet / 상위 공유기 ]
                          │
                     (WAN, DHCP)
                  172.30.1.x/24 대역
                          │
                  ┌──────────────────────┐
                  │   eth0 (WAN Port)    │
                  │   DHCP Client        │
                  │  ──────────────────  │
                  │   Raspberry Pi 4B    │
                  │   OpenWrt 23.05      │ <── *v24.xx 플래싱 오류로 v23.05 채택
                  │  ──────────────────  │
                  │ [ System Core ]      │
                  │  - Procd (Init)      │
                  │  - UCI (Config)      │
                  │                      │
                  │ [ Network Service ]  │
                  │  - network (L3)       │
                  │  - wireless (L2, AP)  │
                  │  - dhcp (DHCP/DNS)│
                  │  - firewall (NAT/Filter)  │
                  │                      │
                  │ [ Custom Service ]   │
                  │  - Mosquitto (Broker)│ <── 포트 1883 개방
                  │  - LEDControl (C)│ <── LED 제어
                  │  ──────────────────  │
                  │   br-lan (LAN, AP)   │
                  │   IP: 192.168.1.1   │
                  │   SSID: RPI-OpenWrt        │
                  └──────────────────────┘
                          │
             192.168.1.0/24 내부망 (Private Network)
              │          │          │
         [Smartphone] [Laptop] [IoT Sensors]

```


## WAN 활성화

```
vi /etc/config/network

config interface 'loopback'
        option device 'lo'
        option proto 'static'
        option ipaddr '127.0.0.1'
        option netmask '255.0.0.0'

config globals 'globals'
        option ula_prefix 'fd69:0f75:9bf8::/48'

config device
        option name 'br-lan'
        option type 'bridge'
        # list ports 'eth0'

config interface 'lan'
        option device 'br-lan'
        option proto 'static'
        option ipaddr '192.168.1.1'
        option netmask '255.255.255.0'
        option ip6assign '60'

config interface 'wan'
        option device 'eth0'
        option proto 'dhcp'
```

---

## opkg 패키지 업데이트

```
opkg update
```

---

## 구성 역할

| 구성요소 | 계층 | 역할 핵심 | 작동 시점 |
|----------|------|-----------|-----------|
| `network` | L3 (IP 관리) | 인터페이스 IP 설정 | 부팅 시 항상 실행 |
| `wireless` | L2 (무선 제어) | 무선 AP 생성 및 인증 (OpenWrt가 자동 제어) | Wi-Fi AP 시작 시 |
| `dhcp` | L3~L7 (DHCP/DNS 서비스) | 내부 단말에 IP 할당 및 DNS 프록시 | AP가 활성화전 실 |
| `firewall` | L3 (NAT/포워딩) | 내부망 트래픽을 외부망으로 변환 (NAT) | 항상 상주, 커널 레벨 |

---

---
## 리뷰
https://kksp12y.tistory.com/category/Network
