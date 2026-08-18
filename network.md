# Network

## 1. OSI, инкапсуляция, декапсуляция и уровни

**L1 / Physical** → сигнал и физическая среда: электричество/оптика/радио, bit rate/baud, шум, затухание.

**L2 / Data Link** → локальная Ethernet-доставка: MAC, frame, switch.

**L3 / Network** → доставка между IP-сетями: IP, routing, router, next hop.

**L4 / Transport** → связь между процессами: TCP/UDP, ports.

**L7 / Application** → HTTP, DNS, TLS и другие прикладные протоколы.

**Инкапсуляция**:

```text
HTTP
↓
TLS
↓
TCP
↓
IP
↓
Ethernet
```

**Декапсуляция** → обратный процесс на принимающей стороне: каждый нижний слой проверяет свой header/trailer, снимает его и передаёт payload выше.

```text
Ethernet
↓
IP
↓
TCP
↓
TLS
↓
HTTP
```

**Frame** → L2.  
**Packet** → IP/L3.  
**Segment** → TCP/L4.



## 2. Ethernet, MAC и switch

**MAC address** → L2-адрес интерфейса.

**NIC** → Network Interface Card, сетевой интерфейс.

**Switch** → пересылает Ethernet frames внутри L2-сегмента.

**MAC table / CAM table** → `MAC → switch port`.

**MAC learning** → switch запоминает `src MAC → ingress port` из входящих frames.

**Flooding** → `dst MAC` в Ethernet frame есть всегда; если switch не знает port для этого MAC в MAC table, frame рассылается по подходящим портам VLAN, кроме входного.

**Broadcast MAC** → `ff:ff:ff:ff:ff:ff`.

**Ethernet II** → dst MAC, src MAC, EtherType, payload, FCS.

**EtherType** → что лежит внутри Ethernet payload.

Примеры:

```text
0x0800 → IPv4
0x86DD → IPv6
0x0806 → ARP
```

**FCS / CRC** → обнаружение ошибок кадра; не криптографическая защита.

**MTU (Maximum Transmission Unit)** → максимальный IP-пакет, передаваемый через L2 без фрагментации; типично Ethernet MTU = 1500.



## 3. IPv4, subnet, mask, prefix

**IPv4 address** → 32-битный L3-адрес интерфейса.

Пример: `192.168.1.10`.

**Subnet** → диапазон IP с общим сетевым префиксом.

Пример: `192.168.1.0/24` содержит адреса `192.168.1.0`–`192.168.1.255`.

**Prefix `/24`** → первые 24 бита сеть, остальные host part.

Пример: в `192.168.1.10/24` сеть — `192.168.1`, host part — `10`.

**Subnet mask** → другая запись prefix: `/24 = 255.255.255.0`.

Пример: `192.168.1.10/255.255.255.0` то же самое, что `192.168.1.10/24`.

**Network address** → адрес подсети.

Пример: для `192.168.1.10/24` network address — `192.168.1.0`.

**Broadcast address** → широковещательный адрес подсети.

Пример: для `192.168.1.10/24` broadcast address — `192.168.1.255`.

**`/32`** → один IPv4.

Пример: `192.168.1.10/32` означает только адрес `192.168.1.10`.

**`/31`** → часто point-to-point link.

Пример: `10.0.0.0/31` даёт два адреса для двух концов link: `10.0.0.0` и `10.0.0.1`.

Сколько адресов можно назначать хостам:

```text
всего адресов в subnet = 2^(32 - prefix)
обычно usable = всего - 2
```

Два адреса обычно не назначают хостам:

- network address;
- broadcast address.

Пример:

```text
192.168.1.0/24
→ всего 256 адресов
→ usable 254 адреса: 192.168.1.1–192.168.1.254
→ 192.168.1.0   = network address
→ 192.168.1.255 = broadcast address
```

Исключения:

```text
/31 → 2 usable адреса для point-to-point link
/32 → 1 адрес, один host route
```



## 4. Gateway и routing

**Routing table** → правила:

```text
destination prefix → next hop / interface
```

Пример:

```text
192.168.1.0/24 → directly connected, eth0
10.0.0.0/8     → via 192.168.1.1, eth0
0.0.0.0/0      → via 192.168.1.1, eth0
```

**Longest Prefix Match** → выбирается самый специфичный маршрут.

Пример:

```text
routing table:
10.0.0.0/8      → via 192.168.1.1
10.20.0.0/16    → via 192.168.1.2
10.20.30.0/24   → via 192.168.1.3
0.0.0.0/0       → via 192.168.1.254

destination: 10.20.30.40
```

Подходят маршруты:

```text
10.0.0.0/8
10.20.0.0/16
10.20.30.0/24
0.0.0.0/0
```

Выбран будет:

```text
10.20.30.0/24 → via 192.168.1.3
```

Потому что `/24` специфичнее, чем `/16`, `/8` и `/0`.

**Default route** → `0.0.0.0/0`.

Пример:

```text
0.0.0.0/0 → via 192.168.1.1, eth0
```

**Default gateway** → next hop для default route.

Пример: `192.168.1.1`.

**Gateway не вычисляется из IP/маски** → он приходит из конфигурации, например DHCP.

Пример: для `192.168.1.10/24` gateway не обязан быть `192.168.1.1`; это просто частый convention.

**Next hop** → следующий L3-узел, которому передаётся packet.

Пример: при маршруте `10.0.0.0/8 → via 192.168.1.1` next hop — `192.168.1.1`.



## 5. ARP / neighbour table

**ARP** → Address Resolution Protocol; сопоставляет локальный next-hop IPv4 с MAC.

```text
нужен gateway 192.168.1.1
↓
ARP: "у кого 192.168.1.1?"
↓
получили MAC gateway
↓
Ethernet dst = MAC gateway
```

**ARP работает внутри локального L2-сегмента.**

**Neighbour table** → кэш `IP → MAC` + состояние соседа.

Если destination в другой subnet, ARP ищет MAC **router/next hop**, а не далёкого сервера.



## 6. IPv4 header, fragmentation и PMTUD

**TTL** → уменьшается на каждом hop; при 0 packet уничтожается.

**Fragmentation** → IPv4 может разбить большой packet на fragments.

**DF / Don't Fragment** → запрет fragmentation.

**PMTUD** → Path MTU Discovery, поиск максимального packet size на пути без fragmentation.

**ICMP (Internet Control Message Protocol)** → сообщения об ошибках/служебная диагностика, в том числе участвует в PMTUD.



## 7. Port, socket, bind и FD

**Port** → L4-номер точки подключения процесса.

**Socket** → kernel object для network endpoint/connection.

**FD / File Descriptor** → integer handle процесса на kernel object, включая socket.

**`bind()`** → привязать socket к локальному `IP:port`.

**`listen()`** → начать ожидать входящие TCP connections.

**`accept()`** → получить отдельный connected socket/FD.

**TCP 4-tuple**:

```text
src IP
src port
dst IP
dst port
```

**5-tuple** → то же + protocol.

**Ephemeral port** → временный client source port, обычно выбранный ОС.



## 8. TCP: базовая модель

**TCP** → надёжный двунаправленный byte stream.

**Handshake**:

```text
SYN
SYN-ACK
ACK
```

Алгоритм:

1. TCP connection — full-duplex: есть отдельный byte stream `client → server` и отдельный byte stream `server → client`.

2. У каждого направления свой sequence space:

```text
client_isn = 1000
server_isn = 5000
```

3. Client отправляет первый segment:

```text
client → server
SYN
SEQ = client_isn = 1000
```

Это задаёт initial sequence number для направления `client → server`.

4. Server отвечает:

```text
server → client
SYN + ACK
SEQ = server_isn = 5000
ACK = client_isn + 1 = 1001
```

`SEQ = 5000` задаёт initial sequence number для направления `server → client`.

`ACK = 1001` подтверждает клиентский `SYN`.

5. Client подтверждает серверный `SYN`:

```text
client → server
ACK
SEQ = client_isn + 1 = 1001
ACK = server_isn + 1 = 5001
```

6. После handshake:

```text
client next SEQ = 1001
server next SEQ = 5001
```

`SYN` занимает один sequence number, хотя не несёт data byte.

Важно:

```text
SEQ относится к своему направлению передачи.
ACK относится к обратному направлению: какой следующий byte ожидаем от peer.
```

**SEQ** → номер байта в stream.

**ACK** → номер следующего ожидаемого байта.

**Retransmission** → повторная отправка потерянных данных.

**RTT** → Round Trip Time.

**RTO** → Retransmission Timeout.

**Fast Retransmit** → ранняя повторная отправка при признаках потери.

**SACK** → Selective Acknowledgement.

**MSS** → Maximum Segment Size, TCP payload без IP/TCP headers.



## 9. TCP windows: `rwnd` и `cwnd`

**TCP window** → ограничение на количество данных, которые можно отправить без получения новых ACK.

В TCP есть два разных ограничения:

**`rwnd` / receive window** → окно получателя; сколько данных peer готов принять в свой receive buffer.

**`cwnd` / congestion window** → окно congestion control; сколько данных sender считает безопасным держать “в полёте” в сети.

Реально отправитель ограничен меньшим из двух:

```text
send window = min(rwnd, cwnd)
```

Пример:

```text
rwnd = 64 KB
cwnd = 16 KB
→ sender может держать in-flight только 16 KB
```

Другой пример:

```text
rwnd = 8 KB
cwnd = 64 KB
→ sender может держать in-flight только 8 KB
```

`rwnd` защищает получателя:

```text
receiver buffer почти заполнен
→ receiver уменьшает advertised window
→ sender замедляется
```

`cwnd` защищает сеть:

```text
loss / timeout / congestion signal
→ congestion control уменьшает cwnd
→ sender отправляет меньше in-flight data
```

**In-flight data** → отправленные bytes, которые ещё не подтверждены ACK.

**Bandwidth-delay product / BDP** → сколько данных нужно держать in-flight, чтобы заполнить канал:

```text
BDP = bandwidth × RTT
```



## 10. TCP states и `ss`

**TCP state** → состояние конкретного TCP socket/connection в state machine ядра.

`ss` показывает эти состояния:

```bash
ss -tna
ss -tna state established
ss -tna state listen
ss -tna state time-wait
```

Основные states:

**LISTEN** → server socket ждёт входящие connections.

**SYN-SENT** → client отправил `SYN`, ждёт `SYN-ACK`.

**SYN-RECV** → server получил `SYN`, отправил `SYN-ACK`, ждёт финальный `ACK`.

**ESTAB / ESTABLISHED** → handshake завершён, данные можно передавать в обе стороны.

**FIN-WAIT-1** → сторона отправила `FIN`, ждёт `ACK` или встречный `FIN`.

**FIN-WAIT-2** → `FIN` подтверждён, сторона ждёт `FIN` от peer.

**CLOSE-WAIT** → peer прислал `FIN`; локальное приложение ещё не закрыло socket.

**LAST-ACK** → локальная сторона отправила `FIN` после `CLOSE-WAIT`, ждёт финальный `ACK`.

**TIME-WAIT** → активно закрывшая сторона ждёт, чтобы старые segments не попали в новую connection.

**CLOSING** → обе стороны почти одновременно отправили `FIN`, ждут подтверждения.

Примеры:

```text
LISTEN    0  4096  0.0.0.0:80        0.0.0.0:*
ESTAB     0  0     10.0.0.2:443      10.0.0.5:53044
TIME-WAIT 0  0     10.0.0.2:443      10.0.0.5:53044
```

Частые симптомы:

```text
много SYN-RECV    → backlog/SYN flood/peer не завершает handshake
много CLOSE-WAIT  → приложение не закрывает socket после FIN от peer
много TIME-WAIT   → много коротких connections; обычно нормально для active close
```



## 11. TCP close: FIN, RST, TIME_WAIT

**FIN** → нормальное закрытие одного направления.

**TCP full duplex** → направления закрываются независимо.

**RST** → немедленный аварийный reset.

**TIME_WAIT** → состояние активно закрывшей стороны; защищает от старых segments и позволяет повторить финальный ACK.



## 12. UDP

**UDP** → datagram protocol без handshake, гарантии доставки, порядка и retransmission.

**Datagram** → отдельное сообщение; границы сообщений сохраняются.

**UDP `connect()`** → handshake не создаёт; фиксирует peer в socket state и упрощает API/фильтрацию.



## 13. DNS

**DNS** → Domain Name System.

**A** → IPv4.

**AAAA** → IPv6.

**CNAME** → alias на другое DNS name.

**Resolver** → компонент, который выполняет DNS resolution.

DNS сообщает адрес; дальше работают routing, ARP, TCP/UDP и т.д.



## 14. DHCP

**DHCP** → Dynamic Host Configuration Protocol.

**DORA**:

```text
Discover
Offer
Request
ACK
```

DHCP Discover у клиента без IP:

```text
Ethernet dst = ff:ff:ff:ff:ff:ff
IP src = 0.0.0.0
IP dst = 255.255.255.255
UDP src = 68
UDP dst = 67
```

Что происходит, когда новый host приходит в сеть:

1. Client ещё не знает свой IP и DHCP server.

2. Client отправляет broadcast `DHCP Discover`:

```text
Ethernet dst = ff:ff:ff:ff:ff:ff
IP src = 0.0.0.0
IP dst = 255.255.255.255
UDP 68 → 67
xid = client-generated transaction id
```

Этот frame получают все хосты в текущем broadcast domain/VLAN, но отвечает только DHCP server.

3. DHCP server отвечает `DHCP Offer`:

```text
xid = тот же transaction id
server предлагает IP
передаёт lease time и options
например: mask, gateway, DNS
```

Если пришло несколько `Offer`, client обычно выбирает первый подходящий.

4. Client отправляет `DHCP Request`:

```text
xid = тот же transaction id
server identifier = выбранный DHCP server
requested IP = выбранный address
просит выдать предложенный IP
```

Обычно это тоже broadcast, чтобы другие DHCP servers увидели `Server Identifier` и поняли, чей offer выбран.

5. DHCP server отправляет `DHCP ACK`:

```text
xid = тот же transaction id
IP выдан
lease активен
client применяет address/mask/gateway/DNS
```

**Lease** → аренда адреса.

**Static lease / reservation** → конкретному MAC DHCP всегда выдаёт конкретный IP.



## 15. DHCP options

DHCP может передать:

**Subnet Mask** → mask/prefix.

**Router** → default gateway.

**DNS Servers** → resolver addresses.

**Lease Time** → время аренды.

**T1/T2** → моменты renewal/rebinding.

**Domain Name / search suffix** → DNS suffix.

**Interface MTU** → MTU.

**NTP Servers** → Network Time Protocol servers.

DHCP только сообщает параметры; ОС сама настраивает interface, routes и resolver.



## 16. VLAN

**VLAN** → Virtual Local Area Network, логическое разделение L2-инфраструктуры на отдельные broadcast domains.

**Broadcast одного VLAN не разливается в другой VLAN.**

**VLAN ≠ subnet.**

VLAN → L2.  
Subnet → L3.

На практике часто:

```text
VLAN 10 ↔ 10.10.10.0/24
VLAN 20 ↔ 10.10.20.0/24
```

Между VLAN нужен L3 routing.



## 17. DHCP relay

**Проблема** → DHCP Discover broadcast не маршрутизируется между подсетями.

**DHCP relay** → принимает локальный DHCP broadcast и пересылает DHCP server'у unicast.

**`giaddr`** → Gateway IP Address; помогает server понять, из какой subnet пришёл client.

**Option 82** → relay может добавить switch port/VLAN/circuit information.



## 18. NAT

**NAT** → Network Address Translation.

Пример исходящего NAT:

```text
client 192.168.1.10:51514 → server 93.184.216.34:443

после NAT:

router public 203.0.113.5:40001 → server 93.184.216.34:443
```

Client ходит на `443`; порт `40001` — внешний source port, который router выбрал для NAT mapping.

Это реальное поле TCP/UDP source port в packet после трансляции, но обычно не listening socket процесса на router.

Router хранит mapping и делает обратную трансляцию на reply:

```text
server 93.184.216.34:443 → router public 203.0.113.5:40001
↓
server 93.184.216.34:443 → client 192.168.1.10:51514
```

**PAT** → Port Address Translation; много внутренних clients делят один public IP через разные внешние ports.



## 19. SNAT, DNAT, MASQUERADE

**SNAT** → Source NAT:

```text
client 192.168.1.10:51514 → server 93.184.216.34:443
→
router public 203.0.113.5:40001 → server 93.184.216.34:443
```

**DNAT** → Destination NAT:

```text
external client 198.51.100.77:53000 → router public 203.0.113.5:443
→
external client 198.51.100.77:53000 → internal server 192.168.1.20:443
```

**Port forwarding** → типичный DNAT scenario.

**MASQUERADE** → SNAT с текущим IP исходящего interface; удобно при dynamic external IP.



## 20. Conntrack / stateful NAT

**conntrack** → runtime-таблица уже увиденных connections/flows.

Она нужна, чтобы router/firewall понимал:

```text
этот packet начинает новый flow
или
это reply к уже известному flow
```

Важно:

```text
DNAT/SNAT/port forwarding rule → статическое правило трансляции
conntrack entry                → динамическая запись о конкретном flow
```

Удобная модель через 5-tuple:

```text
protocol
src IP
src port
dst IP
dst port
```

Для исходящего HTTPS через NAT:

```text
original flow:
tcp 192.168.1.10:51514 → 93.184.216.34:443

translated flow:
tcp 203.0.113.5:40001 → 93.184.216.34:443

reply:
tcp 93.184.216.34:443 → 203.0.113.5:40001
↓ reverse NAT
tcp 93.184.216.34:443 → 192.168.1.10:51514
```

Conntrack/NAT state хранит связь между original/reply tuple и трансляцией адресов/портов.

**Stateful NAT** → знает, какой reverse packet относится к какому уже известному flow.

**UDP** → FIN/handshake нет, state в основном живёт по timeout.



## 21. Hairpin NAT

**Hairpin NAT / NAT loopback** → client из LAN обращается к public IP своего же router и попадает обратно на internal server.

Пример:

```text
LAN client:      192.168.1.10
internal server: 192.168.1.20:443
router public:   203.0.113.5:443
```

Client из LAN идёт не на private IP сервера, а на public address:

```text
192.168.1.10 → 203.0.113.5:443
```

Router применяет DNAT обратно внутрь LAN:

```text
203.0.113.5:443 → 192.168.1.20:443
```

Без hairpin NAT такой доступ изнутри к собственному public IP может не работать.



## 22. CGNAT

**CGNAT** → Carrier-Grade NAT, дополнительный NAT у ISP.

```text
192.168.1.10
↓ home NAT
100.64.x.x
↓ ISP CGNAT
public IP
```

`100.64.0.0/10` → специальный диапазон для carrier-grade NAT.

Следствие:

```text
port forwarding на домашнем router может не работать,
если настоящий public IP находится не на нём, а на NAT у ISP.
```



## 23. STUN

**STUN** → Session Traversal Utilities for NAT.

Client спрашивает STUN server:

> «С какого IP:port ты меня видишь?»

```text
internal = 10.0.0.20:60000
external mapping = 2.2.2.2:45000
```

Важно:

**STUN mapping ≠ гарантированно открытый public socket.**

Это лишь:

> «Вот как NAT отобразил меня при разговоре со STUN server».



## 24. UDP hole punching

Проблема:

```text
Alice за NAT
Bob за NAT
```

У обоих нет заранее настроенного port forwarding, поэтому прямой входящий packet снаружи обычно будет отброшен NAT/firewall.

Но обе стороны хотят установить P2P-связь без relay.

Alice и Bob узнают external mappings через STUN и обмениваются ими через signaling.

Потом обе стороны начинают слать UDP друг другу:

```text
Alice → Bob
Bob → Alice
```

Так NAT на каждой стороне видит исходящий flow к конкретному peer и может разрешить reverse traffic.

**Строгий NAT/firewall** может сказать незнакомому external peer:

> «Я вас не звал».

**Symmetric NAT** → external mapping может зависеть от destination; mapping к STUN server может не совпасть с mapping к реальному peer.



## 25. TURN

**TURN** → Traversal Using Relays around NAT.

Если direct P2P не работает:

```text
Alice
↓
TURN relay
↓
Bob
```

**STUN** → mapping/connectivity assistance.

**TURN** → реальный relay, через него идёт traffic.



## 26. ICE

**ICE** → Interactive Connectivity Establishment.

Это **алгоритм/логика внутри client/library**, например WebRTC stack, а не отдельный server.

ICE собирает candidates:

**host candidate** → local endpoint.

**server-reflexive candidate** → NAT mapping через STUN.

**relay candidate** → endpoint на TURN.

ICE строит **candidate pairs**, даёт им priorities, делает connectivity checks и выбирает рабочую пару.

**Candidate** → возможный endpoint, не готовый route.

**Nomination** → выбор рабочей pair.



## 27. Firewall

**Firewall** → решает, пропустить packet или нет.

**ACCEPT** → пропустить.

**DROP** → молча выбросить.

**REJECT** → отклонить и вернуть ошибку.

**Stateful firewall** → использует conntrack.

Типичная модель:

```text
outgoing NEW → allow
incoming ESTABLISHED → allow
incoming NEW → deny unless explicitly allowed
```



## 28. INPUT, OUTPUT, FORWARD

На Linux:

**INPUT** → packet предназначен самому host.

**OUTPUT** → packet создан local process.

**FORWARD** → packet транзитом проходит через host/router.

**Reverse proxy ≠ FORWARD.**

```text
client → nginx   = INPUT для nginx host
nginx → backend  = OUTPUT для nginx host
```

Это два разных TCP connections.



## 29. Порядок DNAT, routing, firewall и SNAT

Входящий или транзитный packet:

```text
packet arrives
↓
PREROUTING
↓
DNAT
↓
routing decision
↓
INPUT или FORWARD
↓
firewall filtering
↓
POSTROUTING
↓
SNAT / MASQUERADE
↓
network
```

Packet, созданный локальным процессом:

```text
local process sends
↓
OUTPUT
↓
routing decision
↓
firewall filtering
↓
POSTROUTING
↓
SNAT / MASQUERADE
↓
network
```

**DNAT** обычно рано, до routing decision.

**SNAT** обычно поздно, перед отправкой наружу.

Нельзя говорить «NAT всегда раньше firewall» или наоборот.



## 30. Firewall policy и порядок rules

Rules идут сверху вниз.

**Первое совпавшее rule побеждает.**

```text
1. src 10.0.5.20 → dst :5432 → ACCEPT
2. dst :5432                 → DROP
```

**Default ACCEPT** → разрешено всё, что не запрещено.

**Default DROP / default deny** → запрещено всё, что явно не разрешено.

Для server security обычно удобнее default deny.

Что видит вызывающая сторона:

**ACCEPT** → packet пропускается дальше.

Для клиента это выглядит как обычная попытка соединения:

```text
service слушает и отвечает → connection работает
service не слушает        → host обычно вернёт TCP RST / ICMP error
```

**REJECT** → packet запрещён, но firewall явно отвечает ошибкой.

Для клиента это быстрый отказ:

```text
TCP → обычно reset / connection refused
UDP → обычно ICMP port/admin unreachable
```

**DROP** → packet молча выбрасывается, ответа нет.

Для клиента это выглядит как timeout:

```text
TCP SYN → нет SYN-ACK/RST → retries → timeout
UDP     → нет ответа      → timeout на уровне приложения
```



## 31. Bind address

**`127.0.0.1:3000`** → только loopback/local host.

**`192.168.0.10:3000`** → listener только на этом local IP.

**`0.0.0.0:3000`** → wildcard bind на все local IPv4.

**`[::]:3000`** → wildcard IPv6 bind.

Mental model:

**Bind** → «ГДЕ service слушает?»

**Firewall** → «КОМУ туда можно?»

**Routing/NAT** → «КАК туда добраться?»

Bind на private IP полезен как дополнительное ограничение, но не заменяет firewall.



## 32. Reverse proxy и forward proxy

**Reverse proxy** → proxy со стороны server/backends:

```text
Client
↓
Reverse Proxy
↓
Backend
```

Он может делать routing по Host/path, headers, auth, rate limit, TLS termination, load balancing.

**Forward proxy** → proxy со стороны client:

```text
Client
↓
Forward Proxy
↓
Internet
```

Мнемоника:

**Forward proxy скрывает clients.**  
**Reverse proxy скрывает servers.**

Названия не означают, что packet физически идёт «вперёд/назад».



## 33. Gateway vs proxy

**Gateway** → более широкая точка входа/выхода между системами.

Gateway может включать:

- reverse proxy;
- auth;
- rate limiting;
- TLS termination;
- load balancing;
- protocol translation;
- observability;
- иногда routing/NAT.

**Reverse proxy** → одна из возможных функций gateway.



## 34. Load balancer

**Load balancer / LB** → распределяет requests/connections между несколькими backends.

**Round Robin** → по очереди.

**Least Connections** → backend с меньшим числом connections.

**Hash** → выбор по client IP/cookie/key.

Удобно разделять:

**Reverse proxy** → выбрать backend pool/service.

**Load balancer** → выбрать конкретный instance в pool.

Один Nginx/HAProxy/Envoy может делать оба.



## 35. L4 vs L7 load balancing

**L4 LB** → TCP/UDP level; HTTP понимать не обязан.

**L7 LB** → application level, например HTTP.

Может route'ить по:

```text
Host
path
headers
cookie
```

Чтобы читать HTTPS как HTTP, L7 proxy обычно должен terminate TLS.



## 36. TLS: что даёт

**TLS** → Transport Layer Security.

**Confidentiality** → traffic нельзя прочитать без keys.

**Integrity/authenticity of data** → незаметно изменить ciphertext нельзя.

**Server authentication** → client проверяет identity server.

**TLS не «шифрует TCP»** → TLS создаёт защищённые bytes и пишет их в TCP stream.



## 37. Certificate, PKI и trust

**X.509 certificate** → identity + public key + validity + extensions + CA signature.

**PKI** → Public Key Infrastructure.

**CA** → Certificate Authority.

**Leaf certificate** → cert конкретного domain/server.

**Intermediate CA** → промежуточный CA.

**Root CA / trust anchor** → локально доверенный root.

**Trust store** → набор trusted roots.

Chain:

```text
leaf
↓
intermediate
↓
root
```

Root обычно не присылается server'ом; client уже доверяет ему локально.



## 38. CSR и private key

**Private key генерируется у владельца и CA не отправляется.**

**CSR** → Certificate Signing Request; содержит public key и данные запроса, подписанные applicant private key.

CA возвращает certificate, а не private key.



## 39. Certificate formats

**DER** → binary encoding.

**PEM** → Base64 armor вокруг DER.

**PKCS#7** → обычно certificate chain, без private key.

**PKCS#12** → может содержать certificate + chain + private key.

File extension не всегда гарантирует формат.



## 40. TLS 1.3 handshake

После TCP:

```text
ClientHello
↓
ServerHello
↓
Diffie–Hellman
↓
shared secret
↓
handshake traffic keys

[encrypted handshake]

EncryptedExtensions
Certificate
CertificateVerify
Finished

Client Finished

↓
application traffic keys
↓
encrypted HTTP
```



## 41. ClientHello

Содержит, упрощённо:

**SNI** → Server Name Indication, domain name для TLS virtual hosting.

**ALPN** → Application-Layer Protocol Negotiation, например `h2` / `http/1.1`.

**TLS versions**

**cipher suites**

**supported groups**

**key_share / client public DH value**



## 42. ServerHello и cipher suite

Server выбирает TLS parameters.

Примеры TLS 1.3 cipher suites:

```text
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_GCM_SHA256
```

**ChaCha20-Poly1305 / AES-GCM** → symmetric protection TLS records.

**SHA-256** → hash для TLS transcript/key schedule logic.

В TLS 1.3 cipher suite не выбирает certificate signature algorithm или X25519 group; они согласуются отдельно.



## 43. Diffie–Hellman / X25519

**Diffie–Hellman** → способ договориться об общем секрете по открытому каналу; сам по себе не шифрует данные.

**X25519** → современный вариант Diffie–Hellman на elliptic curve, часто используется в TLS 1.3.

```text
Client:
свой private DH key + открытое DH-значение server
→ shared_secret

Server:
свой private DH key + открытое DH-значение client
→ тот же shared_secret
```

Открытые DH-значения передаются по сети и видны наблюдателю.

Private DH keys по сети не передаются.

**Passive MITM / пассивный наблюдатель** → видит оба открытых DH-значения, но не может практически восстановить общий секрет.

**Active MITM / активный посредник** → может подменить открытые DH-значения и создать две разные защищённые сессии:

```text
Client ↔ Attacker
Attacker ↔ Server
```

Поэтому Diffie–Hellman без проверки сертификата не доказывает, что client говорит именно с нужным server.



## 44. CertificateVerify

**Certificate** говорит:

> «Этот public key допустимо считать ключом domain, если chain/hostname/time/usage валидны».

**CertificateVerify** говорит:

> «Server реально владеет private key, который соответствует public key в leaf certificate».

Упрощённо:

```text
signature =
Sign(server_private_key, transcript_hash)
```

`server_private_key` → закрытый ключ сервера, соответствующий public key в leaf certificate.

`transcript_hash` → hash handshake-сообщений, которые стороны уже увидели к моменту `CertificateVerify`.

Client проверяет signature через public key из leaf certificate:

```text
Verify(leaf_certificate_public_key, signature, transcript_hash)
```

В mTLS client тоже отправляет свой `CertificateVerify`, чтобы доказать владение private key своего client certificate.



## 45. Finished

**Finished** → финальная cryptographic check, что стороны:

- получили совместимые handshake secrets;
- видели тот же handshake transcript.

Server Finished → client verifies.  
Client Finished → server verifies.

Откуда берутся ключи:

```text
DH shared_secret
+ handshake transcript
↓
TLS 1.3 key schedule
↓
handshake traffic keys
↓
application traffic keys
```

**Handshake traffic keys** → защищают зашифрованную часть handshake после `ServerHello`.

**Application traffic keys** → защищают application data после завершения handshake, например HTTP.

После этого TLS handshake завершён.



## 46. TLS record protection

После `ServerHello` дальнейшие server handshake messages TLS 1.3 уже защищены handshake traffic keys.

После Finished используются **application traffic keys**.

Если выбран:

```text
TLS_CHACHA20_POLY1305_SHA256
```

TLS records защищаются ChaCha20-Poly1305.



## 47. TLS termination, re-encryption, passthrough

**Termination**:

```text
Client --HTTPS--> Proxy --HTTP--> Backend
```

TLS заканчивается на proxy; private key certificate нужен proxy.

**Termination + re-encryption**:

```text
Client --HTTPS--> Proxy --HTTPS--> Backend
```

Два независимых TLS connections.

**TLS passthrough**:

```text
Client ===== encrypted TLS =====> Backend
            через L4 proxy/LB
```

Proxy не расшифровывает HTTP.



## 48. mTLS

**mTLS** → mutual (взаимный) TLS.

Обычный TLS:

```text
client verifies server
```

mTLS:

```text
client verifies server
+
server verifies client certificate
```

Server отправляет **CertificateRequest**.

Client отвечает:

```text
Certificate
CertificateVerify
Finished
```

mTLS — обычный TLS с дополнительной client certificate authentication.



## 49. mTLS и ClientHello

Специального:

```text
mTLS = true
```

в ClientHello нет.

Начало handshake обычное; client certificate появляется только после `CertificateRequest` от server.



## 50. HTTP Host

```http
Host: api.example.com
```

Позволяет одному IP:port обслуживать несколько HTTP virtual hosts.

Reverse proxy может route'ить:

```text
Host: api.example.com
→ api backend

Host: admin.example.com
→ admin backend
```



## 51. SNI vs HTTP Host

**SNI** → TLS layer, ClientHello, нужен до HTTP для выбора TLS certificate.

**Host** → HTTP layer, виден после расшифровки TLS, нужен для HTTP virtual host/routing.

```text
DNS
↓
TCP
↓
TLS ClientHello: SNI
↓
certificate
↓
TLS handshake
↓
HTTP Host
↓
backend routing
```



## 52. X-Forwarded-For

Через reverse proxy backend видит TCP peer = proxy, а не original client.

```text
Client 1.2.3.4
↓
Nginx 10.0.0.5
↓
Backend
```

Proxy передаёт:

```http
X-Forwarded-For: 1.2.3.4
```

Если proxies несколько, header может содержать chain.

**Security** → нельзя слепо доверять X-Forwarded-For от любого client; client сам может его подделать.

Trusted forwarded headers должны приниматься только от известных proxies.



## 53. X-Forwarded-Proto / Host / Port / Forwarded

**`X-Forwarded-Proto`** → исходная схема:

```http
X-Forwarded-Proto: https
```

Полезно при TLS termination.

**`X-Forwarded-Host`** → original hostname.

**`X-Forwarded-Port`** → original port.

**`Forwarded`** → стандартизированный header:

```http
Forwarded: for=1.2.3.4;proto=https;host=api.example.com
```



## 54. HTTP keep-alive / persistent connection

**Один HTTP request ≠ один TCP connection.**

```text
TCP connection
↓
GET /a
response
↓
GET /b
response
```

Connection reuse уменьшает TCP/TLS handshake overhead.

HTTP/1.1 persistent connection — нормальный режим.



## 55. HTTP/2 multiplexing

**Multiplexing** → несколько независимых HTTP streams внутри одного TCP connection.

```text
TCP connection
├─ stream 1
├─ stream 3
└─ stream 5
```


## 56. HTTP/3 и QUIC

**HTTP/3** → HTTP semantics поверх **QUIC**, а не поверх TCP.

```text
HTTP/1.1
→ TCP

HTTP/2
→ TCP

HTTP/3
→ QUIC
→ UDP
→ IP
```

**QUIC** → transport protocol поверх UDP, который реализует внутри себя:

- connection establishment;
- multiplexed streams;
- flow control;
- congestion control;
- loss recovery;
- TLS 1.3 encryption.

Почему UDP:

```text
kernel/network devices уже умеют пропускать UDP
↓
QUIC можно развивать в userspace быстрее, чем менять TCP в kernel/network stack
```

Ключевое отличие от HTTP/2:

```text
HTTP/2 streams живут внутри одного TCP connection
→ потеря TCP packet может блокировать delivery данных по всем streams

HTTP/3 streams живут внутри QUIC connection
→ loss по одному stream не обязан блокировать остальные streams на transport level
```

Это уменьшает **head-of-line blocking** на transport layer.

QUIC connection identity не равна только 4-tuple:

```text
src IP, src port, dst IP, dst port
```

QUIC использует connection IDs, поэтому connection может переживать смену network path, например переход клиента с Wi‑Fi на mobile network.

Практическая граница:

```text
HTTP/3
→ application protocol version

QUIC
→ encrypted transport over UDP

UDP
→ только базовая datagram delivery; надёжность и streams делает QUIC
```



## 57. Backend connection pools

Reverse proxy может держать pool connections к backends:

```text
Nginx
├─ TCP #1 → backend1
├─ TCP #2 → backend1
├─ TCP #3 → backend2
```

Request берёт свободное connection; после response оно может быть переиспользовано.



## 58. HTTP keep-alive vs TCP keepalive

**HTTP keep-alive** → reuse connection для новых HTTP requests.

**TCP keepalive** → проверить, жив ли давно молчащий peer.

Это разные механизмы.



## 59. Reverse proxy timeouts

**Connect timeout** → сколько ждать установления connection с backend.

Причины проблем: backend down, no listener, firewall, route, overload.

**Read timeout** → connection уже есть, request отправлен, backend слишком долго не присылает data.

Частый итог → `504`.

**Idle / keepalive timeout** → сколько держать неиспользуемое connection открытым.



## 60. 502 vs 504

Грубая полезная модель:

**502 Bad Gateway** → upstream communication **BROKEN**.

Например connection refused/reset, invalid response, upstream TLS failure.

**504 Gateway Timeout** → upstream communication **TOO SLOW**.

Точное mapping зависит от proxy.



## 61. Таймауты слоями

```text
Client
↓
CDN
↓
Cloud LB
↓
Ingress
↓
Service Mesh
↓
App
```

У каждого слоя свой timeout.

Если request стабильно падает ровно через N секунд → искать intermediate proxy/LB с timeout около N.



## 62. Полный путь запроса

```text
DNS
↓
IP
↓
routing
↓
ARP to next hop
↓
TCP handshake
↓
TLS handshake
↓
encrypted HTTP
↓
reverse proxy
↓
HTTP routing / load balancing
↓
backend
```

Если service за NAT:

```text
private host
↓
SNAT/PAT
↓
public Internet
```

Если входящий traffic публикуется внутрь:

```text
public IP:port
↓
DNAT / port forwarding
↓
firewall
↓
private service
```

Если P2P за NAT:

```text
ICE
├─ host
├─ STUN → server-reflexive
└─ TURN → relay
```



## 63. Диагностический маршрут

Если «не работает сеть», быстро пройти:

```text
1. DNS разрешился?
2. Есть route?
3. Правильный next hop?
4. ARP/neighbour жив?
5. TCP handshake проходит?
6. Port слушается?
7. Bind address правильный?
8. Firewall не режет?
9. NAT/DNAT/SNAT ожидаемый?
10. TLS handshake проходит?
11. Certificate/SNI/hostname правильные?
12. Proxy подключается к backend?
13. Backend отвечает до timeout?
14. Forwarded headers настроены корректно?
```
