# Komplexní Síťová Infrastruktura

Tento projekt představuje komplexní model podnikové síťové infrastruktury navržený v souladu s hierarchickým třívrstvým modelem (Core – Distribution – Access) a architekturou Defense-in-Depth. Topologie integruje více dodavatelů a technologií (Cisco IOS, Cisco ASAv, MikroTik RouterOS a Ubuntu Linux) a simuluje reálné prostředí enterprise sítě včetně redundantního směrování, L2 bezpečnosti a klíčových infrastrukturních služeb.

Projekt je zpracován ve formátu pro emulátor **EVE-NG** (`konfigurace/topology.unl`).

---

## 1. Architektura Sítě

Topologie je rozdělena do logických a bezpečnostních vrstev:

```
               +-------------------------------------------------+
               |             Simulovaný Internet (WAN)            |
               |     ISP-1 (MikroTik)       ISP-2 (MikroTik)     |
               +-------------------------------------------------+
                          |                           |
               +-------------------------------------------------+
               |            Hraniční vrstva (Edge Layer)         |
               |        EDGE-R1                     EDGE-R2       |
               |                 (NAT / PAT)                     |
               +-------------------------------------------------+
                          |                           |
               +-------------------------------------------------+
               |             Páteřní vrstva (Core Layer)          |
               |     CORE-R1 <-------------------------> CORE-R2 |
               |          (OSPF Area 0 + MD5 Auth + CoPP)        |
               +-------------------------------------------------+
                     /       |       \       /       |       \
                    /        |        \     /        |        \
               +-------------------------------------------------+
               |         Distribuční vrstva (Distribution)       |
               |     DIST-R1           DIST-R2           DIST-R3 |
               |          (OSPF Area 0 + MD5 Auth + CoPP)        |
               +-------------------------------------------------+
                     \                 |                 /
                      -----------------+-----------------
                                       |
               +-------------------------------------------------+
               |        Bezpečnostní a perimetrická vrstva       |
               |             Cisco ASAv Firewall                 |
               |      (Zóny, Identity & Dynamic NAT, MPF)        |
               +-------------------------------------------------+
                                       |
               +-------------------------------------------------+
               |         Interní stavový firewall & brána        |
               |           Linux Firewall (Ubuntu 20.04)         |
               |       (Netplan 802.1Q, iptables, DHCP Relay)    |
               +-------------------------------------------------+
                     /                 |                 \
               +-------------------------------------------------+
               |           Přístupová vrstva (Access Layer)      |
               |   ACCESS-SW1         ACCESS-SW2      ACCESS-SW3 |
               |     (DAI, DHCP Snooping, Port-Security, STP)    |
               +-------------------------------------------------+
                    |                  |                  |
           +----------------+  +----------------+  +----------------+
           | VLAN 10: Users |  | VLAN 20: Server|  | VLAN 30: WiFi  |
           | VLAN 99: Mgmt  |  | SRV1, SRV2     |  | AP + Klienti   |
           +----------------+  +----------------+  +----------------+
```

---

## 2. Přehled Zařízení a Role v Síti

| Zařízení | Platforma / OS | Role v síti | Management IP / Loopback |
| :--- | :--- | :--- | :--- |
| **ISP-1** | MikroTik RouterOS 7.2.1 | Poskytovatel připojení č. 1, simulace WAN, NAT a DNS cache | 203.0.113.1/30 (eth1) |
| **ISP-2** | MikroTik RouterOS 7.2.1 | Poskytovatel připojení č. 2, simulace WAN, NAT a DNS cache | 203.0.113.5/30 (eth1) |
| **EDGE-R1** | Cisco 7200 (IOS 15.2) | Primární hraniční směrovač, NAT/PAT overload pro odchozí provoz | 1.1.1.1/32 (Loopback0) |
| **EDGE-R2** | Cisco 7200 (IOS 15.2) | Sekundární hraniční směrovač, záložní NAT/PAT brána do internetu | 1.1.1.2/32 (Loopback0) |
| **CORE-R1** | Cisco 7200 (IOS 15.2) | Páteřní směrovač, OSPF Area 0, propagace výchozí cesty, CoPP | 2.2.2.1/32 (Loopback0) |
| **CORE-R2** | Cisco 7200 (IOS 15.2) | Páteřní směrovač (redundance), OSPF Area 0, CoPP | 2.2.2.2/32 (Loopback0) |
| **DIST-R1** | Cisco 3725 (IOS 12.4) | Distribuční směrovač pro přístup k ASAv (uplink Core 1 & 2), CoPP | 3.3.3.1/32 (Loopback0) |
| **DIST-R2** | Cisco 3725 (IOS 12.4) | Distribuční směrovač pro přístup k ASAv (uplink Core 1 & 2), CoPP | 3.3.3.2/32 (Loopback0) |
| **DIST-R3** | Cisco 3725 (IOS 12.4) | Distribuční směrovač pro přístup k ASAv (uplink Core 1 & 2), CoPP | 3.3.3.3/32 (Loopback0) |
| **ASAv** | Cisco ASAv 9.9(2) | Stavový firewall, Modular Policy Framework (MPF), zóny | 4.4.4.4/32, 10.99.99.1 (Mgmt) |
| **Linux FW** | Ubuntu Server 20.04 | L3 brána pro VLANy, stavová filtrace iptables, DHCP Relay | 10.99.99.5, 192.168.99.1/24 |
| **ACCESS-SW1** | Cisco VIOS L2 | Přístupový přepínač pro uživatelskou stanici PC1 a správce MNGT-1 | 192.168.99.11/24 (SVI) |
| **ACCESS-SW2** | Cisco VIOS L2 | Přístupový přepínač pro serverovou farmu (SRV1, SRV2) a PC3 | 192.168.99.12/24 (SVI) |
| **ACCESS-SW3** | Cisco VIOS L2 | Přístupový přepínač pro bezdrátový přístupový bod (WiFi-AP) a PC2 | 192.168.99.13/24 (SVI) |
| **SRV1** | Ubuntu Server 20.04 | Primární DHCP (Failover Master), Primární DNS (BIND9), NTP Master, Apache HTTPS | 192.168.20.10/24 |
| **SRV2** | Ubuntu Server 20.04 | Sekundární DHCP & DNS, FreeRADIUS (AAA), Centralizovaný Rsyslog, SNMP, TFTP | 192.168.20.20/24 |
| **WiFi-AP** | Ubuntu Server 20.04 | Přístupový bod bridgující bezdrátové klienty do VLAN 30 | 192.168.99.20/24 (Mgmt) |

---

## 3. Adresní Plán a Segmentace (VLANy)

### Segmentace Podnikové Sítě
| VLAN ID | Název VLAN | Subnet | Výchozí brána | Účel a popis |
| :--- | :--- | :--- | :--- | :--- |
| **10** | Users | 192.168.10.0/24 | 192.168.10.1 | Kancelářská klientská zařízení (PC1, PC2, PC3) |
| **20** | Servers | 192.168.20.0/24 | 192.168.20.1 | Serverová farma (SRV1, SRV2 - DNS, DHCP, RADIUS, Logy) |
| **30** | WiFi | 192.168.30.0/24 | 192.168.30.1 | Bezdrátoví klienti připojení přes WiFi-AP (simulovaný kvůli omezení WLAN v EVE-NG) |
| **99** | Management | 192.168.99.0/24 | 192.168.99.1 | Izolovaná správa síťových prvků (SSH, SVI přepínačů) |
| **999** | Unused_ports | N/A | N/A | Zabezpečení: Odstavené a nepoužívané přístupové porty |

### Propojovací Linky
| Propojení | Rozhraní | Subnet | IP adresa A | IP adresa B |
| :--- | :--- | :--- | :--- | :--- |
| **ISP-1 - EDGE-R1** | eth1 / fa0/0 | 203.0.113.0/30 | 203.0.113.1 (ISP-1) | 203.0.113.2 (EDGE-R1) |
| **ISP-2 - EDGE-R2** | eth1 / fa0/0 | 203.0.113.4/30 | 203.0.113.5 (ISP-2) | 203.0.113.6 (EDGE-R2) |
| **EDGE-R1 - CORE-R1** | fa1/0 / fa0/0 | 10.0.0.0/30 | 10.0.0.1 (EDGE-R1) | 10.0.0.2 (CORE-R1) |
| **EDGE-R2 - CORE-R2** | fa1/0 / fa0/0 | 10.0.0.4/30 | 10.0.0.5 (EDGE-R2) | 10.0.0.6 (CORE-R2) |
| **CORE-R1 - CORE-R2** | fa4/0 / fa4/0 | 10.0.0.8/30 | 10.0.0.9 (CORE-R1) | 10.0.0.10 (CORE-R2) |
| **CORE-R1 - DIST-R1** | fa1/0 / fa0/0 | 10.0.1.0/30 | 10.0.1.1 (CORE-R1) | 10.0.1.2 (DIST-R1) |
| **CORE-R1 - DIST-R2** | fa2/0 / fa0/0 | 10.0.1.4/30 | 10.0.1.5 (CORE-R1) | 10.0.1.6 (DIST-R2) |
| **CORE-R1 - DIST-R3** | fa3/0 / fa0/0 | 10.0.1.8/30 | 10.0.1.9 (CORE-R1) | 10.0.1.10 (DIST-R3) |
| **CORE-R2 - DIST-R1** | fa1/0 / fa0/1 | 10.0.1.12/30 | 10.0.1.13 (CORE-R2) | 10.0.1.14 (DIST-R1) |
| **CORE-R2 - DIST-R2** | fa2/0 / fa0/1 | 10.0.1.16/30 | 10.0.1.17 (CORE-R2) | 10.0.1.18 (DIST-R2) |
| **CORE-R2 - DIST-R3** | fa3/0 / fa0/1 | 10.0.1.20/30 | 10.0.1.21 (CORE-R2) | 10.0.1.22 (DIST-R3) |
| **DIST-R1 - ASAv** | fa1/0 / Gi0/0 | 10.0.1.24/30 | 10.0.1.25 (DIST-R1) | 10.0.1.26 (ASAv) |
| **DIST-R2 - ASAv** | fa1/0 / Gi0/1 | 10.0.1.28/30 | 10.0.1.29 (DIST-R2) | 10.0.1.30 (ASAv) |
| **DIST-R3 - ASAv** | fa1/0 / Gi0/2 | 10.0.1.32/30 | 10.0.1.33 (DIST-R3) | 10.0.1.34 (ASAv) |
| **ASAv - Linux FW** | Gi0/3 / eth1 | 10.99.99.0/24 | 10.99.99.1 (ASAv) | 10.99.99.5 (Linux FW) |

---

## 4. Klíčové Implementované Technologie

### 4.1. Dynamické Směrování (OSPFv2) a Redundance
- **Backbone Area 0:** Propojuje směrovače CORE-R1, CORE-R2, DIST-R1, DIST-R2, DIST-R3 i ASAv.
- **Kryptografická autentizace MD5:** Všechny spoje v OSPF vyžadují MD5 autentizaci (`area 0 authentication message-digest`, klíč `OspfSecurePass26`), což brání vložení neautorizovaných směrovacích informací.
  - **Poznámka:** Uvědomuji si, že sdílení hesel a jejich verzování je přináší obrovské riziko, ale tento projekt slouží pro účely ukázky mých síťových zkušeností, v praxi bych si nedovolil zveřejnit tyto citlivé údaje. 
- **Point-to-Point sítě:** Linky jsou nastaveny jako `ip ospf network point-to-point`, což eliminuje volbu DR/BDR a zrychluje konvergenci sítě.
- **Propagace výchozí cesty:** CORE-R1 automaticky distribuuje výchozí cestu do OSPF domény (`default-information originate metric 100 metric-type 1`).
- **Plná redundantní matice:** Každý z distribučních směrovačů má aktivní propojení na oba páteřní směrovače (CORE-R1 i CORE-R2), takže výpadek kteréhokoliv prvku nezpůsobí přerušení provozu.

### 4.2. Control Plane Policing - CoPP
Na směrovačích v jádru a distribuci je nakonfigurován mechanismus **CoPP** pomocí MQC (Modular QoS CLI):
- Kritický provoz (SSH správa ze schválených subnetů, směrovací protokol OSPF, diagnostické ICMP) je přiřazován do třídy `CM_COPP_MGMT` s nastavenou propustností 1 Mbps.
- Ostatní nekategorizovaný provoz je omezován na 256 kbps a při překročení zahazován (`police 256000 conform-action transmit exceed-action drop`).
- Tím je procesor síťového prvku stoprocentně chráněn před DoS útoky, zahlcením a pokusy o vyčerpání systémových prostředků.

### 4.3. Zabezpečení Perimetru (Cisco ASAv Firewall)
- **Bezpečnostní úrovně (Security Levels):** Vnější rozhraní směřující k distribučním routerům (`outside-dist1` až `3`) mají úroveň 0, interní management k Linux firewallu úroveň 90.
- **Inspekční mechanizmy (MPF):** Stavová inspekce ICMP, TFTP, DNS a HTTP provozu.
- **Identity NAT & Dynamic NAT:** Vnitřní provoz do infrastruktury prochází bez překladu adres, zatímco klientský provoz směrem ven je dynamicky překládán.
- **Příchozí ACL:** Striktní filtrace povolující pouze autorizovaný administrativní provoz (RADIUS porty 1812/1813, Syslog port 514, SNMP trap 162, TFTP port 69 z routerů na server SRV2).

### 4.4. Interní Stavový Firewall (Linux iptables & Netplan)
Linuxový firewall na bázi Ubuntu 20.04 slouží jako L3 směrovač pro koncové sítě a jako bezpečnostní filtr:
- **Netplan Bridge & 802.1Q subrozhraní:** Propojení přístupových přepínačů přes bridge `br0` se zapnutým Spanning Tree a vytvořenými virtuálními rozhraními pro jednotlivé VLAN (10, 20, 30, 99).
- **Výchozí politika DROP:** Řetězce `INPUT` a `FORWARD` mají výchozí politiku `DROP`.
- **Stavová filtrace (Stateful Inspection):** Povolení odpovědí a návratového provozu pomocí `-m state --state ESTABLISHED,RELATED`.
- **Izolace správy:** Přístup na SSH firewallu je povolen **výhradně** z management VLAN 99.
- **Restrikce komunikace mezi VLAN:**
  - Uživatelé (VLAN 10) a bezdrátoví klienti (VLAN 30) mají povolen přístup k serverům (VLAN 20) **pouze** na specifické služby: DNS (53 UDP/TCP), HTTP/HTTPS (80/443 TCP) a diagnostický ICMP ping.
  - Jakákoliv jiná komunikace mezi VLANami je zahozena, což zabraňuje šíření útoků.
- **DHCP Relay:** Běh služby `isc-dhcp-relay` přeposílajícího broadcastové DHCP dotazy z klientských sítí na servery SRV1 i SRV2.

### 4.5. Zabezpečení Přístupové Vrstvy (L2 Security na Cisco VIOS L2)
- **Dynamic ARP Inspection (DAI):** Ochrana proti útokům typu ARP Spoofing / Poisoning (`ip arp inspection vlan 10,20,30,99` s validací `src-mac dst-mac ip` a rate limitem 15 pps).
- **DHCP Snooping:** Ochrana proti podvrženým DHCP serverům (Rogue DHCP). Trunkové porty a uplinky jsou nastaveny jako `trust`.
- **Port Security:**
  - Omezení na maximálně 2 MAC adresy na port (`maximum 2`).
  - Učení adres pomocí `mac-address sticky`.
  - Režim narušení `restrict` s generováním bezpečnostního logu.
  - Automatické stárnutí neaktivních záznamů po 10 minutách (`aging time 10`).
- **STP PortFast & BPDU Guard:** Globálně i per-port aktivována ochrana proti připojení cizích přepínačů do uživatelských portů (okamžité zablokování portu v případě detekce BPDU).
- **Mitigace VLAN Hopping:** Nativní VLAN na všech trunkech nastavena na nepoužívanou `VLAN 999`, trunk protokoly DTP vypnuty příkazem `switchport nonegotiate`.
- **Odstavení nepoužitých portů:** Všechny volné porty jsou zařazeny do izolované VLAN 999 a administrativně vypnuty (`shutdown`).

### 4.6. Zabezpečení a Správa Síťových Prvků (Hardening)
- **Šifrované SSHv2:** Vypnut nešifrovaný Telnet, vygenerovány 2048bitové RSA klíče.
- **Ochrana proti Brute-Force útokům:** Příkaz `login block-for 120 attempts 3 within 60` uzamkne přihlášení na 2 minuty při třech neúspěšných pokusech za minutu.
- **Vypnutí zranitelných služeb:** Deaktivovány služby `no ip http server`, `no ip finger`, `no ip source-route`, `no ip bootp server`.
- **Restrikce VTY linek:** Přístup na linky VTY je omezen ACL seznamem `ACL_ADMIN_MGMT` pouze z Management sítě (VLAN 99) a serverové sítě.

---

## 5. Infrastrukturní a Serverové Služby (SRV1 a SRV2)

Serverová struktura ve VLAN 20 poskytuje robustní a redundantní místo pro celou podnikovou síť:

### High Availability DHCP (ISC DHCP Server)
- **Aktivní Failover:** SRV1 (Primary) a SRV2 (Secondary) komunikují protokolem `dhcp-failover` na portu 647 (MCLT 1800 s, split 128).
- **Distribuované pooly:** Automatické přidělování konfigurace pro VLAN 10 (Uživatelé), VLAN 20 (Servery), VLAN 30 (WiFi) i VLAN 99 (Management).
- **Vysoká dostupnost:** Při výpadku kteréhokoliv serveru druhý uzel automaticky převezme správu a vydávání IP adres.

### Redundantní DNS (BIND9)
- **Master / Slave architektura:** SRV1 spravuje primární autoritativní zónu pro doménu `lab.local` i reverzní zónu `20.168.192.in-addr.arpa`.
- **Automatický přenos zón:** SRV2 funguje jako sekundární DNS se zabezpečeným přenosem zón z SRV1.
- **Upstream Forwarding:** Rekurzivní dotazy pro veřejný internet jsou forwardovány na zabezpečené upstream servery (8.8.8.8, 1.1.1.1).

### Centralizované Ověřování AAA (FreeRADIUS)
- Na serveru SRV2 běží FreeRADIUS server.
- Síťové prvky (Edge, Core, Dist routery i přístupové přepínače) jsou definovány jako RADIUS klienti (`clients.conf`).
- Uživatelům je při úspěšném přihlášení automaticky přiděleno nejvyšší administrátorské oprávnění přes atribut `Cisco-AVPair = "shell:priv-lvl=15"`.

### Časová Synchronizace (Chrony NTP)
- SRV1 funguje jako lokální referenční NTP master (Stratum 10).
- SRV2 se synchronizuje proti SRV1.
- Všechny směrovače a přepínače synchronizují systémový čas proti serverové farmě pro konzistenci logů.

### Centrální Logování (Rsyslog)
- SRV2 přijímá syslog zprávy přes UDP port 514 ze všech síťových prvků.
- Zprávy jsou dynamicky řazeny do adresářové struktury podle hostname jednotlivých prvků (`/var/log/network/%HOSTNAME%/%PROGRAMNAME%.log`).

### Monitoring a Dohled (SNMPv2c)
- Na všech prvcích je nakonfigurován SNMP démon s omezením na čtení (`RO`) chráněný přístupovým seznamem `ACL_ADMIN_MGMT`.
- SNMP traps jsou automaticky směrovány na dohledový uzel SRV2.

### Automatické Zálohování Konfigurací (TFTP Server)
- Na SRV2 běží služba `tftpd-hpa`.
- Všechny směrovače a přepínače mají v konfiguraci definován příkaz pro automatické odeslání aktuální běžící konfigurace (`copy running-config tftp://192.168.20.20/...`).

### Interní Certifikační Autorita (PKI) a Zabezpečený Web (HTTPS)
- Vytvořena vlastní kořenová certifikační autorita **Lab Root CA** pomocí OpenSSL (SHA-256, 3650 dní).
- Vygenerovány a podepsány serverové certifikáty pro server `srv1.lab.local`.
- Apache2 webový server na SRV1 běží na šifrovaném protokolu HTTPS (port 443) s využitím interního certifikátu.

---

## 6. Seznam Souborů v Této Složce

- `konfigurace/topology.unl`: Exportovaná topologie pro emulátor **EVE-NG**.
- `konfigurace/adresace.txt`: Přehled adresních rozsahů, rozdělení VLAN a loopback rozhraní.
- `konfigurace/ISP-1.txt`, `ISP-2.txt`: Konfigurační skripty pro MikroTik RouterOS směrovače poskytovatelů.
- `konfigurace/EDGE-R1.txt`, `EDGE-R2.txt`: Konfigurace hraničních směrovačů Cisco 7200 (NAT, AAA, Hardening).
- `konfigurace/CORE-R1.txt`, `CORE-R2.txt`: Konfigurace páteřních směrovačů Cisco 7200 (OSPF MD5, CoPP, AAA).
- `konfigurace/DIST-R1.txt`, `DIST-R2.txt`, `DIST-R3.txt`: Konfigurace distribučních směrovačů Cisco 3725 (OSPF, CoPP).
- `konfigurace/ASAv.txt`: Konfigurace bezpečnostního firewallu Cisco ASAv (zóny, NAT, MPF, ACL).
- `konfigurace/firewall.txt`: Konfigurace interního Linux firewallu (Netplan, bridges, iptables pravidla, DHCP relay).
- `konfigurace/ACCESS-SW1.txt`, `ACCESS-SW2.txt`, `ACCESS-SW3.txt`: Konfigurace L2 přepínačů Cisco VIOS L2 (DAI, Snooping, Port-Security).
- `konfigurace/SRV1.txt`: Konfigurace primárního serveru (ISC-DHCP Master, BIND9 Master, Chrony NTP, Apache HTTPS, PKI).
- `konfigurace/SRV2.txt`: Konfigurace sekundárního serveru (ISC-DHCP Slave, BIND9 Slave, FreeRADIUS, Rsyslog, SNMP, TFTP).
- `konfigurace/AP.txt`: Konfigurace bezdrátového přístupového bodu na bázi Linuxu (Netplan bridge pro WiFi klienty).

---

## 7. Ověření Funkčnosti a Diagnostika

Níže jsou uvedeny klíčové příkazy pro ověření stavu jednotlivých subsystémů:

### OSPF Směrování a Redundance
```bash
# Na Core a Dist směrovačích:
show ip ospf neighbor
show ip route ospf
show ip ospf interface brief
```

### Zabezpečení Řídicí Roviny (CoPP)
```bash
# Kontrola čítačů povolených a zahozených paketů na směrovačích:
show policy-map control-plane
```

### L2 Zabezpečení na Přepínačích
```bash
# Kontrola DHCP Snooping a důvěryhodných portů:
show ip dhcp snooping
show ip dhcp snooping binding

# Kontrola Dynamic ARP Inspection:
show ip arp inspection statistics
show ip arp inspection interfaces

# Kontrola zabezpečení portů:
show port-security interface GigabitEthernet0/1
show port-security address
```

### Firewall ASAv
```bash
show access-list
show nat
show service-policy
show conn detail
```

### Služby na Linuxových Serverech
```bash
# Kontrola DHCP failoveru:
sudo systemctl status isc-dhcp-server
cat /var/lib/dhcp/dhcpd.leases | grep failover

# Test DNS rozlišení a přenosu zón:
dig @192.168.20.10 srv1.lab.local
dig @192.168.20.20 axfr lab.local

# Test RADIUS autentizace z konzole SRV2:
radtest admin AdminSecurePass123 127.0.0.1 0 RadiusKey123

# Kontrola příchozích syslogů:
ls -la /var/log/network/
```
