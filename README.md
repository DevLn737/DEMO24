Учётки
ISP root root, administrator P@ssw0rd
BR-R root root, administrator P@ssw0rd
HQ-R root root, administrator P@ssw0rd
HQ-SRV root root administrator P@ssw0rd
BR-SRV root root administrator P@ssw0rd

Полезное  
Выбор ПО на роутеры и серваки  
![](https://github.com/DevLn737/DEMO24/blob/main/%D0%98%D0%B7%D0%BE%D0%B1%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F/Pasted%20image%2020240611182334.png)  
Прежде проверять внутреннюю сеть, необходимо не забыть в настройках proxmox или esxi отключить сетевое устройство, с помощью которого создаётся мост между основным компьютером и виртуальной машиной. И наоборот, возвращать обратно интернет, когда будет происходить установку пакета, или удаление репозитория. Но, как же это сделать? Ответ, думайте, ничего сложного, иначе даже первый модуль не сделать.

Посмотреть последние логи  
```bash
journalctl -xe
```

Узнать текущий DNS через nmcli
![](https://github.com/DevLn737/DEMO24/blob/main/%D0%98%D0%B7%D0%BE%D0%B1%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F/Pasted%20image%2020240617192439.png)

Список используемых пакетов на машинах (*Неполный*)  
network-manager frr dnsmasq iperf3 isc-dhcp-server  
hq-r network-manager frr iperf3 isc-dhcp-server   

### Модуль 1: Выполнение работ по проектированию сетевой инфраструктуры
---
#### Задание 1 - базовая настройка всех устройств

| Имя устройства | Интерфейс | ipv4                   | Маска/префикс      | Шлюз     | NIC             |
| -------------- | --------- | ---------------------- | ------------------ | -------- | --------------- |
| ISP            | ens18     | 9.0.0.1                |                    | 9.0.0.1  | CLI             |
|                | ens19     | 10.0.0.1               | 255.255.255.252/30 | 10.0.0.1 | ISP - HQ        |
|                | ens20     | 11.0.0.1               | 255.255.255.252/30 | 11.0.0.1 | ISP - BR        |
|                | ens21     | DHCP                   |                    | DHCP     | INTERNET        |
| HQ-R           | ens18     | 10.0.0.2               | 255.255.255.252/30 | 10.0.0.1 | HQ - ISP        |
|                | ens19     | 192.168.1.1            | 255.255.255.192/26 |          | HQ - <br>HQ-SRV |
|                | gre       | 100.0.0.1              |                    |          | gre1            |
|                |           |                        |                    |          |                 |
| BR-R           | 11.ens18  | 11.0.0.2               | 255.255.255.252/30 | 11.0.0.1 | BR - ISP        |
|                | ens19     | 192.168.2.1            | 255.255.255.240/28 |          | BR - <br>BR SRV |
|                |           | 200.0.0.1              |                    |          | gre1            |
| HQ-SRV         |           | DHCP<br>(192.168.1.10) |                    |          |                 |
| BR-SRV         |           | 192.168.2.10           |                    |          |                 |


Формулы для рассчёта, их пока что нет.

Для того, чтобы присвоить имя хоста необходимо написать
```bash
hostnamectl set-hostname <имя хоста>
Пример
hostnamectl set-hostname HQ-R
```

Команды  
Обновляем и устанавливаем пакеты на каждом устройстве
```bash
apt update && apt -y dist-upgrade
apt install -y network-manager
```

Разрешаем пересылку пакетов на роутерах HQ-R, BR-R, ISP
```bash
echo net.ipv4.ip_forward=1 >> /etc/systcl.conf
```

Настроить интерфейсы по карте  

после настройки прописать 
```bash
systemctl restart NetworkManager
```

Если не включаются интерфейсы из за ошибки device is strictly unmanaged  
Необходимо проверить файл /etc/network/interfaces  
```bash
nano /etc/network/interfaces
```
И закомментируйте строчки поставив перед ними символ #  
```
#allow-hotplug ens18
#TYT ECHE STROCHKA NO IA YDALIL NECHAINO
```


Проверить работу, можно пинганув с ISP роутеры  
```
ping 10.0.0.2
ping 11.0.0.2
```
Если пинги прошли без проблем, значит всё ок  

nmcli у HQ-R
![](https://github.com/DevLn737/DEMO24/blob/main/%D0%98%D0%B7%D0%BE%D0%B1%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F/Pasted%20image%2020240611200435.png)

nmcli у BR-R
![](https://github.com/DevLn737/DEMO24/blob/main/%D0%98%D0%B7%D0%BE%D0%B1%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F/Pasted%20image%2020240611200536.png)

nmcli у ISP
![](https://github.com/DevLn737/DEMO24/blob/main/%D0%98%D0%B7%D0%BE%D0%B1%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F/Pasted%20image%2020240611200610.png)

#### Задание 2 - Настройка внутренней динамической маршрутизации 
##### Настройка GRE туннеля
Настройки nmtui на HQ-R
Для этого необходимо воспользоваться `nmtui`  
Дальше выбрать add, а тип соединения ip tunnel, третий снизу  

![](https://github.com/DevLn737/DEMO24/blob/main/%D0%98%D0%B7%D0%BE%D0%B1%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F/Pasted%20image%2020240612122448.png)


Настройки nmtui на BR-R
![](https://github.com/DevLn737/DEMO24/blob/main/%D0%98%D0%B7%D0%BE%D0%B1%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F/Pasted%20image%2020240612122806.png)

Проверить можно пинганув c HQ-R роутера BR-R роутер
```bash
ping 200.0.0.1
```
А после c BR-R роутера HQ-R роутер
```bash
ping 100.0.0.1
```
Или аналогично подключившись к аккаунтам administator(для root аккаунта требуются настройки в конфиге sshd)

##### Настройка динамической маршрутизации

Для внутренней динамической маршрутизации будем использовать FRR (Free Range Routing). Выбор протокола маршрутизации зависит от требований к масштабируемости. Рассмотрим два популярных протокола: OSPF и BGP.

**OSPF (Open Shortest Path First)**:
- Хорошо масштабируется в больших корпоративных сетях.
- Иерархическая структура (разделение на области).
- Быстрая конвергенция.

**BGP (Border Gateway Protocol)**:
- Используется преимущественно для меж доменной маршрутизации (между автономными системами).
- Масштабируемый и гибкий.
- Медленная конвергенция по сравнению с OSPF.

Для данной задачи предпочтительнее использовать **OSPF**, так как он лучше подходит для внутренней маршрутизации и обеспечит лучшую масштабируемость и быструю конвергенцию в будущем.

##### Установка и настройка FRR для OSPF
На обоих роутерах (HQ-R и BR-R) необходимо установить frr
 ```bash
apt update 
apt install -y frr frr-pythontools
```
Включить ospf изменив строчку ospfd=yes
```bash
nano /etc/frr/daemons
Находим строчку ospfd=no и меняем на ospfd=yes
```

Настройте OSPF в файле `/etc/frr/frr.conf`
```bash
nano /etc/frr/frr.conf
```

**На HQ-R:**
```
/etc/frr/frr.conf
router ospf network 192.168.1.0/26 area 0 network 100.0.0.0/26 area 0
```
**На BR-R:**
```
/etc/frr/frr.conf
router ospf network 192.168.2.0/28 area 0 network 200.0.0.0/28 area 0
```
Перезапустите FRR на каждом роутере, чтобы применить изменения
```bash
systemctl restart frr
```
Прежде чем проверять, необходимо выполнить задание 3

#### Задание 3 - автоматическое распределение IP-адресов на роутере HQ-R
Установка DHCP сервера
```bash
apt install -y isc-dhcp-server
```

Конфиг DHCP сервера /etc/dhcp/dhcpd.conf
```conf
nano /etc/dhcp/dhcpd.conf

option domain-name-servers 192.168.1.10;

default-lease-time 600;
max-lease-time 7200;

subnet 192.168.1.0 netmask 255.255.255.0 { 
	range 192.168.1.100 192.168.1.200; 
	option routers 192.168.1.1;
}

host server { 
	hardware ethernet 00:11:22:33:44:55; 
	fixed-address 192.168.1.10;
} 
```

Чтобы узнать hardware ethernet, то есть mac адрес порта сервера. Необходимо перейти на HQ-SRV в nmtui, выбрать соединения и в поле device будет указан mac адрес
![](https://github.com/DevLn737/DEMO24/blob/main/%D0%98%D0%B7%D0%BE%D0%B1%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F/Pasted%20image%2020240612093216.png)

- `subnet 192.168.1.0 netmask 255.255.255.0` определяет сеть и маску подсети.
- `range 192.168.1.100 192.168.1.200` определяет диапазон адресов для выделения.
- `option routers 192.168.1.1` указывает шлюз по умолчанию (адрес роутера).
- `option domain-name-servers 192.168.1.10` указывает DNS-сервер (HQ-SRV).
- `host server {...}` резервирует адрес для сервера по его MAC-адресу.
Далее в файле /etc/default/isc-dhcp-server
```
nano /etc/default/isc-dhcp-server
Содержимое должно быть 

INTERFACESv4="ens19"
#INTERFACESv6=""
```
Примечание: ens19(название интерфейса внутренней сети)

После сохранения конфига, необходимо перезапустить dhcp сервер
```bash
systemctl restart isc-dhcp-server
```

Возможно может появится ошибках в логах, что якобы dhcpd мешает запуститься другой dhcpd, необходимо перезагрузить систему и возможно проблема пропадёт.
```bash
reboot
```


#### Задание 4 - Настройте локальные учётные записи на всех устройствах

Для добавления нового пользователя используйте команды `useradd` и `passwd`.

Команда добавит пользователя «admin» и вставит его полное имя, Administrator, в поле комментария.
```plaintext
# useradd -c "Admin" admin -U
# passwd admin
```
`admin` - имя пользователя  
`-c Admin` любая текстовая строка. Используется как поле для имени и фамилии пользователя  
`-U` - cоздание группы с тем же именем, что и у пользователя, и добавление пользователь в эту группу  
`passwd admin` - задать пароль пользователю

**HQ-R**
```plaintext
# useradd -c "Admin" admin -U
# passwd admin
< вводим пароль пользователя >
< повторяем ввод пароля >
```

Создание пользователя `Network admin`
```plaintext
# useradd -c "Network admin" network_admin -U
# passwd network_admin
< вводим пароль пользователя >
< повторяем ввод пароля >
```

**HQ-SRV**
Создание пользователя `Admin`
```plaintext
# useradd -c "Admin" admin -U
# passwd admin
< вводим пароль пользователя >
< повторяем ввод пароля >
```

**BR-R**
Создание пользователя `Branch admin`
```plaintext
# useradd -c "Branch admin" branch_admin -U
# passwd branch_admin
< вводим пароль пользователя >
< повторяем ввод пароля >
```

Создание пользователя `Network admin`
```plaintext
# useradd -c "Network admin" network_admin -U
# passwd network_admin
< вводим пароль пользователя >
< повторяем ввод пароля >
```

**BR-SRV**
Создание пользователя `Branch admin`
```plaintext
# useradd -c "Branch admin" branch_admin -U
# passwd branch_admin
< вводим пароль пользователя >
< повторяем ввод пароля >
```

Создание пользователя `Network admin`
```plaintext
# useradd -c "Network admin" network_admin -U
# passwd network_admin
< вводим пароль пользователя >
< повторяем ввод пароля >
```

**CLI**
Создание пользователя `Admin`
```plaintext
# useradd -c "Admin" admin -U
# passwd admin
< вводим пароль пользователя >
< повторяем ввод пароля >
```

Либо можно создать через gui в графической оболочке

#### Задание 5 - Измерьте пропускную способность сети между двумя узлами HQ-R-ISP

Устанавливаем iperf 3 на HQ-R и ISP
```
apt install -y iperf3
```

На ISP запускаем iperf в режиме сервера
```bash
iperf3 -s 
```

На HQ-R в режиме клиента и вводим адрес порта ISP
```bash
iperf3 -c 10.0.0.1
```

**Скриншот HQ-R**
![](https://github.com/DevLn737/DEMO24/blob/main/%D0%98%D0%B7%D0%BE%D0%B1%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F/Pasted%20image%2020240612105116.png)

**Скриншот ISP**
![](https://github.com/DevLn737/DEMO24/blob/main/%D0%98%D0%B7%D0%BE%D0%B1%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F/Pasted%20image%2020240612105407.png)

#### Задание 6 - Составьте backup скрипты для сохранения конфигурации сетевых устройств

**HQ-R и BR-R**
Создаём директорию если не существует и создаём файл backup-script.sh
```bash
mkdir -p /var/backup
nano /var/backup/backup-script.sh
```

backup-script.sh
```bash
#!/bin/bash

TIMESTAMP=$(date +"%Y%m%d_%H%M%S")

mkdir -p /var/backup/$TIMESTAMP
cp -r /etc/frr /var/backup/$TIMESTAMP
cp -r /etc/NetworkManager/system-connections /var/backup/$TIMESTAMP
cp -r /etc/dhcp /var/backup/$TIMESTAMP

cd /var/backup
tar czfv "./$TIMESTAMP.tar.gz" ./$TIMESTAMP
rm -r /var/backup/$TIMESTAMP

echo "$TIMESTAMP: Backup was successfully done!"
```

Задаём права доступа на и запускаем скрипт
```bash
chmod +x /var/backup/backup-script.sh
/var/backup/backup-script.sh
```

**Пример успешного выполнения скрипта**
![](https://github.com/DevLn737/DEMO24/blob/main/%D0%98%D0%B7%D0%BE%D0%B1%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F/Pasted%20image%2020240612112050.png)


#### Задание 7 - Подключение по SSH для удалённого конфигурирования устройства HQ-SRV

Заходим в настройки sshd

```bash
nano /etc/ssh/sshd_config
```

Найти и изменить строчку
```
Port 2222
```

Перезапустите службу ssh
```bash
systemctl restart ssh
systemctl restart sshd
```

Настраиваем на HQ-R перенаправление порта с 2222 на 22

```bash
iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-ports 2222
iptables -A INPUT -p tcp --dport 2222 -j ACCEPT
```

Сохраняем правила
```bash
mkdir /etc/iptables
sh -c "iptables-save > /etc/iptables/rules.v4"
```

Проверить работу можно прописав
```bash
iptables -t nat -L -v -n 
iptables -L -v -n
```

Вывод
![](https://github.com/DevLn737/DEMO24/blob/main/%D0%98%D0%B7%D0%BE%D0%B1%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F/Pasted%20image%2020240612192646.png)

#### Задание 8 - Настройте контроль доступа до HQ-SRV по SSH со всех устройств, кроме CLI

На HQ-SRV
```bash
iptables -A INPUT -p tcp -s 9.0.0.1 --dport 22 -j DROP
iptables -A INPUT -p tcp --dport -j ACCEPT
mkdir /etc/iptables
sh -c "iptables-save > /etc/iptables/rules.v4"
```
где `9.0.0.1` ip адрес CLI, в текущем случае указан интерфейс ISP, а не CLI, потому что не сделал CLI к этому моменту(что является ошибкой)

Проверить правила
```bash
iptables -L -v -n
```
### Модуль 2: Организация сетевого администрирования
---
#### Задание 1 -  Настройте DNS-сервер на сервере HQ-SRV

Установить легковесный dns сервер dnsmasq и для проверки dnsutils
```
apt update && apt install -y dnsmasq dnsutils
```

Далее добавить в конфиг файл по пути /etc/dnsmasq.conf
Переходим в файл с конфигом
```bash
nano /etc/dnsmasq.conf
```

Сам конфиг
```conf
domain-needed
bogus-priv
no-resolv

local=/hq.work/
local=/branch.work/

address=/hq-r.hq.work/10.0.0.2
address=/hq-srv.hq.work/192.168.1.10

ptr-record=2.0.0.10.in-addr.arpa,hq-r.hq.work
ptr-record=10.1.168.192.in-addr.arpa,hq-srv.hq.work

address=/br-r.branch.work/11.0.0.2
address=/br-srv.branch.work/192.168.2.10

ptr-record=2.0.0.11.in-addr.arpa,br-r.branch.work

server=8.8.8.8
server=1.0.0.1
```

Перезапускаем службу
```bash
systemctl restart dnsmasq
```

> [!NOTE]
> Важно! Убедиться, что у всех в dns прописано именно, 192.168.1.10.
> Это можно проверить в nmcli, и настроить в nmtui

Для проверки можно ввести что-нибудь из этого(что-нибудь, точно да сработает!)
```bash
dig hq-r.hq.work 
dig -x 10.0.0.2 
dig hq-srv.hq.work 
dig -x 192.168.1.10 
dig br-r.branch.work 
dig -x 11.0.0.2
```
#### Задание 2 - Настройте синхронизацию времени между сетевыми устройствами по протоколу NTP.

Устанавливаем на HQ-R пакет ntp
```bash
apt -y install ntp
```

В /etc/ntp.conf необходимо добавить 2 строчки
```bash
nano /etc/ntp.conf
```

```conf
tos stratum 5
interface listen lo:0
```

И перезапустить службу ntp
```bash
systemctl restart ntp
```

На всех остальных устройствах, необходимо также установить ntp и прописать внутри конфиг файла следующие
```bash
nano /etc/ntp.conf
```

HQ-SRV
```
server 192.168.1.1 iburst
```

BR-R, BR-SRV, ISP CLI
```
server 10.0.0.2 iburst
```

Установка московского часового пояса
```bash
timedatectl set-timezone Europe/Moscow
```

Для проверки состояния синхронизации
```bash
ntpq -p
```

Задание 3

=(

Задание 4
```bash
apt install samba
```

=(

Задание 5

=(
#### Задание 6 - Запустите сервис MediaWiki используя docker на сервере HQ-SRV

docker-compose wiki
Создаём тома 
```bash
docker volume create dbvolume
docker volume create images
```

Создаём wiki.yml с содержанием
```yaml
version: '3'
services:
  wiki:
    image: mediawiki
    container_name: wiki
    restart: on-failure
    ports:
      - 8080:80
    depends_on:
      - db
    volumes:
      - images:/var/www/html/images
      - ./LocalSettings.php:/var/www/html/LocalSettings.php
 
  db:
    image: mysql
    container_name: db
    restart: on-failure
    environment:
      MYSQL_DATABASE: mediawiki
      MYSQL_ROOT_PASSWORD: DEP@ssw0rd
    volumes:
      - dbvolume:/var/lib/mysql

volumes:
  dbvolume:
  images:
```
И в той же директории файл конфига
LocalSettings.php
```php
<?php
if ( !defined( 'MEDIAWIKI' ) ) {
    exit;
}
$wgSitename = "test";
$wgMetaNamespace = "Test";
$wgServer = "http://localhost:8080";
$wgEnableEmail = true;
$wgEmailAuthentication = true;


$wgScriptPath = "";
$wgResourceBasePath = $wgScriptPath;
wfLoadSkin( 'Vector' );


## Database settings
$wgDBtype = "mysql";
$wgDBserver = "db";
$wgDBname = "mediawiki";
$wgDBuser = "root";
$wgDBpassword = "DEP@ssw0rd";

  
$wgLanguageCode = "ru";
$wgLocaltimezone = "Europe/Samara";
  

$wgSecretKey = "verysecretkey";
```
Поднимаем контейнеры
```bash
docker compose -f wiki.yml up -d
```
Если надо роняем
```bash
docker compose -f wiki.yml down
```
Вики должна заработать в минимальном состоянии по адресу: http://localhost:8080


