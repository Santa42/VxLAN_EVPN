В Этой заметке хочу рассмотреть построение сетевой фабрики на основе VxLAN EVPN с внешним Firewall.
Все примеры будем выполнять на Cisco Nexus9k

Определим основную задачу сети:

1. Организовать L2 доступ между хостами 
2. Организовать маршрутизацию между сетями в рамках одного сервиса
3. Подключение Firewall к фабрике в качестве шлюза

Все примеры будем выполнять на образах nexus9k

Для решение задачи выбрали топологию Spine-Leaf:

![](img/all.jpg)

По схеме видно, что у нас для примера есть два хоста и сам firewall с которыми мы и будем проводить эксперименты.

Зададим адресацию:
```buildoutcfg
Spine-1 - 10.255.1.101
Spine-2 - 10.255.1.102

Leaf-11 - 10.255.1.11
Leaf-12 - 10.255.1.12
Leaf-21 - 10.255.1.21
Leaf-22 - 10.255.1.22
```

Но для начала необходимо настроить underlay сеть.

## Undrelay

Организуем IP связанность между соседними устройствами:
  ```buildoutcfg
interface loopback0
  description ROUTE_INT
  ip address { IPADD }      ! IP можем использовать с /32
  ```

```buildoutcfg
interface {INTF}
  mtu 9216
  medium p2p     ! Настраиваем Point-to-Point линки, чтобы убрать необходимость поиска DR/BDR между Nexus
  ip unnumbered loopback0    ! для экономии и упрощения работы заимствуем IP с Loopback
  no shutdown
```

Не будем усложнять и настроим на Leaf и Spine протокол OSPF для базовой IP связанности. 

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

interface {INTF}
  ip router ospf UNDERLAY area 0.0.0.0
```
Проверим, что у нас появилась базоавая IP связанность:

```buildoutcfg
show ip route

10.255.1.12/32, ubest/mbest: 1/0
    *via 10.255.1.101, Eth1/51, [110/9], 6d06h, ospf-Underlay, intra
    *via 10.255.1.102, Eth1/52, [110/9], 6d06h, ospf-Underlay, intra
10.255.1.21/32, ubest/mbest: 1/0
    *via 10.255.1.101, Eth1/51, [110/9], 6d06h, ospf-Underlay, intra
    *via 10.255.1.102, Eth1/52, [110/9], 6d06h, ospf-Underlay, intra
10.255.1.22/32, ubest/mbest: 1/0
    *via 10.255.1.101, Eth1/51, [110/9], 6d06h, ospf-Underlay, intra
    *via 10.255.1.102, Eth1/52, [110/9], 6d06h, ospf-Underlay, intra
10.255.1.101/32, ubest/mbest: 1/0
    *via 10.255.1.101, Eth1/51, [110/5], 6d06h, ospf-Underlay, intra
10.255.1.102/32, ubest/mbest: 1/0
    *via 10.255.1.102, Eth1/52, [110/9], 6d06h, ospf-Underlay, intra
```
Как видим от Leaf до других Leaf коммутаторов у нас есть по два пути через 2 Spine.

На данном этапе будем считать настройку базовой Underlay сети выполненной.
Конечно можно настроить еще различные технологии для уменьшения времени сходимости
 и скорости реагирования на какие-либо изменения, но данный материал выходит за рамки данной статьи.
