# Docker

## 1. Что такое Docker

**Docker** — userspace-система, которая управляет container-объектами, их filesystem, runtime-конфигурацией, networking, storage и lifecycle, используя обычные механизмы Linux kernel.

Ключевая граница:

```text
Docker
→ решает, ЧТО должно быть создано и как это связано

container runtime
→ реализует low-level setup процесса

Linux kernel
→ реально создаёт namespaces/cgroups/mounts/interfaces,
  запускает процессы, маршрутизирует packets,
  применяет NAT, доставляет signals и т.д.
```

Основные компоненты:

```text
docker CLI
    ↓
dockerd / Docker Engine
    ↓
containerd
    ↓
shim
    ↓
OCI runtime (обычно runc)
    ↓
Linux kernel
```

Грубое разделение ответственности:

- `docker` — CLI.
- `dockerd` — Docker daemon; управляет Docker-level объектами и orchestration.
- `containerd` — lifecycle manager для container tasks/runtime.
- `shim` — host-side процесс-посредник рядом с конкретным running task.
- `runc` — low-level OCI runtime: создаёт runtime environment и запускает process.
- Linux kernel — делает всю настоящую работу на уровне процессов, namespaces, cgroups, mounts, networking, signals.



## 2. Определения

### Image

**Docker image** — immutable набор:

```text
filesystem data
+
image metadata/config
```

Image не является running process.

Важные свойства:

- read-only;
- content-addressed;
- состоит из layers;
- имеет image ID;
- может иметь tags;
- хранится Docker-managed образом на host;
- может быть скачан из registry.

### Container

**Container** — Docker object, созданный на основе конкретного image, со своим mutable runtime state.

Контейнер включает в модель:

```text
image layers
+
writable layer
+
runtime config
+
namespaces
+
cgroups
+
mounts
+
networking
+
main Linux process
```

Контейнер может существовать без running process (`created`, `exited`).

### Main process / PID 1

Главный process контейнера:

```text
PID 1 inside container PID namespace
```

На host это тот же task может иметь, например:

```text
PID 50228
```

Lifecycle container завязан на этот process:

```text
main process alive
→ container running

main process exits
→ container exited
```

### Dockerfile

Обычный текстовый файл с инструкциями:

```text
INSTRUCTION arguments
```

Примеры:

```dockerfile
FROM ...
RUN ...
COPY ...
ENV ...
WORKDIR ...
USER ...
ENTRYPOINT ...
CMD ...
EXPOSE ...
```

Dockerfile описывает, как построить image.

### Layer

**Layer** — сохранённый набор filesystem-изменений.

Image:

```text
layer 1
layer 2
layer 3
...
```

Layers read-only и могут переиспользоваться между images.

### Writable layer

На каждый container поверх image layers создаётся отдельный writable layer:

```text
image read-only layers
+
container writable layer
```

Он принадлежит конкретному container.

Stop его не удаляет. Remove — удаляет.



## 3. Image → Container: вся цепочка

```text
Dockerfile
   ↓
docker build
   ↓
image
   ├─ filesystem layers
   └─ metadata/config
   ↓
docker create / docker run
   ↓
container object
   ↓
runtime config
   ↓
namespaces + cgroups + mounts + networking
   ↓
Linux process
```

Главная разница:

```text
IMAGE
→ immutable saved data

CONTAINER
→ runtime object based on image
```



## 4. Layers, OverlayFS и Copy-on-Write

Image layers:

```text
lower layer
lower layer
lower layer
```

Container получает:

```text
upper = writable layer
```

OverlayFS conceptually даёт merged view:

```text
lower layers
+
upper writable layer
=
merged root filesystem
```

### Изменение файла

Если файл существует в lower layer и process хочет его изменить:

```text
read lower file
→ copy-up into upper
→ modify upper copy
```

Это **copy-on-write**.

### Удаление файла

Lower layer immutable, поэтому удалить его физически нельзя.

OverlayFS использует whiteout:

```text
lower contains file
+
upper says "hide this path"
→ merged view does not show file
```

### Зачем layers

- reuse;
- меньше сетевого трафика при pull;
- content-addressable storage;
- shared read-only data между containers/images;
- build cache.



## 5. Root filesystem, mounts и mount namespace

### Root filesystem

Для process внутри container `/` — его root filesystem view.

Он складывается из:

```text
image layers
+
writable layer
+
mounts
```

### Mount namespace

Mount namespace даёт process своё дерево mounts.

Два process на одном host могут видеть разные:

```text
/
 /proc
 /sys
 /dev
 /data
```

### Mounts поверх rootfs

Если mount сделан на `/data`:

```text
underlying rootfs /data
     X hidden
mount source
     ↓
visible as /data
```

Mount не удаляет underlying data — он просто перекрывает path в mount tree.



## 6. Volumes, bind mounts, tmpfs

### Bind mount

Существующий host path монтируется внутрь container:

```text
host /srv/data
        ↓
container /data
```

Это те же underlying files.

Lifetime не зависит от container.

### Docker volume

Docker-managed storage object:

```text
Docker volume
→ mounted into container
```

Lifetime отдельно от container.

### tmpfs

Memory-backed filesystem:

```text
RAM
↓
mounted path inside container
```

Исчезает после teardown/stop.

### Главное сравнение

```text
writable layer
→ принадлежит конкретному container

bind mount
→ конкретный host path

volume
→ Docker-managed external storage

tmpfs
→ temporary memory-backed storage
```



## 7. Namespaces: что именно изолируется

Namespace — Linux kernel-механизм, который даёт процессам отдельный context/view определённого ресурса.

### PID namespace

Разные PID views:

```text
host:
PID 50228

container:
same task = PID 1
```

Namespace init process — PID 1 внутри namespace.

### Mount namespace

Отдельное mount tree.

### Network namespace

Отдельные:

```text
interfaces
IP addresses
routing table
sockets
loopback
```

### UTS namespace

Отдельные hostname/domainname.

### IPC namespace

Отдельные IPC resources.

### User namespace

Mapping UIDs/GIDs между namespace и host.

### Важный принцип

Namespace сам по себе **не является контейнером**.

Container = набор нескольких Linux-механизмов + Docker runtime state.



## 8. PID 1: почему он особенный

PID 1 внутри PID namespace — namespace init process.

Он имеет special Linux semantics:

- orphan adoption;
- zombie reaping;
- special signal handling rules.

Если namespace PID 1 умирает:

```text
PID 1 dies
↓
kernel SIGKILLs remaining processes in that PID namespace
```

Это важно для graceful shutdown.

### Shell-wrapper проблема

Плохо:

```text
PID 1 = /bin/sh
PID 7 = app
```

Docker сигналит PID 1, а app может сигнал не получить.

Хорошо:

```text
exec app
```

или exec-form:

```dockerfile
CMD ["app", "--serve"]
```

Тогда:

```text
PID 1 = app
```



## 9. ENTRYPOINT, CMD и argv

### ENTRYPOINT

Основная program/argv prefix.

### CMD

Default command/arguments.

Exec form:

```dockerfile
ENTRYPOINT ["app"]
CMD ["--serve", "--port", "8080"]
```

Результирующий argv:

```text
app --serve --port 8080
```

### Exec form

```dockerfile
CMD ["app", "--serve"]
```

Process запускается напрямую.

### Shell form

```dockerfile
CMD app --serve
```

Conceptually:

```text
/bin/sh -c "app --serve"
```

Это может сделать shell PID 1 и сломать signal forwarding.



## 10. Cgroups: ресурсы, а не изоляция view

Namespace отвечает:

```text
"что process видит?"
```

Cgroup отвечает:

```text
"сколько ресурсов process/group может использовать
и сколько уже использовал?"
```

### CPU

Два разных типа управления:

```text
relative weight
```

и:

```text
quota / period
```

Пример идеи:

```text
period = 100ms
quota = 50ms CPU time
→ 0.5 CPU equivalent
```

Quota считается суммарно по threads/CPUs.

### Memory

Container не получает отдельную физическую RAM.

Есть общий host memory pool, а cgroup ограничивает/account'ит usage.

Cgroup v2:

```text
memory.current
memory.max
```

### PIDs

Ограничивает количество tasks:

```text
processes + threads
```

При достижении лимита новые `fork/clone` могут fail.

### I/O

Контроль block I/O:

```text
BPS
IOPS
weight
```

Ограничивается underlying block device I/O, не «файл по имени».



## 11. Memory limits и OOM

### `--memory`

Docker настраивает cgroup memory limit.

Container продолжает использовать host RAM, но cgroup ограничивает charge.

### Memory pressure

Перед OOM kernel обычно пытается reclaim.

То есть:

```text
usage near limit
↓
reclaim
↓
still cannot satisfy allocation
↓
cgroup OOM
```

### Cgroup OOM

Может произойти даже если на host ещё много свободной RAM:

```text
host free memory > 0
but
container cgroup reached memory.max
```

### Host OOM

Глобальное истощение памяти host.

### OOM killer

Не отдельный daemon.

Kernel выбирает victim.

### Exit code 137

Частое отображение:

```text
128 + SIGKILL(9) = 137
```

Но:

```text
137 ≠ доказательство OOM
```

Любой SIGKILL может дать такой shell-style code.



## 12. Networking prerequisites

### Network namespace

Каждый container обычно получает отдельный network namespace.

### Loopback

```text
127.0.0.0/8
```

— loopback range.

`127.0.0.1` внутри host и container — разные networking contexts.

### Interface

Container обычно видит:

```text
lo
eth0
```

### Veth pair

Pair virtual Ethernet interfaces:

```text
container eth0
<==== veth pair ====>
host-side veth
```

### Linux bridge

Virtual L2 switch.

Host-side veth interfaces подключаются к bridge как ports.

### Routing

L3 decision:

```text
destination IP
→ route lookup
→ output interface / next hop
```

### Forwarding

Host действует как router для packet, который пришёл не к самому host.

### NAT

Меняет address/port information packet.



## 13. Docker bridge network

Типичная topology:

```text
Container A netns             Host netns              Container B netns

eth0 172.18.0.2
   |
 veth
   |
veth-A ----\
            \
             Linux bridge
            /
veth-B ----/
   |
 veth
   |
eth0 172.18.0.3
```

Bridge может иметь gateway IP:

```text
172.18.0.1
```

### Container-to-container

В одной subnet:

```text
A
→ connected route
→ eth0
→ veth
→ bridge
→ veth
→ B
```

Обычно без NAT.

### Container-to-host

Если destination — bridge IP host:

```text
172.18.0.1
```

packet доставляется локально host networking stack.

### Container-to-internet

```text
container
↓
default route
↓
bridge
↓
host routing + forwarding
↓
SNAT/MASQUERADE
↓
external network
```



## 14. NAT, SNAT, DNAT, connection tracking

### SNAT

Меняет source:

```text
172.18.0.2:40000
→
HOST_IP:52001
```

Типичный outbound case.

### MASQUERADE

Разновидность source NAT, использующая address outgoing interface.

### DNAT

Меняет destination:

```text
HOST_IP:8080
→
CONTAINER_IP:80
```

Типичный port publishing case.

### Connection tracking

Router/NAT не «держит TCP stream».

Он хранит state flow/mapping:

```text
inside endpoint
↔
translated endpoint
↔
remote endpoint
```

Reply packets получают reverse translation автоматически.

### Ключевая граница

```text
TCP connection
→ state belongs to endpoint TCP stacks

NAT
→ packet rewriting + tracked flow state

routing
→ chooses path
```

NAT ≠ TCP proxy.



## 15. Port mapping

### Container port

Port socket'а внутри container network namespace.

```text
container :80
```

### Host port

Port host-side endpoint.

```text
host :8080
```

### Публикация

```text
HOST_IP:HOST_PORT
→
CONTAINER_IP:CONTAINER_PORT
```

### CLI

```bash
docker run -p 8080:80 image
```

Смысл:

```text
host :8080
→ container :80
```

Полная форма:

```bash
docker run -p 127.0.0.1:8080:80 image
```

### Bind на 127.0.0.1

Только host loopback entry point.

### Bind на 0.0.0.0

Все подходящие local IPv4 addresses host.

### EXPOSE

```dockerfile
EXPOSE 80
```

Это metadata image.

Не создаёт:

```text
listener
host port
NAT
port publishing
```

Практическое использование: tooling, например `docker run -P`.

### Compose `expose`

Не является ACL.

Containers в одной Docker network могут общаться по listener port без `expose:`.



## 16. Docker DNS и service discovery

### `/etc/resolv.conf`

Resolver configuration внутри container.

На user-defined networks часто:

```text
nameserver 127.0.0.11
```

### `127.0.0.11`

Special loopback endpoint Docker embedded DNS внутри каждого container network namespace.

Не один shared host socket.

```text
Container A:
127.0.0.11
→ Docker-managed DNS handling

Container B:
127.0.0.11
→ Docker-managed DNS handling
```

Одинаковый IP, разные netns.

### Name resolution

```text
backend
↓
Docker DNS
↓
172.18.0.3
↓
normal Linux networking
```

### Network scope

Docker DNS visibility зависит от network membership.

```text
same host
≠ automatically same DNS visibility

same user-defined network
→ Docker name resolution
```

### Compose

Service name — удобное DNS name:

```text
api
db
backend
```

Пример:

```text
api → resolve("db")
→ current db IP
→ connect(db_ip:5432)
```

### IP может меняться

```text
backend → 172.18.0.3
```

после recreate:

```text
backend → 172.18.0.7
```

Поэтому:

```text
stable name
→ current runtime IP
```

лучше, чем hardcoded IP.



## 17. Logs: полная модель

### stdout / stderr

```text
fd 1 = stdout
fd 2 = stderr
```

Это обычные process file descriptors.

### Runtime I/O plumbing

Без TTY conceptually:

```text
container process
  fd 1 → stdout channel/FIFO
  fd 2 → stderr channel/FIFO
  fd 0 ← stdin channel/FIFO
```

Host-side runtime stack/shim держит plumbing.

```text
process
↓
stdio channels
↓
shim/runtime path
↓
Docker logging subsystem
↓
logging driver
```

### Logging driver

Определяет, что делать с stdout/stderr output:

```text
store locally
or
forward externally
```

Примеры:

```text
json-file
local
remote drivers
```

### `docker logs`

Не читает `/proc/PID/fd/1` напрямую.

Идёт через Docker logging subsystem.

Может показывать logs даже после process exit, если driver их сохранил.

### Где хранятся logs

Зависит от driver.

Для file-based driver — Docker-managed storage на host.

Не обязательно container writable layer.

### File logs inside container

Если app делает:

```text
open("/var/log/app.log")
write(...)
```

это обычный filesystem path.

`docker logs` этот файл автоматически не читает.

### Rotation

Для stdout/stderr — responsibility logging driver/config.

Для application-owned `/var/log/*.log` — отдельная responsibility.

Compose per-service:

```yaml
services:
  api:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

Global Docker defaults — daemon config.



## 18. Signals и lifecycle

### Кто получает signal

Docker lifecycle signal обычно идёт main process:

```text
PID 1 inside container
```

Не всему process tree.

### `docker stop`

```text
SIGTERM
↓
wait stop timeout
↓
SIGKILL if still alive
```

Для Linux default stop timeout обычно:

```text
10 seconds
```

### `docker kill`

По умолчанию:

```text
SIGKILL immediately
```

Но можно отправить другой signal.

### SIGTERM

Catchable termination request.

Нормальная graceful semantics:

```text
SIGTERM
→ handler
→ cleanup
→ exit
```

### SIGKILL

```text
cannot catch
cannot ignore
cannot handle
```

Kernel-enforced termination.

### PID 1 special semantics

Namespace init process имеет специальные signal rules.

Важно:

```text
host sees task as PID 50228
container sees same task as PID 1
```

Signal target lookup делается в caller PID namespace, но после lookup kernel знает, что target — namespace init, и применяет его special semantics.



## 19. Graceful shutdown: хорошая последовательность

Полная правильная цепочка:

```text
docker stop
   ↓
SIGTERM
   ↓
PID 1 handler
   ↓
enter shutting-down state
   ↓
stop accepting new work
   ↓
close listening sockets
   ↓
finish in-flight requests
   ↓
signal child processes
   ↓
wait/reap children
   ↓
close established connections
   ↓
flush application buffers
   ↓
perform required writes/sends
   ↓
exit main process
   ↓
container state = exited
```

Параллельно:

```text
stop timeout
   ↓
if PID 1 still alive
   ↓
SIGKILL
```

### Закрытие listener ≠ закрытие connections

До shutdown:

```text
fd 5 → LISTEN :8080
fd 7 → client A
fd 8 → client B
```

После:

```text
close(fd 5)
```

можно оставить:

```text
fd 7
fd 8
```

живыми, чтобы закончить текущую работу.

### Flush

Различать:

```text
userspace buffer
→ write()
→ kernel
```

и:

```text
fsync-like semantics
→ storage durability request
```

Flush userspace buffer ≠ гарантированная physical durability.



## 20. Что происходит при запуске container

```text
1. docker CLI sends request to dockerd

2. dockerd resolves image/config/container settings

3. Docker creates container object

4. Docker prepares:
   - root filesystem
   - mounts
   - runtime config
   - network intent
   - cgroup/resource config

5. containerd/runtime path invoked

6. low-level runtime creates/joins:
   - PID namespace
   - mount namespace
   - UTS namespace
   - IPC namespace
   - user namespace if configured
   - network namespace / namespace hooks as required

7. runtime prepares:
   - rootfs
   - mounts
   - credentials
   - cgroup membership
   - stdio plumbing

8. Docker networking subsystem configures:
   - veth
   - bridge attachment
   - container IP
   - routes
   - DNS environment
   - host NAT/firewall rules as needed

9. runtime starts main Linux process

10. inside container:
    main process = PID 1
```



## 21. Что происходит при остановке container

```text
docker stop
   ↓
dockerd
   ↓
main process
   ↓
SIGTERM
   ↓
application shutdown logic
   ↓
wait up to stop timeout
```

Если process вышел:

```text
runtime records exit
↓
Docker marks container exited
```

Если не вышел:

```text
SIGKILL
↓
main PID 1 dies
↓
kernel kills remaining processes in PID namespace
↓
container exited
```

Container object остаётся.



## 22. Что сохраняется / что исчезает

### После `docker stop`

Сохраняются:

```text
container object
writable layer
metadata/config
Docker-managed logs
volumes
```

Process не работает.

### После `docker rm`

Удаляются:

```text
container object
container writable layer
container-owned runtime state
```

Volume обычно живёт отдельно.

Bind mount data вообще принадлежит host path.



## 23. Минимальная карта знаний

```text
Dockerfile
   ↓
Image
   ↓
Layers
   ↓
OverlayFS
   ↓
Container writable layer
   ↓
Container
   ↓
Runtime config
   ↓
Linux process
   ├─ PID namespace
   ├─ Mount namespace
   ├─ Network namespace
   ├─ UTS/IPC/User namespace
   ├─ Cgroups
   ├─ Root filesystem
   ├─ Mounts
   └─ stdio

Network namespace
   ↓
eth0
   ↓
veth
   ↓
Linux bridge
   ↓
routing/forwarding/NAT
   ↓
host/external network

Docker DNS
   ↓
service name
   ↓
current container IP

Process
   ├─ stdout/stderr
   │    ↓
   │  logging driver
   │    ↓
   │  docker logs / backend
   │
   └─ signals
        ↓
      PID 1
        ↓
      graceful shutdown
```
