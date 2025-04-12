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
> Для HQ-RTR, HQ-SRV, HQ-CLI и BR-RTR IP-адреса связанные с VLAN устанавливаем в пункте с настройкой виртуального коммутатора



Узнаем MAC-адреса маршутизаторов, для подключения через графический интерфейс.

![{835E93B1-6D16-4092-9D20-F83DEF4FD1E7}](https://github.com/user-attachments/assets/81e1fa9b-6fe9-41a1-a880-3505827c9c25)\
**Рисунок 3**

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
**Рисунок  - /etc/hosts**

Устанавливаем Network-Manager, указываем IP адреса и проверяем их
```
sudo apt install network-manager -y
nmtui
ip -c a 
```

![nmtui](https://github.com/user-attachments/assets/a14fd562-c0f9-43c0-9d45-f465b997067d)\
**Рисунок  - nmtui**


![Адреса](https://github.com/user-attachments/assets/0d1d7a1e-dba5-4b0f-a9d4-6a214ce42e29)\
**Рисунок  - IP адреса ISP**

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
**Рисунок**

Cохраняем правило для iptables 
```
sudo apt install iptables-persistent
```

### 3. Локальные учетные записи
>[!NOTE]
>Тут предоставленна настройка на HQ-SRV, но настройка индентична для BR-SRV


### 4. Виртуальный коммутатор HQ

Создаём подинтерфесы для ether2 и ether3
![{12339338-5674-4BAE-81CB-7986E31AD72F}](https://github.com/user-attachments/assets/e2477e86-2605-47c8-b76c-46c1547e0120)\
**Рисунок**

Командами 
```
interface/vlan/add name=vlan100 vlan-id=100 interface=ether2
interface/vlan/add name=vlan200 vlan-id=200 interface=ether2
interface/vlan/add name=vlan999 vlan-id=999 interface=ether3
```

Указываем IP-адреса для созданных подинтерфейсов

![{BDBABCED-39CD-48FA-BF40-2FF3F5790E72}](https://github.com/user-attachments/assets/5d936cd8-a2c8-4913-aa05-e870e24cadd7)\
**Рисунок**

Командами
```
ip/address/add address=192.168.1.2/26 network=192.168.1.0 interface=vlan100
ip/address/add address=192.168.2.1/28 network=192.168.2.0 interface=vlan200
ip/address/add address=192.168.99.4/29 network=192.168.99.0 interface=vlan999
```

На BR-RTR

![{FC4934E3-EC58-4C7D-AB04-207E9264B02F}](https://github.com/user-attachments/assets/4de4628e-14a4-4ce0-991b-b8a9444ab0f1)\
**Рисунок**

Командой
```
ip/address/add address=192.168.99.4/29 network=192.168.99.0 interface=vlan999
```

> [!WARNING]
> Данный выполняется после того как было настроенно на обоих маршутизаторах и необходимо для работы по VLAN999

Переходим в "настройку Виртуальных сетей" у VMware, изменяем IP-адрес и маску, и убираем галочку с работой DHCP-сервера

![{E85B35AA-B4B8-4776-9A7A-80EB8C67988A}](https://github.com/user-attachments/assets/9a0672ee-ca85-4e67-98c9-de24e6a7aaa2)\
**Рисунок**

После чего переходим к настройке адаптеров в "Панель управления"
Выбираем "VMware Network Adapter VMnet1"

![{65500167-E544-4D7C-84BD-F7EF88A4BD7A}](https://github.com/user-attachments/assets/03dcf518-cdc7-4b61-8feb-bbb53553b223)\
**Рисунок**

Переходм в "Свойства",  "Настроить" во вкладку "Дополнительно"
Там выбираем параметр "VLAN ID" и указываем значение 999

![{5D297F7E-3D1F-47A1-8E2D-82D785C5C976}](https://github.com/user-attachments/assets/8c938d5f-0e42-4a34-b19f-f75b3d5de8ae)\
**Рисунок**

После чего теперь в программе winbox можно будет подключаться через сеть управления указывая IP-адрес 

![{46A5F906-650E-4CDD-B2D9-776B66097934}](https://github.com/user-attachments/assets/88136d3d-b8f8-4221-ab7b-9448efb0a614)\
**Рисунок**


Переходим к настройке IP-адресов на HQ-SRV
```
sudo apt install network-manager
nmtui
ip -c a 
```
![{16649FED-EEA4-4FE9-B94E-03F996C634A3}](https://github.com/user-attachments/assets/c9df01b0-2813-47da-bead-a08b00e101b1)\
**Рисунок - nmtui**

![{9CA78156-514A-42A2-A164-5812E6C69CC8}](https://github.com/user-attachments/assets/71d69127-dcba-4e0e-9cc2-6a272522986f)\
**Рисунок - IP-адрес**


### 5. Удаленный доступ sshd (openssh-server)
### 6. Туннель
### 7. Динамическая маршрутизация (OSPF)
### 8. Динамическая трансляция адресов (NAT)
### 9. DHCP-сервер на HQ-RTR
### 10*. DNS-сервер на HQ-SRV (BIND9)
> [!IMPORTANT]
> Вариант настройки без использования доменного контролера Samba и модуля BIND-DLZ\
> Если хотите настроить DNS-сервер после доменного контролера, то переходите ко второму модулю.

**Таблица 3**
|  Устройство  |  Запись  |  Тип   |
|  :---:  |  ---  |  ---  |
|  HQ-RTR  |  hq-rtr.au-team.irpo  |  A,PTR  |
|  BR-RTR  |  br-rtr.au-team.irpo  |  A  |
|  HQ-CLI  |  hq-cli.au-team.irpo  |  A,PTR  |
|  HQ-SRV  |  hq-srv.au-team.irpo  |  A,PTR  |
|  BR-SRV  |  br-srv.au-team.irpo  |  A  |
|  ISP  |  moodle.au-team.irpo  |  CNAME  |
|  ISP  |  wiki.au-team.irpo  |  CNAME  |




### 11. Часовой пояс
> [!TIP]
> этот пукнт может быть выполнен при установки ОС


## Модуль 2. Организация сетевого администрирования операционных систем
### 1. Доменный контролер Samba на HQ-SRV
### 2*. DNS-сервер на HQ-SRV (BIND-DLZ)
> [!IMPORTANT]
> ИМХО Этот пункт должен находится здесь, так как настройка DNS сервера при наличии и отсуствии доменного контролера сильно различается

**Таблица 4**
|  Устройство  |  Запись  |  Тип   |
|  :---:  |  ---  |  :---:  |
|  HQ-RTR  |  hq-rtr.au-team.irpo  |  A,PTR  |
|  BR-RTR  |  br-rtr.au-team.irpo  |  A  |
|  BR-SRV  |  br-srv.au-team.irpo  |  A  |
|  ISP  |  moodle.au-team.irpo  |  CNAME  |
|  ISP  |  wiki.au-team.irpo  |  CNAME  |

> [!NOTE]
> Записи  A, PTR для HQ-SRV и HQ-CLI автоматически создаются, для HQ-SRV после развертывания домена, а для HQ-CLI после ввода машины в домен  

### 3. Сетевое файлое хранилище (NFS сервер)
``` 
sudo apt install mdadm
lsblk 
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
sudo mkfs.ext4 -F /dev/md0
sudo mkdir /raid5
sudo mount /dev/md0 /raid5
```
просмотр точек подключения
```
df -h
```
```
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
sudo echo '/dev/md0 /raid5 ext4 defaults,nofail,discard 0 0' | sudo tee -a /etc/fstab
``` 
```
sudo apt install nfs-kernel-server -y 
sudo systemctl start nfs-kernel-server
sudo systemctl enable nfs-kernel-server

sudo mkdir -p /raid5/nfs
sudo chown nobody:nogroup /raid5/nfs 
sudo chmod 755 /raid5/nfs			
sudo echo '/raid5/nfs 192.168.2.0/28(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
```
```
sudo exportfs -a
```
```
sudo apt install nfs-common -y 
sudo mkdir /mnt/nfs
sudo echo '192.168.3.30:/raid5/nfs /mnt/nfs nfs4 defaults,user,exec,_netdev 0 0' | sudo tee -a /etc/fstab
```
```
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

### 9. Обратный прокси-сервер (nginx) на ~~HQ-RTR~~ ISP
### 10. Яндекс Браузер

> [!TIP]
> для скачивания можно поменять на Мост/Bridged и после вернуть

HQ-CLI
```
sudo apt install yandex-browser-corporate
```

