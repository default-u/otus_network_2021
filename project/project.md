![image](https://user-images.githubusercontent.com/8961955/150671203-ed1e55f7-725e-472b-b713-6c32a15dbae5.png)

В организации для телефонии используется комплекс решений от avaya, в настоящее время все они находятся в серверной в Екб. Также есть два ЦОД в Мск. Маршрутизация построена на BGP, каналы между площадками зарезервированы. Нужно мигрировать телефонные кластера в Мск. и разместить в двух ЦОД. Смена адресации не рассматривается т.к. придется "разбирать" кластера и значительно менять конфигурацию. Это может привести к простою.

Задача:
Настроить L2 связность между устройствами vxlan_clientN.
Настроить L3 связность между устройствами clientN. 
Обмен L2 и L3 маршрутами должен происходить по BGP. 

План: 

1. Добавить в протокол BGP обмен информацией о мак-адресах 
2. "растянуть" используемый в настоящее время влан на три площадки. 
3. выполнить миграцию Екб-Мск вычислительных ресурсов, средствами виртуализации 

Соответствие реальной сети - на схеме выше "площадки" представлены подами, где каждый под - leaf коммутатор со своей AS, и подключенные к нему клиентские устройства. Spine коммутаторы представляют собой core wan маршрутизаторы локальной сети организации, пара таких маршрутизаторов установлена в каждом ЦОД. (Прим.: в данной работе spine1 и spine2 имеют связи с подами AS65011 и AS65012, что не соответствует реальной инфраструктуре, однако позволяет в рамках проектной работы реализовать множественные BGP L3 связи и обеспечить разные пути для одинаковых маршрутов)

Описание BGP L3 пиринга (underlay):

L3 BGP пиринг строится между всеми соседними устройствами. Spine коммутаторы - ядро сети, обеспечивающее передачу L3 маршрутов между подами. Leaf коммутаторы в свою очередь анонсируют L3 маршруты 172.16.N.N и адреса Lo0 для L2 пиринга. 

Описание BGP L2 пиринга (overlay):

L2 пиринг строится между всеми leaf коммутаторами, адреса точек терминации туннелей (lo0 интерфейсы leaf коммутаторов) передаются L3 BGP анонсами. Каждый leaf коммутатор анонсирует мак адреса клиентов, подключенных к нему - адреса 192.168.1.N



**Реализация** **плана**

На всех leaf коммутаторах добавляем конфигурацию:  
nv overlay evpn  
feature bgp  
feature interface-vlan  
feature vn-segment-vlan-based  
feature nv overlay

vlan 20,101  
vlan 20  
  vn-segment 10020  
vlan 101  
  vn-segment 1111  

vrf context VXLAN  
  vni 1111  
  address-family ipv4 unicast  
    route-target import 1111:1111  
    route-target import 1111:1111 evpn  
    route-target export 1111:1111  
    route-target export 1111:1111 evpn  
    route-target both auto  
    route-target both auto evpn  

interface Vlan20  
  no shutdown  
  vrf member VXLAN  
  ip address 192.168.1.**N**/24  
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

router bgp 6501N  
  address-family ipv4 unicast  
    network 10.0.**N.N**/32  
template peer VXLAN_PEER  
    update-source loopback0  
    ebgp-multihop 5  
    address-family l2vpn evpn  
      send-community  
      send-community extended  
      rewrite-evpn-rt-asn  
 neighbor 10.0.**N.N**  
    inherit peer VXLAN_PEER  
    remote-as 6501**N**  
  neighbor 10.0.**N.N**  
    inherit peer VXLAN_PEER  
    remote-as 6501**N**  
  neighbor 10.0.**N.N**  
    inherit peer VXLAN_PEER  
    remote-as 6501**N**  

**rewrite-evpn-rt-asn** - команда, для автоматической подмены номера AS в L2 маршрутах.

`тут будут листинги конфигурации устройств`

**После внесения этой конфигурации необходимо проверить nve пиринг и таблицу маршрутизации evpn**

`Тут будут листинги "sh nve peers" и "sh bgp l2vpn evpn"`

`Тут будут листинги проверки L2 связности`

`Тут будут листинги проверки L3 связности`
