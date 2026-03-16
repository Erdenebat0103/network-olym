# 🏆 Сүлжээний Олимпиадад Бэлдэх Зөвлөмж

> Энэхүү зөвлөгөө нь сүлжээний олимпиадын өмнөх жилийн даалгаварт суурилсан бэлтгэл гарын авлага юм.

---

## 📋 Даалгаварын Ерөнхий Тойм

Байгууллагын сүлжээ **2 сайтад** хуваагдан байрлах ба **ISP-ийн сүлжээгээр** дамжин холбогдоно.

| Хэсэг | Оноо | Тайлбар |
|---|---|---|
| Troubleshoot Site 1 | 10 оноо | 5 асуудал илрүүлж засах |
| Configuration ISP 1 | 5 оноо | ISP сүлжээний протокол тохиргоо |
| Configure Site 2 | 15 оноо | L2, L3, IP services тохиргоо |
| Monitoring | 10 оноо | SNMP monitoring тохиргоо |
| **Нийт** | **40 оноо** | |

---

## 1. 🔧 Layer 2 Technologies (Суурь)

Даалгаварт шаардагдаж байгаа L2 технологиуд:

### 1.1 STP (Spanning Tree Protocol)
- Root bridge сонголт, STP priority тохиргоо
- Site1 дээр VLAN 10 STP root, VLAN 99 STP secondary гэх мэт
- `spanning-tree vlan <id> root primary`
- `spanning-tree vlan <id> root secondary`

### 1.2 VLAN & Trunking
- VLAN үүсгэх, trunk port тохиргоо
- Allowed VLANs зааж өгөх
- Access port зөв VLAN-д оноох

### 1.3 MCLAG / Port-Channel
- Multi-Chassis LAG (Site2 switch-server хооронд)
- LACP тохиргоо
- `channel-group <id> mode active`

### 1.4 HSRP (Hot Standby Router Protocol)
- Primary/standby тохиргоо
- Border1 дээр primary байх
- Interface tracking — uplink тасрахад standby руу шилжих
- `standby <group> ip <virtual-ip>`
- `standby <group> priority <value>`
- `standby <group> track <interface> decrement <value>`

---

## 2. 🌐 Layer 3 Technologies (Routing)

### 2.1 OSPFv2 + OSPFv3 (Dual Stack)
- IPv4 + IPv6 зэрэг ажиллуулах
- Area тохиргоо, network statement
- Passive-interface тохиргоо
- Зөвхөн loopback болон router хоорондийн сүлжээ OSPF-д оролцох

```
router ospf 1
 network <network> <wildcard> area <id>
 passive-interface default
 no passive-interface <interface>

ipv6 router ospf 1
 passive-interface default
 no passive-interface <interface>
```

### 2.2 BGP — IBGP Full Mesh
- ISP роутерүүд хооронд IBGP full mesh (IPv4 + IPv6)
- `next-hop-self` тохиргоо
- Loopback-аар peer хийх, `update-source loopback0`

```
router bgp <AS>
 neighbor <ip> remote-as <AS>
 neighbor <ip> update-source Loopback0
 neighbor <ip> next-hop-self
 address-family ipv6
  neighbor <ipv6> activate
```

### 2.3 BGP — EBGP Peering
- Site ↔ ISP хооронд EBGP
- Prefix filtering — зөвхөн public IP/IPv6 summary зөвшөөрөх

```
ip prefix-list ALLOW-PUBLIC permit <public-range>
route-map FILTER-IN permit 10
 match ip address prefix-list ALLOW-PUBLIC

router bgp <AS>
 neighbor <ip> remote-as <remote-AS>
 neighbor <ip> route-map FILTER-IN in
```

### 2.4 BGP Path Manipulation (Маш чухал!)

#### Outbound Traffic (Site2 → Internet) — Local Preference ашиглах
- Border1 дээр өндөр local-preference оноож outbound traffic-ийг border1-ээр дамжуулах

```
route-map SET-LP permit 10
 set local-preference 200

router bgp <AS>
 neighbor <isp-ip> route-map SET-LP in   ! border1 дээр
```

#### Inbound Traffic (Internet → Site2) — MED ашиглах
- ⚠️ AS prepend ашиглахгүй!
- MED (Multi-Exit Discriminator) эсвэл BGP community ашиглах

```
route-map SET-MED permit 10
 set metric 50    ! border1 дээр бага MED

route-map SET-MED-HIGH permit 10
 set metric 200   ! border2 дээр өндөр MED
```

### 2.5 Default Route зарлах
- BGP-ээр EBGP peer лүүгээ default route зарлах
- ISP-ийн өөрийн summary сүлжээг зарлах

```
router bgp <AS>
 neighbor <ip> default-originate
 network <summary-network> mask <mask>
```

---

## 3. 🔒 IP Services & Tunneling

### 3.1 NAT (Network Address Translation)
- Зөвхөн хэрэглэгчийн сүлжээг public IP range руу NAT хийх
- Site1 ↔ Site2 хооронд NAT хийгдэхгүй

```
access-list 100 deny ip 10.1.0.0 0.0.255.255 10.2.0.0 0.0.255.255
access-list 100 permit ip 10.1.0.0 0.0.255.255 any

ip nat inside source list 100 pool <POOL> overload
ip nat pool <POOL> <start-ip> <end-ip> netmask <mask>
```

### 3.2 GRE Tunnel
- Site1 ↔ Site2 хооронд GRE tunnel үүсгэх
- Tunnel source/destination тохиргоо

```
interface Tunnel0
 ip address <tunnel-ip> <mask>
 tunnel source <public-ip>
 tunnel destination <remote-public-ip>
```

### 3.3 GRE over IPSec
- IPSec transform set, crypto map тохиргоо
- Tunnel protection ашиглах

```
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14

crypto isakmp key <KEY> address <peer-ip>

crypto ipsec transform-set TSET esp-aes 256 esp-sha256-hmac
 mode transport

interface Tunnel0
 tunnel protection ipsec profile <PROFILE>
```

### 3.4 EBGP over GRE Tunnel
- Tunnel interface дээр EBGP neighbor тохиргоо
- Зөвхөн RFC1918 хаягууд зарлагдах

```
ip prefix-list RFC1918 permit 10.0.0.0/8 le 32
ip prefix-list RFC1918 permit 172.16.0.0/12 le 32
ip prefix-list RFC1918 permit 192.168.0.0/16 le 32

route-map ALLOW-RFC1918 permit 10
 match ip address prefix-list RFC1918

router bgp <AS>
 neighbor <tunnel-peer-ip> remote-as <remote-AS>
 neighbor <tunnel-peer-ip> route-map ALLOW-RFC1918 out
```

---

## 4. 🔍 Troubleshooting (10 оноо — Маш чухал!)

Site1 дээр 5 асуудал олж засах шаардлагатай. Түгээмэл алдаанууд:

### Шалгах зүйлс:

| # | Асуудлын төрөл | Шалгах команд |
|---|---|---|
| 1 | **OSPF** — area mismatch, hello/dead timer, network statement | `show ip ospf neighbor`, `show ip ospf interface` |
| 2 | **STP** — priority буруу, root bridge буруу | `show spanning-tree`, `show spanning-tree root` |
| 3 | **HSRP** — group number, virtual IP буруу | `show standby`, `show standby brief` |
| 4 | **VLAN** — trunk allowed vlan дутуу, access port буруу | `show vlan brief`, `show interfaces trunk` |
| 5 | **BGP** — AS number, neighbor address буруу | `show bgp summary`, `show ip bgp neighbors` |
| 6 | **ACL** — буруу direction, буруу permit/deny | `show access-lists`, `show ip interface` |
| 7 | **IP addressing** — subnet mask буруу, IP давхцал | `show ip interface brief` |

### Troubleshoot хийх алхам:
1. `show ip interface brief` — Interface-ийн статус шалгах
2. `show running-config` — Тохиргоо шалгах
3. `show log` — Error message шалгах
4. `ping` / `traceroute` — Connectivity шалгах
5. `show cdp neighbors` — Физик холболт шалгах

---

## 5. 📊 Monitoring (10 оноо)

### 5.1 SNMPv2c тохиргоо (Router/Switch дээр)

```
snmp-server community <COMMUNITY-STRING> RO
snmp-server location <LOCATION>
snmp-server contact <CONTACT>
snmp-server host <monitoring-server-ip> version 2c <COMMUNITY-STRING>
```

### 5.2 Monitoring Server
- **URL:** monitoring.olympiad.mn
- **Credentials:** admin / Admin123
- **PC login:** user / Test123
- Site1 болон Site2-ийн бүх router, switch нэмэх
- SNMPv2-оор хянах

### 5.3 Wireshark
- Wireshark дээрээс SNMP community string олох
- Filter: `snmp` эсвэл `http`
- Password олох: packet capture → Follow TCP/UDP stream

### 5.4 DNS тохиргоо
- monitoring.olympiad.mn нэрийг resolve хийх
- `ip name-server <dns-ip>`
- `ip domain-lookup`

---

## 6. 🛠️ Ашиглах Хэрэгслүүд

| Хэрэгсэл | Зорилго |
|---|---|
| **GNS3 / EVE-NG** | Бүх лабыг энд хийх (Cisco IOSv, IOSvL2) |
| **Wireshark** | Packet analysis, password олох |
| **PuTTY / SecureCRT** | Console/SSH хандалт |
| **Notepad++** | Тохиргооны template бэлдэх |

---

## 7. 📅 Бэлдэх Төлөвлөгөө (4 долоо хоног)

### 1-р долоо хоног: Layer 2 Суурь
- [ ] VLAN үүсгэх, Trunking тохиргоо
- [ ] STP root/secondary тохиргоо
- [ ] HSRP primary/standby + interface tracking
- [ ] MCLAG / Port-Channel (LACP)

### 2-р долоо хоног: Layer 3 Routing
- [ ] OSPFv2 + OSPFv3 dual-stack тохиргоо
- [ ] BGP IBGP full mesh тохиргоо
- [ ] BGP EBGP peering + prefix filtering

### 3-р долоо хоног: IP Services
- [ ] NAT тохиргоо (site хооронд NAT-гүй)
- [ ] GRE Tunnel + IPSec тохиргоо
- [ ] EBGP over GRE tunnel
- [ ] BGP path manipulation (Local Preference, MED)

### 4-р долоо хоног: Monitoring + Дүгнэлт
- [ ] SNMPv2c тохиргоо бүх төхөөрөмж дээр
- [ ] Monitoring server суулгах, төхөөрөмж нэмэх
- [ ] Wireshark packet analysis дасгал
- [ ] Full topology troubleshooting дасгал

---

## 8. 📚 Санал Болгох Нөөцүүд

### Сертификат/Курс:
- **Cisco CCNP ENCOR 350-401** — Даалгаварын ихэнх сэдвүүд энд багтана
- **Cisco CCNP ENARSI 300-410** — BGP, OSPF advanced troubleshooting

### YouTube сувгууд:
- **David Bombal** — GNS3 лаб, BGP, OSPF тайлбар
- **NetworkChuck** — Networking суурь ойлголт
- **Keith Barker (CBT Nuggets)** — HSRP, STP, BGP гүнзгийрүүлсэн

### Лаб платформ:
- **GNS3 Academy** — Үнэгүй лабууд
- **EVE-NG Community** — Виртуал лаб орчин

### RFC & Стандарт:
- **RFC1918** — Private IP хаягийн муж (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)

---

## 9. 💡 Тэмцээний Өдрийн Зөвлөгөө

1. **Диаграммыг сайн ойлгох** — IP хаяг, interface, AS number бүрийг мэдэх
2. **Цагийн менежмент** — Troubleshooting-ийг хурдан хийх, хялбар даалгавраас эхлэх
3. **Дарааллыг баримтлах** — L2 → L3 → Services → Monitoring
4. **Show команд сайн мэдэх:**
   - `show ip ospf neighbor`
   - `show bgp summary`
   - `show ip nat translations`
   - `show standby brief`
   - `show spanning-tree`
   - `show vlan brief`
   - `show interfaces trunk`
   - `show ip route`
   - `show running-config | section <protocol>`
5. **Wireshark filter** сурах — `snmp`, `http`, `bgp`, `ospf`
6. **Тохиргооны template** бэлдэж авах — Цаг хэмнэнэ

---

## 10. 🔑 Чухал Show Командуудын Хураангуй

```
! Layer 2
show vlan brief
show interfaces trunk
show spanning-tree
show spanning-tree root
show etherchannel summary
show standby brief

! Layer 3
show ip route
show ip ospf neighbor
show ipv6 ospf neighbor
show ip bgp summary
show bgp ipv6 unicast summary
show ip bgp neighbors <ip> advertised-routes
show ip bgp neighbors <ip> received-routes

! IP Services
show ip nat translations
show ip nat statistics
show crypto isakmp sa
show crypto ipsec sa
show interfaces Tunnel0

! Monitoring
show snmp
show snmp community
show snmp host
```

---

> **Бэлтгэсэн:** Сүлжээний олимпиадын өмнөх жилийн даалгаварт суурилсан бэлтгэл гарын авлага
>
> **Эх сурвалж даалгавар:** Алтай (SSystems LLC), Улсболд (Mobicom LLC)

---

*Амжилт хүсье! 🎯*