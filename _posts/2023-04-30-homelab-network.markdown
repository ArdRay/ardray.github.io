---
layout: post
title:  "Homelab network"
date:   2023-04-30
mermaid: true
---

## Overview

<div class="mermaid">
    graph TD
        linkStyle default interpolate basis
        wan1[<center>ISP router<br></center>]---router{<center>Mikrotik<br>Router</center>}
        router---|10.10.1.0/24|VLAN100
        router---|10.10.2.0/24|VLAN200
        router---|10.10.3.0/24|VLAN300
        router---|10.10.2.0/24<br>10.10.5.0/24|VLAN200/VLAN500
        subgraph VLAN200/VLAN500
        vmk-ext-01(<center>vmk-ext-01<br><br>10.10.2.9/10.10.5.2</center>)
        end
        subgraph VLAN300
        bm-prox-01(<center>bm-prox-01<br><br>10.10.3.100</center>)
        idrac-prox-01(<center>idrac-prox-01<br><br>10.10.3.224</center>)
        MB14(<center>management<br>laptop<br><br>10.10.3.181</center>)
        end
        subgraph VLAN200
        vmk-man-01(<center>vmk-man-01<br><br>10.10.2.3</center>)
        vmk-cicd-01(<center>vmk-cicd-01<br><br>10.10.2.2</center>)
        vmk-srv-01(<center>vmk-srv-01<br><br>10.10.2.8</center>)
        vmk-prod-01(<center>vmk-prod-01<br><br>10.10.2.7</center>)
        vmk-prod-02(<center>vmk-prod-02<br><br>10.10.2.5</center>)
        vmk-ext-01(<center>vmk-ext-01<br><br>10.10.2.9<br>10.10.5.2</center>)
        end
        subgraph VLAN100
        lexmark(<center>Lexmark printer<br><br>10.10.1.215</center>)
        TV(<center>TV<br><br>dynamic IP</center>)
        Desktop_1(<center>Desktop_1<br><br>dynamic IP</center>)
        Desktop_2(<center>Desktop_2<br><br>dynamic IP</center>)
        MB13(<center>MB13<br><br>dynamic IP</center>)
        end
</div>

Quite straightforward setup. 4 VLANs of different security levels:
- **VLAN500**: Externalisation VLAN. Used only for external traffic between `vmk-ext-01` and `Mikrotik ether5 interface`. Used for better control and monitoring of the traffic.
- **VLAN300**: Management VLAN
- **VLAN200**: Server VLAN
- **VLAN100**: All others. Isolated clients (for the WLAN) without any access to other VLANs.

## Interface

### Ports
```
/interface bridge
add name=bridge vlan-filtering=yes
/interface ethernet
set [ find default-name=sfp-sfpplus1 ] l2mtu=1592
/interface vlan
add interface=bridge name=VLAN100 vlan-id=100
add interface=bridge name=VLAN200 vlan-id=200
add interface=bridge name=VLAN300 vlan-id=300
add interface=bridge name=VLAN500 vlan-id=500
```

### WLAN
```
/interface wireless
set [ find default-name=wlan2 ] band=2ghz-b/g/n channel-width=20/40mhz-XX country=no_country_set \
    default-forwarding=no disabled=no distance=indoors frequency=auto frequency-mode=superchannel \
    installation=indoor mode=ap-bridge name=wlan_100 security-profile=galladoria_security_profile \
    ssid=Galladoria vlan-id=100 vlan-mode=use-tag wireless-protocol=802.11 wps-mode=disabled
set [ find default-name=wlan1 ] band=5ghz-n/ac channel-width=20/40/80mhz-eeeC country=no_country_set \
    disabled=no distance=indoors frequency=auto frequency-mode=manual-txpower mode=ap-bridge name=\
    wlan_300_5G security-profile=xaga_security_profile skip-dfs-channels=all ssid=Xaga vlan-id=300 \
    vlan-mode=use-tag wireless-protocol=802.11 wps-mode=disabled
add default-forwarding=no disabled=no mac-address=DE:2C:6E:1F:54:7D master-interface=wlan_300_5G \
    name=wlan_100_5G security-profile=hurionthex_security_profile ssid=Hurionthex vlan-id=100 \
    vlan-mode=use-tag wps-mode=disabled
```

### VLANs
```
/interface bridge port
add bridge=bridge frame-types=admit-only-vlan-tagged interface=wlan_300_5G pvid=300
add bridge=bridge frame-types=admit-only-untagged-and-priority-tagged interface=ether2 pvid=200
add bridge=bridge frame-types=admit-only-untagged-and-priority-tagged interface=ether4 pvid=300
add bridge=bridge interface=ether5 pvid=500
add bridge=bridge frame-types=admit-only-vlan-tagged interface=wlan_100_5G pvid=100
add bridge=bridge frame-types=admit-only-untagged-and-priority-tagged interface=ether1 pvid=200
add bridge=bridge frame-types=admit-only-vlan-tagged interface=wlan_100 pvid=100
add bridge=bridge frame-types=admit-only-untagged-and-priority-tagged interface=ether6 pvid=100
add bridge=bridge interface=ether3 pvid=200
add bridge=bridge frame-types=admit-only-untagged-and-priority-tagged interface=ether7 pvid=100
add bridge=bridge frame-types=admit-only-untagged-and-priority-tagged interface=ether8 pvid=100
add bridge=bridge frame-types=admit-only-untagged-and-priority-tagged interface=ether9 pvid=100
add bridge=bridge frame-types=admit-only-untagged-and-priority-tagged interface=ether10 pvid=100
/interface bridge settings
set use-ip-firewall=yes
```

```
/interface bridge vlan
add bridge=bridge tagged=bridge untagged=ether1,ether2,ether3 vlan-ids=200
add bridge=bridge tagged=wlan_100,wlan_100_5G,bridge untagged=ether6,ether7,ether8,ether9,ether10 vlan-ids=100
add bridge=bridge tagged=bridge,ether3,wlan_300_5G untagged=ether4 vlan-ids=300
add bridge=bridge tagged=ether5,bridge vlan-ids=500
```

### IP Addresses

```
/ip address
add address=10.10.1.1/24 interface=VLAN100 network=10.10.1.0
add address=10.10.2.1/24 interface=VLAN200 network=10.10.2.0
add address=10.10.3.1/24 interface=VLAN300 network=10.10.3.0
add address=10.10.5.1/24 interface=VLAN500 network=10.10.5.0
```

### DHCP server configuration
```
/ip pool
add name=pool_100 ranges=10.10.1.2-10.10.1.250
add name=pool_200 ranges=10.10.2.2-10.10.2.250
add name=pool_300 ranges=10.10.3.2-10.10.3.250
/ip dhcp-server
add address-pool=pool_100 interface=VLAN100 name=dhcp_100
add address-pool=pool_200 interface=VLAN200 name=dhcp_200
add address-pool=pool_300 interface=VLAN300 name=dhcp_300
/ip dhcp-client
add !dhcp-options interface=sfp-sfpplus1 use-peer-dns=no
/ip dhcp-server network
add address=10.10.1.0/24 gateway=10.10.1.1 netmask=24
add address=10.10.2.0/24 gateway=10.10.2.1 netmask=24
add address=10.10.3.0/24 gateway=10.10.3.1 netmask=24
```

### DNS config configuration

Internal DNS resolvers:
- `vmk-prod-01`
- `vmk-srv-01`

```
/ip dns
set allow-remote-requests=yes cache-max-ttl=1d max-concurrent-queries=200 \
    max-concurrent-tcp-sessions=60 servers=10.10.2.8,10.10.2.7
```
