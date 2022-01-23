![image](https://user-images.githubusercontent.com/8961955/150671203-ed1e55f7-725e-472b-b713-6c32a15dbae5.png)

<p>В организации для телефонии используется комплекс решений от avaya, в настоящее время все они находятся в серверной в Екб. Также есть два ЦОД в Мск. Маршрутизация построена на BGP, каналы между площадками зарезервированы. Нужно мигрировать телефонные кластера в Мск. и разместить в двух ЦОД. Смена адресации не рассматривается т.к. придется &quot;разбирать&quot; кластера и значительно менять конфигурацию. Это может привести к простою.</p>
<p>Задача:
Настроить L2 связность между устройствами vxlan_clientN.
Настроить L3 связность между устройствами clientN. 
Обмен L2 и L3 маршрутами должен происходить по BGP. </p>
<p>План: </p>
<ol>
<li>Добавить в протокол BGP обмен информацией о мак-адресах </li>
<li>&quot;растянуть&quot; используемый в настоящее время влан на три площадки. </li>
<li>выполнить миграцию Екб-Мск вычислительных ресурсов, средствами виртуализации </li>

</ol>
<p>Соответствие реальной сети - на схеме выше &quot;площадки&quot; представлены подами, где каждый под - leaf коммутатор со своей AS, и подключенные к нему клиентские устройства. Spine коммутаторы представляют собой core wan маршрутизаторы локальной сети организации, пара таких маршрутизаторов установлена в каждом ЦОД. (Прим.: в данной работе spine1 и spine2 имеют связи с подами AS65011 и AS65012, что не соответствует реальной инфраструктуре, однако позволяет в рамках проектной работы реализовать множественные BGP L3 связи и обеспечить разные пути для одинаковых маршрутов)</p>
<p>Описание BGP L3 пиринга (underlay):</p>
<p>L3 BGP пиринг строится между всеми соседними устройствами. Spine коммутаторы - ядро сети, обеспечивающее передачу L3 маршрутов между подами. Leaf коммутаторы в свою очередь анонсируют L3 маршруты 172.16.N.N и адреса Lo0 для L2 пиринга. </p>
<p>Описание BGP L2 пиринга (overlay):</p>
<p>L2 пиринг строится между всеми leaf коммутаторами, адреса точек терминации туннелей (lo0 интерфейсы leaf коммутаторов) передаются L3 BGP анонсами. Каждый leaf коммутатор анонсирует мак адреса клиентов, подключенных к нему - адреса 192.168.1.N</p>
<p>&nbsp;</p>
<p><strong>Реализация</strong> <strong>плана</strong></p>
<p>На всех leaf коммутаторах добавляем конфигурацию:
nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay</p>
<p>vlan 20,101
vlan 20
  vn-segment 10020
vlan 101
  vn-segment 1111</p>
<p>vrf context VXLAN
  vni 1111
  address-family ipv4 unicast
    route-target import 1111:1111
    route-target import 1111:1111 evpn
    route-target export 1111:1111
    route-target export 1111:1111 evpn
    route-target both auto
    route-target both auto evpn</p>
<p>interface Vlan20
  no shutdown
  vrf member VXLAN
  ip address 192.168.1.<strong>N</strong>/24
  fabric forwarding mode anycast-gateway</p>
<p>interface Vlan101
  no shutdown
  vrf member VXLAN
  ip forward</p>
<p>interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  member vni 1111 associate-vrf
  member vni 10020
    ingress-replication protocol bgp</p>
<p>router bgp 6501N
  address-family ipv4 unicast
    network 10.0.<strong>N.N</strong>/32
template peer VXLAN_PEER
    update-source loopback0
    ebgp-multihop 5
    address-family l2vpn evpn
      send-community
      send-community extended
      rewrite-evpn-rt-asn
 neighbor 10.0.<strong>N.N</strong>
    inherit peer VXLAN_PEER
    remote-as 6501<strong>N</strong>
  neighbor 10.0.<strong>N.N</strong>
    inherit peer VXLAN_PEER
    remote-as 6501<strong>N</strong>
  neighbor 10.0.<strong>N.N</strong>
    inherit peer VXLAN_PEER
    remote-as 6501<strong>N</strong></p>
<p><strong>rewrite-evpn-rt-asn</strong> - команда, для автоматической подмены номера AS в L2 маршрутах.</p>
<p><code>тут будут листинги конфигурации устройств</code></p>
<p><strong>После внесения этой конфигурации необходимо проверить nve пиринг и таблицу маршрутизации evpn</strong></p>
<p><code>Тут будут листинги &quot;sh nve peers&quot; и &quot;sh bgp l2vpn evpn&quot;</code></p>
<p><code>Тут будут листинги проверки L2 связности</code></p>
<p>`Тут будут листинги проверки L3 связности`</p>
