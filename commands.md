 ISP:
   задание 1
   # hostnamectl set-hostname isp.au-team.irpo
   # bash
   
   задание 2
   # nano /etc/network/interfaces
      auto eth0
      iface eth0 inet dhcp
      auto eth1
      iface eth1 inet static
       address 172.16.1.1/28
      auto eth2
      iface eth2 inet static
       address 172.16.2.1/28
  # systemctl restart networking
  # nano /etc/sysctl.conf 
    и добавляем в конец файла следующую строку
      net.ipv4.ip_forward=1
  # sysctl -p
    далее настроим NAT
  # iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    Реализуем автозагрузку созданных правил
  # iptables-save > /etc/rules.v4
    после ввода данной команды открываем crontab следующей командой
  # crontab -e
    Будет предложен выбор текстового редактора пишем 1 и нажимаем Enter 
    добавляем в конец файла следующую строки:
    @reboot /sbin/iptables-restore < /etc/rules.v4
    @reboot /sbin/sysctl -p
  
  задание 11
  # timedatectl set-timezone Asia/Yekaterinburg
  # timedatectl   
  
HQ-RTR:
  задание 1
  # hostnamectl set-hostname hq-rtr.au-team.irpo
  # bash
  # nano /etc/network/interfaces
    auto eth0 
    iface eth0 inet static 
      address 172.16.1.2/28
      gateway 172.16.1.1
    auto eth1.100
    iface eth1.100 inet static
      address 192.168.1.1/27
      vlan_raw_device eth1
    auto eth1.200
    iface eth1.200 inet static
      address 192.168.2.1/28
      vlan_raw_device eth1
    auto eth1.999
    iface eth1.999 inet static
      address 192.168.99.1/29
      vlan_raw_device eth1
  # systemctl restart networking

  задание 3
  # useradd -m net_admin -u 2026
  # passwd net_admin
  Новый пароль: P@ssword
  Повторный ввод нового пароля: P@ssw0rd
  # usermod -aG sudo net_admin
  # mkdir /home/net_admin
  далее отредактируем файл sudoers
  # nano /etc/sudoers
    находим строку, начинающуюся на “%sudo” и изменяем её значение на
    %sudo  ALL=(ALL:ALL) NOPASSWD: ALL
    проверяем изменения 
  # su net_admin
  # sh-4.4$ sudo ls
  # sh-4.4$ su root
  Пароль: P@ssw0rd

  задание 6
  # nano /etc/network/interfaces
    auto gre1
    iface gre1 inet tunnel
    address 10.0.0.1
    netmask 255.255.255.252
    mode gre
    local 172.16.1.2  "адрес интерфейса HQ-RTR в сторону ISP"
    endpoint 172.16.2.2  "адрес интерфейса BR-RTR в сторону ISP"
    ttl 255
  # systemctl restart networking
    проверить наличие созданного интерфейса можно командой:
  # ip -br a

  задание 7
  # echo "nameserver 8.8.8.8" > /etc/resolv.conf
  # nano /etc/apt/sources.list
    и добавляем следующую строку
    deb [trusted=yes] https://archive.debian.org/debian buster main
  # apt update
    После уже можно устанавливать пакет frr
  # apt install frr -y
    Далее для активации ospf протокола необходимо активировать его демон
  # nano /etc/frr/daemons
    находим строку ospfd=no и переводим её в значение yes
    после чего перезапускаем сервис frr и добавляем его в автозапуск
  # systemctl enable frr
  # systemctl restart frr
   далее входим в интерфейс frr
  # vtysh
    # conf t
    # router ospf
    # network 192.168.1.0/27 area 0
    # network 192.168.2.0/28 area 0
    # network 192.168.99.0/29 area 0
    # exit
    # int gre1
    # ip ospf authentication message-digest
    # ip ospf message-digest-key 1 md5 P@ssword
    # exit
    # do wr mem
    # router ospf
    # network 10.0.0.0/30 area 0
    # do wr mem
  проверить полученные маршруты можно с помощью команды окружения frr
 # do show ip ospf route
 
 задание 8
 для перекидывания пакетов между интерфейсами нужно произвести следующие операции открываем sysctl.conf
 # nano /etc/sysctl.conf
  и добавляем в конец файла следующую строку
 # net.ipv4.ip_forward=1
  сохраняем файл и применяем введенное
 # sysctl -p
  далее настроим NAT
 # iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE 
  Реализуем автозагрузку созданных правил
 # iptables-save > /etc/rules.v4
  после ввода данной команды открываем crontab следующей командой
 # crontab -e
  Будет предложен выбор текстового редактора пишем 1 и нажимаем Enter добавляем в конец файла следующую строки:
  
  @reboot /sbin/iptables-restore < /etc/rules.v4
  @reboot /sbin/sysctl -p

 задание 9
 # apt-get update && apt-get install -y isc-dhcp-server
 # nano /etc/default/isc-dhcp-server
  Его содержимое:
      INTERFACESv4="eth1.200"
      #INTERFACESv6=""
  Перед настройкой конфигурационного файла сохраним его копию, для этого введем следующие команды:
 # cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.bkp
  Так, теперь перейдем к изменению конфигурационного файла:
 # nano /etc/dhcp/dhcpd.conf
  Приведём к виду:
    #option domain-name "example.org";
    #option domain-name-server ns1.example.org, ns2.example.org;
    authoritative;
    subnet 192.168.2.0 netmask 255.255.255.240 {
      range 192.168.2.2 192.168.2.10;
      option domain-name-servers 192.168.1.2;
      option domain-search "au-team.irpo";
      option routers 192.168.2.1;
    }
    Когда мы изменили файл для этого вида, идём дальше.
 # systemctl restart isc-dhcp-server

 задание 11
 # timedatectl set-timezone Asia/Yekaterinburg
 # timedatectl
 
BR-RTR:
 задание 1 
 # hostnamectl set-hostname br-rtr.au-team.irpo
 # bash
 # nano /etc/network/interfaces
   auto eth0
   iface eth0 inet static
     address 172.16.2.2/28
     gateway 172.16.2.1
   auto eth1
   iface eth1 inet static
     address 192.168.3.1/28
 # systemctl restart networking
 
 задание 3
 # useradd -m net_admin -u 2026
 # passwd net_admin
   Новый пароль: P@ssw0rd
   Повторите ввод нового пароля: P@ssw0rd
 # usermod -aG sudo net_admin
 # mkdir /home/net_admin
   далее отредактируем файл sudoers
 # nano /etc/sudoers
   находим строку, начинающуюся на “%sudo” и изменяем её значение на:
     %sudo  ALL=(ALL:ALL) NOPASSWD: ALL
  проверяем изменения:
 # su net_admin
   sudo ls
   su root
   Пароль: P@ssw0rd
   
 задание 6
 # nano /etc/network/interfaces
   после чего вписываем в конец файла данные строки:
     auto gre1
     iface gre1 inet tunnel
       address 10.0.0.2
       netmask 255.255.255.252
     mode gre
     local 172.16.2.2
     endpoint 172.16.1.2
     ttl 255
  # systemctl restart networking
    проверить наличие созданного интерфейса можно командой
  # ip -br a
    Также можно проверить связь с другим роутером, в случае успеха удаленный роутер будет отвечать на пинг запросы
    # ping 10.0.0.1
    
  задание 7
    Для начала нужно указать DNS-сервер для корректной работы установщика apt
  # echo "nameserver 8.8.8.8" > /etc/resolv.conf
    Затем необходимо подключить debian-репозиторий для установки пакета frr.
  # nano /etc/apt/sources.list
    и добавляем следующую строку
      deb [trusted=yes] https://archive.debian.org/debian buster main
  # apt update
    После уже можно устанавливать пакет frr
  # apt install frr -y
    Далее для активации ospf протокола необходимо активировать его демон
  # nano /etc/frr/daemons
    находим строку ospfd=no и переводим её в значение yes
    после чего перезапускаем сервис frr и добавляем его в автозапуск
  # systemctl enable frr
  # systemctl restart frr
   далее входим в интерфейс frr
  # vtysh
    # conf t
    # router ospf
    # network 10.0.0.0/30 area 0
    # network 192.168.3.0/28 area 0
    # exit
    # do wr mem
    # int gre1
    # ip ospf authentication message-digest
    # ip ospf message-digest-key 1 md5 P@ssword
    # exit
    # do wr mem
    после для выхода из интерфейса frr пишем
    quit
    exit
    
  задание 8
  # nano /etc/sysctl.conf
    и добавляем в конец файла следующую строку
  # net.ipv4.ip_forward=1
    сохраняем файл и применяем введенное
  # sysctl -p
    далее настроим NAT
  # iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    Реализуем автозагрузку созданных правил
  # iptables-save > /etc/rules.v4
    после ввода данной команды открываем crontab следующей командо
  # crontab -e
    Будет предложен выбор текстового редактора пишем 1 и нажимаем Enter добавляем в конец файла следующую строки:
  @reboot /sbin/iptables-restore < /etc/rules.v4
  @reboot /sbin/sysctl -p

  задание 11
  # timedatectl set-timezone Asia/Yekaterinburg
  # timedatectl
  
HQ-CLI:
  задание 1
  # hostnamectl set-hostname hq-cli.au-team.irpo
  # bash
  далее добавим vlan-тег к интерфейсу proxmox
  дабл клик по интерфейсу машины (HQ-CLI)-> Hardware -> Network Device (net0) -> ставим на интерфейсе клиента VLAN-тег 200

  задание 9
  Перезагружаем компьютер и смотрим на выданный ip-адрес:
  # ip a
  Пинг внешних источников будет работать, а домен нет т.к у нас не настроен DNS-Server на HQ-SRV. Перейдем к его настройке.

  задание 11
  # timedatectl set-timezone Asia/Yekaterinburg
  # timedatectl

HQ-SRV:
  задание 1
  # hostnamectl hostname hq-srv.au-team.irpo
  # bash
    Открываем файл конфигурации сетевого интерфейса
  # mcedit /etc/net/ifaces/ens18/options
  приводим файл к следующему виду:
  BOOTPROTO=static
  TYPE=eth
  DISABLED=no
    создаем файлы в директории интерфейса и приводим их к следующему виду
  # mcedit /etc/net/ifaces/ens18/ipv4address
    192.168.1.2/27
  # mcedit /etc/net/ifaces/ens18/ipv4route
    default via 192.168.1.1
  далее добавим vlan-тег к интерфейсу proxmox дабл клик по интерфейсу машины
   (HQ-SRV) -> Hardware -> Network Device (net0) -> ставим на интерфейсе клиента VLAN-тег 100
   
  задание 3
  # useradd sshuser -u 2026
  # passwd sshuser
    после чего дважды вводим задаваемый пароль P@ssw0rd
    далее добавляем пользователя в группу wheel
  # usermod -aG sshuser wheel
    после чего меняем файл sudoers
  # mcedit /etc/sudoers
    после чего ищем данную строку и убираем знак комментария “#”
      WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL
  # su sshuser
    sudo ls
    su root

  задание 5
  # mcedit /etc/openssh/sshd_config
    Port 2026
    AllowUsers sshuser
    PermitRootLogin no
    MaxAuthTries 2
    Banner /root/banner
    после сохраняем и закрываем файл
    затем создаем файл баннера, в конце обязательно оставляем пустую строку
  # mcedit /root/banner
      Authorized access only
    перезапускаем службу “sshd”
  # systemctl restart sshd
  # systemctl enable sshd
    проверяем внесенные изменения
  # ssh user@localhost -p 2026
  # ssh sshuser@localhost -p 2026

  задание 10
  # apt-get update && apt-get install -y dnsmasq 
    interface=* 
    server=8.8.8.8
    domain=au-team.irpo
    listen-address=192.168.1.2
    no-resolv
    no-hosts
    
    address=/hq-rtr.au-team.irpo/192.168.1.1
    ptr-record=1.1.168.192.in.addr.arpa,hq-rtr.au-team.irpo
    address=/br-rtr.au-team.irpo/192.168.3.1
    address=/hq-srv.au-team.irpo/192.168.1.2
    ptr-record=3.1.168.192.in.addr.arpa,hq-srv.au-team.irpo 
    address=/hq-cli.au-team.irpo/192.168.2.2
    ptr-record=2.2.168.192.in.addr.arpa,hq-cli.au-team.irpo
    address=/br-srv.au-team.irpo/192.168.3.2
    address=/docker.au-team.irpo/172.16.1.1
    address=/web.au-team.irpo/172.16.2.1
    Далее во избежание конфликтов необходимо отключить DNS службу BIND
 # systemctl disable bind --now
   И далее перезапускаем службу dnsmasq
 # systemctl restart dnsmasq
 # systemctl enable dnsmasq --now
   Настройка DNS произведена
 # systemctl status dnsmasq.service
   Проверка:
 # ping hq-rtr.au-team.irpo

 задание 11
 # timedatectl set-timezone Asia/Yekaterinburg
 # timedatectl

BR-SRV:
 задание 1
 # hostnamectl hostname br-srv.au-team.irpo
 # bash
   Открываем файл конфигурации сетевого интерфейса
 # mcedit /etc/net/ifaces/ens18/options
   приводим файл к следующему виду:    
   BOOTPROTO=static
   TYPE=eth
   DISABLED=no
   создаем файлы в директории интерфейса и приводим их к следующему виду
 # mcedit /etc/net/ifaces/ens18/ipv4address
   192.168.3.2/28
 # mcedit /etc/net/ifaces/ens18/ipv4route
   default via 192.168.3.1

   задание 3
   # useradd sshuser -u 2026
   # passwd sshuser
     после чего дважды вводим задаваемый пароль
     далее добавляем пользователя в группу wheel
   # usermod -aG sshuser wheel
     после чего меняем файл sudoers 
   # mcedit /etc/sudoers
     после чего ищем данную строку и убираем знак комментария “#”
       WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL
   # su sshuser
     sudo ls
     su root

   задание 5
   # mcedit /etc/openssh/sshd_config
      Port 2026
      AllowUsers sshuser
      PermitRootLogin no
      MaxAuthTries 2
      Banner /root/banner
    после сохраняем и закрываем файл
    затем создаем файл баннера, в конце обязательно оставляем пустую строку
  # mcedit /root/banner
      Authorized access only
    перезапускаем службу “sshd”
  # systemctl restart sshd
  # systemctl enable sshd
    проверяем внесенные изменения
  # ssh user@localhost -p 2026
  # ssh sshuser@localhost -p 2026

  задание 11
  # timedatectl set-timezone Asia/Yekaterinburg
  # timedatectl
