# Kubernetes

## 1. Что такое Kubernetes

**Kubernetes** — distributed userspace-система, которая через API хранит желаемую конфигурацию и наблюдаемое состояние объектов, а набор независимых компонентов пытается привести реальную инфраструктуру к требуемому состоянию.

Ключевая граница:

```text
Kubernetes API / controllers / scheduler
→ решают, ЧТО должно существовать, ГДЕ и с какими параметрами

kubelet / runtime / network & storage integrations
→ превращают это решение в локальную конфигурацию Node

Linux kernel / сеть / storage backend
→ реально создают processes, namespaces, cgroups, mounts,
  interfaces, routes, NAT, block I/O и packet forwarding
```

Kubernetes не заменяет Linux. Он в основном программирует уже существующие Linux-механизмы и внешнюю инфраструктуру.

Примеры:

```text
process isolation       → Linux namespaces
CPU/memory limits       → cgroups
filesystem mounts       → Linux mounts
Pod network namespace   → Linux network namespace
Pod interface           → veth / virtual interfaces
cross-node overlay      → VXLAN (в одной из реализаций)
Service forwarding      → netfilter/iptables, IPVS или eBPF
persistent storage      → внешний/local storage backend + mounts/devices
```

## 2. Cluster, Control Plane и Worker Node

### Cluster

**Cluster** — набор машин и компонентов, которыми Kubernetes управляет как одной системой.

Cluster не является одним Linux host и не имеет одного общего kernel.

```text
cluster
├── control-plane node(s)
└── worker node(s)
```

### Control Plane

**Control Plane** — управляющая часть Kubernetes: API, persistent state, scheduling и controller logic.

Основные компоненты:

```text
kube-apiserver
etcd
kube-scheduler
kube-controller-manager
cloud-controller-manager (если нужна cloud-specific интеграция)
```

### Worker Node

**Worker Node** — машина, на которой Kubernetes запускает workload Pods.

Типичные node-side компоненты:

```text
kubelet
container runtime
Pod network implementation / CNI components
Service networking implementation (например kube-proxy)
```

### Control-plane node vs master node

Современный термин: **control-plane node**.

`master node` — старое название, всё ещё часто встречающееся в статьях и старой инфраструктуре.

## 3. kubeadm bootstrap: как физически появляется cluster

`kubeadm` — bootstrap CLI для создания и присоединения Kubernetes Nodes.

На всех машинах обычно устанавливаются как минимум:

```text
kubeadm
kubelet
container runtime
```

`kubectl` нужен там, откуда администратор будет управлять cluster.

### Первая control-plane Node

```bash
kubeadm init
```

или заранее подготовленный config:

```bash
kubeadm init --config kubeadm.yaml
```

После bootstrap первая Node получает Control Plane.

### Worker join

На worker запускается команда вида:

```bash
kubeadm join <control-plane-endpoint>:6443 \
  --token ... \
  --discovery-token-ca-cert-hash ...
```

После этого kubelet Worker Node начинает общаться с `kube-apiserver`, а в API появляется Node object.

### Дополнительная control-plane Node

Для HA Control Plane:

```bash
kubeadm join <control-plane-endpoint>:6443 \
  --token ... \
  --discovery-token-ca-cert-hash ... \
  --control-plane \
  --certificate-key ...
```

Флаг:

```text
--control-plane
```

означает, что машина присоединяется как дополнительная control-plane Node, а не как обычный Worker.

## 4. Зачем kubelet нужен на control-plane Node

**kubelet** — локальный агент Kubernetes Node, который обеспечивает запуск локально заданных или назначенных этой Node Pods и репортит их состояние.

В kubeadm-кластере компоненты Control Plane обычно запускаются как **static Pods**.

```text
systemd
↓
kubelet
↓
static Pod manifests
↓
container runtime
↓
kube-apiserver
kube-controller-manager
kube-scheduler
etcd (при stacked etcd)
```

Static Pod — Pod, конфигурация которого берётся kubelet непосредственно с локального диска, а не сначала приходит через Kubernetes API.

Типичная директория kubeadm:

```text
/etc/kubernetes/manifests/
```

Это решает bootstrap-проблему:

```text
нельзя спросить kube-apiserver,
нужно ли запускать kube-apiserver,
пока kube-apiserver ещё не запущен
```

Поэтому:

```text
Linux/systemd → держит kubelet
kubelet → держит static Pods Control Plane
Control Plane → после запуска управляет cluster, включая эту Node
```

### Почему обычные workloads не запускаются на Control Plane

Control-plane Node обычно имеет taint примерно такого смысла:

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

Это запрещает обычному Pod scheduling на такую Node без соответствующего toleration.

Сам kubelet при этом продолжает обслуживать static Pods Control Plane.

## 5. etcd: persistent memory Control Plane

**etcd** — распределённое key-value хранилище, используемое Kubernetes как persistent backing store API state.

Важно:

```text
etcd хранит Kubernetes API state
≠
etcd хранит содержимое volume, файлы приложения или database pages
```

Controllers, scheduler и kubelet обычно не ходят в etcd напрямую:

```text
controller / scheduler / kubelet
↓
kube-apiserver
↓
etcd
```

### etcd cluster, member, leader, quorum

**etcd member** — один экземпляр etcd, участвующий в одном etcd cluster.

Для трёх members:

```text
etcd-1
etcd-2
etcd-3
```

Raft выбирает одного leader, остальные становятся followers.

**Quorum** — большинство members, необходимое для безопасного commit новых изменений.

Для 3 members:

```text
quorum = 2
```

```text
3 alive → работает
2 alive → quorum есть
1 alive → quorum потерян
```

### Stacked etcd

Типичная kubeadm HA-схема:

```text
cp-1 → apiserver + scheduler + controllers + etcd-1
cp-2 → apiserver + scheduler + controllers + etcd-2
cp-3 → apiserver + scheduler + controllers + etcd-3
```

При `kubeadm join --control-plane` kubeadm добавляет новый etcd member и подготавливает локальный static Pod manifest.

После корректного membership bootstrap сами etcd members через Raft решают:

```text
кто leader
как реплицировать writes
когда write committed
есть ли quorum
```

### External etcd

Можно вынести etcd на отдельные машины:

```text
etcd-1
etcd-2
etcd-3
```

а Control Plane держать отдельно:

```text
cp-1
cp-2
cp-3
```

В kubeadm config это задаётся заранее:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration

etcd:
  external:
    endpoints:
      - https://10.0.0.10:2379
      - https://10.0.0.11:2379
      - https://10.0.0.12:2379
    caFile: /etc/kubernetes/pki/etcd/ca.crt
    certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
    keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
```

Дальше:

```text
kube-apiserver(s)
↓
external etcd cluster
```

## 6. Kubernetes API

**Kubernetes API** — интерфейс, через который создаются, читаются, изменяются и удаляются Kubernetes objects.

Главная endpoint-программа:

```text
kube-apiserver
```

`kubectl`, controllers, scheduler, kubelet и другие компоненты — API clients.

### kube-apiserver

**kube-apiserver** — Control Plane process, который предоставляет Kubernetes API и является центральной точкой доступа к API state.

Он отвечает за:

```text
HTTP(S) API
authentication
 authorization
validation
admission
persistent state access
watch/list responses
```

Он не делает:

```text
Pod scheduling
ReplicaSet reconciliation
container startup
application traffic proxying
```

### Authentication

Определяет:

```text
КТО делает request?
```

### Authorization

Определяет:

```text
МОЖЕТ ЛИ этот caller выполнить это действие
над этим resource в этом scope?
```

### Admission

Дополнительная server-side стадия, которая может разрешать, запрещать или преобразовывать API request согласно configured policy/plugins.

## 7. Kubernetes object и manifest

### Kubernetes object

**Kubernetes object** — сущность, представленная через Kubernetes API.

Примеры:

```text
Pod
Deployment
Service
ConfigMap
Node
PersistentVolume
```

Object не является YAML-файлом и не является Linux process.

### Manifest

**Manifest** — текстовое описание object, которое клиент может отправить Kubernetes API.

```text
manifest file
↓
kubectl / Helm / другой client
↓
API request
↓
Kubernetes object
```

### Основные поля manifest

```yaml
apiVersion: ...
kind: ...
metadata: ...
spec: ...
status: ...
```

**`apiVersion`** — версия Kubernetes API schema для object type.

**`kind`** — тип object.

**`metadata`** — identity и metadata object: name, namespace, labels, annotations, UID и т.д.

**`spec`** — declarative configuration, относящаяся к требуемому состоянию/параметрам object.

**`status`** — API-recorded observed state.

Важно:

```text
status
≠ мгновенная физическая истина мира
```

Он может быть stale, потому что компоненты наблюдают реальность и обновляют status асинхронно.

## 8. kubectl и Declarative model

**kubectl** — CLI client Kubernetes API. Он не является controller, scheduler или runtime.

Примеры:

```bash
kubectl get pods
kubectl describe pod api-123
kubectl apply -f deployment.yaml
kubectl delete -f deployment.yaml
```

`kubectl apply` читает manifest и отправляет API request; дальнейшую reconciliation выполняют Kubernetes components.

### Declarative model, desired state и observed state

**Declarative model** — пользователь задаёт требуемую конфигурацию, а Kubernetes components многократно пытаются привести систему к ней.

Общая идея:

```text
desired state
+
observed state
↓
decision
↓
action
↓
new observed state
```

Важно: не каждое поле `spec` обязательно участвует именно в controller reconciliation.

Например:

```text
spec.replicas
→ controller-level reconciliation

resources.limits.cpu
→ конфигурация, которая переводится в cgroup CPU enforcement
```

## 9. Namespace

**Kubernetes Namespace** — логический scope для namespaced API objects.

Не путать с Linux namespace.

Базовые операции через `kubectl`:

```bash
kubectl get pods -n prod
kubectl apply -f app.yaml -n prod
kubectl config set-context --current --namespace=prod
```

`-n/--namespace` выбирает namespace для namespaced API operation; это не переключает Linux namespace process'а.

Примеры namespaced objects:

```text
Pod
Deployment
Service
ConfigMap
Secret
PVC
```

Примеры cluster-scoped objects:

```text
Node
PersistentVolume
StorageClass
```

## 10. Labels и selectors

### Label

**Label** — metadata `key=value`, добавленная к Kubernetes object.

```text
app=api
env=prod
tier=backend
```

### Selector

**Selector** — условие, выбирающее objects по labels.

```text
app=api
```

может выбрать набор Pods.

Selectors используются для динамической связи объектов.

Важно:

```text
selector
≠ ownership
```

Например Service выбирает Pods через selector, но не является их owner.

## 11. Pod

**Pod** — namespaced Kubernetes object и минимальная единица scheduling/execution Kubernetes.

Pod содержит один или несколько containers, которые:

```text
всегда размещаются на одной Node
делят один Pod network namespace
могут совместно использовать volumes
```

### Pod network

Containers одного Pod имеют общий:

```text
network namespace
Pod IP
loopback
port space
```

То есть два containers одного Pod могут общаться через:

```text
localhost
```

### Filesystem

Containers не получают автоматически общий root filesystem.

Для общего filesystem используется общий volume mount.

### Pod lifecycle / phase

Основное high-level поле `status.phase` может принимать, например:

```text
Pending
Running
Succeeded
Failed
Unknown
```

Это coarse API phase Pod, а не полный список container states/conditions.

`Pending` может означать, что Pod object уже существует, но ещё не scheduled, image ещё не pulled или containers ещё не запущены.

`Running` означает, что Pod назначен Node и как минимум один container запущен/запускается/перезапускается; это не равно `Ready=True`.

`Succeeded`/`Failed` относятся к завершившимся Pods.

### Pod disposable

Pod object считается replaceable.

Новый Pod после замены может получить:

```text
новый UID
новый Pod IP
другую Node
```

Pod не live-migrate'ится как процесс/VM. Обычно старый Pod исчезает, а создаётся новый Pod object.

### Container restart vs Pod replacement

```text
container restart
→ может произойти внутри того же Pod

Pod replacement
→ новый Pod object
```

## 12. Node

**Node** — машина, зарегистрированная в Kubernetes cluster, плюс соответствующий cluster-scoped Node API object.

На Node находятся:

```text
kubelet
container runtime
networking implementation
running Pods
```

Node object содержит observed information, например conditions и resource capacity/allocatable.

### Node Ready

Condition `Ready` отражает, считает ли Control Plane Node работоспособной по информации от kubelet/heartbeats.

```text
Ready=True
Ready=False
Ready=Unknown
```

### Что реально запускается на Node

На Worker Node workload в конце сводится к обычным Linux processes, запущенным container runtime и ограниченным/изолированным Linux mechanisms.

```text
Pod API object
↓
kubelet
↓
runtime
↓
Linux process(es)
```

## 13. kubelet и container runtime

### kubelet

**kubelet** — node-local agent, который обеспечивает execution state Pods, назначенных этой Node, и репортит observed state обратно в API.

Он не выбирает Node для Pod.

```text
assigned Pod
↓
kubelet
↓
container runtime
↓
Linux process
```

### Container runtime

**Container runtime** — программа, которая через runtime interface выполняет low-level lifecycle container'ов.

Примеры:

```text
containerd
CRI-O
```

### CRI

**CRI (Container Runtime Interface)** — Kubernetes interface между kubelet и container runtime.

Концептуально:

```text
kubelet
↓ CRI
runtime
↓
Linux process/namespaces/cgroups/mounts
```

## 14. ReplicaSet

**ReplicaSet** — namespaced object, который поддерживает требуемое количество Pods, matching его selector.

Основные части:

```text
selector → КАКИЕ Pods считать своими/matching
replicas → СКОЛЬКО Pods нужно
template → КАК создать новый Pod
```

Пример:

```text
desired = 3
actual matching Pods = 2
↓
ReplicaSet controller создаёт ещё один Pod object
```

ReplicaSet controller работает внутри `kube-controller-manager`.

ReplicaSet обычно не создают вручную, потому что им управляет Deployment.

## 15. Deployment

**Deployment** — namespaced controller object для declarative lifecycle stateless Pods через ReplicaSets.

Связь:

```text
Deployment
↓ owns/manages
ReplicaSet
↓ owns/manages
Pods
```

### Изменение replicas

```text
replicas: 3 → 5
```

обычно приводит к scale существующего ReplicaSet.

### Изменение Pod template

Изменение `spec.template` создаёт новый ReplicaSet.

```text
Deployment
├── old ReplicaSet
└── new ReplicaSet
```

После этого происходит rollout между ними.

## 16. Rolling Update и rollback

Rolling Update постепенно уменьшает replicas старого ReplicaSet и увеличивает replicas нового.

### maxSurge

Сколько дополнительных Pods сверх `replicas` разрешено временно создать во время rollout.

### maxUnavailable

Сколько Pods могут быть временно недоступны во время rollout.

### rollout status

`kubectl rollout status` наблюдает status rollout; `kubectl` не является самим rollout engine.

### rollback

Rollback делает предыдущий Pod template снова требуемым configuration.

Это не time travel:

```text
не возвращаются прежние Pod UID
не гарантируются прежние Pod IP
не откатываются внешние данные
```

## 17. StatefulSet

**StatefulSet** — namespaced controller object для stateful Pods со стабильными identities.

Он создаёт Pods напрямую, без ReplicaSet.

Например:

```text
db-0
db-1
db-2
```

Стабильное имя не означает, что это всегда один и тот же Pod object.

```text
old db-1 UID != new db-1 UID
```

### Stable network identity

StatefulSet обычно используется вместе с Headless Service, чтобы отдельные ordinal Pods имели стабильные DNS identities.

### Порядок создания и удаления

StatefulSet по умолчанию сохраняет ordered semantics для ordinal Pods: lower ordinal создаётся раньше higher ordinal, а удаление идёт в обратном порядке с учётом readiness/lifecycle правил.

Policy `Parallel` ослабляет ordering creation/deletion, когда приложению не нужна строгая последовательность.

### Storage identity

Через `volumeClaimTemplates` каждая ordinal replica может получить свой PVC:

```text
db-0 → PVC data-db-0
db-1 → PVC data-db-1
db-2 → PVC data-db-2
```

StatefulSet НЕ реплицирует application data между replicas.

Если это PostgreSQL/Cassandra/Kafka, replication semantics реализует сама система или специализированный Operator.

## 18. DaemonSet

**DaemonSet** — controller object, обеспечивающий по одному Pod на каждой подходящей Node.

```text
Node A → Pod
Node B → Pod
Node C → Pod
```

Desired count определяется количеством eligible Nodes, а не полем `replicas`.

Типичные применения:

```text
node monitoring agent
log collector
network agent
storage agent
```

Например Flannel часто разворачивается как DaemonSet, чтобы на каждой Node работал `flanneld`.

## 19. Job и CronJob

### Job

**Job** — object для конечной работы, которая должна успешно завершиться определённое количество раз.

Основные параметры:

```text
completions
parallelism
```

Job controller создаёт Pods напрямую.

Если попытка работы завершилась неуспешно, дальнейшее поведение зависит от Pod `restartPolicy` и Job retry/backoff logic:

```text
container restart inside same Pod
или
new Pod attempt
```

Это разные lifecycle события.

### CronJob

**CronJob** — object, создающий Job objects по расписанию.

Поле schedule задаёт календарное расписание запуска Job. CronJob не запускает Pod напрямую.

Связь:

```text
CronJob
↓
Job
↓
Pod
```

## 20. ConfigMap и Secret

### ConfigMap

**ConfigMap** — namespaced object для несекретной application configuration.

Pod может получить ConfigMap как:

```text
environment variables
files through volume mount
```

Environment variables фиксируются при запуске process; изменение ConfigMap не переписывает environment уже работающего process.

Mounted files могут обновляться позже; приложение само должно перечитать/reload configuration.

### Secret

**Secret** — namespaced object для данных, которые Kubernetes рассматривает как sensitive configuration.

Base64 encoding не является encryption.

Secret не является автоматически полноценным secure secret storage.

Encryption at rest защищает persistent representation в backing storage, но authorized API client всё равно может получить Secret.

## 21. Requests и limits

CPU и memory задаются на container level.

### Request

**Request** — количество ресурса, используемое scheduler'ом при placement и некоторыми resource-control механизмами как заявленная потребность.

Scheduler для обычного resource fit смотрит на:

```text
Node allocatable
-
sum(requests assigned Pods)
```

а не на текущий time-series actual usage.

### CPU units

```text
1000m = 1 CPU
500m  = 0.5 CPU
```

`500m` не означает 50% всей многопроцессорной Node.

### CPU limit → Linux cgroup

Kubernetes CPU limit переводится kubelet/runtime в Linux cgroup CPU bandwidth control.

Conceptually:

```text
limit = 500m
```

может соответствовать примерно:

```text
period = 100ms
quota  = 50ms CPU time
```

Когда quota закончилась, runnable tasks cgroup временно не получают CPU runtime до replenishment.

Это kernel terminology `throttling`; это НЕ уменьшение частоты CPU.

### Memory limit

Memory limit переводится в memory cgroup constraint.

При превышении возможен cgroup-local OOM kill даже если на host ещё есть свободная RAM.

```text
process OOM-killed
↓
container terminates
↓
kubelet может restart container
```

### Node pressure eviction

Отдельно kubelet следит за Node pressure signals и может эвиктить Pods, чтобы освободить ресурсы.

Это не тот же механизм, что kernel OOM.

### QoS classes

Conceptually:

```text
Guaranteed
Burstable
BestEffort
```

QoS вычисляется из requests/limits Pod containers; не задаётся отдельным switch.

Guaranteed не означает «невозможно убить».

## 22. Probes

Probe — периодическая проверка состояния container/application, выполняемая kubelet.

### Readiness probe

Отвечает:

```text
можно ли сейчас отправлять application traffic этому Pod?
```

Failure:

```text
Pod остаётся running
но Ready=False
backend исключается из normal Service traffic
```

### Liveness probe

Отвечает:

```text
нужно ли считать container instance сломанным и restart'ить?
```

Failure threshold может привести к container restart.

### Startup probe

Отвечает:

```text
завершился ли startup приложения?
```

Пока startup probe не succeeded, liveness не должна преждевременно убивать медленно стартующее приложение.

### Mechanisms

Любая из probe purposes может использовать:

```text
HTTP
TCP
exec
```

## 23. Три разных IP-мира

Не смешивать:

```text
1. Node network / underlay
2. Pod network
3. Service virtual network
```

Пример:

```text
Node network:
192.168.1.0/24

cp-1      192.168.1.2
worker-1  192.168.1.3
worker-2  192.168.1.4

Pod network:
10.244.0.0/16

Pod A 10.244.1.10
Pod B 10.244.2.20

Service CIDR:
10.96.0.0/12

Service X 10.96.0.20
```

Node IP обычно существует в реальной physical/cloud network.

Pod IP создаётся Pod network implementation.

ClusterIP — virtual Service address и не обязан существовать как IP конкретного interface.

## 24. CNI

**CNI = Container Network Interface**.

CNI — стандартный interface между container runtime/orchestrator и network plugins для настройки container network namespace.

Важно различать:

```text
CNI specification
→ контракт

CNI/network implementation
→ конкретный software, реализующий Pod networking
```

Примеры network implementations:

```text
Flannel
Calico
Cilium
```

Они могут очень по-разному строить data plane.

## 25. Linux primitives Pod network

### Network namespace

Pod получает отдельный Linux network namespace.

Внутри него есть свои:

```text
interfaces
IP addresses
routes
sockets
loopback
```

### veth pair

**veth pair** — пара виртуальных Ethernet interfaces; frame, отправленный в один конец, появляется на другом.

Один конец можно поместить в Pod network namespace, другой оставить в host networking context.

```text
Pod netns
  eth0
   │
 veth pair
   │
host side veth
```

### Linux bridge

Linux bridge — software Ethernet switch.

Он принимает forwarding decisions по destination MAC, а не по destination IP.

```text
bridge = L2 switch
router = L3 forwarding by IP
```

Network implementation не обязана использовать bridge; это один из возможных datapaths.

## 26. Pod CIDR и IPAM

**Pod CIDR** — диапазон IP-адресов, предназначенный для Pod network.

Например:

```text
cluster Pod CIDR = 10.244.0.0/16
```

Отдельным Nodes могут принадлежать поддиапазоны:

```text
worker-1 → 10.244.1.0/24
worker-2 → 10.244.2.0/24
worker-3 → 10.244.3.0/24
```

**IPAM (IP Address Management)** — механизм распределения IP addresses из соответствующих ranges.

Конкретная network implementation может реализовывать IPAM по-разному.

## 27. Flannel и flanneld

**Flannel** — конкретная реализация Kubernetes Pod network.

**flanneld** — userspace daemon/process Flannel, обычно запущенный на каждой Node через DaemonSet.

```text
DaemonSet
↓
Flannel Pod on each Node
↓
flanneld process
```

Он отдельный от:

```text
kubelet
kube-proxy
```

Задача `flanneld` — получить/согласовать network state и запрограммировать Linux networking своей Node.

Например для VXLAN backend:

```text
create/configure VXLAN interface
configure routes
configure FDB/neighbor state
associate remote Pod subnets with remote Node IPs
```

После настройки packet-by-packet encapsulation/decapsulation выполняет Linux kernel.

## 28. VXLAN

**VXLAN = Virtual eXtensible LAN**.

VXLAN позволяет переносить виртуальный Ethernet traffic поверх обычной IP network.

Conceptually:

```text
inner Ethernet frame
↓
VXLAN encapsulation
↓
UDP
↓
outer IP packet Node A → Node B
```

Например:

```text
Node A underlay IP = 192.168.1.10
Node B underlay IP = 192.168.1.20

Pod A = 10.244.1.7
Pod B = 10.244.2.9
```

Inner traffic:

```text
10.244.1.7 → 10.244.2.9
```

может ехать внутри outer packet:

```text
192.168.1.10 → 192.168.1.20
UDP / VXLAN
```

Physical network знает только реальные Node addresses.

После прихода на Node B Linux VXLAN subsystem decapsulates packet и передаёт inner traffic в virtual datapath.

### FDB

**FDB (Forwarding Database)** — таблица L2 forwarding, связывающая virtual MAC с local/remote destination.

VXLAN может использовать FDB для решения:

```text
этот virtual MAC находится за таким remote VXLAN endpoint
```

### Не вся CNI-сеть = VXLAN

VXLAN — один вариант.

Другие implementations могут использовать:

```text
direct routing
BGP
eBPF
другие overlays
```

Например Flannel `host-gw` может программировать обычные Linux routes без VXLAN encapsulation, если underlay network позволяет.

## 29. Cross-node Pod traffic: полный пример

Пусть:

```text
worker-1 = 192.168.1.10
worker-2 = 192.168.1.20

Pod A = 10.244.1.7
Pod B = 10.244.2.9
```

При Flannel VXLAN:

```text
Pod A
↓
packet dst=10.244.2.9
↓
Linux route on worker-1
↓
VXLAN device
↓
encapsulate
↓
outer packet:
192.168.1.10 → 192.168.1.20
↓
real underlay network
↓
worker-2
↓
decapsulate
↓
Pod B 10.244.2.9
```

Это VPN-подобная идея: virtual network использует real IP connectivity между Nodes как transport.

## 30. CoreDNS и Kubernetes DNS

**CoreDNS** — DNS server, который обычно предоставляет Kubernetes service discovery.

Это обычные Pods, доступные через Service, например:

```text
kube-dns Service
ClusterIP = 10.96.0.10
```

kubelet при создании Pod настраивает DNS config внутри Pod, например `/etc/resolv.conf`:

```text
nameserver 10.96.0.10
```

Application делает обычный Linux DNS lookup:

```text
getaddrinfo("api.default.svc.cluster.local")
```

Resolver отправляет обычный DNS packet к CoreDNS Service.

Kubernetes не подменяет `getaddrinfo()`.

Типичный Service FQDN:

```text
<service>.<namespace>.svc.cluster.local
```

## 31. Service

**Service** — namespaced Kubernetes object, предоставляющий стабильный сетевой endpoint поверх динамического набора backend endpoints.

Service не создаёт Pods.

Типичная связь:

```text
Service selector
↓
matching Pods
↓
EndpointSlice
↓
actual backend IP:ports
```

### Зачем Service

Pod IP может меняться при замене Pod.

Service даёт стабильную identity:

```text
Service DNS name
Service virtual IP (для обычного ClusterIP Service)
```

## 32. Endpoint и EndpointSlice

**Endpoint** — конкретный backend IP+port.

**EndpointSlice** — Kubernetes API object, содержащий набор backend endpoints для Service.

Пример:

```text
Service 10.96.0.20:80

EndpointSlice:
10.244.1.5:8080
10.244.2.7:8080
10.244.3.9:8080
```

EndpointSlice controller — controller внутри `kube-controller-manager`.

EndpointSlice сам не пересылает пакеты.

## 33. ClusterIP

**ClusterIP** — Service type с виртуальным cluster-internal IP.

Пример:

```text
10.96.0.20:80
```

Не надо представлять process:

```text
bind(10.96.0.20)
listen(80)
```

Обычно ClusterIP реализуется packet-processing rules:

```text
dst=10.96.0.20:80
↓
choose backend
↓
DNAT
↓
dst=10.244.2.7:8080
```

После DNAT Pod network доставляет packet к backend Pod.

То есть Service и CNI отвечают за разные части:

```text
Service dataplane:
ClusterIP → Pod IP

Pod network/CNI:
Pod IP → физически нужная Node/Pod
```

## 34. NodePort

**NodePort** — Service exposure через порт на Node addresses.

Например:

```text
<Node-IP>:32080
```

Это не обязательно userspace process, который делает `listen(32080)`.

Conceptually:

```text
client
↓
Node-IP:32080
↓
Linux Service dataplane
↓
backend Pod
```

Backend может находиться на другой Node.

## 35. kube-proxy

**kube-proxy** — node-side Kubernetes component, который в классических implementations наблюдает Services/EndpointSlices и программирует Linux Service dataplane.

```text
Service + EndpointSlice API state
↓
kube-proxy
↓
Linux networking rules
```

Он обычно не находится непосредственно в packet path после того, как rules запрограммированы.

Не путать:

```text
kube-proxy ≠ Ingress Controller
kube-proxy ≠ scheduler
kube-proxy ≠ kubelet
```

## 36. iptables / IPVS / eBPF

Это разные способы реализовать Service semantics.

### iptables / netfilter

Классическая схема:

```text
Service VIP
↓
netfilter rules
↓
DNAT
↓
backend Pod IP
```

### IPVS

**IPVS = Linux IP Virtual Server** — kernel L4 load-balancer mechanism.

Он хранит virtual services и real servers в специализированных таблицах.

### eBPF

eBPF позволяет загружать проверяемые programs в Linux kernel hooks и реализовывать packet processing другим datapath.

Например Cilium может заменить kube-proxy Service dataplane eBPF-реализацией.

Главное:

```text
Service = Kubernetes API abstraction
iptables/IPVS/eBPF = возможные Linux implementations
```

## 37. Service packet path

Пусть:

```text
client Pod = 10.244.4.10:53122
Service    = 10.96.0.20:80
backend    = 10.244.2.7:8080
```

Исходный packet:

```text
src=10.244.4.10:53122
dst=10.96.0.20:80
```

После Service dataplane DNAT:

```text
src=10.244.4.10:53122
dst=10.244.2.7:8080
```

Если backend на другой Node, CNI/Pod network доставляет packet туда.

Connection tracking/NAT state обеспечивает корректный return path, чтобы client socket продолжал видеть connection как связь с Service endpoint.


## 38. Service discovery и Headless Service

Обычный Service DNS обычно резолвится в ClusterIP.

Headless Service:

```yaml
clusterIP: None
```

не получает ClusterIP и может использовать DNS для публикации backend addresses/identities напрямую.

Это особенно полезно для StatefulSet stable network identity.

## 39. Service type=LoadBalancer

`Service type=LoadBalancer` — API intent:

```text
этому Service нужен внешний load-balanced endpoint
```

Сам Kubernetes API не создаёт физический внешний LB из воздуха.

Нужна implementation:

```text
cloud provider integration
MetalLB
kube-vip
K3s ServiceLB
другая реализация
```

## 40. Cloud LoadBalancer и cloud-controller-manager

**cloud-controller-manager** — Control Plane component для cloud-specific controller logic.

Для `Service type=LoadBalancer` cloud integration может сделать:

```text
Service object
↓
cloud-controller-manager/provider controller
↓
cloud API
↓
real cloud LB
↓
external IP/DNS
↓
Service.status.loadBalancer
```

Control path и data path различаются:

```text
CONTROL:
Kubernetes → cloud API → LB configuration

DATA:
client → cloud LB → cluster → Pod
```

Client traffic не проходит через `cloud-controller-manager`.

## 41. Bare metal: MetalLB и kube-vip

На bare metal нет cloud API, который создаст LB.

Implementation должна:

```text
1. выделить IP
2. сделать его реально достижимым в сети
```

### L2 advertisement

Для IPv4 может использоваться ARP:

```text
who has 192.168.10.203?
↓
Node отвечает своим MAC
```

Для IPv6 аналогичную neighbor discovery роль выполняет NDP.

### BGP advertisement

Implementation может объявить router'у route:

```text
192.168.10.203/32
via Node A
```

Тогда router знает next hop для VIP.

## 42. ARP / NDP / BGP

### ARP

IPv4 local L2 neighbor resolution:

```text
какой MAC соответствует этому IPv4 соседу?
```

### NDP

IPv6 neighbor discovery; conceptually решает соответствующую neighbor-level задачу для IPv6.

### BGP

Routing protocol, распространяющий информацию о том, через какой next hop доступен IP prefix.

Главная граница:

```text
ARP/NDP → local L2 neighbor reachability
BGP     → L3 route advertisement
```

Это не Kubernetes mechanisms; Kubernetes network products используют обычные сетевые protocols.

## 43. K3s ServiceLB

K3s может предоставлять встроенную implementation `Service type=LoadBalancer` через ServiceLB.

Conceptually:

```text
Service type=LoadBalancer
↓
K3s ServiceLB controller
↓
DaemonSet
↓
ServiceLB Pods on Nodes
↓
hostPort
```

Типичная идея:

```text
client
↓
Node-IP:443
↓
hostPort
↓
ServiceLB machinery
↓
Service dataplane
↓
backend
```

Это другой approach, чем MetalLB, который может рекламировать отдельный VIP через ARP/BGP.

## 44. Ingress

**Ingress** — namespaced API resource, описывающий HTTP/HTTPS routing rules к Services.

Ingress object сам не proxy.

### Ingress Controller

**Ingress Controller** — отдельная программа/workload, которая читает Ingress resources и конфигурирует реальный L7 proxy.

### TLS

Ingress может ссылаться на Secret с certificate/private key. В типичной схеме TLS connection завершается в Ingress Controller proxy:

```text
client TLS
↓
Ingress Controller
↓ TLS termination
HTTP/upstream connection
↓
backend Service
```

Через SNI client указывает hostname во время TLS handshake, что позволяет proxy выбрать подходящий certificate. Конкретная архитектура может завершать TLS и раньше, например на внешнем cloud L7 load balancer.

Примеры:

```text
Traefik
NGINX Ingress Controller
HAProxy-based implementation
```

Пример:

```text
Host: shop.example.com
Path: /api
↓
api-service
```

Связь:

```text
Ingress
↓ explicit backend reference
Service
↓ selector/EndpointSlice
Pod
```

## 45. Envoy / Traefik / Gateway data plane

Traefik/Envoy могут быть реальными userspace L7 proxies:

```text
accept TCP connection
terminate TLS
read HTTP Host/path
choose upstream
proxy bytes
```

Это принципиально отличается от Service L4 forwarding.

### Gateway API

Gateway API — более современная и расширяемая Kubernetes API-модель для network gateways/routing.

Gateway object тоже не является proxy; нужна конкретная implementation.

### External LB vs Ingress/Gateway

Полезная базовая граница:

```text
external LB / IP publication
→ как traffic вообще вошёл в cluster

Ingress/Gateway proxy
→ куда отправить HTTP/TLS traffic внутри
```

Cloud L7 LB может сдвинуть эту границу и выполнить часть L7 routing во внешней инфраструктуре.

## 46. Полный external traffic path

Типичный вариант:

```text
Browser
↓
DNS
↓
external IP
↓
physical/cloud routing
↓
cloud LB / MetalLB / kube-vip / ServiceLB
↓
Service exposing Ingress Controller
↓
Service dataplane
↓
Traefik/Envoy Pod
↓
Host/path/TLS routing
↓
application Service
↓
Service dataplane
↓
application Pod
↓
application socket/process
```

Control Plane не находится в application packet path:

```text
kube-apiserver      ≠ application proxy
etcd                ≠ application proxy
scheduler           ≠ application proxy
controller-manager  ≠ application proxy
```

## 47. Главное: Kubernetes не создаёт storage из воздуха

Persistent storage имеет два мира:

```text
Kubernetes API abstractions
+
реальный storage backend
```

Реальные bytes предоставляет:

```text
local disk
network filesystem
cloud/network block storage
Ceph/Longhorn/другая distributed storage system
SAN/storage appliance
```

Kubernetes управляет связью workload ↔ storage, но сам не является универсальной системой репликации данных.

## 48. PersistentVolume (PV)

**PV** — cluster-scoped Kubernetes object, представляющий конкретный persistent storage resource.

Одним предложением:

```text
PV = конкретный storage resource, известный Kubernetes
```

PV не является:

```text
файлом
mount'ом
filesystem'ом
Pod'ом
```

## 49. PersistentVolumeClaim (PVC)

**PVC** — namespaced Kubernetes object-запрос на persistent storage с заданными параметрами.

Одним предложением:

```text
PVC = "дай мне storage с такими требованиями"
```

Связь:

```text
Pod
↓ references
PVC
↓ bound to
PV
↓ represents
real storage resource
```

PVC не привязывается к Pod как owner; другой Pod в соответствующем lifecycle может использовать тот же PVC.

## 50. PV/PVC binding

**Binding** — установление связи между конкретным PVC и конкретным PV.

```text
PVC request
↔
matching PV
```

Binding — control-plane связь API objects.

```text
binding ≠ mount
```

## 51. StorageClass

**StorageClass** — cluster-scoped object, описывающий класс/способ предоставления persistent storage через конкретную storage implementation.

```text
PVC
"хочу 100Gi класса fast-ssd"
↓
StorageClass fast-ssd
↓
storage implementation
```

StorageClass не является самим volume.

## 52. Static и dynamic provisioning

### Static provisioning

Storage/PV существуют заранее:

```text
admin creates/has storage
↓
PV exists
↓
PVC appears
↓
binding
```

### Dynamic provisioning

Storage создаётся по запросу:

```text
PVC
↓
StorageClass
↓
storage integration
↓
real volume created
↓
PV representation appears
↓
PVC ↔ PV
```

Dynamic provisioning не означает, что Kubernetes физически создаёт storage сам; storage backend делает это через integration.

## 53. CSI

**CSI = Container Storage Interface**.

CSI — стандартный interface, через который Kubernetes интегрируется с внешними storage systems.

```text
Kubernetes
↓
CSI
↓
concrete CSI driver
↓
storage backend
```

CSI driver — реализация этого interface для конкретной storage system.

Conceptually есть control-plane-side и node-side storage operations.

## 54. Attach / detach

**Attach** — сделать attachable storage device доступным конкретной Node.

**Detach** — убрать эту связь.

Для network/cloud block storage:

```text
storage backend
↓ attach
Node sees block device
```

Например conceptually:

```text
/dev/sdX
```

Attach не равен mount.

## 55. Mount / unmount

**Mount** — Linux operation, которая делает filesystem/mountable resource доступным в mount tree по определённому path.

```text
/dev/sdX
↓ mount
/mnt/data
```

**Unmount** — удаляет эту mount relation.

Для block storage типичный порядок:

```text
attach
↓
device available on Node
↓
mount
↓
filesystem path available
```

Для network filesystem отдельной attach phase как block device может не быть.

## 56. Access modes

Основные Kubernetes access modes:

```text
RWO  = ReadWriteOnce
ROX  = ReadOnlyMany
RWX  = ReadWriteMany
RWOP = ReadWriteOncePod
```

Концептуально:

```text
RWO  → read/write с одной Node
ROX  → read-only с нескольких Nodes
RWX  → read/write с нескольких Nodes
RWOP → read/write только одним Pod
```

Важно:

```text
RWO ≠ один Pod
```

Несколько Pods на одной Node потенциально могут использовать один RWO volume.

Access mode не создаёт backend capability; storage system должна физически поддерживать требуемую модель доступа.


## 57. Storage topology и Node constraints

Storage может быть доступен только на определённых Nodes/зонах.

Например local PV:

```text
worker-03
↓
local SSD
↓
PV X
```

Scheduler обязан учитывать:

```text
Pod requires PV X
↓
PV X only usable on worker-03
↓
Pod must run on worker-03
```

Cloud volume может быть ограничен availability zone.

Scheduler выбирает не просто Node, куда помещаются CPU/memory requests, а Node, где доступны все необходимые resources, включая storage topology.

## 58. Local PersistentVolume YAML

Пример local PV:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-db-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - worker-03
```

Здесь:

```text
/mnt/disks/ssd1
→ path/device на конкретной Worker Node

nodeAffinity
→ говорит, на какой Node этот storage физически существует
```

PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
  namespace: prod
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 100Gi
```

Pod:

```yaml
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: db-data
  containers:
    - name: db
      image: postgres
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
```

Полный path:

```text
Pod
↓
PVC db-data
↓
PV local-db-pv
↓
worker-03:/mnt/disks/ssd1
↓ mount
container:/var/lib/postgresql/data
```

## 59. Network filesystem vs block storage vs local storage

### Network filesystem

```text
Node A ─┐
Node B ─┼→ network filesystem
Node C ─┘
```

Новая Node может mount'ить те же remote data, если backend это разрешает.

### Network/block storage

```text
remote block backend
↓ attach
Node
↓ filesystem
↓ mount
Pod
```

При смене Node тот же volume может detach/attach, если backend/topology это поддерживают.

### Local persistent storage

```text
Node A
↓
local disk
```

Данные физически остаются на этой Node.

Kubernetes не копирует их автоматически на Node B.

## 60. Что происходит со storage при замене Pod на другую Node

### Network filesystem

```text
old Pod on Node A
↓
new Pod on Node B
↓
Node B mounts same remote filesystem
```

Данные не переезжали.

### Network block

```text
unmount Node A
↓
detach from Node A
↓
attach to Node B
↓
mount Node B
↓
new Pod
```

Данные не копируются; меняется attachment того же storage resource.

### Local PV

Pod не может быть произвольно перенесён на другую Node, потому что data существует только на конкретной машине.

## 61. StatefulSet → PVC → PV → backend

Для StatefulSet с тремя replicas:

```text
db-0 → PVC data-db-0 → PV A → volume A
db-1 → PVC data-db-1 → PV B → volume B
db-2 → PVC data-db-2 → PV C → volume C
```

Каждый instance получает свою persistent storage identity.

StatefulSet не делает:

```text
volume A → copy → B → copy → C
```

Data replication — responsibility database/message broker/distributed storage system.

## 62. Нужно ли держать PostgreSQL в Kubernetes

Правильный production вопрос:

```text
не "можно ли?"
а "зачем именно Kubernetes улучшает failure/operations model этой БД?"
```

Практический default для одной критичной PostgreSQL instance без HA:

```text
managed PostgreSQL
или
выделенная VM/DB platform
```

а не Kubernetes только потому, что application уже в Kubernetes.

Причины:

```text
Kubernetes не создаёт вторую копию данных
добавляет CSI/PV/PVC/scheduler/kubelet/networking failure domains
требует Kubernetes expertise для DB debugging
host/kernel/storage tuning всё равно остаётся
```

### Когда DB в Kubernetes оправдана

Если компания сознательно строит Kubernetes-based data platform и имеет:

```text
mature DB Operator
proven storage
backup/restore
failure-domain design
monitoring
team expertise
```

тогда Kubernetes-hosted database может быть нормальным platform choice.

### PostgreSQL HA cluster

Если есть primary + standbys:

```text
primary    → its own PVC/PV
standby-1  → its own PVC/PV
standby-2  → its own PVC/PV
```

PostgreSQL replication/WAL реплицирует database state между instances.

Kubernetes PV subsystem НЕ реплицирует PostgreSQL data между replicas.

### Operator

**Operator** — Kubernetes controller, содержащий application-specific operational logic.

Для PostgreSQL Operator может понимать:

```text
primary/standby roles
failover
promotion
replica recreation
backup orchestration
upgrade sequencing
```

StatefulSet знает generic Pod identity/storage lifecycle, но не database semantics.

## 63. Node classes, Linux tuning и placement stateful workloads

Containers используют host Linux kernel, поэтому требования database/Redis/RabbitMQ к kernel не исчезают.

Node OS можно готовить через:

```text
Ansible
cloud-init
golden images / Packer
Terraform + image pipeline
```

Полезное разделение:

```text
base_linux
↓
kubernetes_worker
↓
stateful_common
↓
postgres_specific / redis_specific / rabbit_specific
```

Пример Node labels:

```text
workload=general
workload=postgres
```

И taint для dedicated Nodes:

```text
workload=postgres:NoSchedule
```

Workload использует:

```text
nodeSelector / nodeAffinity
+
toleration
```

чтобы scheduler размещал его только на подготовленных Nodes.

Главная граница:

```text
Ansible/image pipeline
→ конфигурирует Linux host

Kubernetes scheduling constraints
→ решают, какие Pods могут попасть на этот host
```

## 64. kube-scheduler

**kube-scheduler** — Control Plane component, выбирающий Node для Pod, которому Node ещё не назначена.

Он не создаёт Pod и не запускает container.

```text
Pod exists
spec.nodeName = empty
↓
scheduler
↓
choose Node
↓
record assignment via API
↓
kubelet on chosen Node acts
```

### Filtering

Отбрасываются Nodes, которые не удовлетворяют hard constraints.

Например:

```text
CPU/memory requests
node selectors/affinity
 taints/tolerations
storage topology
```

### Scoring

Подходящие Nodes получают scores, после чего выбирается предпочтительная.

```text
all Nodes
↓ filter
feasible Nodes
↓ score
chosen Node
```

### Scheduler не является Linux scheduler

Он работает на cluster placement level.

Он не делает continuous live migration уже запущенных Pods для балансировки.

## 65. kube-controller-manager

**kube-controller-manager** — Control Plane process, запускающий множество core Kubernetes controllers.

Внутри conceptually находятся:

```text
Deployment controller
ReplicaSet controller
StatefulSet controller
DaemonSet controller
Job controller
CronJob controller
EndpointSlice controller
...
```

Controller-manager обычно работает на уровне Kubernetes API objects, а не напрямую на уровне Linux processes.

## 66. Controller / control loop

**Controller** — логика, которая наблюдает relevant API state и при необходимости вносит изменения, чтобы система приблизилась к требуемому состоянию.

Типичная implementation architecture:

```text
kube-apiserver
↓ LIST/WATCH
local cache
↓ events
work queue
↓
controller worker
↓
reconcile object/key
↓
API write if needed
```

### Cache

Local in-memory representation relevant API objects.

Cache не source of truth и исчезает при process restart.

### Event

Event означает:

```text
"что-то изменилось; это состояние надо пересчитать"
```

а не обязательно готовую команду, которую нужно буквально исполнить.

### Work queue

Внутренняя очередь keys/objects, которые нужно reconcile.

## 67. Watch и resourceVersion

**watch** — Kubernetes API mechanism, позволяющий client получать поток изменений objects после определённой известной точки API state.

Типичная последовательность:

```text
LIST
↓
current objects + resourceVersion
↓
WATCH from that resourceVersion
↓
ADDED / MODIFIED / DELETED
```

**resourceVersion** — непрозрачный identifier версии API state/object, используемый в том числе для LIST/WATCH continuity и optimistic concurrency.

Не надо трактовать resourceVersion как timestamp или число, к которому можно прибавить 1.

Если watch слишком старый/оборвался и продолжить историю нельзя:

```text
re-LIST
↓
rebuild cache
↓
new WATCH
```

Correctness controller не должна требовать идеальной вечной доставки каждого event.

## 68. Reconciliation

**Reconciliation** — процесс, в котором controller смотрит на текущее relevant desired/observed state и выполняет corrective action, если это нужно.

Пример ReplicaSet:

```text
desired = 3
actual = 2
↓
create one Pod object
```

После следующего observation:

```text
desired = 3
actual = 3
↓
no-op
```

Главный принцип:

```text
re-read current state
+
recompute needed action
```

а не:

```text
"я помню, что остановился на шаге 482"
```

Это позволяет переживать:

```text
process restart
API timeout
network interruption
duplicate events
partial failure
```

Reconciliation не является transaction; промежуточные состояния допустимы, если последующие loops способны их исправить.

## 69. Leader election

При HA может быть несколько экземпляров `kube-controller-manager` или `kube-scheduler`.

Чтобы они не выполняли одну active controller/scheduler role одновременно, используется **leader election**.

Conceptually:

```text
controller-manager A → leader
controller-manager B → standby
controller-manager C → standby
```

Kubernetes может использовать `Lease` API object для coordination.

**Lease** — API object, представляющий временное владение leadership role.

Leader периодически renew'ит Lease.

Если renewal прекращается и Lease expires, другой instance может acquire leadership.

### Optimistic concurrency

Если два candidates пытаются обновить один Lease на основе одной старой `resourceVersion`, один write succeeds, другой получает conflict.

### Разные leaders — разные election domains

Не путать:

```text
etcd Raft leader
kube-controller-manager leader
kube-scheduler leader
```

Это независимые роли.

`kube-apiserver` может работать active-active; ему не нужен один глобальный leader для ordinary request handling.

## 70. cloud-controller-manager

**cloud-controller-manager** — Control Plane process для controllers, которым нужна логика конкретного cloud provider.

Он связывает Kubernetes API state с cloud provider API.

Например:

```text
Service type=LoadBalancer
↓
cloud controller
↓
AWS/Azure/GCP API
↓
real cloud resource
```

Он не является application traffic proxy и не запускает containers.

## 71. Как компоненты реально взаимодействуют

Главный архитектурный принцип:

```text
компоненты редко командуют друг другу напрямую;
они координируются через Kubernetes API state
```

Пример:

```text
Deployment controller
↓ writes ReplicaSet object via API

ReplicaSet controller
↓ observes ReplicaSet via API
↓ writes Pod object

scheduler
↓ observes unscheduled Pod
↓ writes Node assignment

kubelet
↓ observes Pod assigned to its Node
↓ starts container
```

API state одновременно является:

```text
persistent state
+
coordination medium
```

## 72. End-to-end: manifest → Linux process

Применение Deployment:

```text
deployment.yaml
↓
kubectl apply
↓
kube-apiserver
↓
etcd-backed API state
↓
Deployment controller
↓
ReplicaSet object
↓
ReplicaSet controller
↓
Pod object
↓
kube-scheduler
↓
Node assignment
↓
kubelet
↓
CRI
↓
container runtime
↓
Linux namespaces/cgroups/mounts
↓
Linux process
```

Обратный observation path:

```text
Linux/runtime result
↓
kubelet
↓
Pod status
↓
kube-apiserver
↓
controllers/other components observe again
```

Kubernetes не «запускает container» одним действием; это серия state transitions и реакций независимых components.

## 73. Helm

**Helm** — отдельный client/tool поверх Kubernetes API для parameterized generation и lifecycle management наборов Kubernetes manifests.

Helm не является Control Plane component.

### Chart

**Chart** — directory/package со структурой Helm application package.

Типично:

```text
Chart.yaml
values.yaml
templates/
```

Chart сам не является Kubernetes object.

### Templates

`templates/` содержат text templates с Go-template expressions:

```text
{{ ... }}
```

До rendering это не обязательно valid Kubernetes manifest.

### values.yaml

Default input parameters Chart.

Effective values формируются из defaults + overrides.

### Release

**Release** — конкретная Helm-managed installation Chart с определённым name, values, generated manifests и revision history.

Helm revision ≠ Deployment revision.

## 74. Helm commands

### install

```bash
helm install RELEASE ./chart
```

```text
Chart + values
↓
render manifests
↓
Kubernetes API
↓
Release revision 1
```

### upgrade

```bash
helm upgrade RELEASE ./chart
```

Рендерит новый desired set manifests и применяет изменения через API.

Kubernetes controllers выполняют actual workload rollouts.

### rollback

```bash
helm rollback RELEASE REVISION
```

Старая Release configuration применяется заново как новая текущая revision.

Это не возвращает физический cluster назад во времени.

### uninstall

```bash
helm uninstall RELEASE
```

Удаляет Helm-managed resources согласно lifecycle/policies; persistent/external resources могут сохраняться в зависимости от configuration.

## 75. Helm validation и dry run

### helm lint

```bash
helm lint ./my-chart
```

Проверяет Chart structure/templates/schema и ловит часть ошибок до изменения cluster.

### helm template

```bash
helm template RELEASE ./my-chart -f values.yaml
```

Локально рендерит окончательные Kubernetes manifests и печатает их, ничего не устанавливая.

Это лучший ответ на вопрос:

```text
"во что именно сейчас собираются мои templates?"
```

### dry-run

```bash
helm install RELEASE ./chart --dry-run --debug
helm upgrade RELEASE ./chart --dry-run
```

Позволяет прогнать install/upgrade-like rendering/validation без обычного deployment.

### values.schema.json

Chart может содержать JSON Schema для validation final merged values.

### Важная лестница

```text
valid template syntax
≠ valid YAML
≠ valid Kubernetes resource
≠ admissible API request
≠ running workload
≠ functioning application
```

### Atomicity

Helm install/upgrade набора разных Kubernetes objects не является универсальной ACID transaction.

Некоторые resources могут уже быть приняты API до ошибки на следующем.

`--atomic` добавляет cleanup/rollback behavior и `--wait`, но не превращает operation в настоящую database transaction: внешние side effects/hook effects могли уже произойти.

## 76. Ownership/control relationships

```text
Deployment → ReplicaSet → Pod
StatefulSet → Pod
DaemonSet → Pod
CronJob → Job → Pod
```

Это lifecycle/controller relationships.

## 77. Non-ownership relationships

```text
Service → Pods
```

через selector + EndpointSlice backend membership.

```text
Ingress → Service
```

через explicit backend reference.

```text
Pod → ConfigMap / Secret
```

через configuration references.

```text
Pod → PVC → PV → storage backend
```

через storage references/binding.

```text
Pod → Node
```

через scheduler assignment.

Один Pod одновременно может быть:

```text
owned by ReplicaSet
selected by Service
referencing ConfigMap
referencing PVC
assigned to Node
```

Это независимые типы отношений.

## 78. Control Plane responsibility map

```text
kube-apiserver
→ API boundary, authn/authz, validation/admission, API state access

etcd
→ persistent distributed backing store

kube-controller-manager
→ core reconciliation/controllers

kube-scheduler
→ Pod → Node placement

cloud-controller-manager
→ cloud-specific reconciliation
```

Node side:

```text
kubelet
→ local Pod execution/reconciliation

container runtime
→ container lifecycle machinery

CNI/network implementation
→ Pod network

kube-proxy/eBPF service implementation
→ Service dataplane
```

## 79. Networking responsibility map

```text
physical/cloud underlay
→ Node-to-Node real connectivity

CNI / Pod network
→ Pod IP connectivity across Nodes

Service dataplane
→ Service VIP/NodePort → backend Pod IP

CoreDNS
→ Service discovery names → Service/backend identities

LoadBalancer implementation
→ external IP/reachability into cluster

Ingress/Gateway proxy
→ HTTP/TLS L7 routing
```

## 80. Storage responsibility map

```text
PVC
→ request

StorageClass
→ class/how to provision

CSI/storage integration
→ bridge between Kubernetes and storage system

PV
→ representation of concrete allocated storage resource

storage backend
→ actual bytes

attach
→ make device available to Node

mount
→ expose filesystem/resource in Linux mount tree
```

## 81. Stateful systems responsibility map

```text
Kubernetes
→ process lifecycle, placement, service identity, API coordination

storage backend
→ persistence/durability of physical bytes

PostgreSQL/Kafka/RabbitMQ/etc.
→ application-level replication, quorum, data semantics

Operator
→ application-specific operational orchestration

Linux kernel
→ CPU, memory, I/O, networking behavior
```

## 82. Object ≠ process

```text
Pod object
≠ Linux process

Service object
≠ listening process

PV object
≠ disk/filesystem itself
```

## 83. Manifest ≠ object ≠ physical reality

```text
YAML manifest
↓ client/API
Kubernetes object
↓ controllers/node components
physical state
```

Эти уровни могут временно расходиться.

## 84. Service IP ≠ Pod IP ≠ Node IP

```text
Node IP
→ real underlay host address

Pod IP
→ Pod network address

ClusterIP
→ virtual Service address
```

## 85. ClusterIP routing ≠ CNI routing

```text
ClusterIP → backend Pod IP
```

делает Service dataplane.

```text
backend Pod IP → actual remote Node/Pod
```

делает Pod network/CNI implementation.

## 86. attach ≠ mount

```text
attach
→ Node получила доступ к storage device

mount
→ filesystem/resource появился по path
```

## 87. RWO ≠ один Pod

```text
RWO = одна Node
RWOP = один Pod
```

## 88. StatefulSet ≠ database HA

StatefulSet даёт:

```text
stable identities
stable PVC relationships
ordered lifecycle options
```

Он не знает:

```text
кто PostgreSQL primary
как Kafka elects leader
как Cassandra реплицирует partitions
```

## 89. Kubernetes replication ≠ data replication

```text
ReplicaSet replicas=3
```

означает три Pods/process instances.

Это не означает, что их internal application data автоматически реплицируется.

## 90. Reconciliation ≠ transaction

Kubernetes допускает промежуточные states.

Controller logic должна быть способна после restart/failure снова посмотреть на current state и продолжить convergence.

## 91. kube-proxy ≠ userspace proxy на каждом packet

В современных классических datapaths kube-proxy в основном программирует kernel networking state; packet forwarding затем делает Linux kernel.

## 92. CNI ≠ конкретно VXLAN

CNI — interface/ecosystem integration point.

VXLAN — один конкретный network mechanism, который может использовать network implementation.

## 93. `kubectl apply Deployment`

```text
manifest
↓
kubectl
↓
kube-apiserver
↓
etcd
↓
Deployment controller
↓
ReplicaSet
↓
ReplicaSet controller
↓
Pod
↓
scheduler
↓
Node
↓
kubelet
↓
runtime
↓
Linux process
```

## 94. Pod A → Service → Pod B на другой Node

При Flannel VXLAN + iptables-like Service dataplane:

```text
Pod A
10.244.1.7
↓
connect(10.96.0.20:80)
↓
Service dataplane DNAT
10.96.0.20:80 → 10.244.2.9:8080
↓
Linux routing sees remote Pod subnet
↓
VXLAN encapsulation
192.168.1.10 → 192.168.1.20
↓
real network
↓
remote Node decapsulation
↓
Pod B 10.244.2.9:8080
```

## 95. Browser → application Pod

```text
Browser
↓
DNS
↓
external IP
↓
cloud LB / MetalLB / kube-vip / ServiceLB
↓
Ingress Controller Service
↓
Service dataplane
↓
Traefik/Envoy
↓
Ingress/Gateway routing
↓
application Service
↓
Service dataplane
↓
application Pod
↓
application process socket
```

## 96. StatefulSet Pod → persistent bytes

```text
StatefulSet
↓
Pod db-0
↓
PVC data-db-0
↓
PV X
↓
CSI/storage integration
↓
storage backend
↓
attach (если нужно)
↓
mount
↓
container path
↓
application filesystem syscalls
```

## 97. HA Control Plane bootstrap

Stacked etcd:

```text
cp-1: kubeadm init
↓
kubelet starts static Pods
↓
apiserver/controllers/scheduler/etcd-1

cp-2: kubeadm join --control-plane
↓
static control-plane Pods + etcd-2

cp-3: kubeadm join --control-plane
↓
static control-plane Pods + etcd-3

etcd members
↓ Raft
leader + quorum
```

## 98. Финальная mental model

Kubernetes удобнее всего держать в голове как набор независимых feedback loops вокруг общего API state.

```text
                    Kubernetes API
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
 controllers         scheduler          kubelet
       │                 │                 │
create/update         assign Node      execute locally
API objects                                │
       │                                   ▼
       │                               runtime + Linux
       │
       ├──────── networking controllers/agents
       │                    ↓
       │      routes/VXLAN/eBPF/netfilter
       │
       └──────── storage controllers/drivers
                          ↓
                 volumes/mounts/backend
```