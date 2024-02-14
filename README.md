# 1.Базовая настройка
- a) Настройте имена устройств согласно топологии
    - a. Используйте полное доменное имя
- b) Сконфигурируйте адреса устройств на свое усмотрение. Для офиса HQ выделена сеть 10.0.10.0/24, для офиса BR выделена сеть 10.0.20.0/24. Данные сети необходимо разделить на подсети для каждого vlan.
- c) На SRV-HQ и SRV-BR, создайте пользователя sshuser с паролем P@ssw0rd
    - a. Пользователь sshuser должен иметь возможность запуска утилиты sudo без дополнительной аутентификации.
    - b. Запретите парольную аутентификацию. Аутентификация пользователя sshuser должна происходить только при помощи ключей.
    - c. Измените стандартный ssh порт на 2023.
    - d. На CLI-HQ сконфигурируйте клиент для автоматического подключения к SRV-HQ и SRV-BR под пользователем sshuser. При подключении автоматически должен выбираться корректный порт. Создайте пользователя sshuser на CLI-HQ для обеспечения такого сетевого доступа.
### Выполнение:
### Назначаем полные доменные имена на устройства:
#### RTR-HQ:

Имя пользователя на **vESR** по умолчанию: **admin** с паролем **password**

- После первого входа на уст-во **vESR** - необходимо задать новый пароль для пользователя **admin**:

```
password P@ssw0rd
commit
confirm
```

![Pasted image 20240212185943](https://github.com/e1mky/dms/assets/102690802/2999626b-00b8-4685-8422-2e794d16b216)


- Задаём полное доменное имя (FQDN):

```
configure terminal
hostname rtr-hq.company.prof
do commit
do confirm
```

- Выполняем подключение к **ISP** для доступа в сеть **Интернет**:
    - Задаём IP-адрес из сети ISP-HQ - 11.11.11.0/24

```
interface gi1/0/1
ip address 11.11.11.11/24
description ISP
exit
```

- - Задаём параметры для DNS:

```
domain name company.prof
domain name-server 11.11.11.1
```

- - Задаём маршрут по умолчанию (шлюз):

```
ip route 0.0.0.0/0 11.11.11.1
```

- Применяем и подтверждаем внесёные измменения:

```
do commit
do confirm
```

- Проверяем доступ до сети Интернет:

![Pasted image 20240212190008](https://github.com/e1mky/dms/assets/102690802/fb0a002e-a1e3-43ef-9286-ba3a48f45f7a)


#### RTR-BR:

Аналогично **RTR-HQ:**

- задаём пароль **P@ssw0rd** для пользователя **admin;**
- задаём полное доменное имя: **RTR-BR.company.prof**;
- выполняем подключение к провайдеру **ISP** для доступа в сеть **Интернет**;
    - сеть ISP-BR: 22.22.22.0/24

Результат:
![Pasted image 20240212190028](https://github.com/e1mky/dms/assets/102690802/0e7028f8-94e2-4d3f-a0b6-5c9c3af06625)


#### SW-HQ | SRV-HQ | CLI-HQ | SW-BR | SRV-BR | CLI-BR:

- на всех остальных устройствах выполняем команду:

```
hostnamectl set-hostname <FQDN>; exec bash
```

где: 
<FQDN> - полное доменное имя, например:
    - sw-hq.company.prof
    - srv-hq.company.prof
    - cli-hq.company.prof
    - sw-br.company.prof
    - srv-br.company.prof
    - cli-br.company.prof


Конфигурируем адреса устройств на свое усмотрение. Для офиса HQ выделена сеть 10.0.10.0/24, для офиса BR выделена сеть 10.0.20.0/24. Данные сети необходимо разделить на подсети для каждого vlan.

Требования к vlan:

- В обоих офисах **серверы** должны находиться во **vlan100**, **клиенты** – во **vlan200**, **management** подсеть – во **vlan300**
- Для каждого **vlan** рассчитайте подсети, выданные для офисов. Количество хостов в каждой подсети **не должно превышать 30-ти**

Таким образом, получаем примерно следующую таблицу разделения сетей на подсети:
### HQ

![Pasted image 20240212190504](https://github.com/e1mky/dms/assets/102690802/1c4a7078-3583-4909-9f20-1abcccf0b615)


### BR

![Pasted image 20240212190636](https://github.com/e1mky/dms/assets/102690802/bb9511a8-606a-4c4e-a3d3-6bf2614bccb8)


Таблица адресации устройств - выглядит следующим образом:

![Pasted image 20240212190722](https://github.com/e1mky/dms/assets/102690802/53d5f935-d2c5-46b5-bbc2-fa4aa2b5c9e9)


#### RTR-HQ:

- **gi1/0/1**  - интерфейс в сторону **ISP**(сконфигурирован для доступа в сеть Интернет ранее);
- **gi1/0/2** - интерфейс в сторону **SW-HQ** (в локальную сеть офиса **HQ**)

Поскольку **SW-HQ** - выступает в роле коммутатора, а сеть разделена на подстети **vlan,** то **RTR-HQ -** будет осуществлять маршрутизацию между **VLAN-ами:**

- для реализации маршрутизации между VLAN-ами - необходимо создать и настроить **sub-interface** (подинтерфейсы):
    - для серверной подсети - **vlan100**:

```
interface gi1/0/2.100
ip firewall disable
description vlan100
ip address 10.0.10.1/27
exit
```

- - для клиентской подсети - **vlan200:**

`interface gi1/0/2.200 ip firewall disable description vlan200 ip address 10.0.10.33/27 exit`

- - для подсети управления - **vlan300**:
```
interface gi1/0/2.300 
ip firewall disable 
description vlan300 
ip address 10.0.10.65/27 
exit
```
- - применяем и подтверждаем внесённые изменения:

```
do commit
do confirm
```

- Проверяем:
![Pasted image 20240212190753](https://github.com/e1mky/dms/assets/102690802/c33471ff-111b-4f23-93f3-3c83ed9ec44c)


![Pasted image 20240212190758](https://github.com/e1mky/dms/assets/102690802/b98a2c40-42ab-4a25-a494-1866d67b3376)


#### RTR-BR:

Аналогично **RTR-HQ:**

- для серверной подсети - **vlan100 - gi1/0/2.100 - 10.0.20.1/27;**
- для клиентской подсети - **vlan200 - gi1/0/2.200 - 10.0.20.33/27;**
- для подсети управления - **vlan300 - gi1/0/2.300 - 10.0.20.65/27**

Результат
![Pasted image 20240212190810](https://github.com/e1mky/dms/assets/102690802/bc9e9237-17cc-4ad3-93c2-45f1b0a7330b)


![Pasted image 20240212190814](https://github.com/e1mky/dms/assets/102690802/c130fed4-d896-4a4d-b46c-357ede18d629)


На данном этапе назначить адреса сможем ещё только на **SRV-HQ** и **SRV-BR**, т.к. **CLI-HQ** и **CLI-BR** получат адреса после настройки **DHCP-сервера**, а на **SW-HQ** и **SW-BR** для назначения адресов - необходимо настроить интерфейс управления средствами **OpenvSwitch** и уже на него назначать адрес из соответствующего vlan (vlan300)

#### SRV-HQ:

В качестве сетевой подсистемы использует **etcnet**

- назначаем **ipv4 - адрес:**

```
echo 10.0.10.2/27 > /etc/net/ifaces/ens33/ipv4address
```

- назначаем адрес **шлюза:**

```
echo default via 10.0.10.1 > /etc/net/ifaces/ens33/ipv4route
```

- перезапускаем службу **network:**

```
systemctl restart network
```

- Проверяем:

![Pasted image 20240212190833](https://github.com/e1mky/dms/assets/102690802/61c00f4e-c05e-45b8-ab3a-edc9ec3a28e6)


_На данном этапе шлюз всеравно будет не доступен, т.к. не настроена коммутация на **SW-HQ**, а весь трафик входящий на **RTR-HQ** блокируется по умолчанию средствами **firewall**_

#### SRV-BR:

Аналогично  **SRV-HQ,** результат:

![Pasted image 20240212190914](https://github.com/e1mky/dms/assets/102690802/a55037c2-6f5e-44fe-bb79-0512e7397c16)


### На SRV-HQ и SRV-BR, создаём пользователя sshuser с паролем P@ssw0rd

#### SRV-HQ | SRV-BR:

- Создаём пользователя **sshuser:**

```
useradd sshuser -m -U -s /bin/bash
```

где:

**-m** - если домашнего каталога пользователя не существует, то он будет создан;

**-U** - создаётся группа с тем же именем, что и у пользователя, и добавляет пользователя в эту группу;

**-s /bin/bash** - задаётся имя регистрационной оболочки пользователя

- Задаём пароль **P@ssw0rd** для пользователя **sshuser**:

```
passwd sshuser
```

- - после чего **дважды** задаём необходимы пароль (**P@ssw0rd**) для пользователя:

![Pasted image 20240212190934](https://github.com/e1mky/dms/assets/102690802/a6d97046-f5a2-4214-ba00-9d5a8788c0c2)


Реализуем возможность запуска утилиты **sudo** без дополнительной аутентификации для пользователя **sshuser**:

- В Альт sudo используется фреймворк control, который задаёт права на выполнение команды sudo. С его помощью можно дать или отнять права на использование команды sudo.
    - Штатное состояние политики:

![Pasted image 20240212190951](https://github.com/e1mky/dms/assets/102690802/b53b09b7-6da7-4684-bc9a-9365e8594791)


_**wheelonly** — только пользователи из группы **wheel** имеют право получить доступ к команде **/usr/bin/sudo**_

- добавляем пользователя **sshuser** в группу **wheel**:

```
usermod -aG wheel sshuser
```

- настраиваем **sudo**, правим конфигурационный файл **/etc/sudoers**:

```
vim /etc/sudoers
```

1. разрешаем использовать **sudo** для всех пользователей входящих в группу **wheel**:
2. разрешаем использовать **sudo** пользователю **sshuser** без дополнительной аутентификации:

![Pasted image 20240212191009](https://github.com/e1mky/dms/assets/102690802/cab85dd8-d781-4842-a9b2-7a1045cccde6)


- Проверяем:
    - Выполняем вход под пользователем **sshuser** и попытаться выполнить **sudo -i** или иную другую команду требующую повышения привелегий до **root**:
        - SRV-HQ:

![Pasted image 20240212191031](https://github.com/e1mky/dms/assets/102690802/8a635327-5713-4c8c-bc4f-f481f50b6df7)


- SRV-BR:
![Pasted image 20240212191043](https://github.com/e1mky/dms/assets/102690802/fffd8f2d-e0ee-4f75-bd16-df4e75722444)

_Вся последующая настройка SSH будет выполнена позднее после получения необходимых сетевых параметров для **CLI-HQ** по **DHCP**, а также сгенерированы и переданы необходимые ключи_

# 2. Настройка динамической трансляции адресов
5. Настройка динамической трансляции адресов

- a) Настройте динамическую трансляцию адресов для обоих офисов. Доступ к интернету необходимо разрешить со всех устройств.

### Выполнение:

Необходимо настройить **NAT** на маршрутизаторах **RTR-HQ** и **RTR-BR**

#### **RTR-HQ**:

- Создадим зону безопасности:
    - **public** - для интерфейса смотрящего в сеть ISP (**gi1/0/1**);

```
security zone public
exit
```

- Добавляем интерфейс в соответствующую зону безопасности:

```
interface gi1/0/1
security-zone public
exit
```

- Применяем и подтверждаем внесённые изменения:

```
do commit
do confirm
```

- Проверяем:

![Pasted image 20240212191240](https://github.com/e1mky/dms/assets/102690802/d94fd51d-8ba8-4df3-b525-8bafcf6a572e)


Для конфигурирования **NAT** и настройки правил зон безопасности потребуется создать профиль адресов подсетей, включающий адреса, которым разрешен выход в публичную сеть, и профиль адреса публичной сети «WAN»

- Создаём профиль **COMPANY** для указания подсетей локальной сети офиса **HQ**:

```
object-group network COMPANY
ip address-range 10.0.10.1-10.0.10.254
exit
```

- Создаём профиль **WAN** и указываем внешний IP-адрес (11.11.11.11):

```
object-group network WAN
ip address-range 11.11.11.11
exit
```

- Конфигурируем сервис **NAT**. Первым шагом задаётся **IP-адрес** публичной сети (**WAN**), используемых для сервиса **NAT**:

```
nat sourсe
pool WAN
ip address-range 11.11.11.11
exit
```

- Создаём набор правил **SNAT**. В атрибутах набора укажем, что правила применяются только для пакетов, направляющихся в публичную сеть – в зону **public**. Правила включают проверку адреса источника данных на принадлежность к пулу **COMPANY**:
```
ruleset SNAT
to zone public 
rule 1 match
source-address COMPANY 
action source-nat pool WAN 
enable 
exit 
exit
```
- Применяем и подтверждаем внесённые изменения:

```
do commit
do confirm
```

- Проверяем:
    - т.к. на данном этапе ещё не настройена коммутация на **SW-HQ** - проверить работоспособность **NAT** можно назначив средствами **iproute2** временно на интерфейс **SW-HQ** на интерфейс,смотрящий в сторону **RTR-HQ** - тегированный подинтерфейс с IP-адресом из подсети для **vlan300:**

```
ip link add link ens33 name ens33.300 type vlan id 300
ip link set dev ens33.300 up
ip addr add 10.0.10.66/27 dev ens33.300
ip route add 0.0.0.0/0 via 10.0.10.65
```

- - затем проверяем доступ в сеть Интернет:

![Pasted image 20240212191312](https://github.com/e1mky/dms/assets/102690802/47760ea6-fff2-425a-9334-4f2e36b693b7)


- также проверяем таблицу **NAT** на **RTR-HQ**:

![Pasted image 20240212191324](https://github.com/e1mky/dms/assets/102690802/44219023-6b96-4d7f-a7bd-cbe1c1a6883f)



_После чего отправляем в перезугрузку SW-HQ, т.к. всё что было назначено средствами iproute2 не пригодится_

#### RTR-BR:

Аналогично  **RTR-HQ,** результат:

![Pasted image 20240212191341](https://github.com/e1mky/dms/assets/102690802/56c4ce1a-19f2-46c6-9d7f-2fbd56eef0ab)


![Pasted image 20240212191348](https://github.com/e1mky/dms/assets/102690802/de3048cf-9f83-45c6-8545-a4efcf4a2034)

# 3. Настройка коммутации

3. Настройка коммутации

- a) В качестве коммутаторов используются SW-HQ и SW-BR.
- b) В обоих офисах серверы должны находиться во vlan100, клиенты – во vlan200, management подсеть – во vlan300.
- c) Создайте management интерфейсы на коммутаторах.
- d) Для каждого vlan рассчитайте подсети, выданные для офисов.Количество хостов в каждой подсети не должно превышать 30-ти.

### Выполнение:

#### SW-HQ:

- назначив средствами **iproute2** временно на интерфейс,смотрящий в сторону **RTR-HQ** - тегированный подинтерфейс с IP-адресом из подсети для **vlan300,** для возможности установки пакета **openvswitch**:

`ip link add link ens33 name ens33.300 type vlan id 300 ip link set dev ens33.300 up ip addr add 10.0.10.66/27 dev ens33.300 ip route add 0.0.0.0/0 via 10.0.10.65 echo nameserver 77.88.8.8 > /etc/resolv.conf`

- обновляем список пакетов и устанавливаем **openvswitch**:

```
apt-get update && apt-get install -y openvswitch
```

- включаем и добавляем в автозагрузку **openvswitch**:

```
systemctl enable --now openvswitch
```

Имеем следующие интерфейсы:

- ens33 - в сторону RTR-HQ;
- ens34 - vlan100;
- ens35 - vlan200;
- ens36 - vlan100;

Сетевая подсистема **etcnet** будет взаимодействовать с **openvswitch**, поэтому в директоии **/etc/net/ifaces/** создаём следующую структуру каталагов:

- создаём каталоги для физических интерфейсов **ens34**, **ens35** и **ens36**:

```
mkdir /etc/net/ifaces/ens3{4,5,6}
```

- создаём также каталог для мостового интерфейса с именем **ovs0**:

```
mkdir /etc/net/ifaces/ovs0
```

- создаём каталог для management интерфейса с именем **mgmt**:

```
mkdir /etc/net/ifaces/mgmt
```

![Pasted image 20240212191438](https://github.com/e1mky/dms/assets/102690802/c62e4189-29ae-4cf9-9b49-8058cc8e5013)


Описываем файлы **options** для каждого интерфейса:

- правим основной файл **options** в котором по умолчанию сказано - удалять настройки заданые через **ovs-vsctl,** т.к. через **etcnet** будет выполнено только создание **bridge** и интерфейса типа **internal** с назначением необходимого IP-адреса, а настройка функционала будет выполнена средствами **openvswitch**:

```
sed -i "s/OVS_REMOVE=yes/OVS_REMOVE=no/g" /etc/net/ifaces/default/options
```

- описываем файл **options** для создания моста с именем **ovs0**:

```
vim /etc/net/ifaces/ovs0/options
```
где:

![Pasted image 20240212191508](https://github.com/e1mky/dms/assets/102690802/bdb4edf4-9ee4-4e26-9f98-e0ca433da1ee)


**TYPE** - тип интерфейса (**bridge**);

**HOST** - интерфейсы которые будут добавлены в **bridge**;

- описываем файл **options** для создания management интерфейса с именем **mgmt**:

```
vim /etc/net/ifaces/mgmt/options
```

![Pasted image 20240212191516](https://github.com/e1mky/dms/assets/102690802/152972b9-ab34-49de-8db8-aadd869308ce)


где:

**TYPE** - тип интерфейса (**internal**);

**BOOTPROTO** - определяет как будут назначаться сетевые параметры (статически);

**CONFIG_IPV4** - определяет использовать конфигурацию протокола IPv4 или нет;

**BRIDGE** - определяет к какому мосту необходимо добавить данный интерфейс;

**VID** - определяет принадлежность интерфейса к VLAN;

- файл **options** для физических интерфейсов **ens34** и **ens35** аналогичен **ens33:**
    - т.к. необходимо просто поднять эти интерфейсы:

```
for i in 4 5 6; do cp /etc/net/ifaces/ens33/options /etc/net/ifaces/ens3$i/; done
```

- назначаем IP-адрес и шлюз на созданный management интерфейс **mgmt** согласно таблеце адресации:

```
echo 10.0.10.66/27 > /etc/net/ifaces/mgmt/ipv4address
```

```
echo default via 10.0.10.65 > /etc/net/ifaces/mgmt/ipv4route
```

- перезапускаем службу **network**:

```
systemctl restart network
```

- Проверяем:

![Pasted image 20240212191534](https://github.com/e1mky/dms/assets/102690802/e4bb27d8-c905-43b8-8aa2-8dea34b52185)


Средствами **openvswitch** настраиваем следующий функционал:

- порт **ens33** смотрящий в сторону **RTR-HQ** - делаем **trunk** и пропускаем все необходимые **VLANs**:

```
ovs-vsctl set port ens33 trunk=100,200,300
```

- порт **ens34** назначаем **тэг 100**:

```
ovs-vsctl set port ens34 tag=100
```

- порт **ens35** назначаем **тэг 200**:

```
ovs-vsctl set port ens35 tag=200
```

- порт **ens36** назначаем **тэг 100**:

```
ovs-vsctl set port ens36 tag=100
```

- включаем модуль ядра **8021q:**

```
modprobe 8021q
```

- Проверяем:

![Pasted image 20240212191555](https://github.com/e1mky/dms/assets/102690802/0afe8e59-4d1e-4cf3-b66d-274dc4eea666)


- также проверяем доступ с **SRV-HQ** в сеть Интернет:

#### SW-BR:

![Pasted image 20240212191604](https://github.com/e1mky/dms/assets/102690802/16397eb0-df93-4b1e-b303-0d6133fa7236)


Аналогично  **SW-HQ,** результат:

![Pasted image 20240212191617](https://github.com/e1mky/dms/assets/102690802/f26e27b8-0813-4033-879a-bec4f6076ed3)


- SRV-BR:

![Pasted image 20240212191628](https://github.com/e1mky/dms/assets/102690802/f40bee37-6096-4f55-9a87-7842fc7a7834)

# 4. Настройка протокола динамической конфигурации хостов
6. Настройка протокола динамической конфигурации хостов

- а) Настройте протокол динамической конфигурации хостов для устройств в подсетях CLI - RTR-HQ
    - i. Адрес сети – согласно топологии
    - ii. Адрес шлюза по умолчанию – адрес маршрутизатора RTR-HQ
    - iii. DNS-суффикс – company.prof
- b) Настройте протокол динамической конфигурации хостов для устройств в подсетях CLI RTR-BR
    - i. Адрес сети – согласно топологии
    - ii. Адрес шлюза по умолчанию – адрес маршрутизатора RTR-BR
    - iii. DNS-суффикс – company.prof

### Выполнение:

#### RTR-HQ:

- Создадим пул адресов с именем «**COMPANY-HQ**» и добавим в данный пул адресов диапазон IP-адресов для выдачи в аренду клиентам. Укажем параметры подсети, к которой принадлежит данный пул, и время аренды для выдаваемых адресов:
    - в качестве адреса **DNS-сервера** указываем адрес **SRV-HQ**:

`configure terminal ip dhcp-server pool COMPANY-HQ network 10.0.10.32/27 default-lease-time 3:00:00 address-range 10.0.10.33-10.0.10.62 excluded-address-range 10.0.10.33                       default-router 10.0.10.33  dns-server 10.0.10.2 domain-name company.prof exit`

- Включаем **DHCP-сервер**:

```
ip dhcp-server
```

- Применяем и подтверждаем внесённые изменения:

```
do commit
do confirm
```

- Проверяем параметры **DHCP-сервера**:

![Pasted image 20240212191930](https://github.com/e1mky/dms/assets/102690802/38130325-83a3-4202-be15-1c736e0b9746)


- Проверяем с **CLI-HQ** - получение сетевых параметров:

![Pasted image 20240212191947](https://github.com/e1mky/dms/assets/102690802/cd0cc128-aa65-4400-944b-9860735a8d15)


- Проверяем с **RTR-HQ**:

![Pasted image 20240212192009](https://github.com/e1mky/dms/assets/102690802/1ed41173-9dc7-4abb-8abe-96ea24f1d172)


#### RTR-BR:

Аналогично **RTR-HQ,** результат:

- Параметры:

![Pasted image 20240212192021](https://github.com/e1mky/dms/assets/102690802/1be21a4c-c0e4-4ef1-b162-7f2f174aaaef)


- **CLI-BR:**

![Pasted image 20240212192033](https://github.com/e1mky/dms/assets/102690802/b01cbd00-8237-49b4-9669-c35096553116)


- **RTR-BR:**

![Pasted image 20240212192046](https://github.com/e1mky/dms/assets/102690802/e2ffc875-e69d-4576-8944-da20730e9735)

# 5. Между маршрутизаторами RTR-HQ и RTR-BR сконфигурируйте защищенное соединение
9. Между маршрутизаторами RTR-HQ и RTR-BR сконфигурируйте защищенное соединение

- a) Все параметры на усмотрение участника.
- b) Используйте парольную аутентификацию.
- c) Обеспечьте динамическую маршрутизацию: ресурсы одного офиса должны быть доступны из другого офиса
- d) Для обеспечения динамической маршрутизации используйте протокол OSPF

### Выполнение:

Для выполнения данного задания будет построен **GRE**-туннель с последующим шифрованием по средством **IPSec:**

#### RTR-HQ:

- Создадим туннель **gre 1**:
    - поместив его в зону безопусности **public;**
    - указав внешний IP-адрес **RTR-HQ**;
    - указав внешний IP-адрес **RTR-BR;**
    - указав IP-адрес туннельного интерфейса **gre 1:**

Explain

`tunnel gre 1   ttl 16   ip firewall disable   local address 11.11.11.11   remote address 22.22.22.22   ip address 172.16.1.1/30   enable exit`

- Также для проверки можно разрешить **ICMP** из зоны **public** в зону **self** (т.е. на RTR-HQ) для проверки связности туннельного интерфейса:
    - И разрешаем трафик **GRE**:

Explain

`security zone-pair public self   rule 1     description "ICMP"     action permit     match protocol icmp     enable   exit   rule 2     description "GRE"     action permit     match protocol gre     enable   exit exit`

- Применияем и подтверждаем внесённые изменения:

```
do commit
do confirm
```

- Проверяем:

![Pasted image 20240212192114](https://github.com/e1mky/dms/assets/102690802/12c4a370-41a0-4983-bc06-a025d048ffab)


#### RTR-BR:

Аналогично **RTR-HQ**, результат:
![Pasted image 20240212192137](https://github.com/e1mky/dms/assets/102690802/5dde80de-9952-4ffe-9022-19b44b431dc6)


Проверяем (работоспособность) связность по туннельному интерфейсу:

- RTR-HQ -> RTR-BR:

![Pasted image 20240212192153](https://github.com/e1mky/dms/assets/102690802/d4522498-0d53-4af4-86f3-d819a095f52a)


- RTR-BR -> RTR-HQ:

![Pasted image 20240212192205](https://github.com/e1mky/dms/assets/102690802/a3039ff3-344d-4c31-aa93-b4a6b110acd2)


#### Выполняем шифрование туннеля средствами **IPsec**

**RTR-HQ | RTR-BR:**

- Создадим профиль протокола **IKE**. В профиле укажем группу **Диффи-Хэллмана 2**, алгоритм шифрования **AES 128 bit**, алгоритм аутентификации **MD5**. Данные параметры безопасности используются для защиты **IKE-соединения**:

`security ike proposal ike_prop1   authentication algorithm md5   encryption algorithm aes128   dh-group 2 exit`

- Создадим политику протокола **IKE**. В политике указывается список профилей протокола **IKE**, по которым могут согласовываться узлы и ключ аутентификации:

```
security ike policy ike_pol1
  pre-shared-key ascii-text P@ssw0rd
  proposal ike_prop1
exit
```

- Создадим шлюз протокола IKE. В данном профиле указывается GRE-туннель, политика, версия протокола и режим перенаправления трафика в туннель:
    - **RTR-HQ:**

`security ike gateway ike_gw1   ike-policy ike_pol1   local address 11.11.11.11   local network 11.11.11.11/32 protocol gre    remote address 22.22.22.22   remote network 22.22.22.22/32 protocol gre    mode policy-based exit`

- - **RTR-BR:**

`security ike gateway ike_gw1   ike-policy ike_pol1   local address 22.22.22.22   local network 22.22.22.22/32 protocol gre    remote address 11.11.11.11   remote network 11.11.11.11/32 protocol gre    mode policy-based exit`

- Создадим профиль параметров безопасности для IPsec-туннеля. В профиле укажем группу Диффи-Хэллмана 2, алгоритм шифрования AES 128 bit, алгоритм аутентификации MD5. Данные параметры безопасности используются для защиты IPsec-туннеля:

`security ipsec proposal ipsec_prop1   authentication algorithm md5   encryption algorithm aes128   pfs dh-group 2 exit`

- Создадим политику для IPsec-туннеля. В политике указывается список профилей IPsec-туннеля, по которым могут согласовываться узлы.

```
security ipsec policy ipsec_pol1
  proposal ipsec_prop1
exit
```

- Создадим IPsec VPN. В VPN указывается шлюз IKE-протокола, политика IP sec-туннеля, режим обмена ключами и способ установления соединения. После ввода всех параметров включим туннель командой enable.

`security ipsec vpn ipsec1   ike establish-tunnel route   ike gateway ike_gw1   ike ipsec-policy ipsec_pol1   enable exit`

- Настраиваем **firewall:**

`security zone-pair public self   rule 3     description "ESP"     action permit     match protocol esp     enable   exit   rule 4     description "AH"     action permit     match protocol ah     enable   exit exit`

- Применияем и подтверждаем внесённые изменения:

```
do commit
do confirm
```

- Проверяем:
    - **RTR-HQ:**

![Pasted image 20240212192235](https://github.com/e1mky/dms/assets/102690802/2f18503c-0644-4cfe-b5b5-fed578f7ab69)


![Pasted image 20240212192242](https://github.com/e1mky/dms/assets/102690802/03bc018a-0a70-48ec-8863-23870d084cf5)


- **RTR-BR:**

![Pasted image 20240212192253](https://github.com/e1mky/dms/assets/102690802/ff66fc5e-e8c6-42e9-b72d-4d0257eabc5b)


![Pasted image 20240212192304](https://github.com/e1mky/dms/assets/102690802/1e8f1a68-9628-4d00-94fb-0b15c4f36e13)


- **tcpdump -i ens34** (на ISP):

![Pasted image 20240212192320](https://github.com/e1mky/dms/assets/102690802/a7f32186-468f-4a0c-a520-fce2756a8e85)


Таким образом, **IPsec работает** - корректно

#### Настраиваем динамическую маршрутизацию для связи локальных сетей через туннельный интерфейс:

#### RTR-HQ:

- Создадим **OSPF-процесс** с идентификатором **1** и перейдём в режим конфигурирования протокола **OSPF**
    - Создадим и включим требуемую область
    - Включим OSPF-процесс

`router ospf 1   area 0.0.0.0     enable   exit   enable exit`

- Для установления соседства с другими маршрутизаторами привяжем их к OSPF-процессу и области.
    - Далее включим на интерфейсе маршрутизацию по протоколу OSPF:
        - интерфейс **gre 1** - для установления соседства
        - интерфейс **gi1/0/2.(100,200,300)** - для обявления локальных сетей

`tunnel gre 1   ip ospf instance 1   ip ospf exit  interface gigabitethernet 1/0/2.100   ip ospf instance 1   ip ospf exit  interface gigabitethernet 1/0/2.200   ip ospf instance 1   ip ospf exit  interface gigabitethernet 1/0/2.300   ip ospf instance 1   ip ospf exit`

- Разрешаем трафик **OSPF**:

`security zone-pair public self   rule 5     description "OSPF"     action permit     match protocol ospf     enable   exit exit`

- Применияем и подтверждаем внесённые изменения:

```
do commit
do confirm
```

#### RTR-BR:

Аналогично **RTR-HQ,** вроверяем:

- Установление соседства:
    - **RTR-HQ:**

![Pasted image 20240212192347](https://github.com/e1mky/dms/assets/102690802/cd2c67aa-e442-4482-b93b-282d0624dc14)


- **RTR-BR:**

![Pasted image 20240212192402](https://github.com/e1mky/dms/assets/102690802/b449a47c-810a-4a96-8c92-e45b69bf61ff)


- Таблицу маршрутизации:
    - **RTR-HQ:**

![Pasted image 20240212192425](https://github.com/e1mky/dms/assets/102690802/a518615b-6b1d-4cbd-86b9-884af44d05b2)


- **RTR-BR:**

![Pasted image 20240212192439](https://github.com/e1mky/dms/assets/102690802/a95a1949-81f8-4995-8fcd-7dd36e7a4566)

# 5.1 Базовая настройка - доработка
Доделать следующие пункты:

- b. Запретите парольную аутентификацию. Аутентификация пользователя sshuser должна происходить только при помощи ключей.
- c. Измените стандартный ssh порт на 2023.
- d. На CLI-HQ сконфигурируйте клиент для автоматического подключения к SRV-HQ и SRV-BR под пользователем sshuser. При подключении автоматически должен выбираться корректный порт. Создайте пользователя sshuser на CLI-HQ для обеспечения такого сетевого доступа.

### Выполнение:

#### CLI-HQ:

- Создаём пользователя **sshuser** с паролем **P@ssw0rd**:
    - открываем **Меню** далее **Администрирование,** затем **Центр управления системой**:

![Pasted image 20240212192518](https://github.com/e1mky/dms/assets/102690802/00222385-b9a3-4638-aa8c-e5177529cf22)


- вводим пароль пользователя **root** нажимаем **ОК**:

![Pasted image 20240212192531](https://github.com/e1mky/dms/assets/102690802/1b97a41a-5973-4be7-9806-b02cafb3d46b)


- в разделе **Пользователя** выбираем **Локальные учётные записи**:

![Pasted image 20240212192547](https://github.com/e1mky/dms/assets/102690802/dc92142f-55c7-4c50-a933-58735a40e6a1)


- вводим имя пользователя **sshuser** нажимаем **Создать**:

![Pasted image 20240212192600](https://github.com/e1mky/dms/assets/102690802/39170ee3-b30d-48ac-be72-261e44f25dc9)


- задаём **пароль** - **P@ssw0rd** для пользователя **sshuser** и нажимаем применить:

![Pasted image 20240212192613](https://github.com/e1mky/dms/assets/102690802/142c8b90-b102-47a6-b525-510e08151b50)


- Выполняем вход из под пользователя **sshuser**:

![Pasted image 20240212192625](https://github.com/e1mky/dms/assets/102690802/3ed5b6f6-e303-4559-a90c-e16107c467cf)


![Pasted image 20240212192638](https://github.com/e1mky/dms/assets/102690802/191c16f9-0fd6-4448-a784-7b2e9b18527a)


- генерируем ключевую пару для доступа по **ssh**:

```
ssh-keygen -t rsa
```

![Pasted image 20240212192651](https://github.com/e1mky/dms/assets/102690802/25961fa8-9b63-4f22-847a-0024f03121c6)


- настраиваем клиент для автоматического подключения к **SRV-HQ** и **SRV-BR** под пользователем **sshuser:**

```
vim .ssh/config
```

![Pasted image 20240212192707](https://github.com/e1mky/dms/assets/102690802/619fa9c3-b4b9-4efc-aaab-067f1c1a9b5b)

где:

**Host** - имя для подключения, таким образом будет достаточно выполнить команду: _**ssh srv-hq.company.prof**_ **-** для подключения к **SRV-HQ** под пользователем **sshuser** используя **2023** порт

- передаём публичный ключ на **SRV-HQ**:

```
ssh-copy-id sshuser@10.0.10.2
```

![Pasted image 20240212192723](https://github.com/e1mky/dms/assets/102690802/64758a79-cb80-4d69-8004-20d00d87f4d2)


### SRV-HQ:

в конфигурационной файле **/etc/openssh/sshd_config** - правим параметры в соответствии с заданием:

```
vim /etc/openssh/sshd_config
```

- - меняем стандартный порт (22) на порт **2023**;

![Pasted image 20240212192742](https://github.com/e1mky/dms/assets/102690802/0dd0afc4-d4b9-4785-bc76-710648444ad8)


- разрешаем аутентификацию по **ключу**:
- запрещаем **парольную аутентификацию**;

![Pasted image 20240212192756](https://github.com/e1mky/dms/assets/102690802/c7c02ac6-d6fd-4193-b929-ea3f7e10c875)


- Перезапускаем службу **sshd**:

```
systemctl restart sshd
```

- Проверяем:
    - работа **ssh** на порту **2023**:

![Pasted image 20240212192811](https://github.com/e1mky/dms/assets/102690802/e86967dc-6cc3-4bee-ae95-cda819e1fb7e)


	- подключение с клиента:

![Pasted image 20240212192828](https://github.com/e1mky/dms/assets/102690802/8ee806c0-ae06-4826-a636-1d991eda87d1)


- передаём публичный ключ на **SRV-BR**:

```
ssh-copy-id sshuser@10.0.20.2
```

![Pasted image 20240212192946](https://github.com/e1mky/dms/assets/102690802/b55baf37-98c0-4dcb-923e-285dcfe348f8)


### SRV-BR:

Аналогично **SRV-HQ,** результат:

![Pasted image 20240212192957](https://github.com/e1mky/dms/assets/102690802/fbbeac01-c94e-4ab4-b9e0-bd781ac7404d)

# 6. Настройка дисковой подсистемы
2. Настройка дисковой подсистемы

- a) На SRV-HQ настройте зеркалируемый LVM том
    - a. Используйте два неразмеченных жестких диска.
    - b. Настройте автоматическое монтирование логического тома.
    - c. Точка монтирования /opt/data.
- b) На SRV-BR сконфигурируйте stripped LVM том.
    - a. Используйте два неразмеченных жестких диска.
    - b. Настройте автоматическое монтирование тома.
    - c. Обеспечьте шифрование тома средствами dm-crypt. Диск должен монтироваться при загрузке ОС без запроса пароля.
    - d. Точка монтирования /opt/data.

### Выполнение:

#### SRV-HQ:

- Используем два неразмеченных жёстких диска: **sdb** и **sdc:**

![Pasted image 20240212193041](https://github.com/e1mky/dms/assets/102690802/5215ca5b-6098-428e-a49b-dabbbcad494d)


- создаём раздел на диске **sdb:**

```
gdisk /dev/sdb
```

- 1 - Создаём пустую **GUID partition table** - нажимаем "**o**" -> "**y**", чтобы удалить все разделы и создать **GPT запись**;
- 2 - Создаём первый и единственный раздел на весь диск. Нажимаем "**n**" -> 1 -> '**enter**' -> '**enter**' -> "**8e00**" - указываем, что данный раздел относится к **LVM**;
- 3 - "**p**" - Теперь можно просмотреть то, что получилось.

![Pasted image 20240212193054](https://github.com/e1mky/dms/assets/102690802/417c9864-a726-4917-a652-1993ff4bb751)


- после чего нажимаем "**w**" -> "**Y**" - подтверждаем и записываем изменения на диск:

![Pasted image 20240212193105](https://github.com/e1mky/dms/assets/102690802/5def3a67-d815-4b7c-a821-2558a16bc5fa)


- аналогичным образом создаём раздел на диске **sdc,** результат:

![Pasted image 20240212193118](https://github.com/e1mky/dms/assets/102690802/555744e6-f88a-4bb3-9624-7a672fae45d1)


- Инициализируем диски для работы с **LVM**:

```
pvcreate /dev/sd{b,c}1
```

где:

**/dev/sd{b,c}1** - это разделы блочных устройств: **sdb** и **sdc**;

- Создаём и добавляем диски в группу **vg01**:

```
vgcreate vg01 /dev/sd{b,c}1
```

где:

**vg01** - имя группы;

**/dev/sd{b,c}1** - это разделы блочных устройств: sdb и sdc, ранее проинициализированные для работы с LVM;

- Создаём логический том в режиме **зеркалирования** (raid1):

```
lvcreate -l 100%FREE -n lvmirror -m1 vg01
```

где:

**-l 100%FREE** - используется все свободное пространство группы **vg01**;

**-n lvmirror** - задаётся имя логическому тому "**lvmirror**";

**-m1** - создает зеркальный логический том с "**зеркальными**" копиями;

**vg01** - имя группы созданной ранее, в которой создаётся логический том с именем **lvmirror**.

- Проверяем:

![Pasted image 20240212193134](https://github.com/e1mky/dms/assets/102690802/136d3867-cbdb-4962-a7f1-63616e3686f0)


- создаём файловую систему на логическом томе **lvmirror**:

```
mkfs.ext4 /dev/vg01/lvmirror
```

- Монтируем логический диск в "**/opt/data**":
    - создаём директорию для монтирования:

```
mkdir /opt/data
```

- - добавляем запись в файл **/etc/fstab** для автоматического монтирования логического тома:

```
echo "UUID=$(blkid -s UUID -o value /dev/vg01/lvmirror) /opt/data ext4 defaults 1 2" | tee -a /etc/fstab
```

- выполняем монтирование:

```
mount -av
```

![Pasted image 20240212193148](https://github.com/e1mky/dms/assets/102690802/13d630c3-3733-4fd9-88f4-033053595550)


- Проверяем:

![Pasted image 20240212193200](https://github.com/e1mky/dms/assets/102690802/98b9ecd9-ac9f-4b26-b515-99661c8c183c)


#### SRV-BR:

- Используем два неразмеченных жёстких диска: **sdb** и **sdc**
- Создаём разделы на двух подключённых дисках "**sdb** и **sdc**" аналогично как и на **SRV-HQ** - через **gdisk,** результат:

![Pasted image 20240212193222](https://github.com/e1mky/dms/assets/102690802/b9a32532-6e24-4703-aca6-319a476bb8d0)


- Аналогично **SRV-HQ** добавляем диски в **LVM** и создаём группу из двух дисков:

```
pvcreate /dev/sd{b,c}1
```

```
vgcreate vg01 /dev/sd{b,c}1
```

- Создаём **striped LVM** том с именем "**lvstriped**":

```
lvcreate -i2 -l 100%FREE -n lvstreped vg01
```

где:

**-i2** - количество полосок (для полосок необходимо использовать диск);

**-l 100%FREE** - используется все свободное пространство группы **vg01**;

**-n lvstreped** - задаётся имя логическому тому "**lvstreped**";

**vg01** - имя группы созданной ранее, в которой создаётся логический том с именем **lvstreped**.

- Проверяем:

![Pasted image 20240212193233](https://github.com/e1mky/dms/assets/102690802/c3e8f801-cc3b-4c67-8402-993b82f6e7b6)


- создаём файловую систему на логическом томе **lvstreped**:

```
mkfs.ext4 /dev/vg01/lvstreped
```

- Обеспечиваем шифрование тома средствами **dm-crypt**:
    - Создаём ключевой файл:

```
dd if=/dev/urandom of=/etc/keys/enc.key bs=1 count=4096
```

- - Выполняем шифрование тома "**lvstreped**" с использованием **ключевого файла** вместо пароля, указывая путь до логического тома и путь до созданного ключа:

```
cryptsetup luksFormat /dev/vg01/lvstreped /etc/keys/enc.key
```

- - - после ввода данной команды необходимо подтверждение - "**YES**":

![Pasted image 20240212193257](https://github.com/e1mky/dms/assets/102690802/48ebffb6-10fd-4b6e-b977-f5406b7dc04c)


- - Далее необходимо открыть только что зашифрованный логический том при помощи созданного ключа, чтобы он появился в списке блочных устройств в выводе команды "**lsblk**":

```
cryptsetup luksOpen -d /etc/keys/enc.key /dev/vg01/lvstreped lvstreped
```

где:

**-d /etc/keys/enc.key** - путь до ключа, которым был зашифрован логический том **lvstreped;**

**/dev/vg01/lvstreped** - путь до логического тома, с которого необходимо снять шифрование;

**lvstreped** - произвольное имя, которое будет назначено для разшифрованного тома, и должно появиться в выводе команды **lsblk**.

![Pasted image 20240212193314](https://github.com/e1mky/dms/assets/102690802/0ee680a1-4eea-4c0c-ab8b-26d051e6683c)


- - Задаём файловую систему на только расшифрованном томе который находится в директории **/dev/mapper/**:

```
mkfs.ext4 /dev/mapper/lvstreped
```

- - Монтируем логический том в "**/opt/data**":
        - создаём директорию для монтирования:

```
mkdir /opt/data
```

- - - добавляем запись в файл **/etc/fstab** для автоматического монтирования логического тома:

```
echo "UUID=$(blkid -s UUID -o value /dev/mapper/lvstreped) /opt/data ext4 defaults 0 0" | tee -a /etc/fstab
```

_P.S: **UUID** указывается **расшифрованного (открытого)** логического тома._

- - выполняем монтирование:

```
mount -av
```

![Pasted image 20240212193327](https://github.com/e1mky/dms/assets/102690802/8799944f-190b-4ed6-b88e-9aa4ee3f1f3a)


- Обеспечиваем монтирование логического тома при загрузке ОС без ввода пароля:

```
echo "lvstreped UUID=$(blkid -s UUID -o value /dev/vg01/lvstreped) /etc/keys/enc.key luks" | tee -a /etc/crypttab
```

_P.S: UUID указывается **зашифрованного (закрытого)** логического тома, чтобы во время загрузки ОС до монтирования выполнялось его открытие при помощи ключа._

- Проверяем:
    - выполнить перезагрузку **reboot**, запуск ОС должен произости без какой либо ошибки на стадии загрузки, после чего проверить примонтированные устройства:

![Pasted image 20240212193342](https://github.com/e1mky/dms/assets/102690802/8ce8b0a3-858a-4bc4-948f-a30ea5f3eb3b)

# 7. Настройка DNS для SRV-HQ и SRV-BR
7. Настройка DNS для SRV-HQ и SRV-BR

- i. Реализуйте основной DNS сервер компании на SRV-HQ
    - a. Для всех устройств обоих офисов необходимо создать записи A и PTR.
    - b. Для всех сервисов предприятия необходимо создать записи CNAME
    - c. Создайте запись test таким образом, чтобы при разрешении имени из левого офиса имя разрешалось в адрес SRV-HQ, а из правого – в адрес SRV-BR.
    - d. Сконфигурируйте SRV-BR, как резервный DNS сервер. Загрузка записей с SRV-HQ должна быть разрешена только для SRV-BR
    - e. Клиенты предприятия должны быть настроены на использование внутренних DNS серверов

### Выполнение:

#### SRV-HQ:

- Устанавливаем пакет **bind** и **bind-utils**:

```
apt-get update && apt-get install -y bind bind-utils
```

- Редактируем конфигурационный файл по пути "**/etc/bind/options.conf**":

```
vim /etc/bind/options.conf
```

![Pasted image 20240212193424](https://github.com/e1mky/dms/assets/102690802/18cf05de-d7de-4bcc-be19-4e8185998fad)


где:

**listen-on { any; };** - Этот параметр определяет адреса и порты, на которых DNS-сервер будет слушать запросы. Значение any означает, что сервер будет прослушивать запросы на всех доступных интерфейсах и IP-адресах.

**allow-query { any; };** -  Определяет список IP-адресов или подсетей, которым разрешено отправлять запросы на этот DNS-сервер. Значение any позволяет принимать запросы от всех клиентов.

**allow-transfer { 10.0.20.2; };** - устанавливает возможность передачи зон для slave-серверов.

- Правим конфигурационный файл "**/etc/net/ifaces/ens33/resolv.conf**":

```
cat <<EOF > /etc/net/ifaces/ens33/resolv.conf
search company.prof
nameserver 127.0.0.1
EOF
```

- Перезапускаем службу **network**:

```
systemctl restart network
```

- Выполняем запуск и добавление в автозагрузку **DNS - сервера**:

```
systemctl enable --now bind
```

- Проверяем:

![Pasted image 20240212193438](https://github.com/e1mky/dms/assets/102690802/c9ad7eb8-4fb5-44f4-b6ef-1efe226e6359)


- Созданиём зону прямого просмотра и обратного просмотра, добавляем в конфигурационный файл "**/etc/bind/local.conf**":

```
vim /etc/bind/local.conf
```

![Pasted image 20240212193451](https://github.com/e1mky/dms/assets/102690802/3186d338-612a-45b9-a9de-5fc1c6ec1b62)


где:

**zone "..." { ... };** : Это начало определения зоны. В кавычках указывается имя зоны, которое следует разрешать на этом сервере.

**type master;** : Это указывает тип зоны. "type master" означает, что эта зона является мастер-зоной, то есть она содержит авторитетные записи, которые могут быть изменены и обновлены на этом сервере.

**file "...";** : Это указывает путь к файлу, который содержит данные зоны. Файлы данных зоны используются для хранения записей DNS, таких как A-записи, CNAME-записи, MX-записи и т. д.

- Копируем пример прямой зоны:

```
cp /etc/bind/zone/{localdomain,company.db}
```

- Копируем пример для зоны обратного просмотра:

```
cp /etc/bind/zone/127.in-addr.arpa /etc/bind/zone/10.0.10.in-addr.arpa.db
```

```
cp /etc/bind/zone/127.in-addr.arpa /etc/bind/zone/20.0.10.in-addr.arpa.db
```

- Задаём права - назначаем владельца:

```
chown root:named /etc/bind/zone/company.db
```

```
chown root:named /etc/bind/zone/10.0.10.in-addr.arpa.db
```

```
chown root:named /etc/bind/zone/20.0.10.in-addr.arpa.db
```

- Настраиваем зону прямого просмотра:

```
vim /etc/bind/zone/company.db
```

![Pasted image 20240212193510](https://github.com/e1mky/dms/assets/102690802/d95579b7-ea30-4118-a506-060ff08b745e)


**Запись типа NS** определяет авторитетные DNS-серверы для определенной зоны.

**Запись типа A** связывает доменное имя с IPv4-адресом. Она используется для преобразования доменного имени в соответствующий IP-адрес.

**Запись типа CNAME** используется для создания псевдонима (алиаса) для другого доменного имени. Она указывает, что одно доменное имя является псевдонимом (алиасом) для другого "канонического" доменного имени.

- Настраиваем зону прямого просмотра для сети 10.0.10.0/24:

```
vim /etc/bind/zone/10.0.10.in-addr.arpa.db
```

![Pasted image 20240212193523](https://github.com/e1mky/dms/assets/102690802/e013f50d-3d9a-4f22-85d6-89fad57c84c3)


- Настраиваем зону прямого просмотра для сети 10.0.20.0/24:

```
vim /etc/bind/zone/20.0.10.in-addr.arpa.db
```

![Pasted image 20240212193536](https://github.com/e1mky/dms/assets/102690802/1eb98c5e-2690-43d7-8aed-0b248cd8a226)


- Утилитой "**named-checkconf**" с ключём "**-z**" проверяем что файл зон не содержитат ошибок и загружаются:

![Pasted image 20240212193557](https://github.com/e1mky/dms/assets/102690802/498294d0-e2f2-4904-94f3-19edc7eb6b02)


- Перезагружаем службу **bind**:

```
systemctl restart bind
```

- Проверяем:
    - Зона прямого просмотра:

![Pasted image 20240212193610](https://github.com/e1mky/dms/assets/102690802/266cdf46-9866-4049-b6cf-d571178926e7)


- Зоны обратного просмотра:

![Pasted image 20240212193621](https://github.com/e1mky/dms/assets/102690802/689974ad-49ee-4f12-ba63-a19d85939364)


#### SRV-BR:

Настраиваем как резервный DNS сервер

- Устанавливаем пакет **bind** и **bind-utils**:

```
apt-get update && apt-get install -y bind bind-utils
```

- Редактируем конфигурационный файл по пути "**/etc/bind/options.conf**":

```
vim /etc/bind/options.conf
```

![Pasted image 20240212193632](https://github.com/e1mky/dms/assets/102690802/6e789151-6173-4338-819d-3064ace5297a)


- добавляем в конфигурационный файл "**/etc/bind/local.conf**" следующую информацию:

```
vim /etc/bind/local.conf
```

![Pasted image 20240212193642](https://github.com/e1mky/dms/assets/102690802/9aedf159-a8df-4d82-9ee1-2736c5b67637)


- Правим конфигурационный файл "**/etc/net/ifaces/ens33/resolv.conf**":

Explain

`cat <<EOF > /etc/net/ifaces/ens33/resolv.conf search company.prof nameserver 10.0.10.2 nameserver 10.0.20.2 EOF`

- Перезапускаем службу **network**:

```
systemctl restart network
```

- Выполняем запуск и добавление в автозагрузку **DNS - сервера**:

```
systemctl enable --now bind
```

- Чтобы bind работал в режиме SLAVE, нужно выполнить:

```
control bind-slave enabled
```

- Проверяем:
    - должны появиться файлы зон по пути "**/etc/bind/zone/slave/**":

![Pasted image 20240212193711](https://github.com/e1mky/dms/assets/102690802/fa3404b5-f137-4d15-a1a2-654bd7736d9d)


- работоспособность с **CLI-BR:**

![Pasted image 20240212193722](https://github.com/e1mky/dms/assets/102690802/401cbcef-effc-4f94-8a2c-1a9fad008b93)


### **SRV-HQ:**

Создайте запись test таким образом, чтобы при разрешении имени из левого офиса имя разрешалось в адрес SRV-HQ, а из правого – в адрес SRV-BR.

- добавляем в конфигурационный файл "**/etc/bind/local.conf**" следующую информацию:

```
vim /etc/bind/local.conf
```

![Pasted image 20240212193737](https://github.com/e1mky/dms/assets/102690802/ca8ce092-0327-4505-a185-8145f96957b6)


- Копируем пример для зоны:

```
cp /etc/bind/zone/{localdomain,test.company.db}
```

- Задаём права - назначаем владельца:

```
chown root:named /etc/bind/zone/test.company.db
```

- Настраиваем зону:

```
vim /etc/bind/zone/test.company.db
```

![Pasted image 20240212193751](https://github.com/e1mky/dms/assets/102690802/49e54cc6-7686-4737-86cb-d39f11d3f99c)


- Перезапускаем службу **bind**:

```
systemctl restart bind
```

### **SRV-BR:**

- добавляем в конфигурационный файл "**/etc/bind/local.conf**" следующую информацию:

```
vim /etc/bind/local.conf
```

![Pasted image 20240212193807](https://github.com/e1mky/dms/assets/102690802/16f7a3cd-7a40-4e29-8732-76b527dc0887)


- Копируем пример для зоны:

```
cp /etc/bind/zone/{localdomain,test.company.db}
```

- Задаём права - назначаем владельца:

```
chown root:named /etc/bind/zone/test.company.db
```

- Настраиваем зону:

```
vim /etc/bind/zone/test.company.db
```

![Pasted image 20240212193822](https://github.com/e1mky/dms/assets/102690802/11a31e8c-475d-4aeb-923c-98b93b5251f0)


- Перезапускаем службу **bind**:

```
systemctl restart bind
```

- Проверяем:
    - с **CLI-HQ**:

![Pasted image 20240212193835](https://github.com/e1mky/dms/assets/102690802/9d5017e3-81e0-4634-9225-6a493684ab3f)


- с **CLI-BR:**

![Pasted image 20240212193845](https://github.com/e1mky/dms/assets/102690802/b8d21419-b3e9-456a-9caa-a1b43b904c1e)

# 8. На сервере SRV-HQ сконфигурируйте основной доменный контроллер на базе FreeIPA
10. На сервере SRV-HQ сконфигурируйте основной доменный контроллер на базе FreeIPA

- a) Создайте 30 пользователей user1-user30.
- b) Пользователи user1-user10 должны входить в состав группы group1.
- c) Пользователи user11-user20 должны входить в состав группы group2.
- d) Пользователи user21-user30 должны входить в состав группы group3.
- e) Разрешите аутентификацию с использованием доменных учетных данных на ВМ CLI-HQ.
- f) Установите сертификат центра сертификации FreeIPA в качестве доверенного на обоих клиентских ПК.

### Выполнение:

#### SRV-HQ:

#### Развёртывание контроллера домена на базе FeeIPA:

- Для ускорения установки можно установить демон энтропии **haveged**:

```
apt-get update && apt-get install -y haveged
```

- Включаем и запускаем службу **haveged**:

```
systemctl enable --now haveged
```

- Установим пакет **FreeIPA**:

```
apt-get install -y freeipa-server
```

- Запускаем интерактивную установку **FreeIPA**:

```
ipa-server-install
```

**1** - отвечаем **no** на вопрос, нужно ли сконфигурировать **DNS-сервер BIND**;

**2, 3, 4** - нужно указать **имя узла** на котором будет установлен сервер **FreeIPA**, **доменное имя** и **пространство Kerberos**;

- _Эти имена нельзя изменить после завершения установки!_

![Pasted image 20240212193919](https://github.com/e1mky/dms/assets/102690802/6c654953-5099-4827-94db-41afe6ef9f41)


**1** - задаётся и подтверждается пароль для **Director Manager**;

- не менее 8 символов;

**2** - задаётся и подтверждается пароль для **администратора** FreeIPA;

![Pasted image 20240212193945](https://github.com/e1mky/dms/assets/102690802/3e56198e-104b-4316-8309-d02e88965bad)


**1** - указывается **имя NetBIOS**;

**2** - Указать, если это необходимо, **NTP-сервер** или пул серверов:

**3** - Далее необходимо проверить информацию о конфигурации и подтвердить ответив **yes**:

![Pasted image 20240212193956](https://github.com/e1mky/dms/assets/102690802/8e5fb0b9-7d6a-457a-827c-6e9cb0c75064)


Начнётся процесс конфигурации. После его завершения будет **выведена следующая информация**:

![Pasted image 20240212194006](https://github.com/e1mky/dms/assets/102690802/48b5d708-4f70-4d6d-9a90-9d68ffe39312)


- Проверяем запущенные службы FreeIPA:

![Pasted image 20240212194018](https://github.com/e1mky/dms/assets/102690802/c3818d8c-a7e8-4215-9e51-a863d64a6f61)


#### Далее выполняем создание необходимых групп и пользователей:

_Пользователей и группы можно накликать и в веб-интерфейсе FreeIPA с CLI-HQ_

- получаем билет **kerberos**:

```
kinit admin
```

- - вводим пароль доменного пользователя **admin:**

![Pasted image 20240212194032](https://github.com/e1mky/dms/assets/102690802/3e3d41fe-951e-46b1-af1c-4f0da1b76f97)


- используя цикл **for** создадим **30** пользователей: **user№** с паролем **P@ssw0rd**, а также сменим срок действия пароля до **2025** года, чтобы при входе из под пользователя на не пришлось менять пароль:

```
for i in {1..30};do 
	echo "P@ssw0rd" | ipa user-add user$i --first=User --last=$i --password;
	ipa user-mod user$i --setattr=krbPasswordExpiration=20251225011529Z;
done
```

- Проверяем:

![Pasted image 20240212194052](https://github.com/e1mky/dms/assets/102690802/bf1e9419-905f-477b-a10a-8d1d60a897cb)


- создадим группы: **group1**, **group2** и **group3**:

```
for i in {1..3}; do
	ipa group-add group$i;
done
```

![Pasted image 20240212194122](https://github.com/e1mky/dms/assets/102690802/f235c241-3855-4949-9e75-5bf773351cf5)


- добавим пользователей в соответствующие группы:
    - **group1:**

```
for i in {1..10}; do
	ipa group-add-member group1 --users=user$i;
done
```

![Pasted image 20240212194139](https://github.com/e1mky/dms/assets/102690802/2fe02052-5d65-4ee7-b4a4-ed0b1188241b)


- - **group2:**

```
for i in {11..20}; do
	ipa group-add-member group2 --users=user$i;
done
```

![Pasted image 20240212194153](https://github.com/e1mky/dms/assets/102690802/9dfd4f1e-ed16-4549-9865-3acc9557a6a8)


- - **group3:**

```
for i in {21..30}; do
	ipa group-add-member group3 --users=user$i;
done
```

![Pasted image 20240212194206](https://github.com/e1mky/dms/assets/102690802/d2e1dbe1-e3ae-4618-909f-3fb3a414f7d1)


#### Разрешаем аутентификацию с использованием доменных учетных данных на ВМ CLI-HQ

Чтобы разрешить аутентификацию с использованием доменных учетных данных на CLI-HQ введём CLI-HQ в домен FreeIPA

#### CLI-HQ:

- Установим необходимые пакеты:

```
apt-get update && apt-get install -y freeipa-client zip
```

- Запустим скрипт настройки клиента в интерактивном режиме:

```
ipa-client-install --mkhomedir
```

- - Скрипт установки должен автоматически найти необходимые настройки на FreeIPA сервере, вывести их и спросить подтверждение для найденных параметров;
    - Затем запрашивается имя пользователя, имеющего право вводить машины в домен, и его пароль (можно использовать администратора по умолчанию, который был создан при установке сервера);
    - Далее сценарий установки настраивает клиент.

![Pasted image 20240212194224](https://github.com/e1mky/dms/assets/102690802/ad415fa4-cf7e-42fb-960c-0d775c9a8d91)


- перезапускаем клиента:

```
reboot
```

- Также после ввода в домен - **CLI-HQ** автоматически доверяет интегрированному **Корневому Центру Сертификации FreeIPA**:

![Pasted image 20240212194241](https://github.com/e1mky/dms/assets/102690802/a4bbaf41-7cd4-43ac-b446-5e0ff404daf7)


![Pasted image 20240212194258](https://github.com/e1mky/dms/assets/102690802/c3931d09-e1ca-4b67-a3d2-ba58cb19518d)


- Выполняем вход из под пользователя **admin** и проверяем ранее **созданные группы**:

![Pasted image 20240212194310](https://github.com/e1mky/dms/assets/102690802/5a3db6a5-40e1-4fef-aa53-51ed13d7d25f)


- И соответствующих пользователей в каждой группе:

![Pasted image 20240212194322](https://github.com/e1mky/dms/assets/102690802/86aba31b-88a1-46dd-b24a-1c00a3185c12)


![Pasted image 20240212194329](https://github.com/e1mky/dms/assets/102690802/3ff45c00-15ea-4659-840b-7943b7beb994)


![Pasted image 20240212194333](https://github.com/e1mky/dms/assets/102690802/8f5ac9cb-9b7e-4ae9-8feb-1d2a3071026a)


- проверим вход на **HQ-CLI** из под доменного пользователя:

![Pasted image 20240212194343](https://github.com/e1mky/dms/assets/102690802/165a9894-b667-4bb3-8d3e-dc1a32b81f8e)


Для добавления Корневого Сертификата на **CLI-BR** - необходимо передать сертификат с **SRV-HQ** расположенный по пути **/etc/ipa/ca.crt**

- данный сертификат на **CLI-BR** - необходимо разместить по пути **/etc/pki/ca-trust/source/anchors/** и выполнить команду **update-ca-trust extract**

# 9. Настройка узла управления Ansible
8. Настройка узла управления Ansible

- a) Настройте узел управления на базе SRV-BR
    - a. Установите Ansible.
- b) Сконфигурируйте инвентарь по пути /etc/ansible/inventory. Инвентарь должен содержать три группы устройств:
    - a. Networking
    - b. Servers
    - c. Clients
- c) Напишите плейбук в /etc/ansible/gathering.yml для сбора информации об IP адресах и именах всех устройств (и клиенты, и серверы, и роутеры). Отчет должен быть сохранен в /etc/ansible/output.yaml, в формате ПОЛНОЕ_ДОМЕННОЕ_ИМЯ – АДРЕС

### Выполнение:

### SRV-BR:

_Настройка генерации **ssh-ключей** и передача их на устройства за исключением **rtr-hq** и **rtr-br** - упущена, далее касательно **ansible**_

- Устанавливаем **ansible**:

```
apt-get install -y ansible sshpass
```

- Описываем инвентарный файл по пути "**/etc/ansible/inventory**":

```
vim /etc/ansible/inventory
```

- - вариант написания инвентарного файла в формате **yml**:

![Pasted image 20240212194430](https://github.com/e1mky/dms/assets/102690802/06026d8f-8616-4a74-b20d-68da58c9271e)


- вариант написания инвентарного файла в формате **ini**:

![Pasted image 20240212194444](https://github.com/e1mky/dms/assets/102690802/a5678b3d-334b-4204-a3a4-fba66209d706)


- Правим конфигурационный файл "**/etc/ansible/ansible.cfg**" для использования по умолчанию только что созданного инвентарного файла:

```
vim /etc/ansible/ansible.cfg
```

![Pasted image 20240212194500](https://github.com/e1mky/dms/assets/102690802/47e50709-77b2-40f7-b7bf-371dd48ac5a7)


- Перейдём в каталог **/etc/ansible** и создадим директорию для переменных **group_vars:**

```
cd /etc/ansible
```

```
mkdir group_vars
```

- Опишим переменные для групп описанных в интентарном файле:
    - для группы **Networking:**

```
vim group_vars/Networking.yml
```

![Pasted image 20240212194517](https://github.com/e1mky/dms/assets/102690802/b85b2b09-2ded-441e-ab91-7a33805afc43)


- - для группы **Servers:**

```
vim group_vars/Servers.yml
```

![Pasted image 20240212194533](https://github.com/e1mky/dms/assets/102690802/e3800f15-cf37-47f5-bdde-8496bb7ab4a7)


- - для группы **Clients:**

```
vim group_vars/Clients.yml
```

![Pasted image 20240212194545](https://github.com/e1mky/dms/assets/102690802/346b7123-3092-467b-801c-283d73d2304d)


- - Также опишим переменные которые относятся ко всем хостам:

```
vim group_vars/all.yml
```

![Pasted image 20240212194558](https://github.com/e1mky/dms/assets/102690802/e2b8c9ad-e4c7-4917-9168-0d50bf8f6b19)


- В результате должна получиться следующая структура:

![Pasted image 20240212194609](https://github.com/e1mky/dms/assets/102690802/bc6f3cee-aa54-4209-b943-94f08f0a51f5)


- Проверяем что ansible может подключиться к хостам:

```
ansible -m ping all
```

![Pasted image 20240212194624](https://github.com/e1mky/dms/assets/102690802/54a68acf-df23-4b1c-9b95-c7659837f15f)

- Пишем **playbook-**сценарий для сбора информации об IP адресах и именах всех устройствах:

```
vim gathering.yml
```

- - _написать playbook-сценарий возможно разными способами, данный пункт выполним даже при помощи ad-hoc команд, поэтому решать каждому самостоятельно какие модули ansible использовать_

```
---
- name: Сбор информации об IP-адресах и именах устройств
  hosts: your_target_group
  gather_facts: yes

  tasks:
    - name: Вывести IP-адреса и имена устройств
      debug:
        var: hostvars[inventory_hostname]['ansible_all_ipv4_addresses'] + [inventory_hostname]

```
 -hosts: your_target_group - замените "your_target_group" на группу хостов, для которых вы хотите собрать информацию.
 -gather_facts: yes - включает сбор фактов о хосте, включая IP-адреса.
 -debug - выводит информацию в формате отладки.
Примечание:
 -ansible_all_ipv4_addresses - это факт, содержащий список всех IPv4-адресов хоста.
 -inventory_hostname - это переменная, содержащая имя текущего хоста.
- Для запуска playbook-сценария:

```
ansible-playbook gathering.yml
```
# 10. Установка и настройка сервера баз данных
4. Установка и настройка сервера баз данных

- a) В качестве серверов баз данных используйте сервера SRV-HQ и SRVBR
- b) Разверните сервер баз данных на базе Postgresql
    - a. Создайте базы данных prod, test, dev
        - i. Заполните базы данных тестовыми данными при помощи утилиты pgbench. Коэффицент масштабирования сохраните по умолчанию.
    - Создайте пользователей produser, testuser, devuser, каждому из пользователей дайте доступ к соответствующей базе данных.
    - b. Разрешите внешние подключения для всех пользователей.
    - c. Сконфигурируйте репликацию с SRV-HQ на SRV-BR

### Выполнение:

#### SVR-HQ:

- Установка пакетов базы данных:

```
apt-get install -y postgresql16 postgresql16-server postgresql16-contrib
```

- Создаём системные базы данных:

```
/etc/init.d/postgresql initdb
```

![Pasted image 20240212194720](https://github.com/e1mky/dms/assets/102690802/b5bdb785-d6c7-461a-9c3b-f421765b961f)


- Включаем и добавляе в автозагрузку PostgreSQL:

```
systemctl enable --now postgresql
```

- Разрешаем доступ к PostgreSQL из сети:

```
vim /var/lib/pgsql/data/postgresql.conf
```

- - в конфигурационном файле находим строку "**listen_addresses = 'localhost'**" и приводим её к следующему виду:

![Pasted image 20240212194733](https://github.com/e1mky/dms/assets/102690802/5ddfab9b-1cfd-4346-96bf-8a022e540054)


- Перезапускаем PostgreSQL:

```
systemctl restart postgresql
```

- Проверяем:

![Pasted image 20240212194743](https://github.com/e1mky/dms/assets/102690802/0816d895-52f3-40e5-b27a-1428c22db521)


#### Создаём базы данных и пользователей с необходимыми правами:

- Для заведения пользователей и создания баз данных, необходимо переключиться в учётную запись "**postgres**":

```
psql -U postgres
```

![Pasted image 20240212194754](https://github.com/e1mky/dms/assets/102690802/fb865d4e-50d3-4bbb-a6e4-130a8e277ef0)


- - зададим пароль для пользователя "**postgres**":

```
ALTER USER postgres WITH ENCRYPTED PASSWORD 'P@ssw0rd';
```

- - Создаём базы данных "**prod**","**test**" и "**dev**":

```
CREATE DATABASE prod;
```

```
CREATE DATABASE test;
```

```
CREATE DATABASE dev;
```

- - Создаём пользователей "**produser**","**testuser**" и "**devuser**":

```
CREATE USER produser WITH PASSWORD 'P@ssw0rd';
```

```
CREATE USER testuser WITH PASSWORD 'P@ssw0rd';
```

```
CREATE USER devuser WITH PASSWORD 'P@ssw0rd';
```

- - Назначаем для каждой базы данных соответствующего владельца:
        - для базы данных "**prod**" назначаем владельцем пользователя "**produser**":

```
GRANT ALL PRIVILEGES ON DATABASE prod to produser;
```

- - - для базы данных "**test**" назначаем владельцем пользователя "**testuser**":

```
GRANT ALL PRIVILEGES ON DATABASE test to testuser;
```

- - - для базы данных "**dev**" назначаем владельцем пользователя "**devuser**":

```
GRANT ALL PRIVILEGES ON DATABASE dev to devuser;
```

- Заполняем базы данных тестовыми данными при помощи утилиты **pgbench:**

```
pgbench -U postgres -i prod
```

```
pgbench -U postgres -i test
```

```
pgbench -U postgres -i dev
```

- Проверяем:

```
psql -U postgres
```

```
\c prod
```

```
\dt+
```

![Pasted image 20240212194817](https://github.com/e1mky/dms/assets/102690802/c4720599-b2c5-4bf4-a37b-7c35750e15ce)


- Аналогично и для других баз данных:

![Pasted image 20240212194827](https://github.com/e1mky/dms/assets/102690802/3ff53df1-78f6-4d73-8447-042a8176a764)


- Настраиваем парольную аутентификацию для удалённого доступа:

```
vim /var/lib/pgsql/data/pg_hba.conf
```

- - добавляем следующую запись:

![Pasted image 20240212194839](https://github.com/e1mky/dms/assets/102690802/d2559c0a-2a83-4ded-ad31-daecefa8ef99)


- перезапускаем PostgreSQL:

```
systemctl restart postgresql
```

#### SRV-BR:

- Установка пакетов базы данных:

```
apt-get install -y postgresql16 postgresql16-server postgresql16-contrib
```

- Проверяем:
    - подключаемся с **SRV-BR** к **SRV-HQ**:
        - из под пользователя "**produser**" к базе данных "**prod**":

![Pasted image 20240212194856](https://github.com/e1mky/dms/assets/102690802/3da1ba3e-bf3d-45dc-9542-6a4a20dc9282)


- из под пользователя "**testuser**" к базе данных "**test**":

![Pasted image 20240212194908](https://github.com/e1mky/dms/assets/102690802/28f50ab5-38c4-47f7-8fd9-92fc98fd9623)


- из под пользователя "**devuser**" к базе данных "**dev**":

![Pasted image 20240212194930](https://github.com/e1mky/dms/assets/102690802/cf6e96d4-e6b6-4edc-9cc5-ffa022910492)


#### Настраиваем репликацию с SRV-HQ на SRV-BR:

#### SRV-HQ:

- Открываем конфигурационный файл "**/var/lib/pgsql/data/postgresql.conf**":

```
vim /var/lib/pgsql/data/postgresql.conf
```

- - редактируем следующие параметры:

![Pasted image 20240212194941](https://github.com/e1mky/dms/assets/102690802/918514b5-2cf8-40f7-985a-151202d3fc24)


![Pasted image 20240212194944](https://github.com/e1mky/dms/assets/102690802/7f75452a-62c6-4fe1-91dd-857fa27fc4c8)


![Pasted image 20240212194948](https://github.com/e1mky/dms/assets/102690802/639b134e-addf-458e-923a-df7b6bb21917)


![Pasted image 20240212194953](https://github.com/e1mky/dms/assets/102690802/323cf4c6-3394-4170-8839-5230402a7015)


где:

**wal_level** указывает, сколько информации записывается в WAL (журнал операций, который используется для репликации);

**max_wal_senders** — количество планируемых слейвов;

**max_replication_slots** — максимальное число слотов репликации; 

**hot_standby** — определяет, можно или нет подключаться к postgresql для выполнения запросов в процессе восстановления;

**hot_standby_feedback** — определяет, будет или нет сервер slave сообщать мастеру о запросах, которые он выполняет.

- Перезапускаем службу postgresql:

```
systemctl restart postgresql
```

#### SRV-BR:

- Останавливаем службу postgres:

```
systemctl stop postgresql
```

- Удаляем содержимое каталога с данными:

```
rm -rf /var/lib/pgsql/data/*
```

- И реплицируем данные с **SRV-HQ** сервера:

```
pg_basebackup -h 10.0.10.2 -U postgres -D /var/lib/pgsql/data --wal-method=stream --write-recovery-conf
```

![Pasted image 20240212195008](https://github.com/e1mky/dms/assets/102690802/1161b7a1-4b90-4049-96be-cb2d4715456a)


- Назначаем владельца:

```
chown -R postgres:postgres /var/lib/pgsql/data/
```

- Снова запускаем сервис postgresql:

```
systemctl start postgresql
```

#### Проверяем репликацию:

#### SRV-HQ:

- Создаём тестовую базу данных:

```
psql -U postgres
```

```
CREATE DATABASE test_replica;
```

![Pasted image 20240212195019](https://github.com/e1mky/dms/assets/102690802/57cfaffd-e2bf-4c31-ae6c-3c8077ebf9ae)


#### SRV-BR:

- Смотрим список всех баз данных:

![Pasted image 20240212195031](https://github.com/e1mky/dms/assets/102690802/23f56ae2-3840-42ed-a027-df472e1c5fd6)

