# Настройка eStor / Dorado V3 для OpenStack: DeviceManager и консоль

[Установка OVA/OVF](README.md) · [Настройка Kolla-Ansible и проверка multipath](CINDER_MULTIPATH_KOLLA_2025_1.md)

Инструкция относится к образу
`Vmware_OceanStor_eStor_V1R1C00T10_single_controller.vmdk` и его упаковке v2.
Это одноконтроллерный эмулятор Dorado V3. Два сетевых пути позволяют проверить
потерю одного пути; выключение ВМ eStor остановит оба.

Команды ниже сверены **29.10.2026 со справкой CLI внутри исходного VMDK**.
Они не выполнялись на работающем эмуляторе. Доступность конкретных дисков,
портов и ресурсов нужно проверить командами `show` на своём экземпляре.
Названия пунктов UI приведены для DeviceManager поколения V3 и могут отличаться
в отдельных сборках T10.

## Что подготовить

| Объект | Значение в примере |
| --- | --- |
| IP управления | `192.168.10.50` |
| Первый сервисный порт, iSCSI A | `172.20.10.50/24` |
| Второй сервисный порт, iSCSI B | `172.20.20.50/24` |
| Disk Domain | `OpenStackDisks` |
| Storage Pool | `OpenStackPool` |
| Группа портов Huawei | `OpenStack-iSCSI` |
| Пользователь REST API | `cinder` |

Адреса условные. Каждый compute и узел Cinder должны иметь доступ к обоим
iSCSI-адресам по TCP 3260; `cinder-volume` — к IP управления по TCP 8088.

У OVA v2 все шесть адаптеров первоначально подключены к одной сети `eStor-Lab`.
В настройках ВМ ESXi сопоставьте два выбранных сервисных адаптера с сетями A и B.
Порты Huawei сопоставляйте с адаптерами ESXi по MAC, а не по предполагаемому
порядку `Network adapter 1/2/3`. Порт управления оставьте в сети управления.
Для первого теста используйте MTU 1500 по всему пути.

## Вариант 1: DeviceManager

### Вход

Откройте `https://<IP-управления>:8088`.
Для заводского состояния в руководстве Huawei указан пользователь `admin`
с паролем `Admin@storage`. Если пароль уже меняли при первом входе, используйте
новый. Это заводские данные из документации, а не проверенный пароль вашего
экземпляра. См. [руководство Huawei Dorado V3](https://www.scribd.com/document/486044516/OceanStor-Dorado-V3-Series-V300R001-Quick-Configuration-Guide).

### Дисковый домен и пул

1. Откройте **Provisioning → Disk Domain → Create**.
2. Задайте имя `OpenStackDisks` и выберите свободные диски эмулятора.
3. Откройте **Provisioning → Storage Pool → Create**.
4. Задайте имя `OpenStackPool`, выберите созданный домен и подходящий RAID
   из предложенных мастером. Для стандартного примера используется RAID6.
5. Дождитесь готовности пула; проверьте доступную ёмкость.

Если подходящие домен и пул уже существуют, используйте их. Пересоздавать их
не нужно; фактическое имя пула затем указывается в XML Cinder. Виртуальные
диски ESXi и диски, показываемые моделью массива в DeviceManager, нельзя
считать взаимно однозначным списком.

### Два сервисных порта

В **System** откройте контроллерную полку, переключитесь на заднюю панель,
выберите сервисный Ethernet-порт и нажмите **Modify**. В некоторых вариантах
интерфейса список доступен через **Provisioning → Port → Ethernet Ports**.

Первому порту назначьте `172.20.10.50`, второму — `172.20.20.50`;
маска обоих — `255.255.255.0`, MTU — `1500`. Оставьте два отдельных порта.
Для узлов в тех же подсетях дополнительные маршруты не нужны.
Проверьте у обоих портов **Link Up**.

Настройка IP непосредственно на Ethernet-порту описана для V3 в
[Huawei SAN Storage Host Connectivity Guide, раздел 5.2.3](https://wiki.colobridge.net/_media/%D0%BA%D0%B0%D1%82%D0%B0%D0%BB%D0%BE%D0%B3_%D1%81%D1%82%D0%B0%D1%82%D0%B5%D0%B9/%D1%84%D0%B0%D0%B9%D0%BB%D0%BE%D0%B2%D1%8B%D0%B5_%D1%81%D0%B8%D1%81%D1%82%D0%B5%D0%BC%D1%8B/huawei_san_storage_host_connectivity_guide_for_vmware_esxi_06.pdf).
Сетевые параметры хоста ESXi из этого руководства не заменяют настройки
Linux-узлов OpenStack: ESXi здесь размещает ВМ эмулятора.

### Группа портов и пользователь

1. Откройте **Provisioning → Port → Port Group → Create**.
2. Создайте `OpenStack-iSCSI` и добавьте оба сервисных Ethernet-порта.
3. В **Settings → Permission Settings → User Management** создайте локального
   пользователя `cinder` с правами администратора и доступом **RESTful**.
4. Если требуется сменить первоначальный пароль, сделайте это до подключения
   Cinder; в XML нужен уже действующий пароль.

Группа портов Huawei задаёт порты для сопоставления LUN с хостом; это отдельный
объект от Port Group в vSphere. Порядок создания описан в
[Huawei HCIA-Storage Lab Manual](https://pdfcoffee.com/hcia-storagelabguidev40pdf-pdf-free.html).

## Вариант 2: консоль самого Huawei

Это альтернатива выполнению тех же действий через UI. Не создавайте повторно
объекты, уже подготовленные в DeviceManager.

### Подключение и правила ввода

С компьютера администратора подключитесь по SSH к IP управления, если SSH
доступен в вашей сети:

```bash
ssh admin@192.168.10.50
```

Ожидается приглашение Huawei вида `admin:/>`. В образе оболочка пользователя
`admin` задана как `/ISM/cli/ismcli`. Если консоль ESXi показывает Linux shell
или установочный мастер, это другое окружение: команды `show/create/change`
ниже вводятся именно в CLI Huawei.

Вводите команды по одной. `admin:/>` копировать не нужно. Обозначения
`DISK_LIST`, `DOMAIN_ID`, `POOL_ID`, `PORT_A`, `PORT_B`, `PORT_GROUP_ID` и
`MGMT_PORT_ID` заменяются найденными значениями; это не переменные оболочки.
Проверить справку непосредственно на устройстве можно так:

```text
help port
```

```text
help disk_domain
```

```text
help storage_pool
```

```text
help port_group
```

### Посмотреть существующую конфигурацию

Модель и версия:

```text
show system general
```

Адреса управления:

```text
show system management_ip
```

Диски, домены и пулы — отдельными командами:

```text
show disk general
```

```text
show disk_domain general
```

```text
show storage_pool general
```

Сервисные Ethernet-порты и их IP:

```text
show port general physical_type=ETH logic_type=Host_Port
```

```text
show port ip
```

Существующие группы портов:

```text
show port_group general
```

Запишите ID двух сервисных портов и их MAC, ID свободных дисков и уже
существующих объектов. Нельзя подставлять management/maintenance-порт вместо
`PORT_A` или `PORT_B`.

### Создать дисковый домен и пул

Только если подходящего домена ещё нет, замените `DISK_LIST` списком свободных
дисков из `show disk general`, перечисленных через запятую:

```text
create disk_domain name=OpenStackDisks disk_list=DISK_LIST hotspare_strategy=low
```

Проверьте результат и получите реальный `DOMAIN_ID`:

```text
show disk_domain general
```

Проверьте доступную ёмкость для RAID6:

```text
show disk_domain available_capacity disk_domain_id=DOMAIN_ID raid_level=RAID6
```

Если выбранные диски позволяют создать такой пул:

```text
create storage_pool name=OpenStackPool disk_domain_id=DOMAIN_ID raid_level=RAID6 capacity=remain
```

`capacity=remain` использует всю оставшуюся доступную ёмкость выбранного домена.
Если свободных дисков или ёмкости недостаточно, сначала проверьте модель
эмулятора и существующие ресурсы; создание нового пула само по себе этого
не исправит.

Получите `POOL_ID`, затем проверьте состояние и ёмкость:

```text
show storage_pool general
```

```text
show storage_pool general pool_id=POOL_ID
```

### Назначить IP сервисным портам

После подстановки ID и согласованных IP, до подключения рабочих LUN:

```text
change port ipv4_address eth_port_id=PORT_A ip=172.20.10.50 mask=255.255.255.0
```

```text
change port ipv4_address eth_port_id=PORT_B ip=172.20.20.50 mask=255.255.255.0
```

При необходимости установить MTU:

```text
change port eth eth_port_id=PORT_A mtu=1500
```

```text
change port eth eth_port_id=PORT_B mtu=1500
```

В T10 адрес задаётся через **`change port ipv4_address ... ip=... mask=...`**.
Команда `change port eth` используется для MTU/режима, а не для назначения IPv4.
Смена IP прерывает соединения этого порта и сбрасывает связанные с изменяемым
IP маршруты. При запросе подтверждения проверяйте, что выбран нужный сервисный
порт.

Проверка каждого порта:

```text
show port ip eth_port_id=PORT_A
```

```text
show port ip eth_port_id=PORT_B
```

```text
show port general physical_type=ETH logic_type=Host_Port
```

### Создать группу из двух портов

```text
create port_group name=OpenStack-iSCSI
```

Получите присвоенный ID группы по её имени:

```text
show port_group general
```

Добавьте оба порта, заменив все три обозначения:

```text
add port_group port port_group_id=PORT_GROUP_ID port_type=ETH port_id_list=PORT_A,PORT_B
```

Проверьте состав:

```text
show port_group port port_group_id=PORT_GROUP_ID port_type=ETH
```

В группе должны отображаться оба выбранных порта. Cinder получает имя
`OpenStack-iSCSI` через `TargetPortGroup` в XML.

### Создать пользователя для REST API

Сначала проверьте, не существует ли он уже:

```text
show user
```

Создайте пользователя из сеанса супер-администратора:

```text
create user type=local user_name=cinder level=admin
```

CLI отдельно запросит пароль и повторный ввод. Пароль в командную строку
не передаётся. Используйте ASCII-пароль, удовлетворяющий политике этой версии;
драйвер Epoxy кодирует исходные учётные данные через `latin-1`.

Разрешите REST и вход для обслуживания учётной записи:

```text
change user user_name=cinder login_method_list=CLI,DeviceManager,RESTful
```

`login_method_list` задаёт весь список разрешённых способов входа. Для уже
существующего пользователя сначала проверьте, какие способы нужно сохранить.

```text
show user user_name=cinder
```

Если при первом входе требуется смена пароля, войдите под `cinder`, выполните
её и внесите окончательный пароль в XML. Не отключайте политику паролей массива
ради обхода этого шага.

### При необходимости изменить IP управления

Этот шаг не нужен, если DeviceManager и SSH уже доступны по правильному адресу.
`MGMT_PORT_ID` берётся из `show system management_ip`. Выполняйте смену через
доступную консоль Huawei: SSH по старому адресу будет разорван.

```text
change system management_ip eth_port_id=MGMT_PORT_ID ip_type=ipv4_address ipv4_address=192.168.10.50 mask=255.255.255.0
```

Если нужен шлюз управления, добавьте параметр `gateway_ipv4` с фактическим
адресом шлюза; в примере адрес шлюза не задан. После изменения:

```text
show system management_ip
```

## Что создаёт Cinder

На этом подготовка хранилища заканчивается: нужны пул, два доступных IP,
группа портов и учётная запись. LUN создаётся через OpenStack; при подключении
тома Huawei-драйвер создаёт/использует Host, инициатор, Host Group, LUN Group
и Mapping View. Не создавайте для этих томов отдельные ручные сопоставления
ко всем compute. Это поведение проверено в
[HuaweiISCSIDriver Epoxy](https://github.com/openstack/cinder/blob/6cc666c01938b398d617391dc676bafc4a95b28c/cinder/volume/drivers/huawei/huawei_driver.py#L81-L120).

Продолжение: [настройка Kolla-Ansible, Cinder и multipath](CINDER_MULTIPATH_KOLLA_2025_1.md).

## Основание для команд CLI

Синтаксис проверен по файлам `disk_domain`, `storage_pool`, `port`, `port_group`,
`user`, `system`, `disk` из `/crash/permitdir/ISM/cli/help/` внутри исходного
VMDK. Эти пути относятся к файловой системе образа при офлайн-разборе;
документ не требует редактировать их на работающем устройстве.

SHA-256 исходного VMDK:

```text
fe2114ae6022e5364d071981a7c58a0159032b16209dddfc06b7da011fa627be
```

Справка подтверждает имена команд и параметров, но не успешное выполнение
операций на конкретном экземпляре T10. Фактические ID, поддержка выбранного
RAID, доступная ёмкость и Link Up проверяются на стенде.
