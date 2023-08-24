Vagrant-стенд c PXE

Цель домашнего задания
Отработать навыки установки и настройки DHCP, TFTP, PXE загрузчика и автоматической загрузки

Описание домашнего задания

1. Следуя шагам из документа https://docs.centos.org/en-US/8-docs/advanced-install/assembly_preparing-for-a-network-install  установить и настроить загрузку по сети для дистрибутива CentOS 8.
В качестве шаблона воспользуйтесь репозиторием https://github.com/nixuser/virtlab/tree/main/centos_pxe 
2. Поменять установку из репозитория NFS на установку из репозитория HTTP.
3. Настроить автоматическую установку для созданного kickstart файла (*) Файл загружается по HTTP.
* 4.  автоматизировать процесс установки Cobbler cледуя шагам из документа https://cobbler.github.io/quickstart/. 
Задание со звездочкой выполняется по желанию.

Формат сдачи ДЗ - vagrant + ansible

Создаем Vagrantfile с 2 ВМ - pxeserver и pxeclient, запускаем командой vagrant up. 
```
neva@Uneva:~$ vagrant status
Current machine states:

pxeserver                 running (virtualbox)
pxeclient                 running (virtualbox)
```

Подключаемся к pxeserver и устанавливаем http сервер. Так как у CentOS 8 закончилась поддержка, для установки пакетов нам потребуется поменять репозиторий. Сделать это можно с помощью следующих команд:

```
[root@pxeserver ~]# sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-Linux-*
[root@pxeserver ~]# sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-Linux-*
[root@pxeserver ~]# yum install httpd
Installed:
  apr-1.6.3-12.el8.x86_64                    apr-util-1.6.1-6.el8.x86_64                                 apr-util-bdb-1.6.1-6.el8.x86_64                                      apr-util-openssl-1.6.1-6.el8.x86_64
  centos-logos-httpd-85.8-2.el8.noarch       httpd-2.4.37-43.module_el8.5.0+1022+b541f3b1.x86_64         httpd-filesystem-2.4.37-43.module_el8.5.0+1022+b541f3b1.noarch       httpd-tools-2.4.37-43.module_el8.5.0+1022+b541f3b1.x86_64
  mailcap-2.1.48-3.el8.noarch                mod_http2-1.15.7-3.module_el8.4.0+778+c970deab.x86_64
```

Далее скачиваем образ CentOS 8.4.2150, монтируем его, создаём директорию iso, копируем в неё всё из скачанного образа и задаём права:

```
wget https://mirror.cs.pitt.edu/centos-vault/8.4.2105/isos/x86_64/CentOS-8.4.2105-x86_64-dvd1.iso
[root@pxeserver ~]# mount -t iso9660 CentOS-8.4.2105-x86_64-dvd1.iso /mnt -o loop,ro
[root@pxeserver ~]# mkdir /iso
[root@pxeserver ~]# cp -r /mnt/* /iso
[root@pxeserver ~]# chmod -R 755 /iso
```

Создаем конфигурационный файл и добавляем в него следующие строчки:

```
[root@pxeserver ~]# vi /etc/httpd/conf.d/pxeboot.conf
Alias /centos8 /iso
#Указываем адрес директории /iso
<Directory /iso>
    Options Indexes FollowSymLinks
    #Разрешаем подключения со всех ip-адресов
    Require all granted
</Directory>
```

Перезапускаем веб-сервер и добавляем его в автозагрузку:
```
[root@pxeserver ~]# systemctl restart httpd
[root@pxeserver ~]# systemctl enable httpd
Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service → /usr/lib/systemd/system/httpd.service.
```

Далее проверяем доступность сервера и каталога. Скриншот1, Скриншот2.


Переходим к настройке TFTP-сервера. Устанавливаем tftp-сервер и запускаем службу:

```
[root@pxeserver ~]# yum install tftp-server
Installed:
  tftp-server-5.2-24.el8.x86_64

Complete!

[root@pxeserver ~]# systemctl start tftp.service
```

Проверяем, в каком каталоге будут храниться файлы, которые будет отдавать TFTP-сервер:

```
[root@pxeserver ~]# systemctl status tftp.service
● tftp.service - Tftp Server
   Loaded: loaded (/usr/lib/systemd/system/tftp.service; indirect; vendor preset: disabled)
   Active: active (running) since Mon 2023-08-21 12:26:27 UTC; 3min 15s ago
     Docs: man:in.tftpd
 Main PID: 12998 (in.tftpd)
    Tasks: 1 (limit: 4953)
   Memory: 192.0K
   CGroup: /system.slice/tftp.service
           └─12998 /usr/sbin/in.tftpd -s /var/lib/tftpboot
```

Видим, что используется каталог /var/lib/tftpboot. Создаём каталог, в котором будем хранить наше меню загрузки:

```
[root@pxeserver ~]# mkdir /var/lib/tftpboot/pxelinux.cfg
```

Создаём меню-файл:
```
[root@pxeserver ~]# vi /var/lib/tftpboot/pxelinux.cfg/default

default menu.c32
prompt 0
#Время счётчика с обратным отсчётом (установлено 15 секунд)
timeout 150
#Параметр использования локального времени
ONTIME local
#Имя «шапки» нашего меню
menu title OTUS PXE Boot Menu
       #Описание первой строки
       label 1
       #Имя, отображаемое в первой строке
       menu label ^ Graph install CentOS 8.4
       #Адрес ядра, расположенного на TFTP-сервере
       kernel /vmlinuz
       #Адрес файла initrd, расположенного на TFTP-сервере
       initrd /initrd.img
       #Получаем адрес по DHCP и указываем адрес веб-сервера
       append ip=enp0s3:dhcp inst.repo=http://10.0.0.20/centos8
       label 2
       menu label ^ Text install CentOS 8.4
       kernel /vmlinuz
       initrd /initrd.img
       append ip=enp0s3:dhcp inst.repo=http://10.0.0.20/centos8 text
       label 3
       menu label ^ rescue installed system
       kernel /vmlinuz
       initrd /initrd.img
       append ip=enp0s3:dhcp inst.repo=http://10.0.0.20/centos8 rescue
```

Распакуем файл syslinux-tftpboot-6.04-5.el8.noarch.rpm:
```
[root@pxeserver ~]# rpm2cpio /iso/BaseOS/Packages/syslinux-tftpboot-6.04-5.el8.noarch.rpm | cpio -dimv
```
После распаковки в каталоге пользователя root будет создан каталог tftpboot из которого потребуется скопировать следующие файлы:

- pxelinux.0
- ldlinux.c32
- libmenu.c32
- libutil.c32
- menu.c32
- vesamenu.c32
```
[root@pxeserver ~]# cd tftpboot
[root@pxeserver tftpboot]# cp pxelinux.0 ldlinux.c32 libmenu.c32 libutil.c32 menu.c32 vesamenu.c32 /var/lib/tftpboot/
```

Также в каталог /var/lib/tftpboot/ нам потребуется скопировать файлы initrd.img и vmlinuz, которые располагаются в каталоге /iso/images/pxeboot/:

```
[root@pxeserver tftpboot]# cp /iso/images/pxeboot/{initrd.img,vmlinuz} /var/lib/tftpboot/
```

Далее перезапускаем TFTP-сервер и добавляем его в автозагрузку:

```
[root@pxeserver tftpboot]# systemctl restart tftp.service
[root@pxeserver tftpboot]# systemctl enable tftp.service
Created symlink /etc/systemd/system/sockets.target.wants/tftp.socket → /usr/lib/systemd/system/tftp.socket.
```

Настройка DHCP-сервера. 
Устанавливаем DHCP-сервер:
```
[root@pxeserver tftpboot]# yum install dhcp-server
Upgraded:
  dhcp-client-12:4.3.6-45.el8.x86_64                                             dhcp-common-12:4.3.6-45.el8.noarch                                             dhcp-libs-12:4.3.6-45.el8.x86_64
Installed:
  dhcp-server-12:4.3.6-45.el8.x86_64

Complete!
```

Правим конфигурационный файл:

```
vi /etc/dhcp/dhcpd.conf

option space pxelinux;
option pxelinux.magic code 208 = string;
option pxelinux.configfile code 209 = text;
option pxelinux.pathprefix code 210 = text;
option pxelinux.reboottime code 211 = unsigned integer 32;
option architecture-type code 93 = unsigned integer 16;

#Указываем сеть и маску подсети, в которой будет работать DHCP-сервер
subnet 10.0.0.0 netmask 255.255.255.0 {
        #Указываем шлюз по умолчанию, если потребуется
        #option routers 10.0.0.1;
        #Указываем диапазон адресов
        range 10.0.0.100 10.0.0.120;

        class "pxeclients" {
          match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
          #Указываем адрес TFTP-сервера
          next-server 10.0.0.20;
          #Указываем имя файла, который надо запустить с TFTP-сервера
          filename "pxelinux.0";
        }
}
```

Перезапускаем DHCP-сервер:

```
[root@pxeserver tftpboot]# systemctl start dhcpd.service
[root@pxeserver ~]# systemctl status dhcpd.service
● dhcpd.service - DHCPv4 Server Daemon
   Loaded: loaded (/usr/lib/systemd/system/dhcpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2023-08-21 15:02:50 UTC; 9s ago
     Docs: man:dhcpd(8)
           man:dhcpd.conf(5)
 Main PID: 4814 (dhcpd)
   Status: "Dispatching packets..."
    Tasks: 1 (limit: 4953)
   Memory: 4.8M
   CGroup: /system.slice/dhcpd.service
           └─4814 /usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid

Aug 21 15:02:50 pxeserver dhcpd[4814]:
Aug 21 15:02:50 pxeserver dhcpd[4814]: No subnet declaration for eth0 (10.0.2.15).
Aug 21 15:02:50 pxeserver dhcpd[4814]: ** Ignoring requests on eth0.  If this is not what
Aug 21 15:02:50 pxeserver dhcpd[4814]:    you want, please write a subnet declaration
Aug 21 15:02:50 pxeserver dhcpd[4814]:    in your dhcpd.conf file for the network segment
Aug 21 15:02:50 pxeserver dhcpd[4814]:    to which interface eth0 is attached. **
Aug 21 15:02:50 pxeserver dhcpd[4814]:
Aug 21 15:02:50 pxeserver dhcpd[4814]: Sending on   Socket/fallback/fallback-net
Aug 21 15:02:50 pxeserver dhcpd[4814]: Server starting service.
Aug 21 15:02:50 pxeserver systemd[1]: Started DHCPv4 Server Daemon.
```

На данном этапе мы закончили настройку PXE-сервера для ручной установки сервера. Давайте попробуем запустить процесс установки вручную, для удобства воспользуемся установкой через графический интерфейс:
В настройках виртуальной машины pxeclient рекомендуется поменять графический контроллер на VMSVGA и добавить видеопамяти. Видеопамять должна стать 20 МБ или больше. 
С такими настройками картинка будет более плавная и не будет постоянно мигать.
Нажимаем ОК, выходим из настроек ВМ и запускаем её.
Выбираем графическую установку:

После этого, будут скачаны необходимые файлы с веб-сервера

Как только появится окно установки, нам нужно будет поочереди пройти по всем компонентам и указать с какими параметрами мы хотим установить ОС:


После установки всех, нужных нам параметров нажимаем Begin installation
После этого начнётся установка системы, после установки всех компонентов нужно будет перезагрузить ВМ и запуститься с диска. 


Если нам не хочется вручную настраивать каждую установку, то мы можем автоматизировать этот процесс с помощью файла автоматиеской установки (kickstart file)

Настройка автоматической установки с помощью Kickstart-файла:

```
[root@pxeserver tftpboot]# vi /iso/ks.cfg

#version=RHEL8
#Использование в установке только диска /dev/sda
ignoredisk --only-use=sda
autopart --type=lvm
#Очистка информации о партициях
clearpart --all --initlabel --drives=sda
#Использование графической установки
graphical
#Установка английской раскладки клавиатуры
keyboard --vckeymap=us --xlayouts='us'
#Установка языка системы
lang en_US.UTF-8
#Добавление репозитория
url -—url=http://10.0.0.20/centos8/BaseOS/
#Сетевые настройки
network  --bootproto=dhcp --device=enp0s3 --ipv6=auto --activate
network  --bootproto=dhcp --device=enp0s8 --onboot=off --ipv6=auto --activate
network  --hostname=otus-pxe-client
#Устанвка пароля root-пользователю (Указан SHA-512 hash пароля 123)
rootpw --iscrypted $6$sJgo6Hg5zXBwkkI8$btrEoWAb5FxKhajagWR49XM4EAOfO/Dr5bMrLOkGe3KkMYdsh7T3MU5mYwY2TIMJpVKckAwnZFs2ltUJ1abOZ.
firstboot --enable
#Не настраиваем X Window System
skipx
#Настраиваем системные службы
services --enabled="chronyd"
#Указываем часовой пояс
timezone Europe/Moscow --isUtc
user --groups=wheel --name=val --password=$6$ihX1bMEoO3TxaCiL$OBDSCuY.EpqPmkFmMPVvI3JZlCVRfC4Nw6oUoPG0RGuq2g5BjQBKNboPjM44.0lJGBc7OdWlL17B3qzgHX2v// --iscrypted --gecos="val"

%packages
@^minimal-environment
kexec-tools

%end

%addon com_redhat_kdump --enable --reserve-mb='auto'

%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end
```

Добавляем параметр в меню загрузки:

```
root@pxeserver tftpboot]# vi /var/lib/tftpboot/pxelinux.cfg/default

default menu.c32
prompt 0
timeout 150
ONTIME local
menu title OTUS PXE Boot Menu
       label 1
       menu label ^ Graph install CentOS 8.4
       kernel /vmlinuz
       initrd /initrd.img
       append ip=enp0s3:dhcp inst.repo=http://10.0.0.20/centos8
       label 2
       menu label ^ Text install CentOS 8.4
       kernel /vmlinuz
       initrd /initrd.img
       append ip=enp0s3:dhcp inst.repo=http://10.0.0.20/centos8 text
       label 3
       menu label ^ rescue installed system
       kernel /vmlinuz
       initrd /initrd.img
       append ip=enp0s3:dhcp inst.repo=http://10.0.0.20/centos8 rescue
       label 4
       menu label ^ Auto-install CentOS 8.4
       #Загрузка данного варианта по умолчанию
       menu default
       kernel /vmlinuz
       initrd /initrd.img
       append ip=enp0s3:dhcp inst.ks=http://10.0.0.20/centos8/ks.cfg inst.repo=http://10.0.0.20/centos8/
```

После внесения данных изменений, можем перезапустить нашу ВМ pxeclient и проверить, что запустится процесс автоматической установки ОС.

```
Затем прописываем всю автоматизацию в Ансибле, добавив в Vagrantfile строки ниже в секцию про pxeserver и запускаем для проверки.


server.vm.provision "ansible" do |ansible|
    ansible.playbook = "Ansible/provision.yml"
    ansible.inventory_path = "Ansible/hosts"
    ansible.host_key_checking = "false"
    ansible.limit = "all"
