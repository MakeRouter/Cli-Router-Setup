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


## Wi-Fi 활성화 명령어

```
sudo rfkill unblock wifi
```

## NetworkManager 비활성화

```
sudo systemctl stop NetworkManager
sudo systemctl disable NetworkManager
sudo systemctl mask NetworkManager

# disable → 부팅 시 자동 실행 방지
# mask → 수동으로도 다시 실행되지 않도록 완전히 비활성화
```

## APT 패키지 업데이트

```
sudo apt update && sudo apt upgrade -y
```

---

## 구성 역할

| 구성요소 | 계층 | 역할 핵심 | 작동 시점 |
|----------|------|-----------|-----------|
| `dhcpcd` | L3 (IP 관리) | 인터페이스 IP 설정 및 라우팅 관리 | 부팅 시 항상 실행 |
| `hostapd` | L2 (무선 제어) | 무선 네트워크(SSID) 생성 및 인증 관리 | Wi-Fi AP 시작 시 |
| `dnsmasq` | L3~L7 (DHCP/DNS 서비스) | 내부 단말에 IP 할당 및 DNS 프록시 | AP가 활성화된 뒤 |
| `iptables-persistent` | L3 (NAT/포워딩) | 내부망 트래픽을 외부망으로 변환 (NAT) | 항상 상주, 커널 레벨 |

---


## HostAP / DHCP / NAT 관련 패키지 설치
```
sudo apt install -y hostapd dnsmasq iptables-persistent dhcpcd5
```

## hostapd / dnsmasq 서비스 중지

```
sudo systemctl stop hostapd || true
sudo systemctl stop dnsmasq || true

# 설치/설정 과정 중 이미 실행 중이 아니어도 오류 없이 진행되도록 하기 위해 true 처리
```

## wlan0 고정 IP 설정

`wlan0` 은 AP 게이트웨이로 사용할 내부 고정 IP(예: `192.168.50.1/24`) 를 가지며,  
`eth0` 는 상위 공유기에서 DHCP IP를 받도록 구성한다.

## dhcpcd 설정 백업 및 기존 wlan0 설정 제거

```
sudo cp -a /etc/dhcpcd.conf /etc/dhcpcd.conf.bak
sudo sed -i "/^interface wlan0$/,/^\$/d" /etc/dhcpcd.conf
```

## dhcpcd.conf 수정
```
sudo nano /etc/dhcpcd.conf
```

### 맨 하단에 입력
```
# === AP(LAN) 측 wlan0 고정 IP ===
interface wlan0
static ip_address=192.168.50.1/24

# AP 모드에서는 wpa_supplicant 사용하지 않음(충돌 방지)
nohook wpa_supplicant
```

## dhcp restart & 동작 확인

```
sudo systemctl restart dhcpcd
sudo systemctl status dhcpcd
```

---

## AP 구성을 위한 hostapd 설정

```
mkdir -p /etc/hostapd
cp -a /etc/default/hostapd /etc/default/hostapd.bak 2>/dev/null || true
```

## hostapd.conf 수정

```
sudo nano /etc/hostapd/hostapd.conf
```

### 맨 하단에 입력

```
# === 라즈베리파이 3B+ 2.4GHz AP (KR) ===
interface=wlan0
driver=nl80211

ssid=Pi_AP
country_code=KR

# 2.4GHz
hw_mode=g
channel=6
ieee80211n=1
wmm_enabled=1

# 보안 (WPA2-PSK)
auth_algs=1
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_passphrase=00000011
rsn_pairwise=CCMP

# 내부 단말 간 통신 허용 (필요 시 1로 isolation)
ap_isolate=0
```

## hostapd 데몬 설정 경로 지정
> hostapd 서비스가 사용할 설정 파일 위치를 지정한다.

```
sudo nano /etc/default/hostapd
```

### DAEMON_CONF 수정
```
DAEMON_CONF="/etc/hostapd/hostapd.conf”
```

---
## hostapd 서비스 활성화 및 시작
> hostapd 는 기본적으로 mask 상태일 수 있으므로 unmask 작업이 필요할 수 있다.

```
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
```
