В Этой заметке хочу рассмотреть построение сетевой фабрики на основе VxLAN EVPN с внешним Firewall.
Все примеры будем выполнять на Cisco Nexus9k

Определим основную задачу сети:

1. Организовать L2 доступ между хостами 
2. Организовать маршрутизацию между сетями в рамках одного сервиса. 1 VRF - 1 сервис
3. Подключение Firewall для настройки правил между сервисами. Дополнительно firewall должен работать в качестве шлюза по умолчанию


Все примеры будем выполнять на образах nexus9k

Для решение задачи выбрали топологию Spine-Leaf:

![](img/all.jpg)

По схеме видно, что для примера есть два хоста и сам firewall с которыми мы и будем проводить эксперименты.

Зададим адресацию:
```buildoutcfg
Spine-1 - 10.255.1.101
Spine-2 - 10.255.1.102

Leaf-11 - 10.255.1.11
Leaf-12 - 10.255.1.12
Leaf-21 - 10.255.1.21
Leaf-22 - 10.255.1.22
```

Для начала необходимо настроить underlay сеть и обеспечить базовую IP связанность между всеми устройствами внутри VxLAN фабрики

### Undrelay

Организуем IP связанность между соседними устройствами:
  ```buildoutcfg
interface loopback0
  description ROUTE_INT
  ip address 10.255.1.11      ! IP можем использовать с /32
  ```

```buildoutcfg
interface ethernet1/1
  no switchport
  mtu 9216
  medium p2p     ! Настраиваем Point-to-Point линки, чтобы убрать необходимость поиска DR/BDR между Nexus
  ip unnumbered loopback0    ! для экономии и упрощения работы заимствуем IP с Loopback
  no shutdown
```

Не будем усложнять сеть и настроим на Leaf и Spine протокол OSPF для IP связанности. 

На Nexus необходимо включить feature OSPF и создать процесс OSPF:
```buildoutcfg
feature ospf

router ospf UNREFLAY
  router-id {ID} ! для каждого устройства задаем ID для упрощения дальнейшего TrobleShoting`a
```
Далее на каждом интерфейсе включаем процесс OSPF. Все устройства 
поместим в area 0:
```buildoutcfg
interface loopback0
  ip router ospf UNDERLAY area 0.0.0.0

interface ethernet1/1-2
  ip router ospf UNDERLAY area 0.0.0.0
```
Проверим, что у нас появилась базоавая IP связанность:

```buildoutcfg
Leaf22# sh ip route
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.255.1.11/32, ubest/mbest: 2/0
    *via 10.255.1.101, Eth1/4, [110/81], 00:00:03, ospf-UNDERLAY, intra
    *via 10.255.1.102, Eth1/3, [110/81], 00:00:03, ospf-UNDERLAY, intra
10.255.1.12/32, ubest/mbest: 2/0
    *via 10.255.1.101, Eth1/4, [110/81], 00:00:03, ospf-UNDERLAY, intra
    *via 10.255.1.102, Eth1/3, [110/81], 00:00:03, ospf-UNDERLAY, intra
10.255.1.21/32, ubest/mbest: 2/0
    *via 10.255.1.101, Eth1/4, [110/81], 00:00:03, ospf-UNDERLAY, intra
    *via 10.255.1.102, Eth1/3, [110/81], 00:00:03, ospf-UNDERLAY, intra
10.255.1.22/32, ubest/mbest: 2/0, attached
    *via 10.255.1.22, Lo0, [0/0], 00:02:20, local
    *via 10.255.1.22, Lo0, [0/0], 00:02:20, direct
10.255.1.101/32, ubest/mbest: 1/0
    *via 10.255.1.101, Eth1/4, [110/41], 00:00:06, ospf-UNDERLAY, intra
10.255.1.102/32, ubest/mbest: 1/0
    *via 10.255.1.102, Eth1/3, [110/41], 00:00:03, ospf-UNDERLAY, intra
```
Как видим от Leaf до других Leaf коммутаторов у нас есть по два пути через 2 Spine.

Так же нам потребуется подключить Firewall. Для этого необходимо включить две фичи:
```buildoutcfg
feature vpc
feature lacp
```
Далее настроим домаен VPC между парами Leaf коммутаторов. Домен VPC должен быть одинаковым на обоих устройстваз в паре Nexus. На данном этапе
настроим только базовые настройки, дальнейшие нюансы связанные с работой VxLAN будем добавлять по мере настройки фабрики:

```buildoutcfg
vpc domain 2
  peer-keepalive destination 192.168.2.1 source 192.168.2.2 ! данные адреса настроены на интерфейсе mgmt
!
! создаем channel-group 7 mode active на интерфейсах между Nexus
!
interface port-channel7 
  vpc peer-link ! указываем, что этот port-channel является служебным линком между парой устройств 
```
Проверим, что VPC синхронизировался и все ок:
```buildoutcfg
vPC domain id                     : 1
Peer status                       : peer adjacency formed ok
vPC keep-alive status             : peer is alive
Configuration consistency status  : success
Per-vlan consistency status       : success
Type-2 consistency status         : success
vPC role                          : primary
Number of vPCs configured         : 0
Peer Gateway                      : Disabled
Dual-active excluded VLANs        : -
Graceful Consistency Check        : Enabled
Auto-recovery status              : Disabled
Delay-restore status              : Timer is off.(timeout = 30s)
Delay-restore SVI status          : Timer is off.(timeout = 10s)
Operational Layer3 Peer-router    : Disabled

```
Далее настраиваем LACP в сторону Firewall. На каждом Leaf настройка должна быть идентична:
```buildoutcfg
interface Ethernet1/6
  switchport mode trunk
  channel-group 5 mode active
!
!
interface port-channel5
  switchport mode trunk
  vpc 5  ! для каждого Port-channel настройка уникальная
```
Проверим, что не возникло никаких проблем с VPC:
```buildoutcfg
vPC status
----------------------------------------------------------------------------
Id    Port          Status Consistency Reason                Active vlans
--    ------------  ------ ----------- ------                ---------------
5     Po5           up     success     success               1
```

На данном этапе будем считать настройку базовой Underlay сети выполненной.
Конечно можно настроить еще различные технологии для уменьшения времени сходимости
 и скорости реагирования на какие-либо изменения, но данный материал выходит за рамки данной статьи.


Для настройки Overlay сети необходимо на Spine коммутаторах включить BGP с поддержкой семейства l2vpn evpn:

```buildoutcfg
feature bgp
nv overlay evpn
```

Далее необходимо настроить BGP пиринг между Leaf и Spine. Для упрощения настройки и оптимизации распространения маршрутной информации
Spine настраиваем в качестве Route-Reflector server. Все Leaf пропишим в конфиге через шаблоны, чтобы оптимизировать настройку.

Таким образом конфиг на Spine выглядит так:

```buildoutcfg
router bgp 65001
  template peer LEAF 
    remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
  neighbor 10.255.1.11
    inherit peer LEAF
  neighbor 10.255.1.12
    inherit peer LEAF
  neighbor 10.255.1.21
    inherit peer LEAF
  neighbor 10.255.1.22
    inherit peer LEAF
```
Конфиг на Leaf коммутаторе выглядит аналогичным образом:

```buildoutcfg
router bgp 65001
  template peer SPINE
    remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.255.1.101
    inherit peer SPINE
  neighbor 10.255.1.102
    inherit peer SPINE
```

На Spine проверим что установили пиринг с каждым Leaf:
```buildoutcfg
Spine1# sh bgp l2vpn evpn summary
<.....>
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.255.1.11     4 65001       7       8        6    0    0 00:01:45 0
10.255.1.12     4 65001       7       7        6    0    0 00:01:16 0
10.255.1.21     4 65001       7       7        6    0    0 00:01:01 0
10.255.1.22     4 65001       6       6        6    0    0 00:00:47 0
```

Подготовили основу и теперь наконец-то можем перейти непосредствено к настройке VxLAN для огранизации L2 каналов между хостами.

Дальнейшая настройка будет производиться только на стороне Leaf коммутаторов. 
Включим feature для настройки VxLAN, так же включим feature для ассоциации номера Vlan с номером VNI(Virtual Network Index):
```buildoutcfg
feature nv overlay
feature vn-segment-vlan-based
```
И включим интерфейс NVE, который отвечает за работу VxLAN:
```buildoutcfg
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
```
Однако, если мы проверим nve peers, то он окажется пустым. Тут необходимо вернуться к первональномй схеме. Мы видим, что
Leaf работают в парах и объединены VPC доменом. От сюда получается следующеся ситуация:

Хост отправляет один пакет на Leaf-21, второй пакет на Leaf-22. С точки зрения хоста оба Leaf выступают в качестве одного коммутатора.
Далее интерфейсе NVE должен построить тоннель до другого Leaf и передать пакет, полученный от хоста и возникает вопрос к какому Leaf 
надо строить тоннель. Потому что на практике адрес хоста доступен через два разных Leaf коммутатора.

Чтобы решить эту ситуации Leaf понимали что они имеют дело с VPC - для Cisco Nexus на Loopback задается secondary адресс. 
Этот адрес должен быть одинаковым на обоих коммутаторах в домене VPC:
```buildoutcfg
interface loopback0
 ip add 10.255.1.20/32 secondary
``` 

Сразу заведем Vlan и VNI для хостов. Настроим L2 тоннель между хостами
```buildoutcfg
vlan 10
  vn-segment 10000

interface nve1
  member vni 10000
    ingress-replication protocol bgp
```
Теперь проверим nve peers и таблицу для BGP evpn:

```buildoutcfg
Leaf11# sh nve peers
Interface Peer-IP          State LearnType Uptime   Router-Mac
--------- ---------------  ----- --------- -------- -----------------
nve1      10.255.1.20      Up    CP        00:00:41 n/a                 ! Видим что peer доступен с secondary адреса


Leaf11# sh bgp l2vpn evpn

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 10.255.1.11:32777    (L2VNI 10000)                       ! От кого именно пришет этот l2VNI
*>l[3]:[0]:[32]:[10.255.1.10]/88                                              ! EVPN route-type 3 - показывает нашего соседа, котороый так же знает об l2VNI10000
                      10.255.1.10                       100      32768 i
*>i[3]:[0]:[32]:[10.255.1.20]/88
                      10.255.1.20                       100          0 i
* i                   10.255.1.20                       100          0 i

Route Distinguisher: 10.255.1.21:32777
* i[3]:[0]:[32]:[10.255.1.20]/88
                      10.255.1.20                       100          0 i
*>i                   10.255.1.20                       100          0 i

Route Distinguisher: 10.255.1.22:32777
*>i[3]:[0]:[32]:[10.255.1.20]/88
                      10.255.1.20                       100          0 i
* i                   10.255.1.20                       100          0 i
```

Выше видим только маршруты 3 типа, которые рассказывают о peer(Leaf), однако где же наши хосты?

Для того чтобы увидеть наши хосты необходимо настроить EVPN route-type 2, которые будут рассказывать об MAC или MAC/IP хостов

```buildoutcfg
evpn
  vni 10000 l2
    route-target import auto
    route-target export auto
```

Так же сделаем ping с одного хоста до другого и в BGP l2route evpn появляются следующая информация:

```buildoutcfg
Firewall2# ping 192.168.10.1
PING 192.168.10.1 (192.168.10.1): 56 data bytes
36 bytes from 192.168.10.2: Destination Host Unreachable
Request 0 timed out
64 bytes from 192.168.10.1: icmp_seq=1 ttl=254 time=215.555 ms
64 bytes from 192.168.10.1: icmp_seq=2 ttl=254 time=38.756 ms
64 bytes from 192.168.10.1: icmp_seq=3 ttl=254 time=42.484 ms
64 bytes from 192.168.10.1: icmp_seq=4 ttl=254 time=40.983 ms

Leaf11# sh bgp l2vpn evpn
<......>

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 10.255.1.11:32777    (L2VNI 10000)
*>l[2]:[0]:[0]:[48]:[5001.0007.0007]:[0]:[0.0.0.0]/216                      ! видим что это evpn route-type 2 и mac адресс хоста 1
                      10.255.1.10                       100      32768 i
*>i[2]:[0]:[0]:[48]:[5001.0008.0007]:[0]:[0.0.0.0]/216                      ! видим что это evpn route-type 2 и mac адресс хоста 2
                      10.255.1.20                       100          0 i
* i                   10.255.1.20                       100          0 i
*>l[3]:[0]:[32]:[10.255.1.10]/88
                      10.255.1.10                       100      32768 i
*>i[3]:[0]:[32]:[10.255.1.20]/88
                      10.255.1.20                       100          0 i
* i                   10.255.1.20                       100          0 i

Route Distinguisher: 10.255.1.21:32777
* i[2]:[0]:[0]:[48]:[5001.0008.0007]:[0]:[0.0.0.0]/216
                      10.255.1.20                       100          0 i
*>i                   10.255.1.20                       100          0 i
* i[3]:[0]:[32]:[10.255.1.20]/88
                      10.255.1.20                       100          0 i
*>i                   10.255.1.20                       100          0 i

Route Distinguisher: 10.255.1.22:32777
* i[2]:[0]:[0]:[48]:[5001.0008.0007]:[0]:[0.0.0.0]/216
                      10.255.1.20                       100          0 i
*>i                   10.255.1.20                       100          0 i
*>i[3]:[0]:[32]:[10.255.1.20]/88
                      10.255.1.20                       100          0 i
* i                   10.255.1.20                       100          0 i
```

Отлично, теперь у у нас появилась L2 свзянность