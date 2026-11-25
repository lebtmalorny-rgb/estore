# Huawei eStor T10: сеть и iSCSI multipath для Cinder Epoxy 2025.1

Дата подготовки: 10.11.2026.

Исходное состояние: T10 уже загружен, в консоли видно `admin:/>`, мастер
первоначальной настройки пропущен. Лаборатория состоит из ВМ, FC отсутствует.
Цель — подключить T10 к Cinder по iSCSI и проверить два пути к тестовому LUN.

Инструкция относится к ранее разобранному образу
`Vmware_OceanStor_eStor_V1R1C00T10_single_controller.vmdk`:
**Dorado6000 V3 / V300R003C00**, один контроллер. Настройки OpenStack ниже
даны для **Kolla-Ansible 2025.1**. Для другой системы развёртывания содержимое
конфигов применимо, но способ их доставки и управления сервисами отличается.

Повторный запуск мастера для изменения IP не требуется. Не запускайте заново
инициализацию образа: адреса меняются штатными командами Huawei.

Синтаксис CLI ранее сверялся со справкой внутри T10; настройки драйвера —
с документацией и исходниками Epoxy. Команды изменения конфигурации,
развёртывание Cinder и проверка I/O на данном экземпляре не выполнялись.
Поддержка семейства Dorado драйвером не подтверждает полную совместимость
всех функций эмулятора.

## 1. Адреса и значения для замены

Все адреса ниже — **пример**. До выполнения команд замените их адресами
своей лаборатории. Для каждого дополнительного узла нужен собственный IP.

| Назначение | T10 | compute01 | storage01 с cinder-volume |
| --- | --- | --- | --- |
| Управление / REST | `192.168.10.50/24` | — | Доступ к management TCP 8088 |
| iSCSI A | `172.20.10.50/24` | `172.20.10.11/24` | `172.20.10.21/24` |
| iSCSI B | `172.20.20.50/24` | `172.20.20.11/24` | `172.20.20.21/24` |

Пример шлюза управления — `192.168.10.1`. Для прямой связи узлов внутри
каждой iSCSI-подсети отдельный шлюз не нужен.

| Обозначение | Что подставить |
| --- | --- |
| `MGMT_PORT_ID` | ID порта из `show system management_ip` |
| `PORT_A`, `PORT_B` | ID двух разных сервисных Ethernet-портов |
| `PORT_GROUP_ID` | ID группы `OpenStack-iSCSI` |
| `OpenStackPool` | Имя существующего или специально созданного пула |
| `INVENTORY_HOSTNAME` | Имя выбранного узла Cinder в inventory Ansible |
| `REPLACE_WITH_*_IQN` | Реальный IQN соответствующего узла OpenStack |
| `REPLACE_WITH_PASSWORD` | Действующий пароль пользователя `cinder` массива |
| `TEST_VM` | Имя или UUID тестовой ВМ OpenStack |

Обозначения в командах Huawei — не переменные shell. Подставляйте значения
вручную, вводите команды по одной, приглашение `admin:/>` не копируйте.

## 2. Посмотреть существующую конфигурацию T10

В консоли Huawei:

```text
show system general
show system management_ip
show port general physical_type=ETH logic_type=Host_Port
show port ip
show port_group general
show disk_domain general
show storage_pool general
```

Сохраните исходные IP и ID портов. Существующие исправные пул и группу портов
можно использовать повторно. Если команда отличается в вашей сборке,
посмотрите встроенную справку: `help system`, `help port`, `help port_group`.

## 3. Задать IP управления после пропущенного мастера

Если текущий адрес уже подходит, этот шаг пропустите. Смену адреса выполняйте
через консоль ВМ: SSH-соединение со старым IP будет разорвано.

Подставьте реальный ID порта управления и свой свободный IP:

```text
change system management_ip eth_port_id=MGMT_PORT_ID ip_type=ipv4_address ipv4_address=192.168.10.50 mask=255.255.255.0
```

Если нужен шлюз, добавьте в конец этой же команды
`gateway_ipv4=192.168.10.1`, заменив адрес фактическим шлюзом.

Проверка:

```text
show system management_ip
```

Откройте DeviceManager по адресу `https://192.168.10.50:8088`.
С узла `cinder-volume` этот адрес должен быть доступен для REST API.

## 4. Подключить виртуальные адаптеры к сетям

В подготовленной OVA v2 шесть адаптеров первоначально подключены к одной
сети `eStor-Lab`. Администратор гипервизора должен сопоставить адаптер
управления с management-сетью, а два выбранных сервисных адаптера —
с сетями iSCSI A и B. Сопоставляйте порты по MAC и фактической конфигурации,
не по предположению, что Network adapter 2 обязательно соответствует PORT_A.

На каждом compute и узле Cinder настройте доступ к обеим iSCSI-подсетям.
Это сеть самих узлов OpenStack: создание tenant-сети Neutron такой доступ
само по себе не предоставляет. Для начала используйте MTU 1500 по всему пути.

Две отдельные виртуальные сети позволяют проверять отказ одного пути.
Если оба пути проходят через один общий uplink, отказ uplink прервёт оба.
Остановка единственной ВМ T10 также остановит оба пути.

## 5. Настроить два сервисных iSCSI-порта

Используйте сервисные Ethernet-порты из раздела 2. Порт управления
для этих команд не выбирайте.

```text
change port ipv4_address eth_port_id=PORT_A ip=172.20.10.50 mask=255.255.255.0
change port ipv4_address eth_port_id=PORT_B ip=172.20.20.50 mask=255.255.255.0
change port eth eth_port_id=PORT_A mtu=1500
change port eth eth_port_id=PORT_B mtu=1500
show port ip eth_port_id=PORT_A
show port ip eth_port_id=PORT_B
show port general physical_type=ETH logic_type=Host_Port
```

В T10 IPv4 задаётся через `change port ipv4_address`, а MTU — через
`change port eth`. Смена IP прерывает соединения выбранного порта.
У обоих портов проверьте адреса и Link Up.

Если группы `OpenStack-iSCSI` ещё нет:

```text
create port_group name=OpenStack-iSCSI
show port_group general
```

Из вывода возьмите ID группы. Добавьте в неё оба порта, если они ещё не добавлены:

```text
add port_group port port_group_id=PORT_GROUP_ID port_type=ETH port_id_list=PORT_A,PORT_B
show port_group port port_group_id=PORT_GROUP_ID port_type=ETH
```

Эта группа Huawei используется для сопоставления LUN с хостом.
Она является отдельным объектом от сетевой Port Group в vSphere.

С каждого compute и узла Cinder проверьте TCP-доступность обоих targets:

```bash
nc -vz 172.20.10.50 3260
nc -vz 172.20.20.50 3260
```

С узла Cinder дополнительно:

```bash
nc -vz 192.168.10.50 8088
```

Успешный TCP connect проверяет доступность порта, но ещё не чтение/запись LUN.

## 6. Подготовить пул и пользователя REST

В `show storage_pool general` найдите подходящий пул с доступной ёмкостью.
Его точное имя затем укажите в XML. В примере это `OpenStackPool`.

Если пула нет, создайте в DeviceManager Disk Domain из свободных дисков,
затем Storage Pool в этом домене. Выбирайте RAID из доступных для данной
модели и числа дисков вариантов. Не удаляйте существующие пулы ради примера.
Диски, которые показывает модель СХД, не обязаны соответствовать VMDK один к одному.

Проверьте существующих пользователей:

```text
show user
```

Если пользователя `cinder` ещё нет, создайте его из сеанса супер-администратора:

```text
create user type=local user_name=cinder level=admin
```

CLI запросит пароль отдельно. Используйте пароль, соответствующий политике
массива; ASCII-символы совместимы с кодированием учётных данных драйвера Epoxy.
Затем разрешите REST и вход для обслуживания учётной записи:

```text
change user user_name=cinder login_method_list=CLI,DeviceManager,RESTful
show user user_name=cinder
```

`login_method_list` заменяет весь список разрешённых способов входа.
Если используется существующий пользователь, сохраните необходимые ему методы.
Если при первом входе требуется смена пароля, выполните её до подключения
Cinder и используйте окончательный пароль в XML.

LUN, Host, инициаторы и сопоставления для тестового тома будут создаваться
драйвером при операциях OpenStack. Вручную сопоставлять будущие тома всем
compute не требуется.

## 7. Какие файлы настроить в Kolla-Ansible

Файлы слева находятся на deploy-узле. Не редактируйте только рабочие файлы
внутри контейнеров: Kolla может перезаписать их при следующем применении.

| Файл на deploy-узле | Назначение |
| --- | --- |
| `/etc/kolla/globals.yml` | Включение iSCSI, multipathd и доставки Huawei XML |
| `/etc/kolla/config/cinder/cinder_huawei_estor.xml` | REST, пул, IQN и группа портов массива |
| `/etc/kolla/config/cinder/INVENTORY_HOSTNAME/cinder.conf` | Backend на выбранном узле Cinder |
| `/etc/kolla/config/nova.conf` | Multipath для подключения томов к ВМ |
| `/etc/kolla/config/multipath.conf` | Необязательная замена штатной политики multipath |

В этой схеме используется **один активный cinder-volume для backend T10**.
Проверенный Huawei-драйвер не объявляет поддержку active-active. Если Cinder
уже работает в HA, размещение backend и `cinder_cluster_name` нужно согласовать
с конфигурацией сервисов; отключение HA-precheck не добавляет поддержки драйверу.

### globals.yml

Добавьте, сохранив остальные настройки:

```yaml
enable_cinder: "yes"
enable_cinder_backend_iscsi: "yes"
enable_multipathd: "yes"
cinder_backend_huawei: "yes"
cinder_backend_huawei_xml_files:
  - cinder_huawei_estor.xml
```

Переменная называется `cinder_backend_huawei`, без `enable_`.
Локальная LVM-группа `cinder-volumes` для внешнего Huawei backend не нужна.
Не отключайте уже используемые Ceph/LVM/другие backends.
Убедитесь, что необходимые узлы входят в группы inventory `iscsid` и `multipathd`.

### IQN узлов

Когда `iscsid` развёрнут, на каждом compute и узле Cinder выполните:

```bash
docker exec iscsid cat /etc/iscsi/initiatorname.iscsi
```

В XML копируется значение после `InitiatorName=`. IQN разных узлов должны
различаться. Если `iscsid` отсутствует, сначала разверните его штатной ролью
Kolla. Для cinder-backup с локальным iSCSI-подключением нужен также IQN его узла.

### cinder_huawei_estor.xml

Создайте `/etc/kolla/config/cinder/cinder_huawei_estor.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config>
  <Storage>
    <Product>Dorado</Product>
    <Protocol>iSCSI</Protocol>
    <RestURL>https://192.168.10.50:8088/deviceManager/rest/</RestURL>
    <UserName>cinder</UserName>
    <UserPassword>REPLACE_WITH_PASSWORD</UserPassword>
  </Storage>
  <LUN>
    <LUNType>Thin</LUNType>
    <StoragePool>OpenStackPool</StoragePool>
  </LUN>
  <iSCSI>
    <DefaultTargetIP>172.20.10.50</DefaultTargetIP>
    <Initiator Name="REPLACE_WITH_COMPUTE01_IQN"
               TargetPortGroup="OpenStack-iSCSI"/>
    <Initiator Name="REPLACE_WITH_CINDER01_IQN"
               TargetPortGroup="OpenStack-iSCSI"/>
  </iSCSI>
</config>
```

Для Dorado V3 значение `Product` — именно `Dorado`, тип LUN — `Thin`.
Добавьте отдельную запись `Initiator` для каждого подключающего тома узла.
Требуется точное совпадение IQN: `Name="*"` не заменяет перечисление узлов.

Для двух путей используется `TargetPortGroup` с двумя исправными портами.
Просто перечислить два IP в `DefaultTargetIP` недостаточно: резервная ветка
выбора адресов драйвера использует первый из них.

В пароле XML символы `&` и `<` записываются как `&amp;` и `&lt;`.
Ограничьте права на исходный XML и не сохраняйте действующие секреты в Git.
Kolla доставляет его внутрь cinder-volume как
`/etc/cinder/cinder_huawei_estor.xml`, с владельцем `cinder` и правами `0600`.
Драйверу нужна возможность записи: он переписывает учётные данные в формате
`!$$$` + Base64. Это кодирование, не шифрование.

В примере оставлен лабораторный режим TLS драйвера без проверки сертификата.
Для проверки сертификата добавьте в `Storage` параметры `SSLCertVerify=true`
и `SSLCertPath` с путём к доверенному CA, доступному внутри контейнера.

### cinder.conf

В `/etc/kolla/config/cinder/INVENTORY_HOSTNAME/cinder.conf`:

```ini
[DEFAULT]
enabled_backends = estor_iscsi

[estor_iscsi]
volume_driver = cinder.volume.drivers.huawei.huawei_driver.HuaweiISCSIDriver
volume_backend_name = estor_iscsi
cinder_huawei_conf_file = /etc/cinder/cinder_huawei_estor.xml
use_multipath_for_image_xfer = true
enforce_multipath_for_image_xfer = true
```

Если `enabled_backends` уже содержит другие секции, добавьте `estor_iscsi`
через запятую, сохранив их. Флаг доставки XML в globals.yml сам по себе
секцию backend не создаёт. При пакетной установке это содержимое относится
к `/etc/cinder/cinder.conf`, XML — к пути из `cinder_huawei_conf_file`.

### nova.conf

В `/etc/kolla/config/nova.conf`:

```ini
[libvirt]
volume_use_multipath = true
volume_enforce_multipath = true
```

Флаги `enforce_*` требуют доступного multipathd, поэтому сначала обеспечьте
его работу. Они не проверяют наличие именно двух исправных путей к каждому LUN.
При пакетной установке настройки Nova относятся к `/etc/nova/nova.conf`.

### Политика multipath и её доставка

Для первого подключения можно использовать штатную политику Kolla:

```text
defaults {
    user_friendly_names no
    find_multipaths yes
    skip_kpartx yes
}

blacklist {
}
```

Пользовательский `/etc/kolla/config/multipath.conf` заменяет штатный файл
целиком. Профиль `HUAWEI/XSG1` и параметры ALUA проверяются по фактическим
SCSI inquiry и состоянию путей эмулятора; профиль физического Dorado V6
не следует автоматически считать подходящим T10.

В ранее проверенном upstream-срезе Kolla 2025.1 доставка multipath.conf
предусмотрена для `multipathd` и `nova_compute`, но отсутствует в списке
`config_files` шаблона `cinder_volume`. Проверьте свой fork и образ.
Для согласованной пользовательской политики Cinder может потребоваться
доставка файла через роль и config.json в `/etc/multipath.conf` контейнера.
Эта адаптация конкретного fork данным документом не внедряется.

## 8. Сформировать и применить конфигурацию

Перед применением замените placeholders, согласуйте IP, IQN, пул, группу
портов и размещение backend. Если нужны изменения доставки multipath.conf,
сначала внесите их в используемую роль Kolla.

На deploy-узле из окружения вашей Kolla-Ansible, заменив путь inventory:

```bash
kolla-ansible genconfig -i /path/to/inventory --tags iscsi,multipathd,cinder,nova
```

Проверьте сгенерированные cinder.conf, nova.conf, config.json и XML.
Содержимое XML с паролем не публикуйте в выводах диагностики. Затем:

```bash
kolla-ansible prechecks -i /path/to/inventory --tags iscsi,multipathd,cinder,nova
```

После устранения замечаний для существующего развёртывания:

```bash
kolla-ansible reconfigure -i /path/to/inventory --tags iscsi,multipathd,cinder,nova
```

`reconfigure` может перезапускать контейнеры. Новые сервисы разворачиваются
принятым в вашем окружении циклом `deploy`; одно изменение globals.yml
не означает, что отсутствующие контейнеры уже созданы.

## 9. Создать тестовый том и проверить два пути

На узле с настроенным OpenStack CLI и загруженными учётными данными:

```bash
openstack volume service list
openstack volume pool list --detail
```

Backend `estor_iscsi` должен быть `up`, пул — присутствовать.
Для нового лабораторного типа и тома:

```bash
openstack volume type create estor-iscsi
openstack volume type set --property volume_backend_name=estor_iscsi estor-iscsi
openstack volume create --size 1 --type estor-iscsi t10-test
openstack volume show t10-test
```

Дождитесь `available`, затем подключите к тестовой ВМ:

```bash
openstack server add volume TEST_VM t10-test
```

На compute, где работает эта ВМ:

```bash
docker exec iscsid iscsiadm -m session
docker exec multipathd multipath -ll
docker exec nova_compute multipath -ll
```

Ожидается один WWID тестового LUN, одна DM multipath-карта и два исправных
SCSI-пути через разные target IP. Статус тома `in-use` сам по себе этого
не доказывает. Обычный attach к ВМ подключает LUN на compute, поэтому отсутствие
этого LUN на узле cinder-volume в такой момент не является ошибкой.

Для отдельной проверки пути Cinder image-transfer создайте том из доступного
Glance-образа, указав достаточный размер; наблюдайте сеансы и multipath-карту
на узле Cinder во время копирования. После операции временное подключение
может быть снято.

Для проверки потери пути:

1. Сопоставьте тестовый том с WWID на compute и устройством внутри гостя.
2. Запустите I/O на специально подготовленном тестовом томе и убедитесь,
   что исходно оба пути исправны.
3. Администратор гипервизора отключает ровно один сервисный адаптер T10,
   сверив MAC. Management и второй сервисный адаптер остаются подключены.
4. Проверьте потерю одного пути, сохранение DM-карты и продолжение I/O.
5. Подключите адаптер обратно и дождитесь восстановления пути.

Такой тест проверяет отказ сетевого пути к одной ВМ эмулятора. Он не доказывает
отказоустойчивость контроллеров, хоста ESXi или физической СХД.

## 10. Если что-то не работает

| Симптом | Первая проверка |
| --- | --- |
| DeviceManager недоступен | Management IP, маска/шлюз, сеть адаптера, TCP 8088 |
| Backend `down` | REST, права и пароль пользователя, имя пула, XML и журнал Cinder |
| Том создаётся, attach не работает | Оба target IP:3260 с нужного compute, его IQN и журнал Cinder |
| Видно только один путь | Точный IQN в XML, TargetPortGroup, состав группы, Link Up и multipath Nova |
| Два SCSI-устройства без DM-карты | multipathd, blacklist/find_multipaths и совпадение WWID |
| Нет multipath.conf в Cinder | Доставка файла используемой ролью и содержимое образа контейнера |

Для уточнения настройки T10 достаточно начать с вывода:

```text
show system management_ip
show port general physical_type=ETH logic_type=Host_Port
show port ip
```

## Основание и дополнительные материалы

- Справка CLI исходного T10: `/crash/permitdir/ISM/cli/help/` внутри VMDK;
  ранее проверены команды `system`, `port`, `port_group`, `user`, `storage_pool`.
- [Huawei driver, Cinder 2025.1](https://docs.openstack.org/cinder/2025.1/configuration/block-storage/drivers/huawei-storage-driver.html).
- [Выбор target IP и группы портов, Cinder stable/2025.1](https://github.com/openstack/cinder/blob/stable/2025.1/cinder/volume/drivers/huawei/rest_client.py).
- [Обработка XML, Cinder stable/2025.1](https://github.com/openstack/cinder/blob/stable/2025.1/cinder/volume/drivers/huawei/huawei_conf.py).
- [Kolla-Ansible Cinder guide, 2025.1](https://docs.openstack.org/kolla-ansible/2025.1/reference/storage/cinder-guide.html).
- [Доставка Huawei XML, проверенный срез Kolla](https://github.com/openstack/kolla-ansible/blob/7222f0fb4b78ab668834a7d548661ea5c76bd099/ansible/roles/cinder/templates/cinder-volume.json.j2).
- [Настройки Nova multipath](https://docs.openstack.org/nova/2025.1/configuration/config.html#libvirt.volume_enforce_multipath).
- Дополнительные локальные инструкции: [подготовка eStor](ESTOR_SETUP.md)
  и [Kolla / Cinder / multipath](CINDER_MULTIPATH_KOLLA_2025_1.md).

Предыдущая проверка полных исходников выполнена для Cinder
`6cc666c01938b398d617391dc676bafc4a95b28c` и Kolla-Ansible
`7222f0fb4b78ab668834a7d548661ea5c76bd099`. В текущей сессии дополнительно
сопоставлены документация Cinder 2025.1 и выбор target IP ветки stable/2025.1.
Собственный fork и ранние сборки Epoxy могут отличаться от этих срезов.
