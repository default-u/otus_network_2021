![otus-source](https://user-images.githubusercontent.com/8961955/150805319-1219d171-d450-403d-8b9e-bfe96867e6ef.jpg)

В организации для телефонии используется комплекс решений от avaya, в настоящее время все они находятся в серверной в Екб. (AS65013 на схеме выше) Также есть два ЦОД в Мск. Маршрутизация построена на BGP, каналы между площадками зарезервированы. Нужно мигрировать телефонные кластера в Мск. и разместить в двух ЦОД. Смена адресации не рассматривается т.к. придется "разбирать" кластера и значительно менять конфигурацию. Это может привести к простою.

Задача:
Настроить L2 связность между устройствами vxlan_clientN.
Настроить L3 связность между устройствами clientN. 
Обмен L2 и L3 маршрутами должен происходить по BGP. 

План: 

1. Добавить в протокол BGP обмен информацией о мак-адресах 
2. "растянуть" используемый в настоящее время влан на три площадки. 
3. выполнить миграцию Екб-Мск вычислительных ресурсов, средствами системы виртуализации, в данной работе под "миграция", следует понимать - перемещение VPC с адресом 192.168.1.34 из AS65013 в AS65011 с сохранением L2 связности  

Соответствие реальной сети - на схеме выше "площадки"/ЦОД-ы представлены подами, где каждый под - leaf коммутатор со своей AS, и подключенные к нему клиентские устройства. Spine коммутаторы представляют собой core wan маршрутизаторы локальной сети организации, пара таких маршрутизаторов установлена в каждом ЦОД. (Прим.: в данной работе spine1 и spine2 имеют связи с подами AS65011 и AS65012, что не соответствует реальной инфраструктуре, однако позволяет в рамках проектной работы реализовать множественные BGP L3 связи и обеспечить разные пути для одинаковых маршрутов)

Описание BGP L3 пиринга (underlay):

L3 BGP пиринг строится между всеми соседними устройствами. Spine коммутаторы - ядро сети, обеспечивающее передачу L3 маршрутов между подами. Leaf коммутаторы в свою очередь анонсируют L3 маршруты 172.16.N.N и адреса Lo0 для L2 пиринга. 

Описание BGP L2 пиринга (overlay):

L2 пиринг строится между всеми leaf коммутаторами, адреса точек терминации туннелей (lo0 интерфейсы leaf коммутаторов) передаются L3 BGP анонсами. Каждый leaf коммутатор анонсирует evpn пирам мак адреса своих клиентов - адреса 192.168.1.N



**Реализация** **плана**

На всех leaf коммутаторах добавляем конфигурацию:  
nv overlay evpn  
feature bgp  
feature interface-vlan  
feature vn-segment-vlan-based  
feature nv overlay

vlan 20,101  
vlan 20  
&nbsp;&nbsp;&nbsp;&nbsp;vn-segment 10020  
vlan 101  
&nbsp;&nbsp;&nbsp;&nbsp;vn-segment 1111  

vrf context VXLAN  
&nbsp;&nbsp;&nbsp;&nbsp;vni 1111  
&nbsp;&nbsp;&nbsp;&nbsp;address-family ipv4 unicast  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;route-target import 1111:1111  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;route-target import 1111:1111 evpn  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;route-target export 1111:1111  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;route-target export 1111:1111 evpn  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;route-target both auto  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;route-target both auto evpn  

interface Vlan20  
&nbsp;&nbsp;&nbsp;&nbsp;no shutdown  
&nbsp;&nbsp;&nbsp;&nbsp;vrf member VXLAN  
&nbsp;&nbsp;&nbsp;&nbsp;ip address 192.168.1.**N**/24  
&nbsp;&nbsp;&nbsp;&nbsp;fabric forwarding mode anycast-gateway  

interface Vlan101  
&nbsp;&nbsp;&nbsp;&nbsp;no shutdown  
&nbsp;&nbsp;&nbsp;&nbsp;vrf member VXLAN  
&nbsp;&nbsp;&nbsp;&nbsp;ip forward  

interface nve1  
&nbsp;&nbsp;&nbsp;&nbsp;no shutdown  
&nbsp;&nbsp;&nbsp;&nbsp;host-reachability protocol bgp  
&nbsp;&nbsp;&nbsp;&nbsp;source-interface loopback0  
&nbsp;&nbsp;&nbsp;&nbsp;member vni 1111 associate-vrf  
&nbsp;&nbsp;&nbsp;&nbsp;member vni 10020  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ingress-replication protocol bgp  

router bgp 6501N  
&nbsp;&nbsp;&nbsp;&nbsp;address-family ipv4 unicast  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;network 10.0.**N.N**/32  
&nbsp;&nbsp;&nbsp;&nbsp;template peer VXLAN_PEER  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;update-source loopback0  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ebgp-multihop 5  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;address-family l2vpn evpn  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;send-community  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;send-community extended  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;rewrite-evpn-rt-asn  
&nbsp;&nbsp;&nbsp;&nbsp;neighbor 10.0.**N.N**  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inherit peer VXLAN_PEER  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;remote-as 6501**N**  
&nbsp;&nbsp;&nbsp;&nbsp;neighbor 10.0.**N.N**  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inherit peer VXLAN_PEER  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;remote-as 6501**N**  
&nbsp;&nbsp;&nbsp;&nbsp;neighbor 10.0.**N.N**  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;inherit peer VXLAN_PEER  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;remote-as 6501**N**  

**rewrite-evpn-rt-asn** - команда, для автоматической подмены номера AS в L2 маршрутах.

<details>  
<summary>spine1  </summary>
<pre><code>
vdc spine1 id 1  
  limit-resource vlan minimum 16 maximum 4094  
  limit-resource vrf minimum 2 maximum 4096  
  limit-resource port-channel minimum 0 maximum 511  
  limit-resource u4route-mem minimum 248 maximum 248  
  limit-resource u6route-mem minimum 96 maximum 96  
  limit-resource m4route-mem minimum 58 maximum 58  
  limit-resource m6route-mem minimum 8 maximum 8  
  
feature bgp  
feature vn-segment-vlan-based  
feature nv overlay  
  
no password strength-check  
username admin password 5 $5$At6j7sFH$QFuvnLdpsRlAQvI3B/KqQWkCBp3twsYZcAvg/Jesiy.  role network-admin  
ip domain-lookup  
copp profile strict  
snmp-server user admin network-admin auth md5 0x722863b7ae9ae350b0fe9606202c0538 priv 0x722863b7ae9ae350b0fe9606202c0538 localizedkey  
rmon event 1 description FATAL(1) owner PMON@FATAL  
rmon event 2 description CRITICAL(2) owner PMON@CRITICAL  
rmon event 3 description ERROR(3) owner PMON@ERROR  
rmon event 4 description WARNING(4) owner PMON@WARNING  
rmon event 5 description INFORMATION(5) owner PMON@INFO  
  
vlan 1  
  
vrf context management  
  
interface Ethernet1/1  
  description to leaf1  
  no switchport  
  ip address 10.1.1.1/30  
  no shutdown  
  
interface Ethernet1/2  
  description to leaf2  
  no switchport  
  ip address 10.1.2.1/30  
  no shutdown  
  
interface Ethernet1/3  
  description to leaf3  
  no switchport  
  ip address 10.1.3.1/30  
  no shutdown  
  
interface Ethernet1/4  
  
interface Ethernet1/5  
  
interface Ethernet1/6  
  
interface Ethernet1/7  
  no switchport  
  ip address 10.0.0.1/29  
  no shutdown  
  
interface mgmt0  
  vrf member management  
  
interface loopback0  
  ip address 10.0.1.1/28  
line console  
line vty  
boot nxos bootflash:/nxos.9.2.2.bin   
router bgp 65001  
  router-id 10.0.1.1  
  bestpath as-path multipath-relax  
  address-family ipv4 unicast  
    maximum-paths 10  
  neighbor 10.0.0.2  
    remote-as 65002  
    address-family ipv4 unicast  
  neighbor 10.0.0.3  
    remote-as 65003  
    address-family ipv4 unicast  
  neighbor 10.1.1.2  
    remote-as 65011  
    address-family ipv4 unicast  
  neighbor 10.1.2.2  
    remote-as 65011  
    address-family ipv4 unicast  
  neighbor 10.1.3.2  
    remote-as 65012  
    address-family ipv4 unicast  
  
  
!  
  
  
!end</code></pre>
</details>  
 
<details>  
<summary>spine2  </summary>
<pre><code>
vdc spine2 id 1  
  limit-resource vlan minimum 16 maximum 4094  
  limit-resource vrf minimum 2 maximum 4096  
  limit-resource port-channel minimum 0 maximum 511  
  limit-resource u4route-mem minimum 248 maximum 248  
  limit-resource u6route-mem minimum 96 maximum 96  
  limit-resource m4route-mem minimum 58 maximum 58  
  limit-resource m6route-mem minimum 8 maximum 8  
  
feature bgp  
feature interface-vlan  
feature vn-segment-vlan-based  
feature nv overlay  
  
no password strength-check  
username admin password 5 $5$UAiNjKt1$9pR8KDkHWS4OUVkN9jiMAKdcR0rsbrLgfD9aekSuCy.  role network-admin  
ip domain-lookup  
copp profile strict  
snmp-server user admin network-admin auth md5 0xb1b14953e1da70a8ccc8c18d3e0982b0 priv 0xb1b14953e1da70a8ccc8c18d3e0982b0 localizedkey  
rmon event 1 description FATAL(1) owner PMON@FATAL  
rmon event 2 description CRITICAL(2) owner PMON@CRITICAL  
rmon event 3 description ERROR(3) owner PMON@ERROR  
rmon event 4 description WARNING(4) owner PMON@WARNING  
rmon event 5 description INFORMATION(5) owner PMON@INFO  
  
vlan 1  
  
vrf context management  
  
interface Vlan1  
  
interface Ethernet1/1  
  description to leaf1  
  no switchport  
  ip address 10.2.1.1/30  
  no shutdown  
  
interface Ethernet1/2  
  description to leaf2  
  no switchport  
  ip address 10.2.2.1/30  
  no shutdown  
  
interface Ethernet1/3  
  description to leaf3  
  no switchport  
  ip address 10.2.3.1/30  
  no shutdown  
  
interface Ethernet1/4  
  
interface Ethernet1/5  
  
interface Ethernet1/6  
  
interface Ethernet1/7  
  no switchport  
  ip address 10.0.0.2/29  
  no shutdown  
  
interface mgmt0  
  vrf member management  
  
interface loopback0  
  ip address 10.0.1.2/28  
line console  
line vty  
boot nxos bootflash:/nxos.9.2.2.bin   
router bgp 65002  
  router-id 10.0.1.2  
  bestpath as-path multipath-relax  
  address-family ipv4 unicast  
    maximum-paths 10  
  neighbor 10.0.0.1  
    remote-as 65001  
    address-family ipv4 unicast  
  neighbor 10.0.0.3  
    remote-as 65003  
    address-family ipv4 unicast  
  neighbor 10.2.1.2  
    remote-as 65011  
    address-family ipv4 unicast  
  neighbor 10.2.2.2  
    remote-as 65011  
    address-family ipv4 unicast  
  neighbor 10.2.3.2  
    remote-as 65012  
    address-family ipv4 unicast  
  
  
!  
  
  
!end</code></pre>
</details>  

 
<details>  
<summary>spine3  </summary>
<pre><code>
vdc spine3 id 1  
  limit-resource vlan minimum 16 maximum 4094  
  limit-resource vrf minimum 2 maximum 4096  
  limit-resource port-channel minimum 0 maximum 511  
  limit-resource u4route-mem minimum 248 maximum 248  
  limit-resource u6route-mem minimum 96 maximum 96  
  limit-resource m4route-mem minimum 58 maximum 58  
  limit-resource m6route-mem minimum 8 maximum 8  
  
feature bgp  
feature vn-segment-vlan-based  
feature nv overlay  
  
no password strength-check  
username admin password 5 $5$Zyqyx4gx$opAI6.22o34.DYvDrs0spdtxsgWZm8RkrG748cfUb79  role network-admin  
ip domain-lookup  
copp profile strict  
snmp-server user admin network-admin auth md5 0x5de038599eb457d83922c64111096ee1 priv 0x5de038599eb457d83922c64111096ee1 localizedkey  
rmon event 1 description FATAL(1) owner PMON@FATAL  
rmon event 2 description CRITICAL(2) owner PMON@CRITICAL  
rmon event 3 description ERROR(3) owner PMON@ERROR  
rmon event 4 description WARNING(4) owner PMON@WARNING  
rmon event 5 description INFORMATION(5) owner PMON@INFO  
  
vlan 1  
  
vrf context management  
  
interface Ethernet1/1  
  
interface Ethernet1/2  
  
interface Ethernet1/3  
  
interface Ethernet1/4  
  no switchport  
  ip address 10.3.4.1/30  
  no shutdown  
  
interface Ethernet1/5  
  
interface Ethernet1/6  
  
interface Ethernet1/7  
  no switchport  
  ip address 10.0.0.3/29  
  no shutdown  
  
interface mgmt0  
  vrf member management  
  
interface loopback0  
  ip address 10.0.2.3/28  
line console  
line vty  
boot nxos bootflash:/nxos.9.2.2.bin   
router bgp 65003  
  router-id 10.0.2.3  
  bestpath as-path multipath-relax  
  address-family ipv4 unicast  
    maximum-paths 10  
  neighbor 10.0.0.1  
    remote-as 65001  
    address-family ipv4 unicast  
  neighbor 10.0.0.2  
    remote-as 65002  
    address-family ipv4 unicast  
  neighbor 10.3.4.2  
    remote-as 65013  
    address-family ipv4 unicast  
  
  
!  
  
  
!end</code></pre>
</details>  
  
  
<details>  
<summary>leaf1  </summary>
<pre><code>
vdc leaf1 id 1  
  limit-resource vlan minimum 16 maximum 4094  
  limit-resource vrf minimum 2 maximum 4096  
  limit-resource port-channel minimum 0 maximum 511  
  limit-resource u4route-mem minimum 248 maximum 248  
  limit-resource u6route-mem minimum 96 maximum 96  
  limit-resource m4route-mem minimum 58 maximum 58  
  limit-resource m6route-mem minimum 8 maximum 8  
  
cfs eth distribute  
nv overlay evpn  
feature bgp  
feature interface-vlan  
feature vn-segment-vlan-based  
feature hsrp  
feature lacp  
feature vpc  
feature nv overlay  
  
no password strength-check  
username admin password 5 $5$F12LEqdY$NY06tRPG98gZM02iwpaD5NduJj66BaZDyRtTjqivmm8  role network-admin  
ip domain-lookup  
copp profile strict  
snmp-server user admin network-admin auth md5 0x60a27ad08a02b0931b3d489fdda3c23a priv 0x60a27ad08a02b0931b3d489fdda3c23a localizedkey  
rmon event 1 description FATAL(1) owner PMON@FATAL  
rmon event 2 description CRITICAL(2) owner PMON@CRITICAL  
rmon event 3 description ERROR(3) owner PMON@ERROR  
rmon event 4 description WARNING(4) owner PMON@WARNING  
rmon event 5 description INFORMATION(5) owner PMON@INFO  
  
fabric forwarding anycast-gateway-mac 0001.0001.0001  
vlan 1-2,20,101  
vlan 2  
  name client-vlan  
vlan 20  
  vn-segment 10020  
vlan 101  
  vn-segment 1111  
  
ip prefix-list eBGP seq 10 permit 0.0.0.0/0 le 32   
vrf context VPC-vrf  
  address-family ipv4 unicast  
vrf context VXLAN  
  vni 1111  
  address-family ipv4 unicast  
    route-target import 1111:1111  
    route-target import 1111:1111 evpn  
    route-target export 1111:1111  
    route-target export 1111:1111 evpn  
    route-target both auto  
    route-target both auto evpn  
vrf context management  
vpc domain 1  
  peer-keepalive destination 192.168.2.2 source 192.168.2.1 vrf VPC-vrf  
  
  
interface Vlan1  
  
interface Vlan2  
  no shutdown  
  no ip redirects  
  ip address 172.16.1.2/29  
  fabric forwarding mode anycast-gateway  
  hsrp version 2  
  hsrp 1   
    preempt delay minimum 30   
    priority 20  
    ip 172.16.1.1  
  
interface Vlan20  
  no shutdown  
  vrf member VXLAN  
  ip address 192.168.1.1/24  
  fabric forwarding mode anycast-gateway  
  
interface Vlan101  
  no shutdown  
  vrf member VXLAN  
  ip forward  
  
interface port-channel1  
  switchport mode trunk  
  switchport trunk allowed vlan 2  
  vpc 1  
  
interface port-channel47  
  description VPC keepalive  
  no switchport  
  vrf member VPC-vrf  
  ip address 192.168.2.1/30  
  
interface port-channel48  
  description VPC peer-link  
  switchport mode trunk  
  spanning-tree port type network  
  vpc peer-link  
  
interface nve1  
  no shutdown  
  host-reachability protocol bgp  
  source-interface loopback0  
  member vni 1111 associate-vrf  
  member vni 10020  
    ingress-replication protocol bgp  
  
interface Ethernet1/1  
  description to spine1  
  no switchport  
  ip address 10.1.1.2/30  
  no shutdown  
  
interface Ethernet1/2  
  description to spine2  
  no switchport  
  ip address 10.2.1.2/30  
  no shutdown  
  
interface Ethernet1/3  
  switchport mode trunk  
  switchport trunk allowed vlan 20  
  
interface Ethernet1/4  
  
interface Ethernet1/5  
  switchport mode trunk  
  switchport trunk allowed vlan 2  
  channel-group 1 mode active  
  
interface Ethernet1/6  
  description VPC peer link  
  switchport mode trunk  
  channel-group 48 mode active  
  
interface Ethernet1/7  
  description VPC keepalive  
  no switchport  
  channel-group 47 mode active  
  no shutdown  
  
interface mgmt0  
  vrf member management  
  
interface loopback0  
  ip address 10.0.1.11/28  
line console  
line vty  
boot nxos bootflash:/nxos.9.2.2.bin   
router bgp 65011  
  router-id 10.0.1.11  
  bestpath as-path multipath-relax  
  address-family ipv4 unicast  
    network 10.0.1.11/32  
    network 172.16.1.0/29  
    maximum-paths 10  
  template peer VXLAN_PEER  
    update-source loopback0  
    ebgp-multihop 5  
    address-family l2vpn evpn  
      send-community  
      send-community extended  
      rewrite-evpn-rt-asn  
  neighbor 10.0.1.12  
    remote-as 65011  
    update-source loopback0  
    address-family l2vpn evpn  
      send-community  
      send-community extended  
  neighbor 10.0.1.13  
    inherit peer VXLAN_PEER  
    remote-as 65012  
  neighbor 10.0.2.14  
    inherit peer VXLAN_PEER  
    remote-as 65013  
  neighbor 10.1.1.1  
    remote-as 65001  
    address-family ipv4 unicast  
  neighbor 10.2.1.1  
    remote-as 65002  
    address-family ipv4 unicast  
  neighbor 172.16.1.3  
    remote-as 65011  
    address-family ipv4 unicast  
      next-hop-self  
  
  
!  
  
  
!end</code></pre>
</details>  
  

<details>  
<summary>leaf2  </summary>
<pre><code>
vdc leaf2 id 1  
  limit-resource vlan minimum 16 maximum 4094  
  limit-resource vrf minimum 2 maximum 4096  
  limit-resource port-channel minimum 0 maximum 511  
  limit-resource u4route-mem minimum 248 maximum 248  
  limit-resource u6route-mem minimum 96 maximum 96  
  limit-resource m4route-mem minimum 58 maximum 58  
  limit-resource m6route-mem minimum 8 maximum 8  
  
cfs eth distribute  
nv overlay evpn  
feature bgp  
feature interface-vlan  
feature vn-segment-vlan-based  
feature hsrp  
feature lacp  
feature vpc  
feature nv overlay  
  
no password strength-check  
username admin password 5 $5$icUr49nY$ZckMifx8m3jNavkjkYkLYPuamhdfCoF6x31egxjiDvA  role network-admin  
ip domain-lookup  
copp profile strict  
snmp-server user admin network-admin auth md5 0x42f75d98a183220dd1a1e9fadc737d44 priv 0x42f75d98a183220dd1a1e9fadc737d44 localizedkey  
rmon event 1 description FATAL(1) owner PMON@FATAL  
rmon event 2 description CRITICAL(2) owner PMON@CRITICAL  
rmon event 3 description ERROR(3) owner PMON@ERROR  
rmon event 4 description WARNING(4) owner PMON@WARNING  
rmon event 5 description INFORMATION(5) owner PMON@INFO  
  
fabric forwarding anycast-gateway-mac 0002.0002.0002  
vlan 1-2,20,101  
vlan 2  
  name client-vlan  
vlan 20  
  vn-segment 10020  
vlan 101  
  vn-segment 1111  
  
ip prefix-list eBGP seq 10 permit 0.0.0.0/0 le 32   
vrf context VPC-vrf  
  address-family ipv4 unicast  
vrf context VXLAN  
  vni 1111  
  address-family ipv4 unicast  
    route-target import 1111:1111  
    route-target import 1111:1111 evpn  
    route-target export 1111:1111  
    route-target export 1111:1111 evpn  
    route-target both auto  
    route-target both auto evpn  
vrf context management  
vpc domain 1  
  peer-keepalive destination 192.168.2.1 source 192.168.2.2 vrf VPC-vrf  
  
  
interface Vlan1  
  
interface Vlan2  
  no shutdown  
  no ip redirects  
  ip address 172.16.1.3/29  
  fabric forwarding mode anycast-gateway  
  hsrp version 2  
  hsrp 1   
    preempt delay minimum 30   
    priority 10  
  
interface Vlan20  
  no shutdown  
  vrf member VXLAN  
  ip address 192.168.1.2/24  
  fabric forwarding mode anycast-gateway  
  
interface Vlan101  
  no shutdown  
  vrf member VXLAN  
  ip forward  
  
interface port-channel1  
  switchport mode trunk  
  switchport trunk allowed vlan 2  
  vpc 1  
  
interface port-channel2  
  switchport mode trunk  
  switchport trunk allowed vlan 20  
  
interface port-channel47  
  description VPC keepalive  
  no switchport  
  vrf member VPC-vrf  
  ip address 192.168.2.2/30  
  
interface port-channel48  
  description VPC peer-link  
  switchport mode trunk  
  spanning-tree port type network  
  vpc peer-link  
  
interface nve1  
  no shutdown  
  host-reachability protocol bgp  
  source-interface loopback0  
  member vni 1111 associate-vrf  
  member vni 10020  
    ingress-replication protocol bgp  
  
interface Ethernet1/1  
  no switchport  
  ip address 10.1.2.2/30  
  no shutdown  
  
interface Ethernet1/2  
  no switchport  
  ip address 10.2.2.2/30  
  no shutdown  
  
interface Ethernet1/3  
  switchport mode trunk  
  switchport trunk allowed vlan 20  
  channel-group 2 mode active  
  
interface Ethernet1/4  
  
interface Ethernet1/5  
  switchport mode trunk  
  switchport trunk allowed vlan 2  
  channel-group 1 mode active  
  
interface Ethernet1/6  
  description VPC peer link  
  switchport mode trunk  
  channel-group 48 mode active  
  
interface Ethernet1/7  
  description VPC keepalive  
  no switchport  
  channel-group 47 mode active  
  no shutdown  
  
interface mgmt0  
  vrf member management  
  
interface loopback0  
  ip address 10.0.1.12/28  
line console  
line vty  
boot nxos bootflash:/nxos.9.2.2.bin   
router bgp 65011  
  router-id 10.0.1.12  
  bestpath as-path multipath-relax  
  address-family ipv4 unicast  
    network 10.0.1.12/32  
    network 172.16.1.0/29  
    maximum-paths 10  
  template peer VXLAN_PEER  
    update-source loopback0  
    ebgp-multihop 5  
    address-family l2vpn evpn  
      send-community  
      send-community extended  
      rewrite-evpn-rt-asn  
  neighbor 10.0.1.11  
    remote-as 65011  
    update-source loopback0  
    address-family l2vpn evpn  
      send-community  
      send-community extended  
  neighbor 10.0.1.13  
    inherit peer VXLAN_PEER  
    remote-as 65012  
  neighbor 10.0.2.14  
    inherit peer VXLAN_PEER  
    remote-as 65013  
  neighbor 10.1.2.1  
    remote-as 65001  
    address-family ipv4 unicast  
  neighbor 10.2.2.1  
    remote-as 65002  
    address-family ipv4 unicast  
  neighbor 172.16.1.2  
    remote-as 65011  
    address-family ipv4 unicast  
      next-hop-self  
  
  
!  
  
  
!end</code></pre>
</details>  
  
 
<details>  
<summary>leaf3  </summary>
<pre><code>
vdc leaf3 id 1  
  limit-resource vlan minimum 16 maximum 4094  
  limit-resource vrf minimum 2 maximum 4096  
  limit-resource port-channel minimum 0 maximum 511  
  limit-resource u4route-mem minimum 248 maximum 248  
  limit-resource u6route-mem minimum 96 maximum 96  
  limit-resource m4route-mem minimum 58 maximum 58  
  limit-resource m6route-mem minimum 8 maximum 8  
  
nv overlay evpn  
feature bgp  
feature interface-vlan  
feature vn-segment-vlan-based  
feature nv overlay  
  
no password strength-check  
username admin password 5 $5$Pj7v9o54$CUmU.uXo21XZRLNF0F9g9FSsTPFD6fgVcQI7ysyvLI0  role network-admin  
ip domain-lookup  
copp profile strict  
snmp-server user admin network-admin auth md5 0x139add3535ee3dbb9654deed03bb8ba7 priv 0x139add3535ee3dbb9654deed03bb8ba7 localizedkey  
rmon event 1 description FATAL(1) owner PMON@FATAL  
rmon event 2 description CRITICAL(2) owner PMON@CRITICAL  
rmon event 3 description ERROR(3) owner PMON@ERROR  
rmon event 4 description WARNING(4) owner PMON@WARNING  
rmon event 5 description INFORMATION(5) owner PMON@INFO  
  
fabric forwarding anycast-gateway-mac 0003.0003.0003  
vlan 1,20,101  
vlan 20  
  vn-segment 10020  
vlan 101  
  vn-segment 1111  
  
route-map NH_UNCHANGED permit 10  
  set ip next-hop unchanged  
vrf context VXLAN  
  vni 1111  
  address-family ipv4 unicast  
    route-target import 1111:1111  
    route-target import 1111:1111 evpn  
    route-target export 1111:1111  
    route-target export 1111:1111 evpn  
    route-target both auto  
    route-target both auto evpn  
vrf context management  
  
  
interface Vlan1  
  
interface Vlan20  
  no shutdown  
  vrf member VXLAN  
  ip address 192.168.1.3/24  
  fabric forwarding mode anycast-gateway  
  
interface Vlan101  
  no shutdown  
  vrf member VXLAN  
  ip forward  
  
interface nve1  
  no shutdown  
  host-reachability protocol bgp  
  source-interface loopback0  
  member vni 1111 associate-vrf  
  member vni 10020  
    ingress-replication protocol bgp  
  
interface Ethernet1/1  
  no switchport  
  ip address 10.1.3.2/30  
  no shutdown  
  
interface Ethernet1/2  
  no switchport  
  ip address 10.2.3.2/30  
  no shutdown  
  
interface Ethernet1/3  
  
interface Ethernet1/4  
  
interface Ethernet1/5  
  no switchport  
  ip address 172.16.3.1/30  
  no shutdown  
  
interface Ethernet1/6  
  
interface Ethernet1/7  
  switchport mode trunk  
  switchport trunk allowed vlan 20  
  spanning-tree bpdufilter enable  
  
interface mgmt0  
  vrf member management  
  
interface loopback0  
  ip address 10.0.1.13/28  
line console  
line vty  
boot nxos bootflash:/nxos.9.2.2.bin   
router bgp 65012  
  router-id 10.0.1.13  
  bestpath as-path multipath-relax  
  address-family ipv4 unicast  
    network 10.0.1.13/32  
    network 172.16.3.0/30  
    maximum-paths 10  
  template peer VXLAN_PEER  
    update-source loopback0  
    ebgp-multihop 5  
    address-family l2vpn evpn  
      send-community  
      send-community extended  
      rewrite-evpn-rt-asn  
  neighbor 10.0.1.11  
    inherit peer VXLAN_PEER  
    remote-as 65011  
  neighbor 10.0.1.12  
    inherit peer VXLAN_PEER  
    remote-as 65011  
  neighbor 10.0.2.14  
    inherit peer VXLAN_PEER  
    remote-as 65013  
  neighbor 10.1.3.1  
    remote-as 65001  
    address-family ipv4 unicast  
  neighbor 10.2.3.1  
    remote-as 65002  
    address-family ipv4 unicast  
  
  
!  
  
  
!end</code></pre>
</details>  
  
<details>  
<summary>leaf4  </summary>
<pre><code>
vdc leaf4 id 1  
  limit-resource vlan minimum 16 maximum 4094  
  limit-resource vrf minimum 2 maximum 4096  
  limit-resource port-channel minimum 0 maximum 511  
  limit-resource u4route-mem minimum 248 maximum 248  
  limit-resource u6route-mem minimum 96 maximum 96  
  limit-resource m4route-mem minimum 58 maximum 58  
  limit-resource m6route-mem minimum 8 maximum 8  
  
nv overlay evpn  
feature bgp  
feature interface-vlan  
feature vn-segment-vlan-based  
feature nv overlay  
  
no password strength-check  
username admin password 5 $5$vysaK09z$n/.fFxBUb/WegOIg7WIfDQX2qPNGLPvZFc1g6mVW9hA  role network-admin  
ip domain-lookup  
copp profile strict  
snmp-server user admin network-admin auth md5 0x35664887a49ee2d41b31d241a439febe priv 0x35664887a49ee2d41b31d241a439febe localizedkey  
rmon event 1 description FATAL(1) owner PMON@FATAL  
rmon event 2 description CRITICAL(2) owner PMON@CRITICAL  
rmon event 3 description ERROR(3) owner PMON@ERROR  
rmon event 4 description WARNING(4) owner PMON@WARNING  
rmon event 5 description INFORMATION(5) owner PMON@INFO  
  
fabric forwarding anycast-gateway-mac 0004.0004.0004  
vlan 1,20,101  
vlan 20  
  vn-segment 10020  
vlan 101  
  vn-segment 1111  
  
ip prefix-list l3 seq 5 permit 0.0.0.0/0   
ip prefix-list l3 seq 10 permit 172.16.4.1/32   
route-map GLOBAL-TO-VRF permit 10  
  match ip address prefix-list l3   
route-map NH_UNCHANGED permit 10  
  set ip next-hop unchanged  
vrf context VXLAN  
  vni 1111  
  address-family ipv4 unicast  
    route-target import 1111:1111  
    route-target import 1111:1111 evpn  
    route-target export 1111:1111  
    route-target export 1111:1111 evpn  
    route-target both auto  
    route-target both auto evpn  
vrf context management  
  
  
interface Vlan1  
  
interface Vlan20  
  no shutdown  
  vrf member VXLAN  
  ip address 192.168.1.4/24  
  fabric forwarding mode anycast-gateway  
  
interface Vlan101  
  no shutdown  
  vrf member VXLAN  
  ip forward  
  
interface nve1  
  no shutdown  
  host-reachability protocol bgp  
  source-interface loopback0  
  member vni 1111 associate-vrf  
  member vni 10020  
    ingress-replication protocol bgp  
  
interface Ethernet1/1  
  switchport mode trunk  
  switchport trunk allowed vlan 20  
  
interface Ethernet1/2  
  
interface Ethernet1/3  
  no switchport  
  ip address 10.3.4.2/30  
  no shutdown  
  
interface Ethernet1/4  
  
interface Ethernet1/5  
  no switchport  
  ip address 172.16.4.1/30  
  no shutdown  
  
interface Ethernet1/6  
  
interface Ethernet1/7  
  switchport mode trunk  
  switchport trunk allowed vlan 20  
  spanning-tree bpdufilter enable  
  
  
interface mgmt0  
  vrf member management  
  
interface loopback0  
  ip address 10.0.2.14/28  
line console  
line vty  
boot nxos bootflash:/nxos.9.2.2.bin   
router bgp 65013  
  router-id 10.0.2.14  
  address-family ipv4 unicast  
    network 10.0.2.14/32  
    network 172.16.4.0/30  
  template peer VXLAN_PEER  
    update-source loopback0  
    ebgp-multihop 5  
    address-family l2vpn evpn  
      send-community  
      send-community extended  
      rewrite-evpn-rt-asn  
  neighbor 10.0.1.11  
    inherit peer VXLAN_PEER  
    remote-as 65011  
  neighbor 10.0.1.12  
    inherit peer VXLAN_PEER  
    remote-as 65011  
  neighbor 10.0.1.13  
    inherit peer VXLAN_PEER  
    remote-as 65012  
  neighbor 10.3.4.1  
    remote-as 65003  
    address-family ipv4 unicast  
  
  
!  
  
  
!end</code></pre>
</details>  
  
**После внесения этой конфигурации выполним миграцию одного из серверов с IP 192.168.1.34 из AS65013 в AS65011, а также добавим новый клиент в AS65012, после чего проверим nve пиринг и таблицу маршрутизации**
  
После миграции 192.168.1.34 и добавления нового клиента в AS65012, схема сети будет иметь вид:  
![image](https://user-images.githubusercontent.com/8961955/150806498-77721e50-8fa2-44b4-81dd-0c04c67dfa7f.png)


`Тут будут листинги "sh nve peers" и "sh bgp l2vpn evpn"`  
  
<details>  
<summary>leaf4  </summary>
![image](https://user-images.githubusercontent.com/8961955/150807220-d7cccbf0-5990-44fa-a82d-45dc9358c535.png)  
![image](https://user-images.githubusercontent.com/8961955/150807319-e0095c6e-d130-44b1-ab76-8391d2dc96e9.png)  
</details>  


`Тут будут листинги проверки L2 связности`

`Тут будут листинги проверки L3 связности`
