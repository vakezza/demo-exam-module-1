# Модуль 1 — настройка сети по шагам

**Порядок:** адреса и VLAN → шлюзы и GRE → OSPF → NAT и DHCP → SSH и DNS → отчёт. После каждого шага проверь результат. Если на стенде шаг уже выполнен, проверь его и переходи дальше.

Это рабочий пример по образцу из репозитория. На экзамене сначала сверь адреса, имена портов и дополнительные требования с выданным вариантом. Linux: команды настройки выполняй от root через `su -`. EcoRouter: `en` → `conf t` для настройки, `end` → `write memory` для сохранения. Команды `show` запускай после `end`.

## 1. Таблица адресов

| Сегмент | Адреса | Шлюз для конечной машины |
|---|---|---|
| ISP → HQ-RTR | ISP 172.16.1.1/28, HQ-RTR 172.16.1.2/28 | HQ-RTR → 172.16.1.1 |
| ISP → BR-RTR | ISP 172.16.2.1/28, BR-RTR 172.16.2.2/28 | BR-RTR → 172.16.2.1 |
| VLAN 100 | HQ-RTR 10.10.100.1/27, HQ-SRV 10.10.100.2/27 | 10.10.100.1 |
| VLAN 200 | HQ-RTR 10.10.200.1/28, HQ-CLI по DHCP (.2–.14) | 10.10.200.1 |
| VLAN 999 | HQ-RTR 10.10.30.1/29, HQ-SW .2/29 при необходимости | 10.10.30.1 |
| BR-RTR → BR-FW | BR-RTR 10.20.10.1/30, BR-FW 10.20.10.2/30 | BR-FW → 10.20.10.1 |
| BR-SRV | BR-FW 10.20.20.1/28, BR-SRV 10.20.20.2/28 | 10.20.20.1 |
| BR-CLI | BR-FW 10.20.30.1/29, BR-CLI 10.20.30.2/29 | 10.20.30.1 |
| GRE | HQ-RTR 10.10.10.1/30, BR-RTR 10.10.10.2/30 | — |

Маски: /27 = 255.255.255.224, /28 = 255.255.255.240, /29 = 255.255.255.248, /30 = 255.255.255.252. /27 даёт 32 адреса всего, /28 — 16, /29 — 8, /30 — 4. Адрес сети и broadcast устройствам не назначаются.

## 2. Альт Linux и ISP

На каждой Linux-машине выполни `hostnamectl set-hostname ИМЯ.au-team.irpo` (имя строчными, например `hq-srv`) и `timedatectl set-timezone Europe/Moscow`. Проверь `hostname`, `timedatectl`.

**Статический IP.** Узнай интерфейс через `ip -br link`. В каталоге `/etc/net/ifaces/ИМЯ_ИНТЕРФЕЙСА/` проверь `options`: `TYPE=eth`, `BOOTPROTO=static`, `CONFIG_IPV4=yes`, `DISABLED=no`. В `ipv4address` запиши адрес с маской. В `ipv4route` запиши `default via АДРЕС_ШЛЮЗА` для конечных машин. В `resolv.conf` запиши `search au-team.irpo` и `nameserver 10.10.100.2` для BR-SRV и BR-CLI. На HQ-SRV после запуска DNS укажи `nameserver 127.0.0.1`. Для редактирования можно использовать `nano`, если есть, или `vi` (i → ввод → Esc → :wq). Применяй с консоли ВМ: `systemctl restart network`. Проверь `ip -br a`, `ip route` и ping своего шлюза.

**ISP.** На внешнем интерфейсе `BOOTPROTO=dhcp`. На двух внутренних — статика из таблицы, без default route. Пример имён из материалов: ens3 наружу, ens4 к HQ, ens5 к BR. Проверь, что `ip route` показывает default через провайдера. В `/etc/net/sysctl.conf` выставь `net.ipv4.ip_forward = 1` и выполни `sysctl -p /etc/net/sysctl.conf`. Проверь `sysctl net.ipv4.ip_forward`.

Если внешний интерфейс действительно ens3, выполни на ISP:
~~~bash
iptables -t nat -C POSTROUTING -o ens3 -j MASQUERADE || iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
iptables -t nat -nvL POSTROUTING
~~~

## 3. EcoRouter и VLAN

На каждом роутере задай `hostname HQ-RTR` или `hostname BR-RTR`, затем `ip domain-name au-team.irpo`. В том же режиме конфигурации:
~~~text
username net_admin
password P@ssw0rd
role admin
exit
ntp timezone utc+3
~~~

Сначала посмотри `show ip interface brief`, `show port brief`, `show running-config`. Если интерфейсы отсутствуют, создай их по таблице ниже. Для каждого логического интерфейса нужен IP и привязка к физическому порту.

| Роутер | Интерфейс | Порт по примеру | Кадры |
|---|---|---|---|
| HQ-RTR | isp | ge0 | untagged |
| HQ-RTR | vl100, vl200, vl999 | ge1 | dot1q 100, 200, 999 |
| BR-RTR | isp | ge0 | untagged |
| BR-RTR | fw | ge1 | untagged |

Шаблон для HQ-RTR на порту к ISP:
~~~text
interface isp
ip address 172.16.1.2/28
exit
port ge0
service-instance uplink
encapsulation untagged
connect ip interface isp
exit
exit
~~~

Для VLAN создай `interface vl100` с адресом 10.10.100.1/27. На порту `ge1` создай `service-instance v100`, выполни `encapsulation dot1q 100`, `rewrite pop 1`, `connect ip interface vl100`. Аналогично сделай 200 и 999 со своими номерами, именами и IP. На BR-RTR создай `isp` и `fw` по шаблону untagged, со своими портами и адресами.

**HQ-SW с Open vSwitch.** Проверь `ovs-vsctl show`. Порт к роутеру должен пропускать VLAN 100,200,999, а порты к серверу и клиенту должны быть access в VLAN 100 и 200. Пример для пустого OVS. Замени PORT_RTR, PORT_SRV и PORT_CLI реальными именами:
~~~bash
ovs-vsctl --may-exist add-br br-hq
ovs-vsctl --may-exist add-port br-hq PORT_RTR
ovs-vsctl set port PORT_RTR vlan_mode=trunk trunks=100,200,999
ovs-vsctl --may-exist add-port br-hq PORT_SRV
ovs-vsctl set port PORT_SRV vlan_mode=access tag=100
ovs-vsctl --may-exist add-port br-hq PORT_CLI
ovs-vsctl set port PORT_CLI vlan_mode=access tag=200
~~~

Если нужен IP управления коммутатором, добавь внутренний OVS-порт в VLAN 999 и назначь IP ему. Проверь ping HQ-SRV → 10.10.100.1. Сохрани вывод `ovs-vsctl show` и конфигурацию привязок EcoRouter для отчёта.

## 4. GRE между HQ и BR

Настрой default route и проверь, что маршрутизаторы пингуют WAN-адреса друг друга через ISP. Внешние адреса в команде `ip tunnel` идут в порядке **свой → чужой**.

| Команда в `conf t` | HQ-RTR | BR-RTR |
|---|---|---|
| Маршрут | `ip route 0.0.0.0 0.0.0.0 172.16.1.1` | `ip route 0.0.0.0 0.0.0.0 172.16.2.1` |
| Туннель | `interface tunnel.0` | `interface tunnel.0` |
| Внутренний IP | `ip address 10.10.10.1/30` | `ip address 10.10.10.2/30` |
| GRE | `ip tunnel 172.16.1.2 172.16.2.2 mode gre` | `ip tunnel 172.16.2.2 172.16.1.2 mode gre` |

Выполни `end`, `write memory`. С HQ-RTR пингуй 10.10.10.2, с BR-RTR — 10.10.10.1. Если WAN пингуется, а туннель нет, сверь порядок WAN-адресов и прохождение GRE (IP-протокол 47).

## 5. OSPF и BR-FW

На HQ-RTR:
~~~text
router ospf 0
network 10.10.100.0/27 area 0
network 10.10.200.0/28 area 0
network 10.10.30.0/29 area 0
network 10.10.10.0/30 area 0
passive-interface default
no passive-interface tunnel.0
area 0 authentication
exit
interface tunnel.0
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
exit
~~~

На BR-RTR используй тот же блок, но вместо трёх HQ-сетей добавь `network 10.20.10.0/30 area 0`. Оставь сеть GRE. В разборе используется соседство BR-RTR ↔ BR-FW: для него добавь `no passive-interface fw` в настройке OSPF и повтори две команды MD5 под `interface fw`. Проверь формулировку выданного задания: образец одновременно включает BR-FW в OSPF и ограничивает обмен туннелем. Если в твоём варианте соседство на линке к BR-FW явно запрещено, следуй выданной схеме маршрутизации.

**BR-FW.** Для Ideco веб-интерфейс из разбора — `https://10.20.30.1:8443` с BR-CLI. Если доступа ещё нет, начни с консоли ВМ. В настройках сети задай адреса из таблицы, default через BR-RTR и имя `br-fw.au-team.irpo`. Топология образца показывает общий BR-NET, а условие требует две IP-подсети. На общем L2 нужны оба адреса шлюза на этом сегменте, если прошивка поддерживает дополнительный адрес. При раздельных портах или VLAN задай по одному шлюзу на соответствующий интерфейс.

В Ideco открой **Сервисы → OSPF**. Для соседства с BR-RTR укажи интерфейс к BR-RTR, область 0, MD5, Key ID 1 и пароль из задания. Анонсируй две локальные BR-сети через пассивные интерфейсы или экспорт подключённых маршрутов по возможностям версии. Настрой правила для трафика между офисами и исходящего в Интернет. Для OSPF на линке нужен IP-протокол 89. Для Xfirewall используй соответствующие настройки интерфейсов, OSPF и политик его версии.

**Проверка:** `show ip ospf neighbor` должен показать FULL с другой стороной GRE, а при включённом соседстве с BR-FW — также на линке fw со стороны BR-RTR. В `show ip route` ищи удалённые сети. Из BR-SRV проверь ping 10.10.100.2. Если маршруты есть, а ping нет, проверь обратный маршрут и политику BR-FW.

## 6. NAT и DHCP

На обоих роутерах в `conf t` выставь `ip nat outside` на интерфейсе `isp`. На HQ-RTR задай `ip nat inside` на `vl100`, `vl200`, `vl999`. На BR-RTR — на `fw`. После каждого интерфейса вводи `exit`.

На HQ-RTR:
~~~text
ip nat pool HQ 10.10.0.1-10.10.200.254
ip nat source dynamic inside-to-outside pool HQ overload interface isp
~~~
На BR-RTR:
~~~text
ip nat pool BR 10.20.0.1-10.20.30.254
ip nat source dynamic inside-to-outside pool BR overload interface isp
~~~

После сохранения с клиента проверь доступ к внешнему IP и `show ip nat translations` на роутере. Если связь между офисами ломается только после NAT, проверь, что межофисный трафик идёт через GRE и транслируется только трафик, выходящий через ISP.

**DHCP только на HQ-RTR:**
~~~text
ip pool HQCLI 10.10.200.2-10.10.200.14
dhcp-server 1
pool HQCLI 1
mask 255.255.255.240
gateway 10.10.200.1
dns 10.10.100.2
domain-name au-team.irpo
exit
exit
interface vl200
dhcp-server 1
end
write memory
show dhcp-server 1 detailed
show dhcp-server clients vl200
~~~

На HQ-CLI включи DHCP, обнови подключение с консоли и посмотри `ip -br a`, `ip route`, `cat /etc/resolv.conf`. Должны быть IP .2–.14, шлюз .1 и DNS 10.10.100.2. Если адреса нет, сначала проверь access VLAN 200, trunk и привязку DHCP к `vl200`. Запиши полученный IP для DNS и сохрани вывод DHCP для отчёта.

## 7. SSH на HQ-SRV и BR-SRV

На каждом сервере проверь `id sshuser` и `getent passwd 2026`. Если пользователя нет и UID свободен:
~~~bash
useradd -m -u 2026 -s /bin/bash sshuser
passwd sshuser
~~~

Введи учебный пароль из задания. Если sshuser уже существует, проверь его UID. Если UID 2026 занят другой учёткой, сначала выясни, для чего она нужна. Через `visudo` добавь:
~~~text
sshuser ALL=(ALL:ALL) NOPASSWD: ALL
~~~

В /etc/openssh/sshd_config на Альт (или по фактическому пути) в глобальной части до первого `Match` задай:
~~~text
MaxAuthTries 2
Banner /etc/openssh/banner
~~~

В файл /etc/openssh/banner запиши `Authorized access only`. Если выданное задание требует особый порт или ограничения входа, внеси их там же. Проверь синтаксис `sshd -t`, затем `systemctl restart sshd`. С клиента выполни `ssh sshuser@IP_СЕРВЕРА`, убедись, что виден баннер, и выполни `sudo -n id`. Ожидаемый результат — uid=0.

## 8. DNS на HQ-SRV

Установи `bind` и `bind-utils`, если отсутствуют. Посмотри `/etc/bind/named.conf` и его include. В существующий блок `options` добавь или обнови `forwarders { 77.88.8.8; };`. Разреши запросы и рекурсию из подсетей задания, убедись, что BIND слушает 127.0.0.1 и 10.10.100.2. В подключённый файл зон (в разборе `/etc/bind/rfc1912.conf`) добавь, если зон ещё нет:
~~~text
zone "au-team.irpo" {
    type master;
    file "/etc/bind/zone/au-team.irpo.zone";
};
zone "10.10.in-addr.arpa" {
    type master;
    file "/etc/bind/zone/10.10.in-addr.arpa";
};
~~~

Прямая зона /etc/bind/zone/au-team.irpo.zone. **В этом примере HQ-CLI получил 10.10.200.2. Замени .2 фактическим адресом в обеих зонах.** Если аренда изменится, обнови DNS или закрепи выдачу за MAC в DHCP.
~~~dns
$TTL 300
@ IN SOA hq-srv.au-team.irpo. hostmaster.au-team.irpo. (
    1 3600 900 604800 300 )
@      IN NS hq-srv.au-team.irpo.
hq-rtr IN A  10.10.100.1
hq-srv IN A  10.10.100.2
hq-cli IN A  10.10.200.2
br-rtr IN A  10.20.10.1
br-fw  IN A  10.20.10.2
br-srv IN A  10.20.20.2
br-cli IN A  10.20.30.2
docker IN A  172.16.1.1
web    IN A  172.16.2.1
~~~

Обратная зона /etc/bind/zone/10.10.in-addr.arpa:
~~~dns
$TTL 300
@ IN SOA hq-srv.au-team.irpo. hostmaster.au-team.irpo. (
    1 3600 900 604800 300 )
@     IN NS  hq-srv.au-team.irpo.
1.100 IN PTR hq-rtr.au-team.irpo.
2.100 IN PTR hq-srv.au-team.irpo.
2.200 IN PTR hq-cli.au-team.irpo.
~~~

Проверь и запусти:
~~~bash
named-checkconf
named-checkzone au-team.irpo /etc/bind/zone/au-team.irpo.zone
named-checkzone 10.10.in-addr.arpa /etc/bind/zone/10.10.in-addr.arpa
systemctl enable --now bind
systemctl restart bind
~~~

С клиента:
~~~bash
dig @10.10.100.2 br-srv.au-team.irpo +short
dig @10.10.100.2 -x 10.10.100.2 +short
dig @10.10.100.2 example.org +short
getent hosts hq-srv.au-team.irpo
~~~

Ожидай 10.20.20.2, hq-srv.au-team.irpo., внешний IP и успешное разрешение через системный DNS. Если сервер не отвечает из BR, проверь маршруты и доступ к DNS по UDP/TCP 53. Если меняешь файл зоны, увеличивай serial.

## 9. Проверка перед сдачей

- [ ] Имена, IP, маски, шлюзы и время проверены на всех машинах. Время HQ-SW выставлять не требуется по образцу.
- [ ] HQ-SRV пингует шлюз VLAN 100, HQ-CLI получил адрес по DHCP и пингует шлюз VLAN 200.
- [ ] GRE пингуется в обе стороны, OSPF в FULL и видны удалённые сети.
- [ ] HQ и BR видят друг друга и выходят в Интернет. NAT виден на роутерах.
- [ ] SSH показывает баннер, UID sshuser — 2026, sudo работает без пароля.
- [ ] DNS проверен из обоих офисов по имени и обратной записи.
- [ ] Выполнен `write memory` на обоих EcoRouter. Настройки BR-FW применены.
- [ ] В отчёте есть схема, таблица IP, описание VLAN и DHCP и читаемые результаты проверок. Для пунктов, требующих отдельных документов, укажи номер и краткое название. Итоговый файл назови `ФамилияУчастникаМодуль1` и сохрани на рабочем месте.

**Если что-то не работает:** IP и маска → ping своего шлюза → маршрут до цели → GRE/OSPF → правила BR-FW → DNS. Если по IP связь есть, а по имени нет, начни с `dig @10.10.100.2` и проверки DNS у клиента.

Материалы: [образец задания и разбор модуля 1](https://github.com/Ondre1212/-), [документация EcoRouter](https://docs.ecorouter.ru/), [настройка OSPF Ideco](https://docs.ideco.ru/pdf/v20/ru-ngfw-settings-services-ospf.pdf). Команды сверены с материалами, проверка на живом стенде не проводилась.

