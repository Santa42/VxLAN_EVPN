## Проект Network

1. [Общее описание сети](./README.md#Basic)
   1. [Модули для работы сети на Cisco Nexus](./README.md#Feature)
2. [Работа Underlay сети](./README.md#Underlay)
   1. [Настройка Интерфейсов](./README.md#Intfunderlay)
   1. [Настройка OSPF](./README.md#OSPFunderlay)
3. [Настройка Overlay сети](./README.md#Overlay)
   1. [Настройка NVE](./README.md#NVEOverlay)
   1. [Настройка iBGP для l2route](./README.md#l2routeOverlay)
4. [Заведение сервиса](./README.md#Service)
   1. [Настройка VLAN](./README.md#VLANService)
   1. [Настройка VRF для сервиса](./README.md#VRFService)
   1. [Настройка interface VLAN](./README.md#IntVLANService)
   1. [Настройка BGP](./README.md#BGPService)
5. [Подключение физических серверов](./README.md#Po)
   1. [Настройка VPC](./README.md#VPCPo)
   
## <a name="Basic"></a> Общее описание сети
Сетевое облако построено на cisco Nexus9000 C93108TC-FX(медь) и Nexus9000 93180YC-EX chassis(sfp).

Сетевая топология облака: Spine-leaf

В качестве Spine работает Nexus9000 93180YC-EX chassis - 2 штуки (в дальйшем необходимо заменить на серию 9500)

В качестве Leaf работает Nexus9000 93180YC-EX chassis и cisco Nexus9000 C93108TC-FX(медь)

Смысл топологии разделить сеть на два основных уровня:
- Underlay - Основа сети. Работает между всеми nexus. Основной протокол маршрутизации OSPF
- Overlay - сеть для работы приложений. Основной протокол VXLAN+EVPN(BGP). Дополнительно работает протоколо iBGP для связи с Checkpoint

Разделение сети поволяет производить изменение в любой ее части без влияния на вторую.

### <a name="Feature"></a> Модули для работы сети на Cisco Nexus:

##### Spine:
* nv overlay evpn
* feature ospf
* feature bgp
* feature bfd

##### Leaf:

* nv overlay evpn
* feature ospf
* feature bgp
* feature interface-vlan
* feature vn-segment-vlan-based
* feature nv overlay
* feature bfd
* feature lacp (опционально для настройки Port-channel)
* feature vpc (опционально для настройки Port-channel c разными Nexus)

## <a name="Underlay"></a> Работа Underlay сети

Схема Underlay сети:

![](img/Network_Underlay.jpg)

Каждый Leaf подключен к двум Spine. Для упрощения настройки и управления сетью на каждом устройстве настроен только один IP
из сети 10.255.1.0/24. IP адррес настраивается на интерфейсе loopback2. Адреса на физических интерфейсах настриваются 
командой `ip unnumbered loopback2 `

### <a name="Intfunderlay"></a> Настройка интерфейсов

Настройка физического интерфейса:
  ```buildoutcfg
interface { Interf }
  description { description }
  mtu 9216
  medium p2p
  ip unnumbered loopback2
  no shutdown
  ```

Настройка loopback2:
  ```buildoutcfg
interface loopback2
  description ROUTE_INT
  ip address { IPADD }
  ```
### <a name="OSPFunderlay"></a> Настройка OSPF в Underlay

OSPF в Underlay необходим только для IP связанности между всеми устройствами в облаке
и работает в режиме point-to-point для избавления от лишней рассылки LSA и выборов DR/DBR.

Настройка OSPF в связки с BFD(отслеживание каналов между устройствами)
```buildoutcfg
router ospf Underlay
  bfd
  router-id { IPADD }
  name-lookup
```
Добавление интерфейсов в OSPF:
```buildoutcfg
interface { Interf }
  ip router ospf UNDERLAY area 0.0.0.0
  ip ospf bfd
```

## <a name="Overlay"></a>  Настройка Overlay сети
Схема Overlay сети:

![](img/Network_Overlay.jpg)

### <a name="NVEOverlay"></a>Настройка NVE

Виртуальный интерфейсы, который отвечает за инкапсуляцию данных клиентов внтрь UDP, для передачи через облако.
```buildoutcfg
interface nve1
  no shutdown
  source-interface loopback2
  host-reachability protocol bgp
```

### <a name="l2routeOverlay"></a>Настройка iBGP для l2route

В данном случае протокол BGP необходим для распространения MAC-адресов клиентов по облаку. 

Так как к каждому Spine подключены все Leaf, то Spine настраивается в качетве `route-reflector-server`. 
Для упрощения настройка на Spine заведен шаблон для подключения leaf устройств - `template peer VXLAN`:
```buildoutcfg
router bgp 65000
  address-family l2vpn evpn
  template peer VXLAN
    remote-as 65000
    update-source loopback2
    address-family l2vpn evpn
      send-community both
      route-reflector-client
  neighbor { IPADD_Leaf }
    inherit peer VXLAN
    description { NAME_Leaf }
```

Leaf подлючается только к Spine, поэтому сосодство BGP устанавливается только с двумся Spine. Для возможного расширения,
сосед так же задан через шаблон - `template peer VXLANNRR`:
```buildoutcfg
router bgp 65000
  address-family l2vpn evpn
  template peer VXLANNRR
    remote-as 65000
    update-source loopback2
    address-family l2vpn evpn
      send-community both
  neighbor { IPADD_Spine }
    inherit peer VXLANNRR
    description { NAME_Spine }
```

## <a name="Service"></a> Заведение сервиса:

Заведение сервиса состоит из трех этапов:
1. Заведение VLAN
2. Создание VRF
3. Включении маршрутизации между сервисами

### <a name="VLANService"></a> Настройка VLAN

VLAN имеет значение только в рамках одного Nexus (или пары, если они работают в связке VPC)

VLAN:
```buildoutcfg
vlan { vlan }
  name { name_vlan }
  vn-segment { vni }
```
Добавляем `vni` в настройку nve1:
```buildoutcfg
interface nve1
  member vni { vni }
    ingress-replication protocol bgp
```

### <a name="VRFService"></a> Настройка VRF для сервиса

VRF необходим для отдельния сервисов друг от друга. Маршрутизацию между VRF происходит на Checkpoint.
В VRF добавляются Interface Vlan, принадлежащие только одному сервису

Дополнительно для работы L3 связанности VRF на всех nexus необходимо добавить дополнительный (служебный) `vni`.
```buildoutcfg
vlan { vlan_L3 }
  name { name_vlan_L3}
  vn-segment { vni_L3 }
```
Далее создаем `vrf context`
```buildoutcfg
vrf context { vrf_name }
  vni { vni_L3 }
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn
```

Добавляем служебный `vni` в настройку nve:

```buildoutcfg
interface nve1
  member vni { vni_L3 } associate-vrf
```

И настраиваем маршрутизацию служебного `vni` через `interface vlan { vlan_L3 }`

```buildoutcfg
interface { vlan_L3 }
  no shutdown
  mtu 9216
  vrf member { vrf_name }
  no ip redirects
  ip forward
  no ipv6 redirects
```

### <a name="IntVLANService"></a> Настройка interface VLAN 

Заведение самого Vlan идентично настройке служебного VLAN:

```buildoutcfg
vlan { vlan }
  name { name_vlan}
  vn-segment { vni }
```

Для каждого vlan настраивается interface vlan с привязкой к VRF сервиса:

```buildoutcfg
interface vlan { vlan}
  description { name_vlan_service }
  no shutdown
  vrf member { vrf_name }
  no ip redirects
  ip address { IPADD_vlan }
  no ipv6 redirects
  fabric forwarding mode anycast-gateway
```
В каждом VLAN на всех Leaf должен быть одинаковый ip address.

Для работы одинакового адреса на всех Leaf должна быть прописана глобальная настройка:
`fabric forwarding anycast-gateway-mac 0000.0000.1111`

Для работы L2 свзяанности между серверами через сетевую фабрику надо прописать `vni` каждого vlan в настройку интерфейса `nve`
```buildoutcfg
interface nve1
  member vni { vni }
    ingress-replication protocol bgp
```

### <a name="BGPService"></a> Настройка BGP

Для маршрутизации между сервисами на Leaf (nx9300-[8,k]-0[1-2]), подключенных к Checkpoint необходимо настроить BGP для анонсирования IP сети сервиса:

```buildoutcfg
router bgp 65000
  vrf { vrf_name }
    address-family ipv4 unicast
      redistribute direct route-map { name_service }
    neighbor { IPADD_check}
      remote-as 65000
      update-source { vlan_check}
      address-family ipv4 unicast
        next-hop-self
```
Для работы маршрутизации на на Leaf (nx9300-[8,k]-0[1-2]) настраивается долнительный VLAN {vlan_check }  
и intreface vlan {vlan_check } для связи с checkpoint(актуальный только этих Leaf).

Для анонсирования маршрутов задается route-map с проверкой по prefix-list
```buildoutcfg
ip prefix-list { name_service } seq { seq } permit { NET_SERVICE }

route-map { name_service } permit 10
  match ip address prefix-list { name_service }
```

Соответсвенно на Checkpoint необходимо завести интерфейс и IP для каждого VRF. `Не забыть настроить BGP`

## <a name="Po"></a> Подключение физических серверов

Подключение серверов выполняется с использованием протокола LACP. Сервер подключается к одной паре Nexus.

Так как на Nexus отсутствует возможность кластеризации, используется технология VPC для синхронизации данных
между двумя нодами для работы протокола LACP. 

###  <a name="VPCPo"></a> Настройка VPC:

Для работы VPC, пара nexus должна видеть друг-друга по ip - `peer-keepalive` и связана как минимум
одним линком - `peer-link` 

Настройка vpc: 
```buildoutcfg
vpc domain { vpo_domain }
  peer-switch
  peer-keepalive destination { ip_neigh } source { ip_local }
  delay restore 180
  peer-gateway
  layer3 peer-router
  ip arp synchronize
```
Настройка peer-link:
```buildoutcfg
interface { intf }
  description peer-link
  switchport
  switchport mode trunk
  vpc peer-link
```
В настройке Porr-channel в сторону физического сервера необходимо добавить команду `vpc { vpc }`

Номер `vpc` желательно делать таким же, как номер интерфейса Port-channel.


