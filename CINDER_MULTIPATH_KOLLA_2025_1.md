# eStor: iSCSI multipath для Cinder и Nova в Kolla-Ansible 2025.1

[Установка OVA/OVF](README.md) · [Подготовка eStor через UI или консоль Huawei](ESTOR_SETUP.md)

Это шаблон лабораторного подключения одноконтроллерного eStor T10 к OpenStack
Epoxy 2025.1. Адреса ещё не согласованы; замените IP, IQN и имя узла inventory.
Проверены исходники Cinder и Kolla-Ansible ветки `stable/2025.1`, указанные
в конце документа. Развёртывание, REST-вызовы к эмулятору, создание томов
и отказ пути на стенде не выполнялись.

Образ содержит Dorado V3. Он позволяет исследовать подключение по iSCSI,
но не подтверждает совместимость с Dorado V6. Два пути к одной ВМ хранилища
не обеспечивают переживание отказа этой ВМ или её ESXi-хоста.

## 1. Сеть и объекты хранилища

| Назначение | eStor | Пример compute01 | Пример storage01 |
| --- | --- | --- | --- |
| Управление / REST | `192.168.10.50` | — | Доступ к TCP 8088 |
| iSCSI A | `172.20.10.50/24` | `172.20.10.11/24` | `172.20.10.21/24` |
| iSCSI B | `172.20.20.50/24` | `172.20.20.11/24` | `172.20.20.21/24` |

Каждому compute и узлу `cinder-volume` нужны маршруты к обоим target IP
и TCP 3260. Это сеть узлов OpenStack; создание tenant-сети Neutron само по себе
такой доступ не предоставляет. Для проверки сетевого отказа сети A/B должны
иметь соответствующее разделение. Два IP через один общий uplink позволяют
проверить логические пути, но отказ uplink прервёт оба.

На eStor заранее подготовьте:

- storage pool `OpenStackPool`;
- два сервисных Ethernet-порта с указанными IP;
- группу портов Huawei `OpenStack-iSCSI`, содержащую оба порта;
- пользователя `cinder` с правами администратора и доступом RESTful.

Пошаговые действия, включая точные команды CLI T10:
[ESTOR_SETUP.md](ESTOR_SETUP.md).

В этом примере backend eStor обслуживает **один активный `cinder-volume`**.
Проверенный Huawei-драйвер не объявляет `SUPPORTS_ACTIVE_ACTIVE=True`.
Не запускайте один backend на нескольких узлах, предполагая, что multipath
даёт поддержку Cinder active-active. Если существующий Cinder уже работает
в HA, размещение отдельного backend и `cinder_cluster_name` нужно согласовать
с конфигурацией его сервисов; отключение precheck само по себе это не решает.
См. [требования Kolla к Cinder HA](https://docs.openstack.org/kolla-ansible/2025.1/reference/storage/cinder-guide.html#ha).

## 2. Включить iSCSI и multipath в Kolla

На deploy-узле добавьте в `/etc/kolla/globals.yml`, сохранив остальные настройки:

```yaml
enable_cinder: "yes"
enable_cinder_backend_iscsi: "yes"
enable_multipathd: "yes"

cinder_backend_huawei: "yes"
cinder_backend_huawei_xml_files:
  - cinder_huawei_estor.xml
```

Название переменной — `cinder_backend_huawei`, без префикса `enable_`.
Не отключайте существующие Ceph/LVM/другие backend. Для внешнего iSCSI
`enable_cinder_backend_iscsi` включает необходимый путь настройки `iscsid`;
создавать локальную LVM-группу `cinder-volumes` для Huawei не требуется.
См. [Cinder в Kolla-Ansible](https://docs.openstack.org/kolla-ansible/2025.1/reference/storage/cinder-guide.html)
и [переменные проверенной ветки](https://github.com/openstack/kolla-ansible/blob/7222f0fb4b78ab668834a7d548661ea5c76bd099/ansible/group_vars/all.yml).

Проверьте, что нужные узлы включены в группы inventory `iscsid` и `multipathd`.
В стандартном inventory они охватывают compute/storage; в собственном
inventory это может быть изменено.

## 3. Получить IQN узлов

На каждом compute и узле Cinder, когда контейнер `iscsid` уже развёрнут:

```bash
docker exec iscsid cat /etc/iscsi/initiatorname.iscsi
```

Сохраните значение после `InitiatorName=`. У разных узлов должны быть разные
IQN. В XML нужна запись для каждого узла, с которого том будет подключаться.
Если `cinder-backup` использует локальное iSCSI-подключение, учитывается и его
узел. Если `iscsid` ещё отсутствует, сначала разверните сервис iSCSI штатной
ролью Kolla, затем получите сгенерированный IQN.

## 4. Huawei XML

На deploy-узле создайте файл
`/etc/kolla/config/cinder/cinder_huawei_estor.xml`:

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

`Product=Dorado` — имя семейства для драйвера, применяемое и к Dorado V3.
Пул должен существовать; его имя и имя группы портов должны совпадать с UI/CLI.
Для Dorado используется `Thin`.

В проверенном коде инициатор ищется по **точному IQN**. `Name="*"` не заменяет
перечисление узлов. Для нескольких путей нужен `TargetPortGroup`: без него
резервный выбор из `DefaultTargetIP` использует только первый адрес, даже если
указать несколько через точку с запятой.
См. [реализацию выбора target IP](https://github.com/openstack/cinder/blob/6cc666c01938b398d617391dc676bafc4a95b28c/cinder/volume/drivers/huawei/rest_client.py#L1287-L1359).

Ограничьте доступ к исходному XML пользователем, запускающим Ansible.
Символы пароля `&` и `<` в XML записываются как `&amp;` и `&lt;`.
Реальный пароль в Git не сохраняйте. В примере используется лабораторный
режим SSL по умолчанию драйвера: без `SSLCertVerify` проверка сертификата
отключена. Для проверяемого TLS настройте `SSLCertVerify=true`, доверенный CA
и доступный внутри контейнера `SSLCertPath`.

Kolla копирует XML в `/etc/cinder/cinder_huawei_estor.xml` внутри контейнера
с владельцем `cinder` и правами `0600`. Не заменяйте это прямым read-only
bind mount: при первом чтении драйвер переписывает username/password
в формате `!$$$` + Base64. Это кодирование, а не шифрование.
См. [доставку XML](https://github.com/openstack/kolla-ansible/blob/7222f0fb4b78ab668834a7d548661ea5c76bd099/ansible/roles/cinder/templates/cinder-volume.json.j2)
и [обработку конфигурации драйвером](https://github.com/openstack/cinder/blob/6cc666c01938b398d617391dc676bafc4a95b28c/cinder/volume/drivers/huawei/huawei_conf.py).

## 5. Backend Cinder

На deploy-узле создайте или дополните
`/etc/kolla/config/cinder/<inventory_hostname>/cinder.conf`, где
`<inventory_hostname>` — имя выбранного узла `cinder-volume` из inventory:

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

Если в `enabled_backends` уже перечислены другие секции, добавьте `estor_iscsi`
через запятую, сохранив их. Например, `rbd-1` превращается в
`rbd-1,estor_iscsi`, если существующая секция действительно называется `rbd-1`.

Host-specific файл применяется после общих override. В Kolla флаг доставки
Huawei XML сам по себе не создаёт секцию backend.
См. [порядок слияния cinder.conf](https://github.com/openstack/kolla-ansible/blob/7222f0fb4b78ab668834a7d548661ea5c76bd099/ansible/roles/cinder/tasks/config.yml).

## 6. Nova и политика multipath

В `/etc/kolla/config/nova.conf` на deploy-узле:

```ini
[libvirt]
volume_use_multipath = true
volume_enforce_multipath = true
```

В проверенной Kolla `enable_multipathd` уже выставляет `volume_use_multipath`;
override явно фиксирует требуемое поведение. `volume_enforce_multipath` требует
доступного multipathd и может остановить запуск compute при его отсутствии.
Эти флаги не гарантируют, что у LUN обнаружено два пути: это проверяется после
attach. См. [параметры Nova](https://docs.openstack.org/nova/2025.1/configuration/config.html#libvirt.volume_enforce_multipath).

Стандартный шаблон multipathd проверенной ветки содержит:

```text
defaults {
    user_friendly_names no
    find_multipaths yes
    skip_kpartx yes
}

blacklist {
}
```

Для первого подключения можно начать со штатной политики. Если нужен свой
`/etc/kolla/config/multipath.conf`, он заменит штатный файл целиком: сохраните
нужные defaults. Hardware-профиль `HUAWEI/XSG1` и ALUA-параметры подбираются
по реальным SCSI inquiry и состоянию путей; готовый профиль физического V6
нельзя считать проверенным для этого эмулятора.

### Доставка файла в контейнеры

Проверка по конкретному upstream commit показала:

| Контейнер | Доставка `multipath.conf` ролью |
| --- | --- |
| `multipathd` | Предусмотрена; пользовательский файл выбирается целиком |
| `nova_compute` | Предусмотрена в проверенной ветке `stable/2025.1` |
| `cinder_volume` | В проверенном шаблоне `config_files` такой записи нет |

Следовательно, включение `enable_multipathd` не доказывает наличие правильной
политики внутри Cinder. Проверьте свой fork и образ контейнера. Если файл
отсутствует или нужен согласованный custom-конфиг, настройте доставку
в роли Cinder: копирование выбранного источника в конфигурационный каталог
`cinder-volume` и запись назначения `/etc/multipath.conf` в `config_files`.
Это необходимая адаптация конкретной сборки, она не внедрена данным документом.
Наличие минимального файла `defaults { }` в образе тоже не доказывает
совпадение эффективной политики с демоном.

Проверьте исходники
[роли Nova](https://github.com/openstack/kolla-ansible/blob/7222f0fb4b78ab668834a7d548661ea5c76bd099/ansible/roles/nova-cell/tasks/config.yml),
[шаблона Cinder](https://github.com/openstack/kolla-ansible/blob/7222f0fb4b78ab668834a7d548661ea5c76bd099/ansible/roles/cinder/templates/cinder-volume.json.j2)
и [шаблона multipathd](https://github.com/openstack/kolla-ansible/blob/7222f0fb4b78ab668834a7d548661ea5c76bd099/ansible/roles/multipathd/templates/multipath.conf.j2).

Glance требует отдельной настройки локального multipath, если его Cinder store
сам подключает iSCSI-тома. Для Glance file/RBD/S3/Swift из этой схемы такое
требование не следует.

## 7. Применение

До применения замените все `REPLACE_WITH_*`, согласуйте адреса/маршруты,
имя узла, размещение backend и доставку `multipath.conf`. Если `iscsid`
ещё не развёрнут, сначала разверните его ролью Kolla и заполните реальные IQN.

Команды выполняются вручную на deploy-узле из окружения вашей Kolla-Ansible.
`/path/to/inventory` заменяется настоящим путём. Сначала сформируйте конфиги:

```bash
kolla-ansible genconfig -i /path/to/inventory --tags iscsi,multipathd,cinder,nova
```

Проверьте полученные `cinder.conf`, `nova.conf`, XML и `config.json` соответствующих
сервисов. XML содержит секреты: не публикуйте его содержимое в журнале проверки.
Затем выполните prechecks:

```bash
kolla-ansible prechecks -i /path/to/inventory --tags iscsi,multipathd,cinder,nova
```

После устранения замечаний примените настройки в согласованное окно:

```bash
kolla-ansible reconfigure -i /path/to/inventory --tags iscsi,multipathd,cinder,nova
```

`reconfigure` может пересоздавать/перезапускать затронутые контейнеры.
Для нового развёртывания используйте принятый в вашем окружении цикл `deploy`.
Команды этого раздела при подготовке документа не запускались.

## 8. Проверка backend и тома

На узле с настроенным OpenStack CLI и загруженными учётными данными:

```bash
openstack volume service list
```

```bash
openstack volume pool list --detail
```

Проверьте `cinder-volume` с backend `estor_iscsi`: сервис должен быть `up`,
пул — присутствовать. Затем создайте новый лабораторный тип и том:

```bash
openstack volume type create estor
```

```bash
openstack volume type set --property volume_backend_name=estor_iscsi estor
```

```bash
openstack volume create --type estor --size 1 estor-test
```

```bash
openstack volume show estor-test
```

После перехода тома в `available` подключите его к своей тестовой ВМ;
`TEST_VM` замените её именем или UUID:

```bash
openstack server add volume TEST_VM estor-test
```

На compute, где работает ВМ, проверьте сеансы и пути:

```bash
docker exec iscsid iscsiadm -m session
```

```bash
docker exec multipathd multipath -ll
```

```bash
docker exec nova_compute multipath -ll
```

Файл политики внутри сервисов проверяется на их соответствующих узлах:

```bash
docker exec nova_compute cat /etc/multipath.conf
```

```bash
docker exec cinder_volume cat /etc/multipath.conf
```

Требуемый результат — один WWID тестового LUN, один DM multipath-device
и два исправных SCSI-пути через разные target IP. Один лишь статус
`in-use` у тома этого не доказывает. Не считайте отсутствие путей в
`cinder_volume` после обычного attach к ВМ ошибкой: такой том подключён
на compute. Для проверки Cinder image-transfer нужна отдельная операция
создания тома из образа и наблюдение путей на узле Cinder во время копирования.

## 9. Проверка потери одного пути

1. Используйте только новый тестовый том; сопоставьте его UUID, WWID и устройство
   в госте. Организуйте I/O на тестовой файловой системе и зафиксируйте результат.
2. Убедитесь, что оба пути исправны.
3. Отключите ровно один выбранный сервисный адаптер ВМ eStor в ESXi,
   предварительно сверив MAC. Management и второй сервисный адаптер остаются
   подключены.
4. Проверьте потерю одного пути, сохранение multipath-device и продолжение I/O.
5. Подключите адаптер обратно и дождитесь восстановления пути.
6. Сохраните состояние путей и журнал I/O до, во время и после отказа.

Тест не доказывает отказоустойчивость контроллеров, ESXi или физической СХД.
Команды массового logout, удаления mapping и очистки multipath для этого
теста не требуются.

## 10. Если подключение не получилось

| Симптом | Что проверить сначала |
| --- | --- |
| Backend `down` | Доступ к management:8088, права REST, пароль, имя пула, читаемость и возможность записи XML внутри Cinder |
| Том создаётся, attach не работает | Доступ с конкретного compute к обоим IP:3260, его реальный IQN и журнал `cinder-volume` |
| Только один путь | Точное совпадение IQN в XML, `TargetPortGroup`, оба участника группы и Link Up, multipath в Nova |
| Два SCSI-диска без DM-карты | Работа multipathd, доступ к нему из контейнера, политика blacklist/find_multipaths, WWID обоих путей |
| Нет `/etc/multipath.conf` в Cinder | Доставка файла в вашей версии роли/образа; само по себе это ещё не устанавливает причину ошибки I/O |
| Пул/порт не создаётся в eStor | Справка именно T10, реальные ID, свободные диски, доступная ёмкость и ограничения модели |

## Проверенные версии и источники

- Cinder `stable/2025.1`: `6cc666c01938b398d617391dc676bafc4a95b28c`.
- Kolla-Ansible `stable/2025.1`: `7222f0fb4b78ab668834a7d548661ea5c76bd099`.
- [Huawei driver, OpenStack 2025.1](https://docs.openstack.org/cinder/2025.1/configuration/block-storage/drivers/huawei-storage-driver.html).
- [Kolla-Ansible Cinder guide, 2025.1](https://docs.openstack.org/kolla-ansible/2025.1/reference/storage/cinder-guide.html).

Это срез веток с исправлениями после первоначального выпуска Epoxy;
собственный fork или ранняя версия Kolla 20.x могут отличаться.
