# DemoExamSSA2025
Вариант по демострационному экзамену 2025 

## Модуль 0. Подготовка виртуальных машин
### 1. Виртуальные машины

**Таблица №1**
|  Машина  |  RAM, ГБ  |  CPU  |  HDD/SSD, ГБ  |  Операционная система  |
|  :---:  |  :---:  |  :---:  |  :---:  |  :---:  |
|  ISP  |  1  |  1  |  10  |  OC Debian 12 (No GUI)  |
|  HQ-RTR  |  1  |  1  |  0.128  |  CHR 7.18 (RouterOS)  |
|  BR-RTR  |  1  |  1  |  0.128  |  CHR 7.18 (RouterOS)  |
|  HQ-SRV  |  2  |  1  |  10  |  OC Debian 12 (No GUI)  |
|  BR-SRV  |  2  |  1  |  10  |  OC Debian 12 (No GUI)  |
|  HQ-CLI  |  3  |  2  |  15  |  OC Debian 12 (Cinnamon)  |
|  Итого  |  10  |  7  |  45.256  |  -  |  

### 2. Проверка репозиториев Debian
Для того чтобы выполнить задания необходимо наличие официальных репозиториев для стандартных пакетов (не находящихся на дисковом образе Debian) и стороних репозиториев для использования актуальных версий пакетов Samba, Docker, Moodle и Яндекс браузера.

Перечень файла /etc/apt/sources.list [репозиториев Debian](https://wiki.debian.org/SourcesList)
```
deb http://deb.debian.org/debian bookworm main non-free non-free-firmware
deb http://deb.debian.org/debian bookworm-updates main non-free non-free-firmware
deb http://security.debian.org/debian-security bookworm-security main non-free non-free-firmware
```

Для начало на каждую машину установим Network Manager, (который по умолчанию есть в дисковом образе) и который добавит автоматически профили для подключения по DHCP к сети
```
sudo apt install network-manager -y
```
> [!NOTE]
> На время будут подключен в качестве сетевого адаптера Мост/Bridged на машинах для скачивания и проверки репозиториев


Добавление [репозиториев Moodle](https://docs.moodle.org/405/en/Installing_Moodle_on_Debian_based_distributions) на HQ-SRV
```
sudo apt -y install lsb-release ca-certificates curl
curl -sSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb
sudo dpkg -i /tmp/debsuryorg-archive-keyring.deb
sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/deb.sury.org-php.gpg] https://ftp.mpi-inf.mpg.de/mirrors/linux/mirror/deb.sury.org/repositories/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list'
wget https://r.mariadb.com/downloads/mariadb_repo_setup
chmod +x mariadb_repo_setup
sudo ./mariadb_repo_setup
```

Добавление [репозиториев Docker](https://docs.docker.com/engine/install/debian/) на BR-SRV
```
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings	
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
```

Не обязательно, добавление [репозиториев Samba](https://samba.tranquil.it/doc/en/samba_config_server/debian/server_install_samba_debian.html) на HQ-SRV
```
wget -qO-  https://samba.tranquil.it/tissamba-pubkey.gpg | sudo tee /usr/share/keyrings/tissamba.gpg > /dev/null
sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/tissamba.gpg] https://samba.tranquil.it/debian/samba-4.20/ $(lsb_release -c -s) main" > /etc/apt/sources.list.d/tissamba.list'
sudo apt update
```

Не обязательно, добавление репозиториев Яндекс браузера на HQ-CLI
```
sudo apt install curl
curl -fsSL https://repo.yandex.ru/yandex-browser/YANDEX-BROWSER-KEY.GPG | gpg --dearmor | sudo tee /usr/share/keyrings/yandex.gpg > /dev/null
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/yandex.gpg] http://repo.yandex.ru/yandex-browser/deb stable main" | sudo tee /etc/apt/sources.list.d/yandex-browser.list
sudo apt update
```

> [!IMPORTANT]
> При проверки доступности репозиториев Яндекс может выводится, что данный репозиорий не найден\
> Стоит тогда повторить обновление репозиториев

![Проверка репозиторий](https://github.com/user-attachments/assets/ed1cb455-dfbf-48fb-b2b3-822a274eb459)\
**Рисунок 1**

### 3. Подключение виртуальных машин  

![Топология](https://github.com/user-attachments/assets/6e7b0b78-a47c-467f-a61b-08e421c27acf)\
**Рисунок 2 - Топология сети**

> [!WARNING]
> 1. Виртуальный коммутатор будет реализован на HQ-RTR.
> 2. Подключение у HQ-CLI и HQ-SRV будет использоваться один локальный сигмент HQ-Net, но будут использоваться теги у интерфейсов.
> 3. на HQ-RTR и BR-RTR будут использованны дополнительные адаптеры Только локальный/Host-Only который будут использоваться в качестве управления через winbox и будут настроенны в качестве VLAN 999.


## Модуль 1. Сетевая инфраструктура 
### 1. Базовая настройка

**Таблица 2**
|  Имя устройства   |  IP-адрес  |  Шлюз по умолчанию  |
|  ---  |  ---  |  ---  |
|  ISP ens33  |  192.168.x.x/24  |  192.168.x.x  |
|  -  ens34  |  172.16.4.1/28  |  -  |
|  -  ens35  |  172.16.5.1/28  |  -  |
|  HQ-RTR  ether1  |  172.16.4.2/28  |  172.16.4.1  |
|  -  vlan100 (ether2)  |  192.168.1.1/26  |  -  |
|  -  vlan200 (ether2)  |  192.168.2.1/28  |  -  |
|  -  vlan999 (ether3)  |  192.168.99.4/29  |  192.168.99.1  |
|  BR-RTR  ether1  |  172.16.5.2/28  |  172.16.5.1  |
|  -  ether2  |  192.168.3.1/27  |  -  |
|  -  vlan999 (ether3)  |  192.168.99.5/29  |  192.168.99.1  |
|  HQ-SRV ens33.100  |  192.168.1.62/26  |  192.168.1.1  |
|  HQ-CLI ens33.200  |  192.168.2.14/28  |  192.168.2.1  |
|  BR-SRV ens33  |  192.168.3.30/27  |  192.168.3.1  |


> [!WARNING]
> Настройка ISP проводится в следующем отдельном пункте, но указываем IP-адреса в таблицу для отчёта\
> Для HQ-RTR, HQ-SRV, HQ-CLI и BR-RTR IP-адреса связанные с VLAN устанавливаем в пункте с настройкой виртуального коммутатора и DHCP-сервера



Узнаем MAC-адреса маршутизаторов, для подключения через графический интерфейс.

![{835E93B1-6D16-4092-9D20-F83DEF4FD1E7}](https://github.com/user-attachments/assets/81e1fa9b-6fe9-41a1-a880-3505827c9c25)\
**Рисунок 3 - MAC-адрес**

![{A91A916F-F875-40E0-93AD-428FB5616221}](https://github.com/user-attachments/assets/3030f19f-38d0-4fb7-9ad8-4a9826471c6d)\
**Рисунок 4 - окно входа**

При первом подключение к Микротику пароль не требуется, но для использования терминала потребуется установить пароль

![{362CBBA7-DA8D-4EEC-A316-68CB9809DD63}](https://github.com/user-attachments/assets/75db4935-759b-4996-ae49-74953079d7d6)\
**Рисунок 5**

Теперь устанавливаем IP-адреса для маршутизаторов

Для HQ-RTR\
![{B25E7E4C-C6A5-4C06-B021-61057047D616}](https://github.com/user-attachments/assets/0ba8d36c-9867-4270-8562-cc2488d354b9)\
**Рисунок 6**

Командой
```
ip/address/add address=172.16.4.2/28 network=172.16.4.0 interface=ether1
```

BR-RTR\
![{A4E10ED6-56FF-45F9-996E-7A47004495F9}](https://github.com/user-attachments/assets/e04cc1f3-4201-4960-a447-f35fb15d8ce9)\
**Рисунок 7**

Командами
```
ip/address/add address=172.16.5.2/28 network=172.16.5.0 interface=ether1
ip/address/add address=192.168.3.1/27 network=192.168.3.0 interface=ether2
```

Изменяем имя маршутизаторов\
![{3B435704-FC83-47DA-9DBD-5BB9ECD79CF2}](https://github.com/user-attachments/assets/bc30c7b2-c93f-4639-89b0-963038f8a306)\
**Рисунок 8**

Командой
```
system/identity/set name=HQ-RTR.au-team.irpo
```

Для BR-SRV
```
sudo apt install network-manager
nmtui
ip -c a 
```
![{16DD9159-44BD-4EB9-9BAA-378B7650A6E5}](https://github.com/user-attachments/assets/18267998-d8ce-4d63-a0e0-eaf68cc213f9)\
**Рисунок 9 - nmtui**

![{E6C1803A-1580-438B-8D26-E4D241BA4B77}](https://github.com/user-attachments/assets/1073a629-d51d-43b6-af61-0114d5bfdf3c)\
**Рисунок 10 - IP-адрес**


Изменяем имя виртуальной машины
```
sudo hostnamectl set-hostname BR-SRV.au-team.irpo
sudo reboot
```


Также вносим изменение в файл /etc/hosts, для коректной работы
```
sudo nano /etc/hosts
127.0.1.1  ISP  BR-SRV.au-team.irpo
ctrl+x
y
enter
```
![{FD805E78-E06E-4F2D-A5AB-E256971D828C}](https://github.com/user-attachments/assets/5387a946-1ca5-4ed0-a014-610ca0b671fd)\
**Рисунок 11**


### 2. Конфигурация ISP

Изменяем имя виртуальной машины
```
sudo hostnamectl set-hostname ISP.au-team.irpo
sudo reboot
```

Также вносим изменение в файл /etc/hosts, для коректной работы
```
sudo nano /etc/hosts
127.0.1.1  ISP  ISP.au-team.irpo
ctrl+x
y
enter
```
![hosts](https://github.com/user-attachments/assets/cef966ec-3287-4b75-b2ee-ce065f76a356)\
**Рисунок 12 - /etc/hosts**

Устанавливаем Network-Manager, указываем IP адреса и проверяем их
```
sudo apt install network-manager -y
nmtui
ip -c a 
```

![nmtui](https://github.com/user-attachments/assets/a14fd562-c0f9-43c0-9d45-f465b997067d)\
**Рисунок 13 - nmtui**


![Адреса](https://github.com/user-attachments/assets/0d1d7a1e-dba5-4b0f-a9d4-6a214ce42e29)\
**Рисунок 14 - IP адреса ISP**

Устанавливаем iptables и настраиваем маршутизацию из внешной сети 
```
sudo apt install iptables
sudo sh -c 'echo 1 > /proc/sys/net/ipv4/ip_forward'
sudo iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE
```

Раскоменчиваем параметр net.ipv4.ip_forward=1
```
sudo nano /etc/sysctl.conf
net.ipv4.ip_forward=1
ctrl+z
y
enter
```

![ip_forward](https://github.com/user-attachments/assets/a6ac726e-434b-4faa-ae87-b5db2d3391a3)\
**Рисунок 15**

Cохраняем правило для iptables 
```
sudo apt install iptables-persistent
```

### 3. Локальные учетные записи
>[!NOTE]
>Тут предоставленна настройка на HQ-RTR, но настройка индентична для BR-RTR

Для HQ-RTR/BR-RTR\
Переходим в "System" во вкладку "Users" 
Создаем нового пользователя net_user\
![изображение](https://github.com/user-attachments/assets/5b498467-9174-463a-9abb-093223206c39)\
**Рисунок 16- создание пользователя**

Командой 
```
/user/add name=net_admin group=full password="P@$$word"
```

>[!NOTE]
>Тут предоставленна настройка на HQ-SRV, но настройка индентична для BR-SRV

Для HQ-SRV/BR-SRV\
Создаем пользователя, вводим пароль P@$$w0rd для него и добавляем в группу sudo.
```
sudo adduser sshuser -u 1010
sudo usermod -aG sudo sshuser
```

Для испозьзования sudo без аунтификации переходим в файл sudoers и добавляем строчку в самом конце.
```
sudo visudo
sshuser ALL=(ALL:ALL) NOPASSWD:ALL
ctrl+z
y
enter
```

![изображение](https://github.com/user-attachments/assets/1eb79afb-127e-41c9-b401-14f707199156)\
**Рисунок 17 - sudoers**


### 4. Виртуальный коммутатор HQ

Создаём подинтерфесы для ether2 и ether3
![{12339338-5674-4BAE-81CB-7986E31AD72F}](https://github.com/user-attachments/assets/e2477e86-2605-47c8-b76c-46c1547e0120)\
**Рисунок 18**

Командами 
```
interface/vlan/add name=vlan100 vlan-id=100 interface=ether2
interface/vlan/add name=vlan200 vlan-id=200 interface=ether2
interface/vlan/add name=vlan999 vlan-id=999 interface=ether3
```

Указываем IP-адреса для созданных подинтерфейсов

![{BDBABCED-39CD-48FA-BF40-2FF3F5790E72}](https://github.com/user-attachments/assets/5d936cd8-a2c8-4913-aa05-e870e24cadd7)\
**Рисунок 19**

Командами
```
ip/address/add address=192.168.1.2/26 network=192.168.1.0 interface=vlan100
ip/address/add address=192.168.2.1/28 network=192.168.2.0 interface=vlan200
ip/address/add address=192.168.99.4/29 network=192.168.99.0 interface=vlan999
```

На BR-RTR

![{FC4934E3-EC58-4C7D-AB04-207E9264B02F}](https://github.com/user-attachments/assets/4de4628e-14a4-4ce0-991b-b8a9444ab0f1)\
**Рисунок 20**

Командой
```
ip/address/add address=192.168.99.4/29 network=192.168.99.0 interface=vlan999
```

> [!WARNING]
> Данный выполняется после того как было настроенно на обоих маршутизаторах и необходимо для работы по VLAN999

Переходим в "настройку Виртуальных сетей" у VMware, изменяем IP-адрес и маску, и убираем галочку с работой DHCP-сервера

![{E85B35AA-B4B8-4776-9A7A-80EB8C67988A}](https://github.com/user-attachments/assets/9a0672ee-ca85-4e67-98c9-de24e6a7aaa2)\
**Рисунок 21**

После чего переходим к настройке адаптеров в "Панель управления"
Выбираем "VMware Network Adapter VMnet1"

![{65500167-E544-4D7C-84BD-F7EF88A4BD7A}](https://github.com/user-attachments/assets/03dcf518-cdc7-4b61-8feb-bbb53553b223)\
**Рисунок 22**

Переходим в "Свойства" > "Настроить" > "Дополнительно"
Там выбираем параметр "VLAN ID" и указываем значение 999

![{5D297F7E-3D1F-47A1-8E2D-82D785C5C976}](https://github.com/user-attachments/assets/8c938d5f-0e42-4a34-b19f-f75b3d5de8ae)\
**Рисунок 22**

После чего теперь в программе winbox можно будет подключаться через сеть управления указывая IP-адрес 

![{46A5F906-650E-4CDD-B2D9-776B66097934}](https://github.com/user-attachments/assets/88136d3d-b8f8-4221-ab7b-9448efb0a614)\
**Рисунок 24**


Переходим к настройке IP-адресов на HQ-SRV
```
sudo apt install network-manager
nmtui
ip -c a 
```
![{16649FED-EEA4-4FE9-B94E-03F996C634A3}](https://github.com/user-attachments/assets/c9df01b0-2813-47da-bead-a08b00e101b1)\
**Рисунок 25 - nmtui**

![{9CA78156-514A-42A2-A164-5812E6C69CC8}](https://github.com/user-attachments/assets/71d69127-dcba-4e0e-9cc2-6a272522986f)\
**Рисунок 26 - IP-адрес**


### 5. Удаленный доступ sshd (openssh-server)

Устанавливаем openssh-server и переходим в файл sshd_config
```
sudo apt install openssh-server -y
sudo nano /etc/ssh/sshd_config
```

Изменяем порт на 2025, ограничиваем количество попыток до двух (MaxAuthTries)
```
Port 2024
MaxAuthTries 2
```
![изображение](https://github.com/user-attachments/assets/e86aabc5-de61-4be3-804c-a0a04897ef53)\
**Рисунок 27**

Добавляем параметр AllowUsers sshuser, чтобы запретить вход с других пользователей, кроме sshuser
```
AllowUsers sshuser
```

![изображение](https://github.com/user-attachments/assets/187634c5-2f81-4cfc-a5ca-f08c05c3a5f7)\
**Рисунок 28**

Сохранаяем 
```
ctrl+z
y
enter
```
И перезагружаем службу sshd
```
sudo systemctl restart sshd
```
### 6. Туннель
> [!NOTE]
> Настройка будет происходить одновременно на двух маршрутизаторах

Переходим в "Interfaces" > "GRE Tunnel" и создаём на новый туннель.\
На HQ-RTR\
![изображение](https://github.com/user-attachments/assets/03f8b823-32f5-4e15-9876-2c20b9ca8145)\
**Рисунок 29**

На BR-RTR\
![изображение](https://github.com/user-attachments/assets/70c73afb-fffb-468f-9a44-62fa1a5e66ee)\
**Рисунок 30**

Командами\
Для HQ-RTR\
```
interface/gre/add name=HQ-BR local-address=172.16.4.2 remote-address=172.16.5.2 allow-fast-path=no ipsec-secret="P@$$w0rd"
```
Для BR-RTR\
```
interface/gre/add name=BR-HQ local-address=172.16.5.2 remote-address=172.16.4.2 allow-fast-path=no ipsec-secret="P@$$w0rd"
```

Устанавливаем IP-адрес для GRE-туннеля\
Для HQ-RTR\
![изображение](https://github.com/user-attachments/assets/d4178b69-186f-4f2f-80db-71ab327a0fa2)\
**Рисунок 31**

Для BR-RTR\
![изображение](https://github.com/user-attachments/assets/d13d3c76-dcc1-4305-bfb1-1b4cd6ef5016)\
**Рисунок 32**

Командами\
Для HQ-RTR
```
ip/address/add address=10.10.10.1/30 network=10.10.10.0 interface=HQ-BR
```
Для BR-RTR
```
ip/address/add address=10.10.10.2/30 network=10.10.10.0 interface=BR-HQ
```

### 7. Динамическая маршрутизация (OSPF)
>[!NOTE]
> Настройка индетнична для HQ-RTR и BR-RTR

Переходим в "Routing" > "OSPF"\
Создаём "Instances"\
![изображение](https://github.com/user-attachments/assets/66a4bb20-6bae-464b-a6ee-38dab3c434be)\
**Рисунок 33**

Создаём "Area"\
![изображение](https://github.com/user-attachments/assets/555832f0-ddba-405d-83ec-c58464a1b309)\
**Рисунок 34**

Создаём "Interfaces Templates"\
![изображение](https://github.com/user-attachments/assets/7c36610c-5e77-436e-bbdd-e49076d32fb6)\
**Рисунок 35**

Командами
```
routing/ospf/instance/add
routing/ospf/area/add area-id=10.10.10.0 instance=ospf-instance-1
routing/ospf/interface-template/add area=ospf-area-1 networks="10.10.10.0/30, 192.168.1.0/26, 192.168.2.0/28, 192.168.3.0/27"
```

### 8. Динамическая трансляция адресов (NAT)
>[!NOTE]
> Настройка индетнична для HQ-RTR и BR-RTR

Переходим в "IP", в раздел "Firewall", во вкладку "NAT" и создаём правило\
![изображение](https://github.com/user-attachments/assets/9ebaf955-3da8-45b8-b787-654aa29477de)\
**Рисунок 36**

![изображение](https://github.com/user-attachments/assets/4202a80b-c9a1-412f-913c-5ee1084eddd3)\
**Рисунок 37**


Командой
```
ip/firewall/nat/add chain=srcnat action=masquerade
```

### 9. DHCP-сервер на HQ-RTR
Переходим в "IP" > "Pool" и создаём пул адресов\
![изображение](https://github.com/user-attachments/assets/1335a3ba-da09-4894-98a9-4ce78578d602)\
**Рисунок 38**

Командой
```
ip/pool/add name=hq-pool ranges="192.168.2.2-192.168.2.14"
```

После чего переходим в "IP" > "DHCP Server" > "DHCP "и создаем сервер, указывая интерфейс vlan200 и пул hq-pool\
![изображение](https://github.com/user-attachments/assets/0dcb139b-ea52-4d09-9c73-fb9f3657940b)\
**Рисунок 38**
```
ip/dhcp-server/add name=dhcp-server interface=vlan200 address-pool=hq-pool
```

Переоходим в соседнюю вкладку "Network" и указываем параметры для DHCP сервера\
Параметры "DNS Server" и "Domain" можно заполнить заранее\
![изображение](https://github.com/user-attachments/assets/f90c1cc7-7fda-43e3-9d44-17badb004d00)

**Рисунок 40**

Командой
```
ip/dhcp-server/network/add address=192.168.2.1/28 gateway=192.168.2.0 netmask=28 dns-server=192.168.1.62 domain="au-team.irpo"
```

Переходим к HQ-CLI и настраиваем DHCP-клиент на нём\
![изображение](https://github.com/user-attachments/assets/12f2dd54-5ebc-45b7-b2df-cbcc27795746)\
**Рисунок 41**

### 10*. DNS-сервер на HQ-SRV (BIND9)
> [!IMPORTANT]
> Вариант настройки без использования доменного контролера Samba и модуля BIND-DLZ\
> Если хотите настроить DNS-сервер после доменного контролера, то переходите ко второму модулю.

Временно указываем открытый ДНС-сервер например Яндекса (77.88.8.8), для того чтобы скачать пакет bind9\
![изображение](https://github.com/user-attachments/assets/27e531cc-b9a6-42d9-9df8-c556419765a0)\
**Рисунок 42**

Скачиваем пакет bind9
```
sudo apt update
sudo apt install bind9 -y
```
После чего ставим IP-адрес HQ-SRV (или 127.0.0.1) и указываем домен\
![изображение](https://github.com/user-attachments/assets/f546537b-f115-45d9-8bf4-536858ce468d)\
**Рисунок 43**

>[!NOTE]
> Указываем данный ДНС-сервер и домен также на ISP, BR-SRV

Запускаем и добавляет автозагрузка
```
sudo systemctl start named
sudo systemctl enable named
```
Редактируем файл /etc/bind/named.conf.options
```
sudo nano /etc/bind/named.conf.options
ctrl+z
y
enter
```
![изображение](https://github.com/user-attachments/assets/ca4a2df4-f887-47f0-8c38-ca914a56c802)\
**Рисунок 44 - /etc/bind/named.conf.options**

```
sudo nano /etc/bind/named.conf.local
ctrl+z
y
enter
```
![изображение](https://github.com/user-attachments/assets/d6aff391-a236-43ee-a4e4-615fe37cda0d)\
**Рисунок 45 - /etc/bind/named.conf.local**


Копируем файлы c зонами localhost для того, чтобы на основе сделать новые две зоны: прямую и рекурсивную 
```
sudo cp /etc/bind/db.local /etc/bind/db.irpo
sudo cp /etc/bind/db.127 /etc/bind/db.172
```
Редактируем файл под прямую зону 
```
sudo nano /etc/bind/db.irpo
ctrl+z
y
enter
```
![изображение](https://github.com/user-attachments/assets/56372b32-f51c-43da-99f8-b88933684b21)\
**Рисунок 46 - db.irpo**

И рекурсивную зону
```
sudo nano /etc/bind/db.127
ctrl+z
y
enter
```
![изображение](https://github.com/user-attachments/assets/8489a772-6f64-4de6-9540-02519ef0b3fb)\
**Рисунок 47 - db.127**

Перезапускаем ДНС-сервер
```
sudo systemctl restart named
```

**Таблица 3**
|  Устройство  |  Запись  |  Тип   |
|  :---:  |  ---  |  ---  |
|  HQ-RTR  |  hq-rtr.au-team.irpo  |  A,PTR  |
|  BR-RTR  |  br-rtr.au-team.irpo  |  A  |
|  HQ-CLI  |  hq-cli.au-team.irpo  |  A,PTR  |
|  HQ-SRV  |  hq-srv.au-team.irpo  |  A,PTR  |
|  BR-SRV  |  br-srv.au-team.irpo  |  A  |
|  ISP  |  isp.au-team.irpo  |  A  |
|  ISP  |  moodle.au-team.irpo  |  CNAME  |
|  ISP  |  wiki.au-team.irpo  |  CNAME  |


### 11. Часовой пояс
> [!TIP]
> Этот пукнт может быть выполнен при установки ОС
> Здесь часовой пояс указывается автора 

Для ISP/HQ-SRV/HQ-CLI/BR-SRV
```
timedatectl set-timezone Asia/Irkutsk
```
Для HQ-RTR/BR-RTR
Заходим в "System" > "Clock"\
![изображение](https://github.com/user-attachments/assets/0fe62699-83f5-40a0-a900-e63eb589723e)\
**Рисунок 48**

Командой
```
system/clock/set time-zone-name=Asia/Irkutsk
```

## Модуль 2. Организация сетевого администрирования операционных систем
### 1. Доменный контролер Samba на HQ-SRV
Временно указываем открытый ДНС-сервер например Яндекса (77.88.8.8), для того чтобы скачать пакеты свящанные с samba-ad-dc \
![изображение](https://github.com/user-attachments/assets/27e531cc-b9a6-42d9-9df8-c556419765a0)\
**Рисунок 49**

Устанавливаем пакеты с samba-ad-dc
```
sudo apt update
sudo apt install samba winbind libnss-winbind krb5-user smbclient ldb-tools python3-cryptography
```

Изменяем файл /etc/krb5.conf
> [!WARNING]
> Домен указывается ВЫСОКИМ РЕГИСТРОМ

```
sudo nano /etc/krb5.conf
  default_realm = AU-TEAM.IRPO
  dns_lookup_kdc = true
  dns_lookup_realm = false
ctrl+z
y
enter
```
![изображение](https://github.com/user-attachments/assets/da65ae21-a9a2-41e8-b2c0-2695f3e6ebb0)\
**Рисунок 50**

Удаляем файл /etc/samba/smb.conf и останавливаем службы 
```
sudo rm -f /etc/samba/smb.conf
sudo systemctl stop samba winbind nmbd smbd
```
Изменение роли на контролер домена
```
sudo samba-tool domain provision --realm=AU-TEAM.IRPO --domain AU-TEAM --server-role=dc
```

![изображение](https://github.com/user-attachments/assets/c8fd6994-fdc8-4c64-b4db-692a9f95a188)\
**Рисунок 51 - удачная установка домена**

Указываем пароль для администратора
```
sudo samba-tool user setpassword administrator
```
В файле /etc/samba/smb.conf указываем общедоступный ДНС в качестве сервера пересылки(Яндекс - 77.88.8.8) 
```
sudo nano /etc/samba/smb.conf
  dns forwarder = 77.88.8.8
ctrl+z
y
enter
```
![изображение](https://github.com/user-attachments/assets/0f2dd8c2-99eb-4ec5-ad4e-4ef2505c1ec6)\
**Рисунок 52**

После чего ставим IP-адрес HQ-SRV (или 127.0.0.1) и указываем домен\
![изображение](https://github.com/user-attachments/assets/f546537b-f115-45d9-8bf4-536858ce468d)\
**Рисунок 53**

Удаляем ненужный файл и создаём символьную ссылку
```
sudo rm -f /var/lib/samba/private/krb5.conf
sudo ln -s /etc/krb5.conf /var/lib/samba/private/krb5.conf
```

Активируем службу samba-ad-dc чтобы включалась вместе с машиной
```
sudo systemctl disable samba winbind nmbd smbd
sudo systemctl unmask samba-ad-dc
sudo systemctl enable samba-ad-dc
```

Перезапускаем машину и проверяем работу
```
kinit administrator
klist
```
![изображение](https://github.com/user-attachments/assets/ff963a29-45a6-44a0-9dc6-cddb32207575)\
**Рисунок 54**

Создаём пользователей user№.hq и вводим пароля
```
sudo samba-tool user create user1.hq
sudo samba-tool user create user2.hq
sudo samba-tool user create user3.hq
sudo samba-tool user create user4.hq
sudo samba-tool user create user5.hq
```
Создаем группу и добавлем туда пользователей
```
samba-tool group add hq
samba-tool group addmembers hq user1.hq
samba-tool group addmembers hq user2.hq
samba-tool group addmembers hq user3.hq
samba-tool group addmembers hq user4.hq
samba-tool group addmembers hq user5.hq
```

Настройка для ввода машины HQ-CLI в домен AU-TEAM.IRPO
```
sudo apt install sssd-ad sssd-tools realmd adcli
sudo realm -v discover HQ-SRV.au-team.irpo
sudo realm join -v HQ-SRV.au-team.irpo
```
![изображение](https://github.com/user-attachments/assets/6c8ab7c5-406d-478a-bb91-e739fed59c7c)
**Рисунок 55 - Машина удачно добавленна в домен**


> [!IMPORTANT]
> Данное заание будет дополненно позже


### 2*. DNS-сервер на HQ-SRV (BIND-DLZ)
> [!IMPORTANT]
> ИМХО Этот пункт должен находится здесь, так как настройка DNS сервера при наличии и отсуствии доменного контролера сильно различается

Перевод с SAMBA-INTERNAL на BIND-DLZ\
Устанавливаем bind9
```
sudo apt install bind9 -y
```

Редактируем файл /etc/bind/named.conf.options
```
sudo nano /etc/bind/named.conf.options
  forwarders { 77.88.8.8;};
  allow-query {  any;};
  dnssec-validation no;
  auth-nxdomain no;
  tkey-gssapi-keytab "/var/lib/samba/bind-dns/dns.keytab";sev
  minimal-responses yes;
ctrl+z
y
enter
```

![изображение](https://github.com/user-attachments/assets/fc17b45f-4188-4cd5-9f70-caea96d0c495)\
**Рисунок 56**

Редактируем файл /etc/bind/named.conf.local
```
sudo nano /etc/bind/named.conf.local
dlz "au-team.irpo" {
  database "dlopen /usr/lib/x86_64-linux-gnu/samba/bind9/dlz_bind9_18.so";
};
ctrl+z
y
enter
```
![изображение](https://github.com/user-attachments/assets/15850bb5-624a-4a33-9300-9b1d1bfe4407)\
**Рисунок 57**

Изменяем файл /etc/default/named
```
sudo nano /etc/default/named
OPTIONS="-4 -u bind"
ctrl+z
y
enter
```
![изображение](https://github.com/user-attachments/assets/26c69cec-6402-4fb5-bc0c-28d618812002)\
**Рисунок 58**

Изменяем параметр в файле /etc/samba/smb.conf, чтобы за ДНС-сервер отвечал bind9
```
sudo nano /etc/samba/smb.conf
server services = -dns
# dns forwarder = 77.88.8.8
ctrl+z
y
enter
```
![изображение](https://github.com/user-attachments/assets/d37fa431-24ce-48cd-95b7-dbf3fd2abf11)\
**Рисунок 59**

Создаём папку для ДНС-сервера и переключаем режим на BIND9_DLZ
```
sudo mkdir /var/lib/samba/bind-dns/dns
samba_upgradedns --dns-backend=BIND9_DLZ
```
![изображение](https://github.com/user-attachments/assets/3e7007ce-47b6-41e7-a976-171277c25dbf)\
**Рисунок 60 - обновление ДНС**

Создаём обратную зону и перезапускаем для активации её
```
sudo samba-tool dns zonecreate localhost 168.192.in-addr.arpa -U administrator
sudo systemct restart samba-ad-dc bind9 
```
> [!NOTE]
> Запись A для HQ-SRV автоматически создаются

Добавляем записи
```
#HQ-SRV
sudo samba-tool dns add localhost 168.192.in-addr.arpa 62.1 PTR HQ-SRV.au-team.irpo -U administrator

#HQ-RTR
sudo samba-tool dns add localhost au-team.irpo HQ-RTR A 192.168.1.1 -U administrator
sudo samba-tool dns add localhost au-team.irpo HQ-RTR A 192.168.2.1 -U administrator
sudo samba-tool dns add localhost 168.192.in-addr.arpa 1.1 PTR HQ-RTR.au-team.irpo -U administrator
sudo samba-tool dns add localhost 168.192.in-addr.arpa 1.2 PTR HQ-RTR.au-team.irpo -U administrator

#HQ-CLI
sudo samba-tool dns add localhost au-team.irpo HQ-CLI A 192.168.2.14 -U administrator
sudo samba-tool dns add localhost 168.192.in-addr.arpa 14.2 PTR HQ-CLI.au-team.irpo -U administrator

#BR-RTR
sudo samba-tool dns add localhost au-team.irpo BR-RTR A 192.168.3.1 -U administrator

#BR-SRV
sudo samba-tool dns add localhost au-team.irpo BR-SRV A 192.168.3.30 -U administrator

#ISP
sudo samba-tool dns add localhost au-team.irpo ISP A 172.16.4.1 -U administrator
sudo samba-tool dns add localhost au-team.irpo ISP A 172.16.5.1 -U administrator
sudo samba-tool dns add localhost au-team.irpo wiki CNAME ISP.au-team.irpo -U administrator
sudo samba-tool dns add localhost au-team.irpo moodle CNAME ISP.au-team.irpo -U administrator
```

**Таблица 4**
|  Устройство  |  Запись  |  Тип   |
|  :---:  |  ---  |  :---:  |
|  HQ-RTR  |  hq-rtr.au-team.irpo  |  A,PTR  |
|  HQ-CLI  |  hq-cli.au-team.irpo  |  A,PTR  |
|  BR-RTR  |  br-rtr.au-team.irpo  |  A  |
|  BR-SRV  |  br-srv.au-team.irpo  |  A  |
|  ISP  |  isp.au-team.irpo  |  A  |
|  ISP  |  moodle.au-team.irpo  |  CNAME  |
|  ISP  |  wiki.au-team.irpo  |  CNAME  |


### 3. Сетевое файлое хранилище (NFS сервер)
> [!IMPORTANT]
> Перед начало следует выключить и добавить 3 виртуальных жестких диска

Добавим 3 виртуальных жестких дисках по 1 ГБ в VmWare Workstation для BR-SRV\
Переходим "Edit virtual machine settings" > "Add.." > "Hard Disk"\
![изображение](https://github.com/user-attachments/assets/b79d9ac1-a4ae-4f61-84fa-9c24287c14ee)\
**Рисунок 61**

Включаем и ставит пакет mdadm
``` 
sudo apt install mdadm -y
```

Просматриваем имеющийся диски и запоминаем их имена
```
lsblk
```
![изображение](https://github.com/user-attachments/assets/677a6e25-b792-41bf-9c29-a4ca35e08b9a)\
**Рисунок 62 - список дисков**

Создаём raid-массив и форматируем его в файловой системе ext4
```
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
sudo mkfs.ext4 -F /dev/md0
```
Создаём папку и создаём точку подключения
```
sudo mkdir /raid5
sudo mount /dev/md0 /raid5
```
Просмотр точек подключения
```
df -h
```
![изображение](https://github.com/user-attachments/assets/7479f413-bcac-43cf-81b0-18d7dc373095)\
**Рисунок 63**

Считываем характериски массиав и записываем в файл /etc/mdadm/mdadm.conf, чтобы массив автоматически подключался после перезагрузки 
```
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
sudo echo '/dev/md0 /raid5 ext4 defaults,nofail,discard 0 0' | sudo tee -a /etc/fstab
```

Устанавливаем пакет nfs-kernel-server и запускаем его
```
sudo apt install nfs-kernel-server -y 
sudo systemctl start nfs-kernel-server
sudo systemctl enable nfs-kernel-server
```
Создаём папку которая будет использоваться как сетевая папка и выдаём особые права
```
sudo mkdir -p /raid5/nfs
sudo chown nobody:nogroup /raid5/nfs 
sudo chmod 755 /raid5/nfs
```

Прописываем параметры для того что можно было подключится к сетевой папке и принимаем их
```		
sudo echo '/raid5/nfs 192.168.2.0/28(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -a
```

На HQ-CLI\
Скачиваем пакет nfs-common и подключаем сетевую папку
```
sudo apt install nfs-common -y 
sudo mkdir /mnt/nfs
sudo echo '192.168.3.30:/raid5/nfs /mnt/nfs nfs4 defaults,user,exec,_netdev 0 0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
```
### 4. Служба сетевого времени (NTP сервер)
### 5. Служба Ansible на BR-SRV
### 6. Docker compose на BR-SRV
> [!TIP]
> для скачивания можно поменять на Мост/Bridged и после вернуть
Устанавливаем Docker
```
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose docker-compose-plugin
```

```
sudo usermod -aG docker $USER
sudo chown $USER /var/run/docker.sock
```
Создаём конфигурационный файл для MediaWiki
```
sudo nano wiki.yml
```
> [!WARNING]
> Внимательно следите за отступами, оступы делаются пробелом, а не "TAB"

Пример конфигурационного файла
```
services:
 MediaWiki:
  container_name: wiki
  image: mediawiki
  restart: always
  ports:
   - 8080:80
  links:
   - database
  volumes:
   - images:/var/www/html/images
   #- ./LocalSettings.php:/var/www/html/LocalSettings.php
 database:
  container_name: db
  image: mysql
  restart: always
  environment:
   MYSQL_DATABASE: mediawiki
   MYSQL_USER: wiki
   MYSQL_PASSWORD: DEP@ssw0rd
   MYSQL_RANDOM_ROOT_PASSWORD: 'yes'	
  volumes:
    - dbvolume:/var/lib/mysql

volumes:
 images:
 dbvolume:
  external: true
```

Создаём отдельный раздел для БД Вики
```
sudo docker volume create --name=dbvolume
```
Запускаем докер указывая конфигурационный файл
```
docker compose -f wiki.yml up
```

### 7. Статическая трансляция портов
### 8. Сервис Moodle на HQ-SRV
> [!WARNING]
> Задание будет переработано 

### 9. Обратный прокси-сервер (nginx) на ~~HQ-RTR~~ ISP
### 10. Яндекс Браузер

> [!TIP]
> для скачивания можно поменять на Мост/Bridged и после вернуть

HQ-CLI
```
sudo apt install yandex-browser-corporate
```

