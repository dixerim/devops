# Kubernetes

## 1. Что такое Kubernetes

**Kubernetes** — оркестратор контейнерных workloads: distributed userspace-система, которая через API хранит desired state, планирует Pods на Nodes и набором controllers/agents приводит реальную инфраструктуру к этому состоянию.

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
Service networking implementation: обычно kube-proxy; иногда kube-proxy replacement от CNI/eBPF implementation
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

Для 2 members:

```text
quorum = 2
```

```text
2 alive → работает
1 alive → quorum потерян
```

Почему не `quorum = 1`: quorum — это большинство от configured members, а не “сколько сейчас осталось живых”.

Общая формула:

```text
quorum = floor(N / 2) + 1
```

Для `N = 2`:

```text
floor(2 / 2) + 1 = 2
```

Поэтому etcd cluster из двух members не переживает потерю одного member. Для отказоустойчивости обычно используют нечётное количество voting members, например `3` или `5`.

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

### Controllers, controller-manager и reconciliation

Kubernetes object сам ничего не делает.

Изменения выполняют controllers — циклы управления, которые наблюдают API state и пишут обратно в API новые/изменённые objects.

`kube-controller-manager` — Control Plane process, внутри которого работают core controllers:

```text
Deployment controller
ReplicaSet controller
StatefulSet controller
DaemonSet controller
Job controller
CronJob controller
EndpointSlice controller
Node controller
PersistentVolume controller
Attach/Detach controller
...
```

Главная модель:

```text
desired state в API
↓
controller смотрит current state
↓
controller создаёт/обновляет/удаляет API objects
↓
другие components доводят это до runtime
```

Пример:

```text
Deployment object
↓ observed by Deployment controller
ReplicaSet object
↓ observed by ReplicaSet controller
Pod objects
↓ observed by scheduler/kubelet
Linux processes
```

Поэтому `Deployment` и `ReplicaSet` не являются процессами. Это API objects, за которыми стоят controllers.

## 9. Namespace

**Kubernetes Namespace** — логический scope для namespaced API objects.

Не путать с Linux namespace.

Минимальный YAML:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```

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

Минимальный YAML:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
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

Node lifecycle наблюдает node controller внутри `kube-controller-manager`: он следит за Node heartbeats/status, выставляет conditions/taints и запускает eviction logic при потере Node.

### Node Ready

Condition `Ready` отражает, считает ли Control Plane Node работоспособной по информации от kubelet/heartbeats.

```text
Ready=True
Ready=False
Ready=Unknown
```

kubelet регулярно отправляет heartbeats в API: чаще всего обновляет lightweight `Lease` object в `kube-node-lease` namespace, а также периодически обновляет `Node.status`.

Если heartbeats перестают приходить, node controller сначала перестаёт считать Node надёжно наблюдаемой:

```text
kubelet heartbeats пропали
→ Node Ready=Unknown
→ Node получает taints вроде node.kubernetes.io/unreachable
→ Pods на этой Node больше не считаются надёжно живыми
→ после timeout control plane начинает eviction/rescheduling через controllers
```

Control Plane при этом не знает, умерла ли машина, сеть до неё или только kubelet. Он видит потерю heartbeats и действует консервативно: перестаёт отправлять туда новые Pods и пытается восстановить desired state на других Nodes.

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

**ReplicaSet** — объект в namespace с desired state: сколько Pods должно существовать и какой selector/template использовать.

Реальную работу выполняет ReplicaSet controller внутри `kube-controller-manager`.

Основные части:

```text
selector → КАКИЕ Pods считать своими
replicas → СКОЛЬКО Pods нужно
template → КАК создать новый Pod
```

Пример YAML для static local PV:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: api-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: example/api:1.0
```

Пример:

```text
desired = 3
actual matching Pods = 2
↓
ReplicaSet controller создаёт ещё один Pod object
```

ReplicaSet обычно не создают вручную, потому что им управляет Deployment.

## 15. Deployment

**Deployment** — объект в namespace с desired state для stateless workload: сколько replicas нужно и какой Pod template должен быть запущен.

Реальную работу выполняет Deployment controller: он наблюдает Deployment object и создаёт/обновляет ReplicaSets через Kubernetes API.

Минимальный YAML:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: example/api:1.0
          ports:
            - containerPort: 8080
```

Связь:

```text
Deployment
↓ owner reference / управляется Deployment controller
ReplicaSet
↓ owner reference / управляется ReplicaSet controller
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

**StatefulSet** — объект в namespace с desired state для stateful Pods со стабильными identities.

Реальную работу выполняет StatefulSet controller внутри `kube-controller-manager`.

Он создаёт Pods напрямую, без ReplicaSet.

Пример YAML для PVC, который может связаться с PV выше:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: db
          image: postgres:17
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

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

**DaemonSet** — объект в namespace с desired state: по одному Pod на каждой подходящей Node.

Реальную работу выполняет DaemonSet controller внутри `kube-controller-manager`: он создаёт/удаляет Pods при появлении, исчезновении или изменении подходящих Nodes.

Пример YAML для static local PV:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      containers:
        - name: agent
          image: example/node-agent:1.0
```

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

**Job** — объект для конечной работы, которая должна успешно завершиться определённое количество раз.

Реальную работу выполняет Job controller внутри `kube-controller-manager`: он создаёт Pods и следит за успешными/неуспешными attempts.

Пример YAML для PVC, который может связаться с PV выше:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate-db
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: example/app:1.0
          command: ["./app", "migrate"]
```

Основные параметры:

```text
completions
parallelism
```

Job controller создаёт Pods напрямую.

Если попытка работы завершилась неуспешно, поведение зависит от Pod `restartPolicy`.

Для Job обычно используются только:

```text
restartPolicy: OnFailure
restartPolicy: Never
```

`restartPolicy: Always` для Job не подходит: Job должен когда-то завершиться.

При `OnFailure`:

```text
container process exit code != 0
→ kubelet перезапускает container внутри того же Pod
→ Pod object обычно остаётся тем же
→ container restart count растёт
```

При `Never`:

```text
container process exit code != 0
→ container не перезапускается
→ Pod становится Failed
→ Job controller создаёт новый Pod attempt, если backoff/лимиты позволяют
```

То есть:

```text
OnFailure → retry внутри того же Pod через container restart
Never     → retry через новый Pod object
```

Это разные lifecycle события.

### CronJob

**CronJob** — объект, создающий Job objects по расписанию.

Реальную работу выполняет CronJob controller внутри `kube-controller-manager`: он смотрит на расписание и создаёт Job objects в нужные моменты.

Пример YAML:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
spec:
  schedule: "*/15 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: example/app:1.0
              command: ["./app", "cleanup"]
```

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

**ConfigMap** — объект в namespace для несекретной application configuration.

ConfigMap хранит набор string keys и values.

Пример:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "prod"
  LOG_LEVEL: "info"
  nginx.conf: |
    server {
      listen 8080;
    }
```

Здесь:

```text
APP_MODE   → key со строковым value
LOG_LEVEL  → key со строковым value
nginx.conf → key, который удобно смонтировать как file name
```

ConfigMap сам не меняет приложение. Он только хранит данные в Kubernetes API.

Pod может получить ConfigMap как:

```text
environment variables → key/value становятся env vars container process
files through volume mount → keys становятся file names, values становятся file contents
```

Environment variables фиксируются при запуске process; изменение ConfigMap не переписывает environment уже работающего process.

Mounted files могут обновляться позже; приложение само должно перечитать/reload configuration.

### Secret

**Secret** — объект в namespace для данных, которые Kubernetes рассматривает как sensitive configuration.

Secret похож на ConfigMap по способам использования, но предназначен для паролей, tokens, certificates и других sensitive values.

Пример:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USER: "app"
  DB_PASSWORD: "secret-password"
```

`stringData` удобно писать руками: Kubernetes примет обычные строки и сохранит их в `data` в base64-представлении.

Pod может получить Secret как:

```text
environment variables → key/value становятся env vars container process
files through volume mount → keys становятся file names, values становятся file contents
```

Base64 encoding не является encryption.

Secret не является автоматически полноценным secure secret storage.

Encryption at rest защищает persistent representation в backing storage, но authorized API client всё равно может получить Secret.

## 21. Requests и limits

Requests и limits задаёт автор workload manifest: разработчик, platform team, Helm chart, Operator или другой client Kubernetes API.

Они задаются на container level в `spec.containers[].resources`.

Пример:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: example/api:1.0
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

Кто использует эти значения:

```text
requests → scheduler решает, на какую Node можно поставить Pod
limits   → kubelet/runtime настраивают Linux cgroups для container
```

Если в Pod несколько containers, requests/limits задаются отдельно для каждого container, а scheduler учитывает сумму requests по Pod.

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

Упрощённо:

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

Отдельно kubelet следит за Node pressure signals и может эвиктить Pods, чтобы освободить ресурсы на Node.

Это не тот же механизм, что kernel OOM.

Типичные signals:

```text
memory.available
nodefs.available / nodefs.inodesFree
imagefs.available / imagefs.inodesFree
pid.available
```

Когда signal пересекает configured threshold, kubelet считает, что Node под давлением:

```text
memory.available слишком низкий
или
disk/inodes заканчиваются
или
PID space заканчивается
```

Дальше kubelet пытается освободить ресурсы:

```text
1. local reclaim, если возможно
   например garbage collection unused images/containers

2. если reclaim не помог
   выбрать Pods для eviction

3. graceful terminate выбранные Pods
   с учётом eviction grace period

4. Pod получает status Failed / reason Evicted
```

Для memory pressure порядок выбора обычно учитывает QoS и превышение requests:

```text
BestEffort → первые кандидаты
Burstable  → если использует больше request
Guaranteed → обычно последние кандидаты
```

Важно:

```text
Node pressure eviction
→ kubelet proactive action до полной катастрофы Node

kernel OOM
→ ядро уже не смогло выделить память и убивает process
```

### QoS classes

QoS class вычисляется Kubernetes автоматически по requests/limits всех containers в Pod.

Он не задаётся отдельным полем.

Возможные классы:

```text
Guaranteed
Burstable
BestEffort
```

### Guaranteed

Pod получает `Guaranteed`, если у каждого container заданы CPU и memory request/limit, и для каждого ресурса:

```text
request == limit
```

Пример:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### Burstable

Pod получает `Burstable`, если он не `Guaranteed`, но хотя бы у одного container задан request или limit.

Пример:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

### BestEffort

Pod получает `BestEffort`, если ни у одного container не заданы ни requests, ни limits.

Пример:

```yaml
resources: {}
```

Практический смысл при memory pressure:

```text
BestEffort → первые кандидаты на eviction
Burstable  → дальше, особенно если usage > request
Guaranteed → обычно последние кандидаты
```

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

Схема:

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

После прихода на Node B ядро Linux снимает VXLAN-обёртку и передаёт внутренний packet дальше в виртуальную сеть Pod'ов.

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

**FQDN (Fully Qualified Domain Name)** → полное DNS-имя.

Типичный Service FQDN:

```text
<service>.<namespace>.svc.cluster.local
```

## 31. Service

**Service** — namespaced Kubernetes object, предоставляющий стабильный сетевой endpoint поверх динамического набора backend endpoints.

Service не создаёт Pods.

Пример YAML:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

Если у Service есть selector, EndpointSlice controller внутри `kube-controller-manager` поддерживает EndpointSlice objects со списком подходящих backend Pods.

Сам Service dataplane реализует не Service object, а node-side implementation: обычно kube-proxy, иногда eBPF/kube-proxy replacement.

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

**NodePort** — тип Service, который делает Service доступным через один и тот же порт на каждой Node.

Например:

```text
<Node-IP>:32080
```

Запрос на `NodeIP:nodePort` попадает на выбранную Node, где правила Service networking перенаправляют его к одному из backend Pods.

Пример YAML:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-nodeport
spec:
  type: NodePort
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 32080
```

Это не обязательно userspace process, который делает `listen(32080)`.

Схема:

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

Пример YAML:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-lb
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
```

Частый production-сценарий: `LoadBalancer` дают не каждому application Service, а edge-компоненту, например Ingress Controller.

```yaml
# внешний вход в cluster
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
spec:
  type: LoadBalancer
  selector:
    app: ingress-nginx
  ports:
    - port: 80
      targetPort: 80
    - port: 443
      targetPort: 443
```

Тогда цепочка выглядит так:

```text
mysite.com
↓ DNS
external LoadBalancer IP
↓
Service type=LoadBalancer для Ingress Controller
↓
Ingress Controller Pod
↓
Ingress/Gateway rules
↓
обычные ClusterIP Services приложения
↓
application Pods
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

IPv6 neighbor discovery; по смыслу решает такую же задачу поиска L2-соседа для IPv6.

### BGP

Routing protocol, распространяющий информацию о том, через какой next hop доступен IP prefix.

Главная граница:

```text
ARP/NDP → local L2 neighbor reachability
BGP     → L3 route advertisement
```

Это не собственные механизмы Kubernetes. Сетевые решения для Kubernetes используют обычные сетевые протоколы.

## 43. K3s ServiceLB

K3s может предоставлять встроенную implementation `Service type=LoadBalancer` через ServiceLB.

Схема:

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

Пример YAML:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

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
→ внешний IP/DNS и L4-доставка трафика к edge Service targets, обычно к Pods Ingress/Gateway Controller

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

## 47. Persistent storage: API objects и реальный backend

Persistent storage имеет два мира:

```text
Kubernetes API abstractions
+
реальный storage backend
```

Главное: Kubernetes не создаёт storage из воздуха.

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

Пример YAML:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-1
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
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
                - worker-1
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

Пример YAML:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 10Gi
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

В static provisioning `storageClassName`, `accessModes` и размер PVC должны совпасть с подходящим PV. В примере выше PVC не выбирает `/mnt/disks/ssd1` напрямую, но его требования подходят к `local-pv-1`, поэтому persistent volume controller может связать:

```text
PVC data
↓ storageClassName=local-storage, size=10Gi, RWO
PV local-pv-1
↓
/mnt/disks/ssd1 на worker-1
```

## 50. PV/PVC binding

**Binding** — установление связи между конкретным PVC и конкретным PV.

```text
PVC request
↔
matching PV
```

Binding — control-plane связь API objects.

Реальную работу binding выполняет persistent volume controller внутри `kube-controller-manager`: он сопоставляет PVC с подходящим PV и записывает эту связь в API.

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

Пример YAML для dynamic provisioning:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: example.com/csi-driver
parameters:
  type: ssd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

StorageClass не является самим volume.

StorageClass обычно используется dynamic provisioner'ом. В CSI-сценарии это чаще external CSI provisioner, а не сам `kube-controller-manager`.

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

Обычно это выглядит так:

```text
PVC создан
↓
external provisioner видит PVC + StorageClass
↓
просит storage backend создать volume
↓
создаёт PV object
↓
persistent volume controller связывает PVC ↔ PV
```

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

У CSI есть операции со стороны Control Plane и операции со стороны Node.

Что ставить в новый cluster:

```text
Kubernetes сам по себе не даёт production storage.
После установки cluster обычно нужно выбрать и установить storage implementation.
```

Выбор зависит от того, где работает cluster:

```text
AWS
→ обычно ставят AWS EBS CSI для RWO block volumes
→ если нужен RWX/shared filesystem, смотрят AWS EFS CSI

GCP/GKE
→ обычно GCE Persistent Disk CSI

Azure/AKS
→ Azure Disk CSI для block volumes
→ Azure File CSI для shared filesystem

OpenStack
→ Cinder CSI

VMware/vSphere
→ vSphere CSI

Bare metal / свои VM без cloud storage
→ Longhorn, OpenEBS, Ceph/Rook или внешний NFS/Ceph

Уже есть NFS
→ NFS CSI может создавать/mount'ить PVC поверх NFS backend

Нужен distributed storage внутри cluster
→ часто смотрят Longhorn, OpenEBS или Rook/Ceph
```

Примеры `provisioner` names, которые потом встречаются в `StorageClass`:

```text
AWS EBS CSI              → ebs.csi.aws.com
AWS EFS CSI              → efs.csi.aws.com
GCE Persistent Disk CSI  → pd.csi.storage.gke.io
Azure Disk CSI           → disk.csi.azure.com
Azure File CSI           → file.csi.azure.com
OpenStack Cinder CSI     → cinder.csi.openstack.org
VMware vSphere CSI       → csi.vsphere.vmware.com
NFS CSI                  → nfs.csi.k8s.io
Ceph RBD CSI             → rbd.csi.ceph.com
CephFS CSI               → cephfs.csi.ceph.com
Longhorn CSI             → driver.longhorn.io
```

Главный практический вопрос:

```text
откуда физически будут браться bytes?
```

После этого выбирают integration:

```text
block volume
→ обычно attach к Node
→ filesystem mount в Pod
→ часто RWO

shared filesystem
→ mount с разных Nodes
→ может поддерживать RWX

local storage
→ физически привязан к конкретной Node
→ Kubernetes не реплицирует bytes сам
```

CSI driver не обязан сам хранить данные. Часто он только говорит внешней системе:

```text
create volume
attach volume to Node
mount volume
expand volume
snapshot volume
delete volume
```

## 54. Attach / detach

**Attach** — сделать attachable storage device доступным конкретной Node.

**Detach** — убрать эту связь.

Attach/detach orchestration выполняет attach/detach controller внутри `kube-controller-manager`, а конкретную операцию к storage backend делает volume plugin/CSI driver.

Для network/cloud block storage:

```text
storage backend
↓ attach
Node sees block device
```

Например:

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

## 60. Storage и смена Node

Базовая идея:

```text
Pod object disposable
PVC object обычно остаётся тем же
PV/backend storage остаётся тем же
меняется только место, где workload пытается mount/use этот storage
```

Дальше поведение зависит от типа storage backend.

### Network filesystem

```text
old Pod on Node A
↓
new Pod on Node B
↓
Node B mounts same remote filesystem
```

Данные физически не переезжают между Nodes. Они лежат в remote filesystem, а новая Node просто подключает тот же backend.

Обычно это модель NFS/EFS/CephFS/других shared filesystem.

Важно: если access mode/backend разрешает несколько Nodes, старый и новый Pod потенциально могут видеть одни и те же bytes. Корректность concurrent access — ответственность filesystem и приложения.

### Network block

```text
old Pod on Node A stops
↓
kubelet unmounts volume on Node A
↓
attach/detach controller + CSI detach volume from Node A
↓
CSI attach same volume to Node B
↓
kubelet mounts filesystem on Node B
↓
new Pod starts with same data
```

Данные не копируются. Меняется attachment одного и того же remote block volume.

Обычно такой volume нельзя одновременно read/write mount'ить на нескольких Nodes как обычный filesystem. Поэтому важны detach/attach ordering, access mode и backend constraints.

### Local PV

```text
old Pod on worker-03
↓
local PV points to worker-03:/mnt/disks/ssd1
↓
new Pod also must be scheduled to worker-03
```

Pod не может быть произвольно перенесён на другую Node, потому что data существует только на конкретной машине.

Если worker-03 умерла, Kubernetes не создаст копию local data на worker-07. Нужна application-level replication, backup/restore или storage system, которая сама реплицирует данные.

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

## 62. kube-scheduler

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

После filtering остаются feasible Nodes: на них Pod в принципе можно поставить.

Scoring выбирает лучшую Node среди допустимых.

```text
all Nodes
↓ filter
feasible Nodes
↓ score
chosen Node
```

Scheduler plugins начисляют Nodes баллы по разным критериям, затем результаты нормализуются, умножаются на веса и суммируются.

Упрощённо:

```text
Node score =
  resources score
+ affinity/preference score
+ topology/spread score
+ image locality score
+ другие plugin scores
```

Примеры факторов:

```text
resource fit/shape
→ насколько удачно Pod укладывается по CPU/memory requests

preferred node affinity
→ мягкие предпочтения пользователя, в отличие от hard nodeSelector/required affinity

pod affinity / anti-affinity
→ ближе или дальше от других Pods с нужными labels

topology spread constraints
→ распределить replicas по zones/nodes/racks более равномерно

taints/tolerations
→ hard filtering для NoSchedule, но некоторые эффекты могут влиять и на предпочтительность

image locality
→ Node, где image уже есть, может получить бонус
```

Filtering отвечает на вопрос: можно ли поставить Pod на Node.

Scoring отвечает на вопрос: какая из допустимых Nodes лучше.

Scoring не гарантирует идеальную глобальную оптимизацию cluster. Scheduler принимает решение для конкретного pending Pod на основе текущего API state и своей configuration.

### Scheduler не является Linux scheduler

Он работает на cluster placement level.

Он не делает continuous live migration уже запущенных Pods для балансировки.

## 63. kube-controller-manager

**kube-controller-manager** — Control Plane process, запускающий множество core Kubernetes controllers.

Внутри по смыслу находятся:

```text
Deployment controller
ReplicaSet controller
StatefulSet controller
DaemonSet controller
Job controller
CronJob controller
EndpointSlice controller
Node controller
PersistentVolume controller
Attach/Detach controller
...
```

Controller-manager обычно работает на уровне Kubernetes API objects, а не напрямую на уровне Linux processes.

## 64. Как работает controller

**Controller** — логика, которая наблюдает нужные Kubernetes API objects и при необходимости вносит изменения, чтобы система приблизилась к desired state.

Это общая модель для многих Kubernetes controllers:

```text
Deployment controller
ReplicaSet controller
StatefulSet controller
Job controller
EndpointSlice controller
...
```

Например ReplicaSet controller видит:

```text
ReplicaSet wants replicas=3
matching Pods сейчас 2
↓
controller создаёт ещё один Pod object через API
```

Типичная внутренняя схема controller:

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

Локальная in-memory копия нужных API objects, которую controller поддерживает через `LIST/WATCH`.

Cache нужна, чтобы controller не перечитывал всё из API на каждое решение.

Cache не source of truth. Источник истины — Kubernetes API/etcd. При restart process cache строится заново.

### Event

Event в этой схеме — сигнал от watch/cache:

```text
"что-то изменилось; это состояние надо пересчитать"
```

а не обязательно готовую команду, которую нужно буквально исполнить.

### Work queue

Внутренняя очередь keys/objects, которые controller должен обработать.

Обычно event не исполняется напрямую. Он кладёт key в work queue, а worker позже делает reconcile: заново читает текущее состояние и решает, нужно ли что-то менять.

## 65. Watch и resourceVersion

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

## 66. Reconciliation

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

## 67. Leader election

В HA Control Plane может быть несколько экземпляров `kube-controller-manager` или `kube-scheduler`.

Чтобы они не выполняли одну active controller/scheduler role одновременно, используется **leader election**.

Схема:

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

## 68. cloud-controller-manager

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

## 69. Как компоненты реально взаимодействуют

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

## 70. End-to-end: manifest → Linux process

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



## 71. `kubectl apply Deployment`

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

## 72. Pod A → Service → Pod B на другой Node

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

## 73. Browser → application Pod

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

## 74. StatefulSet Pod → persistent bytes

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