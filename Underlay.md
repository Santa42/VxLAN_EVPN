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



Но для начала необходимо настроить underlay сеть.

##Unrelay

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
Проверим что у нас появилась базоавая IP связанность:

```buildoutcfg
show ip route


```

