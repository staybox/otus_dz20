# OTUS ДЗ VPN (Centos 7)
-----------------------------------------------------------------------
### Домашнее задание
```
Цель: Студент получил навыки работы с VPN, RAS.
1. Между двумя виртуалками поднять vpn в режимах
- tun
- tap
Прочуствовать разницу.

2. Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на виртуалку

3*. Самостоятельно изучить, поднять ocserv и подключиться с хоста к виртуалке 
```

1. Между двумя виртуалками поднять vpn в режимах TUN и TAP

Отличия:
TUN наиболее часто используемый интерфейс для построения VPN. TUN работает на L3 уровне, и используется когда необходимо объеденить разные сайты с разными подсетями, а также в случае RAS.
TAP интерфейс работает на L2 уровне, т.е. виден ARP траффик если слушать TAP интерфейс, пример:
```
[root@openvpnServerOffice1 vagrant]# tcpdump -itap0 -nv
tcpdump: listening on tap0, link-type EN10MB (Ethernet), capture size 262144 bytes
16:16:43.265290 IP (tos 0x0, ttl 64, id 31461, offset 0, flags [DF], proto ICMP (1), length 84)
    10.10.10.2 > 10.10.10.1: ICMP echo request, id 5788, seq 1, length 64
16:16:43.265351 IP (tos 0x0, ttl 64, id 39909, offset 0, flags [none], proto ICMP (1), length 84)
    10.10.10.1 > 10.10.10.2: ICMP echo reply, id 5788, seq 1, length 64
16:16:48.275326 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 10.10.10.2 tell 10.10.10.1, length 28
16:16:48.276209 ARP, Ethernet (len 6), IPv4 (len 4), Reply 10.10.10.2 is-at b2:e6:83:17:29:59, length 28
```
TAP интерфейс совместно с бридж-интерфейсом могут использоваться когда надо объединить два и более филиалов с одинаковой подсетью. В таком случае можно использовать виртуальные интерфейсы (dummy) для объединения с туннельным (tap).

#### Примеры конфигурационных файлов сервера и клиента

Пример конфигурационного файла для сервера (TUN):
```
port 13555
proto udp
dev tun

ca /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key /etc/openvpn/server.key
dh /etc/openvpn/dh.pem

server 10.0.0.0 255.255.255.0
route 192.168.20.0 255.255.255.0
push "route 192.168.20.0 255.255.255.0"

ifconfig-pool-persist ipp.txt
client-to-client
client-config-dir /etc/openvpn/ccd

keepalive 10 120
comp-lzo
persist-key
persist-tun
status /var/log/openvpn/openvpn-status.log
log /var/log/openvpn/openvpn.log
verb 3
```

Пример конфигурационного файла для клиента (TUN):
```
dev tun
proto udp
remote 192.168.0.2 13555
client
resolv-retry infinite
ca /etc/openvpn/ca.crt
cert /etc/openvpn/client.crt
key /etc/openvpn/client.key
route 192.168.10.0 255.255.255.0
persist-key
persist-tun
comp-lzo
verb 3
status /var/log/openvpn/openvpn-status.log
status-version 3
log-append /var/log/openvpn/openvpn-client.log
```

Пример конфигурационного файла для сервера (TAP без бридж интерфейса):
```
dev tap
ifconfig 10.10.10.1 255.255.255.0 # Туннельная подсеть
topology subnet 
secret /etc/openvpn/static.key # Общий ключ для соединения
comp-lzo # Метод компрессии
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3 # Уровень логирования
```

Пример конфигурационного файла для сервера (TAP без бридж интерфейса):
```
dev tap
remote 192.168.0.1
ifconfig 10.10.10.2 255.255.255.0 # Туннельная подсеть
topology subnet
route 192.168.10.0 255.255.255.0 
secret /etc/openvpn/static.key # Общий ключ для соединения
comp-lzo # Метод компрессии
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3 # Уровень логирования
```

Особенности:
- Архитектурно в OpenVPN всегда один сервер, а все остальные клиенты (даже если Site-to-Site). 
- Если мы строим Site-to-Site с другим филиалом (вторым или третьим) то на сервере создаем еще один инстанс OpenVPN со своим конфигурационным файлом, в котором будут другие настройки.
- Не рекомендуется использовать OpenVPN в серьезном продакшн окружении в силу того что OpenVPN работает в User Space, в силу чего на скоростях свыше 20 Мбит происходят патери пакетов, в особенности при использовании UDP протокола. При использовании TCP происходит повторная переотправка, однако это создает дополнительную нагрузку. Таким образом не рекомендуется использовать для передачи больших объемов данных (UDP). Рекомендуется только для домашнего использования, в котором не критична потеря данных (если UDP). При использовании протокола TCP потеря пакетов невелируется природой TCP протокола.
- Серьезной разницы в скорости между TAP и TUN интерфейсами не замечено и зависит от производительности тестового стенда.


#### Испытания с помощью программы iperf3

Данные iperf3 (UDP):
```
[root@office2Computer vagrant]# iperf3 -c 192.168.10.2 -b 50M -u -t 0 -P 1
Connecting to host 192.168.10.2, port 5201
[  4] local 192.168.20.2 port 44277 connected to 192.168.10.2 port 5201
[ ID] Interval           Transfer     Bandwidth       Total Datagrams
[  4]   0.00-1.00   sec  5.57 MBytes  46.7 Mbits/sec  4346  
[  4]   1.00-2.00   sec  5.85 MBytes  49.1 Mbits/sec  4560  
[  4]   2.00-3.00   sec  5.94 MBytes  49.8 Mbits/sec  4629  
[  4]   3.00-4.00   sec  5.98 MBytes  50.2 Mbits/sec  4664  
[  4]   4.00-5.00   sec  5.99 MBytes  50.2 Mbits/sec  4666  
[  4]   5.00-6.00   sec  5.99 MBytes  50.2 Mbits/sec  4668  
[  4]   6.00-7.00   sec  5.97 MBytes  50.1 Mbits/sec  4658  
[  4]   7.00-8.00   sec  5.90 MBytes  49.5 Mbits/sec  4598  
[  4]   8.00-9.00   sec  5.96 MBytes  49.9 Mbits/sec  4643  
[  4]   9.00-10.00  sec  6.01 MBytes  50.4 Mbits/sec  4685  
[  4]  10.00-11.00  sec  5.93 MBytes  49.7 Mbits/sec  4622  
[  4]  11.00-12.00  sec  6.01 MBytes  50.4 Mbits/sec  4682  
[  4]  12.00-13.00  sec  5.91 MBytes  49.6 Mbits/sec  4611  
[  4]  13.00-14.00  sec  6.01 MBytes  50.4 Mbits/sec  4685  
[  4]  14.00-15.00  sec  5.91 MBytes  49.6 Mbits/sec  4610  
[  4]  15.00-16.00  sec  5.97 MBytes  50.1 Mbits/sec  4654  
[  4]  16.00-17.00  sec  5.96 MBytes  50.0 Mbits/sec  4649  
[  4]  17.00-18.00  sec  5.92 MBytes  49.6 Mbits/sec  4612  
[  4]  18.00-19.00  sec  6.01 MBytes  50.4 Mbits/sec  4683  
[  4]  19.00-20.00  sec  5.96 MBytes  50.0 Mbits/sec  4644  
[  4]  20.00-21.00  sec  6.02 MBytes  50.5 Mbits/sec  4695  
[  4]  21.00-22.00  sec  5.92 MBytes  49.6 Mbits/sec  4612  
[  4]  22.00-23.00  sec  5.94 MBytes  49.8 Mbits/sec  4631  
[  4]  23.00-24.00  sec  6.00 MBytes  50.3 Mbits/sec  4678  
[  4]  24.00-25.00  sec  5.92 MBytes  49.7 Mbits/sec  4618  
[  4]  25.00-26.00  sec  5.91 MBytes  49.6 Mbits/sec  4610  
[  4]  26.00-27.00  sec  5.99 MBytes  50.3 Mbits/sec  4671  
[  4]  27.00-28.00  sec  5.99 MBytes  50.2 Mbits/sec  4666  
[  4]  28.00-29.00  sec  5.99 MBytes  50.3 Mbits/sec  4671  
[  4]  29.00-30.00  sec  5.88 MBytes  49.3 Mbits/sec  4585  
[  4]  30.00-31.00  sec  5.99 MBytes  50.2 Mbits/sec  4666  
[  4]  30.00-31.00  sec  5.99 MBytes  50.2 Mbits/sec  4666  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Jitter    Lost/Total Datagrams
[  4]   0.00-31.00  sec   186 MBytes  50.4 Mbits/sec  0.000 ms  0/145130 (0%)  
[  4] Sent 145130 datagrams


Time: Sun, 31 May 2020 07:42:45 GMT
Accepted connection from 192.168.10.1, port 39488
      Cookie: office2Computer.1590910965.732613.7e
[  5] local 192.168.10.2 port 5201 connected to 192.168.10.1 port 44277
Starting Test: protocol: UDP, 1 streams, 1345 byte blocks, omitting 0 seconds, 10 second test
[ ID] Interval           Transfer     Bandwidth       Jitter    Lost/Total Datagrams
[  5]   0.00-1.00   sec  4.05 MBytes  34.0 Mbits/sec  0.055 ms  972/4129 (24%)  
[  5]   1.00-2.00   sec  4.48 MBytes  37.6 Mbits/sec  0.053 ms  1285/4777 (27%)  
[  5]   2.00-3.00   sec  4.49 MBytes  37.6 Mbits/sec  0.051 ms  1131/4629 (24%)  
[  5]   3.00-4.00   sec  4.90 MBytes  41.1 Mbits/sec  0.111 ms  845/4663 (18%)  
[  5]   4.00-5.00   sec  4.90 MBytes  41.1 Mbits/sec  0.051 ms  847/4667 (18%)  
[  5]   5.00-6.00   sec  4.69 MBytes  39.3 Mbits/sec  0.142 ms  1013/4667 (22%)  
[  5]   6.00-7.00   sec  4.45 MBytes  37.3 Mbits/sec  0.062 ms  1189/4656 (26%)  
[  5]   7.00-8.00   sec  4.54 MBytes  38.1 Mbits/sec  0.058 ms  1062/4599 (23%)  
[  5]   8.00-9.00   sec  4.80 MBytes  40.3 Mbits/sec  0.049 ms  900/4644 (19%)  
[  5]   9.00-10.00  sec  4.60 MBytes  38.6 Mbits/sec  0.112 ms  1099/4683 (23%)  
[  5]  10.00-11.00  sec  4.76 MBytes  39.9 Mbits/sec  0.035 ms  913/4625 (20%)  
[  5]  11.00-12.00  sec  4.94 MBytes  41.4 Mbits/sec  0.116 ms  829/4681 (18%)  
[  5]  12.00-13.00  sec  4.62 MBytes  38.8 Mbits/sec  0.037 ms  1008/4612 (22%)  
[  5]  13.00-14.00  sec  4.45 MBytes  37.4 Mbits/sec  0.119 ms  1211/4682 (26%)  
[  5]  14.00-15.00  sec  4.29 MBytes  36.0 Mbits/sec  0.081 ms  1267/4609 (27%)  
[  5]  15.00-16.00  sec  4.29 MBytes  36.0 Mbits/sec  0.139 ms  1046/4389 (24%)  
[  5]  16.00-17.00  sec  4.31 MBytes  36.1 Mbits/sec  0.081 ms  1556/4916 (32%)  
[  5]  17.00-18.00  sec  4.46 MBytes  37.4 Mbits/sec  0.049 ms  1135/4613 (25%)  
[  5]  18.00-19.00  sec  4.50 MBytes  37.8 Mbits/sec  0.095 ms  1167/4678 (25%)  
[  5]  19.00-20.00  sec  4.38 MBytes  36.7 Mbits/sec  0.072 ms  1234/4649 (27%)  
[  5]  20.00-21.00  sec  4.63 MBytes  38.9 Mbits/sec  0.075 ms  1085/4696 (23%)  
[  5]  21.00-22.00  sec  4.69 MBytes  39.3 Mbits/sec  0.110 ms  954/4611 (21%)  
[  5]  22.00-23.00  sec  4.86 MBytes  40.8 Mbits/sec  0.075 ms  843/4631 (18%)  
[  5]  23.00-24.00  sec  4.40 MBytes  36.9 Mbits/sec  0.104 ms  1247/4675 (27%)  
[  5]  24.00-25.00  sec  4.51 MBytes  37.8 Mbits/sec  0.113 ms  1105/4622 (24%)  
[  5]  25.00-26.00  sec  4.74 MBytes  39.8 Mbits/sec  0.056 ms  912/4609 (20%)  
[  5]  26.00-27.00  sec  4.53 MBytes  38.0 Mbits/sec  0.076 ms  1142/4672 (24%)  
[  5]  27.00-28.00  sec  4.57 MBytes  38.4 Mbits/sec  0.072 ms  1101/4665 (24%)  
[  5]  28.00-29.00  sec  4.48 MBytes  37.6 Mbits/sec  0.088 ms  1179/4672 (25%)  
[  5]  29.00-30.00  sec  4.57 MBytes  38.3 Mbits/sec  0.089 ms  1023/4584 (22%)  
[  5]  30.00-31.00  sec  4.65 MBytes  39.0 Mbits/sec  0.072 ms  1042/4665 (22%)  
^C[  5]  31.00-31.31  sec  1.60 MBytes  43.3 Mbits/sec  0.119 ms  214/1458 (15%)  
- - - - - - - - - - - - - - - - - - - - - - - - -
Test Complete. Summary Results:
[ ID] Interval           Transfer     Bandwidth       Jitter    Lost/Total Datagrams
[  5]   0.00-31.31  sec  0.00 Bytes  0.00 bits/sec  0.119 ms  33556/145128 (23%)  
CPU Utilization: local/receiver 11.0% (0.2%u/10.9%s), remote/sender 0.0% (0.0%u/0.0%s)
```

Пример iperf3 (TCP):
```
[root@openvpnServerOffice1 vagrant]# iperf3 -s
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 10.10.10.2, port 33330
[  5] local 10.10.10.1 port 5201 connected to 10.10.10.2 port 33332
[ ID] Interval           Transfer     Bandwidth
[  5]   0.00-1.00   sec  17.4 MBytes   146 Mbits/sec                  
[  5]   1.00-2.00   sec  18.0 MBytes   152 Mbits/sec                  
[  5]   2.00-3.00   sec  18.7 MBytes   156 Mbits/sec                  
[  5]   3.00-4.00   sec  19.3 MBytes   162 Mbits/sec                  
[  5]   4.00-5.01   sec  19.2 MBytes   160 Mbits/sec                  
[  5]   5.01-6.00   sec  19.2 MBytes   162 Mbits/sec                  
[  5]   6.00-7.01   sec  19.3 MBytes   161 Mbits/sec                  
[  5]   7.01-8.01   sec  19.4 MBytes   163 Mbits/sec                  
[  5]   8.01-9.00   sec  19.3 MBytes   162 Mbits/sec                  
[  5]   9.00-10.00  sec  19.0 MBytes   160 Mbits/sec                  
[  5]  10.00-10.06  sec   940 KBytes   136 Mbits/sec                  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth
[  5]   0.00-10.06  sec  0.00 Bytes  0.00 bits/sec                  sender
[  5]   0.00-10.06  sec   190 MBytes   158 Mbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 10.10.10.2, port 33334
[  5] local 10.10.10.1 port 5201 connected to 10.10.10.2 port 33336
[ ID] Interval           Transfer     Bandwidth
[  5]   0.00-1.00   sec  10.8 MBytes  90.4 Mbits/sec                  
[  5]   1.00-2.00   sec  12.2 MBytes   102 Mbits/sec                  
[  5]   2.00-3.01   sec  11.4 MBytes  95.2 Mbits/sec                  
[  5]   3.01-4.01   sec  11.9 MBytes  99.6 Mbits/sec                  
[  5]   4.01-5.00   sec  11.5 MBytes  96.9 Mbits/sec                  
[  5]   5.00-6.00   sec  12.5 MBytes   104 Mbits/sec                  
[  5]   6.00-7.00   sec  12.1 MBytes   102 Mbits/sec                  
[  5]   7.00-8.00   sec  12.0 MBytes   101 Mbits/sec                  
[  5]   8.00-9.00   sec  11.5 MBytes  96.0 Mbits/sec                  
[  5]   9.00-10.00  sec  12.0 MBytes   100 Mbits/sec                  
[  5]  10.00-10.05  sec   738 KBytes   126 Mbits/sec                  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth
[  5]   0.00-10.05  sec  0.00 Bytes  0.00 bits/sec                  sender
[  5]   0.00-10.05  sec   118 MBytes  98.8 Mbits/sec                  receiver
-----------------------------------------------------------
```

- В случае использования TAP интерфейса и использования одной и той же подсети но в разных филиалах необходимо использовать бридж интерфейс (необходимо использовать на обоих концах):

```Необходимо установить пакеты bridge-utils и net-tools```

Скрипт для создания бридж интерфейса:
```
#!/bin/bash
br="br0"
tap="tap0"
eth="eth0"
eth_ip="IP ADDRESS"
eth_netmask="255.255.255.0"
eth_broadcast="BROADCAST ADDRESS"
openvpn --mktun --dev $tap
brctl addbr $br
brctl addif $br $eth
brctl addif $br $tap
ifconfig $tap 0.0.0.0 promisc up
ifconfig $eth 0.0.0.0 promisc up
ifconfig $br $eth_ip netmask $eth_netmask broadcast $eth_broadcast
```

Скрипт для удаления бридж интерфейса:
```
#!/bin/bash
br="br0"
tap="tap0"
ifconfig $br down
brctl delbr $br
openvpn --rmtun --dev $tap
```

В этом случае настройки конфигурационных файлов будут отличаться.

Пример конфигурационного файла на сервере:
```
proto udp
port 1194
dev tap0 ## the '0' is extremely important
server-bridge 192.168.4.65 255.255.255.0 192.168.4.128 192.168.4.200
push "route 192.168.4.0 255.255.255.0"
ca /etc/openvpn/cookbook/ca.crt
cert /etc/openvpn/cookbook/server.crt
key /etc/openvpn/cookbook/server.key
dh /etc/openvpn/cookbook/dh1024.pem
tls-auth /etc/openvpn/cookbook/ta.key 0
persist-key
persist-tun
keepalive 10 60
user nobody
group nobody
daemon
log-append /var/log/openvpn.log
```


```
client

proto udp

port 1194

remote my.openvpn.server.com #Внешний адрес вашего сервера

dev tap0

nobind

tun-mtu 1500

ping 10

persist-key

persist-tun

ca /etc/openvpn/ca.crt

cert /etc/openvpn/client.crt

key /etc/openvpn/client.key

```

- TAP эмулирует Ethernet устройство и работает на канальном уровне модели OSI, оперируя кадрами Ethernet.

- TUN (сетевой туннель) работает на сетевом уровне модели OSI, оперируя IP пакетами.

- TAP используется для создания сетевого моста, тогда как TUN для маршрутизации.



TAP:

Преимущества:

    ведёт себя как настоящий сетевой адаптер (за исключением того, что он виртуальный);
    может осуществлять транспорт любого сетевого протокола (IPv4, IPv6, IPX и прочих);
    работает на 2 уровне, поэтому может передавать Ethernet-кадры внутри тоннеля;
    позволяет использовать мосты.

Недостатки:

    в тоннель попадает broadcast-трафик, что иногда не требуется;
    добавляет свои заголовки поверх заголовков Ethernet на все пакеты, которые следуют через тоннель;
    в целом, менее масштабируем из-за предыдущих двух пунктов;
    не поддерживается устройствами Android и iOS (по информации с сайта OpenVPN).

TUN:

Преимущества:

    передает только пакеты протокола IP (3й уровень);
    сравнительно (отн. TAP) меньшие накладные расходы и, фактически, ходит только тот IP-трафик, который предназначен конкретному клиенту.

Недостатки:

    broadcast-трафик обычно не передаётся;
    нельзя использовать мосты.



2. Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на виртуалку

Фактически это почти тоже самое что и из первого задания, только нам необходимо пробросить порт 1194 в виртуальную машину (делаем через вагрант).


Таблица маршрутизации с хостовой машины и доступность удаленной машины в туннеле:
```
[andy@noname vpn]$ ip r
default via 192.168.1.1 dev enp5s0 
10.10.10.0/24 via 10.10.10.5 dev tun0 
10.10.10.5 dev tun0 proto kernel scope link src 10.10.10.6 
192.168.1.0/24 dev enp5s0 proto kernel scope link src 192.168.1.33 metric 100 
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 
[andy@noname vpn]$ 
[andy@noname vpn]$ ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.802 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.838 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=0.871 ms
```

Логи подключения:
```
[andy@noname vpn]$ sudo openvpn --config ras-client-config/client.conf 
[sudo] пароль для andy: 
Sat Jun  6 23:39:32 2020 OpenVPN 2.4.8 x86_64-redhat-linux-gnu [Fedora EPEL patched] [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on Nov  1 2019
Sat Jun  6 23:39:32 2020 library versions: OpenSSL 1.0.2k-fips  26 Jan 2017, LZO 2.06
Sat Jun  6 23:39:32 2020 WARNING: No server certificate verification method has been enabled.  See http://openvpn.net/howto.html#mitm for more info.
Sat Jun  6 23:39:32 2020 TCP/UDP: Preserving recently used remote address: [AF_INET]127.0.0.1:1194
Sat Jun  6 23:39:32 2020 Socket Buffers: R=[87380->87380] S=[16384->16384]
Sat Jun  6 23:39:32 2020 Attempting to establish TCP connection with [AF_INET]127.0.0.1:1194 [nonblock]
Sat Jun  6 23:39:32 2020 TCP connection established with [AF_INET]127.0.0.1:1194
Sat Jun  6 23:39:32 2020 TCP_CLIENT link local: (not bound)
Sat Jun  6 23:39:32 2020 TCP_CLIENT link remote: [AF_INET]127.0.0.1:1194
Sat Jun  6 23:39:32 2020 TLS: Initial packet from [AF_INET]127.0.0.1:1194, sid=07c17be5 0c98945c
Sat Jun  6 23:39:32 2020 VERIFY OK: depth=1, CN=Easy-RSA CA
Sat Jun  6 23:39:32 2020 VERIFY OK: depth=0, CN=server
Sat Jun  6 23:39:32 2020 Control Channel: TLSv1.2, cipher TLSv1/SSLv3 ECDHE-RSA-AES256-GCM-SHA384, 2048 bit RSA
Sat Jun  6 23:39:32 2020 [server] Peer Connection Initiated with [AF_INET]127.0.0.1:1194
Sat Jun  6 23:39:33 2020 SENT CONTROL [server]: 'PUSH_REQUEST' (status=1)
Sat Jun  6 23:39:33 2020 PUSH: Received control message: 'PUSH_REPLY,route 10.10.10.0 255.255.255.0,topology net30,ping 10,ping-restart 120,ifconfig 10.10.10.6 10.10.10.5,peer-id 0,cipher AES-256-GCM'
Sat Jun  6 23:39:33 2020 OPTIONS IMPORT: timers and/or timeouts modified
Sat Jun  6 23:39:33 2020 OPTIONS IMPORT: --ifconfig/up options modified
Sat Jun  6 23:39:33 2020 OPTIONS IMPORT: route options modified
Sat Jun  6 23:39:33 2020 OPTIONS IMPORT: peer-id set
Sat Jun  6 23:39:33 2020 OPTIONS IMPORT: adjusting link_mtu to 1627
Sat Jun  6 23:39:33 2020 OPTIONS IMPORT: data channel crypto options modified
Sat Jun  6 23:39:33 2020 Data Channel: using negotiated cipher 'AES-256-GCM'
Sat Jun  6 23:39:33 2020 Outgoing Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
Sat Jun  6 23:39:33 2020 Incoming Data Channel: Cipher 'AES-256-GCM' initialized with 256 bit key
Sat Jun  6 23:39:33 2020 ROUTE_GATEWAY 192.168.1.1/255.255.255.0 IFACE=enp5s0 HWADDR=90:2b:34:35:37:89
Sat Jun  6 23:39:33 2020 TUN/TAP device tun0 opened
Sat Jun  6 23:39:33 2020 TUN/TAP TX queue length set to 100
Sat Jun  6 23:39:33 2020 /sbin/ip link set dev tun0 up mtu 1500
Sat Jun  6 23:39:33 2020 /sbin/ip addr add dev tun0 local 10.10.10.6 peer 10.10.10.5
Sat Jun  6 23:39:33 2020 /sbin/ip route add 10.10.10.0/24 via 10.10.10.5
Sat Jun  6 23:39:33 2020 WARNING: this configuration may cache passwords in memory -- use the auth-nocache option to prevent this
Sat Jun  6 23:39:33 2020 Initialization Sequence Completed
```

Чтобы запустить подключение из linux к виртуальной машине необходимо выполнить команду: ```sudo openvpn --config config/ras-client/client.conf```

В папке ```vagrantfiles``` находятся 3 вагрант файла.

#### Как запустить:

```git clone git@github.com:staybox/otus_dz20.git && cd otus_dz20``` далее скопировать в корневую папку нужный вагрант файл и ```vagrant up```