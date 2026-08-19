# GitLab CI/CD


## 1. Создание pipeline

```text
Git repository
    │
    │ push / tag / Merge Request
    ▼
GitLab решает: создавать pipeline или нет
    │
    │ workflow:rules
    ▼
Pipeline
    │
    ├── Stage
    │    ├── Job
    │    └── Job
    │
    ├── Stage
    │    └── Job
    │
    └── ...
         │
         │ job подбирает Runner
         ▼
       Runner
         │
         ├── shell executor
         ├── docker executor
         └── ...
              │
              ▼
           выполняет script
              │
              ├── variables
              ├── cache
              ├── artifacts
              ├── services
              └── environment/deployment
```

Ключевая мысль:

**GitLab CI/CD — это система, которая по событиям Git создаёт pipeline, формирует граф jobs, назначает jobs на Runner'ы, передаёт им контекст и хранит результаты выполнения.**



## 2. Pipeline → Stage → Job

### Pipeline

**Pipeline** — один экземпляр выполнения CI/CD-конфигурации для конкретного Git-события и конкретного Git ref/commit.

Pipeline может возникнуть, например, из:

- push в branch;
- push tag;
- Merge Request event;
- schedule;
- manual/API trigger;
- upstream pipeline.

Pipeline имеет собственный статус:

```text
created
pending
running
success
failed
canceled
skipped
...
```

Pipeline — контейнер для jobs и их зависимостей.



### Stage

**Stage** — логическая группа jobs и одновременно базовый механизм порядка выполнения.

Например:

```yaml
stages:
  - lint
  - test
  - build
  - deploy
```

Без дополнительных зависимостей GitLab рассматривает stages последовательно:

```text
lint
 ↓
test
 ↓
build
 ↓
deploy
```

Jobs внутри одного stage могут выполняться параллельно.

```text
          ┌─ unit_test
test ─────┤
          └─ integration_test
```



### Job

**Job** — минимальная единица работы CI/CD, которую GitLab передаёт Runner'у.

Пример:

```yaml
test:
  stage: test
  script:
    - go test ./...
```

Job описывает:

- что выполнить;
- когда выполнить;
- на каком Runner;
- какие variables использовать;
- какие artifacts/cache получить или сохранить;
- какое environment изменить;
- от каких jobs зависеть.



### `script`, `before_script`, `after_script`

```text
before_script
     ↓
script
     ↓
after_script
```

`script` — основная работа job.

`before_script` — подготовительные команды.

`after_script` — завершающие команды; важно помнить, что у него отдельная семантика выполнения/ошибок и его нельзя считать обычным `finally`, полностью эквивалентным конструкции языка программирования.



## 3. Runner

### Что такое Runner

**GitLab Runner** — отдельная программа-агент, которая получает jobs от GitLab и физически их выполняет.

Разделение:

```text
GitLab
→ планирует и хранит CI state

Runner
→ исполняет job
```

GitLab сам shell-команды job не исполняет.



### Как job попадает на Runner

Упрощённо:

```text
job создан
   ↓
GitLab ищет подходящий Runner
   ↓
совпадают требования job и Runner
   ↓
Runner забирает job
   ↓
executor создаёт execution environment
   ↓
script выполняется
```



### Executors

**Executor** определяет, *как Runner создаёт среду выполнения job*.

Основные модели, которые обсуждали:

#### Shell executor

```text
Runner host
   ↓
shell
   ↓
job process
```

Job выполняется непосредственно в ОС Runner-хоста.

Плюсы:

- просто;
- быстро;
- можно использовать локально установленные инструменты и caches.

Минус:

- слабая изоляция;
- job получает существенно более прямой контакт с host.

#### Docker executor

```text
Runner
   ↓
Docker
   ↓
job container
```

Runner создаёт контейнер из указанного `image`, помещает туда checkout проекта и запускает команды job.

Контейнер job временный, но Docker/BuildKit state самого Runner может быть persistent.

### Runner tags

Runner может иметь:

```text
tags:
  docker
  linux
  amd64
```

Job:

```yaml
build:
  tags:
    - docker
```

GitLab подберёт Runner, удовлетворяющий требованиям job.

#### Почему job висит `pending`

Одна из первых вещей при debugging:

```text
Есть ли вообще Runner?
Runner online?
Runner разрешён для проекта?
Совпадают tags?
Runner принимает untagged jobs?
Runner protected?
Подходит ли protected status ref?
Есть свободный concurrency slot?
```

`pending` часто означает не проблему `script`, а то, что **job некому выполнить**.

## 4. Variables

### CI/CD variables

Variable — значение, которое GitLab передаёт job и которое обычно появляется внутри execution environment как environment variable.

```yaml
variables:
  APP_NAME: billing
```

В job:

```bash
echo "$APP_NAME"
```

### Где variables могут задаваться

Мы рассматривали:

```text
predefined GitLab variables
instance/group/project variables
pipeline variables / inputs
.gitlab-ci.yml variables
job variables
downstream variables
```

Нужно помнить о precedence: при совпадении имени выигрывает источник с более высоким приоритетом.

### Predefined variables

GitLab сам создаёт большой набор контекстных переменных:

```text
CI_PROJECT_ID
CI_PROJECT_NAME
CI_PROJECT_PATH
CI_COMMIT_SHA
CI_COMMIT_BRANCH
CI_COMMIT_TAG
CI_DEFAULT_BRANCH
CI_PIPELINE_SOURCE
CI_REGISTRY
CI_REGISTRY_IMAGE
CI_MERGE_REQUEST_IID
...
```

Это API контекста pipeline/job.

Особенно важна:

```text
CI_PIPELINE_SOURCE
```

Она позволяет понять, **почему появился pipeline**.

Например:

```text
push
merge_request_event
pipeline
schedule
web
...
```

### Protected variable

**Protected variable** доступна только pipeline'ам на protected refs согласно правилам GitLab.

Это механизм ограничения того, где секрет вообще может появиться.

Принцип:

```text
feature branch
→ production secret НЕ должен попадать

protected main/tag
→ может получить production secret
```



### Masked variable

**Masked** относится к отображению значения в job log.

Это не то же самое, что protected.

```text
Protected
→ где variable разрешено использовать

Masked
→ стараемся не показать значение в логах
```

Нельзя путать security boundary и log redaction.



### Secrets

CI variable может хранить секрет, но секреты архитектурно лучше получать из специализированного secret storage, когда инфраструктура этого требует.

Мы обсуждали Vault: job может аутентифицироваться и получить secret/file непосредственно во время выполнения, вместо хранения всего секрета в YAML.

Главное правило:

**секрет не должен находиться в repository, template или Docker image.**



## 5. Artifacts и Cache

Это одна из самых важных пар понятий.

### Artifact

**Artifact** — результат job, который GitLab сохраняет и делает доступным другим jobs / пользователю.

Например:

```yaml
build:
  script:
    - go build -o app ./cmd/app
  artifacts:
    paths:
      - app
```

Модель:

```text
build job
   ↓
создал app
   ↓
upload artifact
   ↓
GitLab хранит artifact
   ↓
следующая job может скачать
```

Artifact имеет lifecycle / expiration.



### Cache

**Cache** — механизм ускорения повторных jobs за счёт сохранения повторно используемых файлов.

Например:

```text
Go module cache
npm cache
compiler cache
...
```

Cache не следует воспринимать как надёжный канал передачи обязательного результата между jobs.

#### Главная разница

```text
ARTIFACT
= результат работы, который нужен дальше

CACHE
= оптимизация; отсутствие cache не должно ломать корректность pipeline
```

Если следующая job **обязана** получить файл предыдущей — это artifact, а не cache.



### Cache key

Cache идентифицируется key.

```text
key
→ определяет, какие jobs разделяют cache
```

Плохой key:

```text
слишком общий
→ unrelated jobs портят/вытесняют cache
```

Слишком специфичный:

```text
каждый commit имеет отдельный cache
→ почти нет reuse
```



## 6. Управление выполнением jobs

### `rules`

`rules` решает, должна ли job попасть в pipeline и с каким поведением.

Пример:

```yaml
rules:
  - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

Важно:

**`rules` вычисляются при построении pipeline, а не когда job уже начала выполняться.**



### `when`

Определяет режим запуска job.

Например:

```text
on_success
always
manual
never
delayed
```

Manual job:

```yaml
deploy_prod:
  when: manual
```

требует явного запуска пользователем.



### Branch / tag / Merge Request pipelines

Нельзя сваливать их в одну сущность.

```text
branch pipeline
→ pipeline для branch/ref

tag pipeline
→ pipeline для tag

MR pipeline
→ pipeline в контексте Merge Request
```

Для MR pipeline ключевой контекст:

```text
CI_PIPELINE_SOURCE == "merge_request_event"
```



### `workflow:rules`

Очень важное различие:

```text
workflow:rules
→ должен ли существовать PIPELINE вообще

job rules
→ должна ли существовать конкретная JOB внутри уже создаваемого pipeline
```

Пример:

```yaml
workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

Это позволяет явно определить допустимые типы pipeline и избежать нежелательных duplicate pipelines.



### `needs`

`needs` задаёт **граф выполнения** jobs.

Без `needs`:

```text
Stage A полностью
      ↓
Stage B
```

С `needs` job может стартовать сразу после необходимых ей jobs, не ожидая завершения всех jobs предыдущего stage.

```text
build-a ─────→ test-a
build-b ─────────────→ test-b
```

Это DAG-модель pipeline.



### `dependencies`

`dependencies` исторически управляет тем, **из каких предыдущих jobs скачивать artifacts**.

Ключевая мысль:

```text
needs
→ scheduling / DAG dependency

dependencies
→ artifact download selection
```

Современный `needs` также умеет связываться с artifacts, поэтому эти механики нельзя механически дублировать без понимания.



## 7. Docker и Registry в GitLab CI

### `image`

При Docker executor:

```yaml
image: golang:...
```

означает image, из которого Runner создаёт контейнер job.

Это **не image, который мы обязательно собираем**.

```text
job image
→ среда выполнения CI job

application image
→ продукт docker build
```



### `services`

`services` — дополнительные контейнеры, которые Runner запускает рядом с job container.

Например:

```yaml
services:
  - postgres:...
```

или:

```yaml
services:
  - docker:dind
```

Job и service получают сетевую связность, чтобы job могла обращаться к сервису.



### GitLab Container Registry

Registry хранит OCI/Docker images проекта.

Типичный flow:

```text
docker build
    ↓
image
    ↓
tag
    ↓
docker push
    ↓
GitLab Container Registry
```

Predefined variables:

```text
CI_REGISTRY
CI_REGISTRY_IMAGE
CI_REGISTRY_USER
CI_REGISTRY_PASSWORD
```



### Docker-in-Docker и Docker socket

Эту тему мы грызли отдельно и глубоко.

#### Docker socket binding

Схема:

```text
job container
    │
    │ /var/run/docker.sock
    ▼
HOST dockerd
```

Docker CLI внутри job фактически управляет **хостовым Docker daemon**.

Это очень мощный доступ.

Если daemon rootful, пользователь, способный отправлять произвольные Docker API requests, практически получает путь к host-level control: может попросить daemon создать privileged container, mount host filesystem и т.д.

Поэтому:

**docker.sock — не «просто доступ к Docker images». Это control API Docker daemon.**

User namespace remapping/rootless Docker могут менять security properties, но socket всё равно остаётся крайне чувствительным интерфейсом.



#### DinD

Docker-in-Docker:

```text
Runner host Docker
   │
   ├── job container
   │      └── docker CLI
   │
   └── docker:dind service
          └── dockerd
                └── build/run containers
```

Job CLI разговаривает не с host dockerd, а с dockerd внутри service container.

Но для классического DinD service часто требуется `privileged`.

**Важный вывод.**

`privileged` **не означает «контейнер автоматически находится во всех namespace хоста»**.

Namespaces и privileges — разные механизмы.

Однако privileged container получает очень широкие capabilities/device access и резко ослабленные ограничения, поэтому считать его безопасной песочницей нельзя.



#### Runner в контейнере не создаёт автоматически дополнительную security boundary

Если GitLab Runner сам запущен в Docker, но для создания job/service containers использует:

```text
/var/run/docker.sock → host dockerd
```

то новые контейнеры создаёт **host dockerd**.

Они не становятся «контейнерами внутри Runner-контейнера».

```text
Runner container
      │
      │ docker.sock
      ▼
host dockerd
      │
      ├── job container
      └── dind service container
```

Это sibling containers.



#### Сильная boundary — отдельная VM/kernel

Для untrusted/privileged CI workloads мы пришли к идее:

```text
physical host
   ↓
VM
   ↓
Runner
   ↓
Docker workloads
```

Тогда даже privileged workload находится внутри отдельного guest kernel.

Обсуждали KVM/microVM/Firecracker. Важна не конкретная марка виртуалки, а принцип:

**для сильной изоляции нужен отдельный kernel boundary, а не ещё один Docker namespace.**

Firecracker — KVM-based microVM technology; в Proxmox чаще практичнее сначала использовать обычные VM, а microVM вводить, если действительно нужна плотность/скорость старта.



#### Rootless Docker

Rootless Docker запускает daemon и containers без root privileges host'а, используя user namespaces и другие Linux-механизмы.

Это уменьшает последствия компрометации daemon/container, но имеет ограничения и не превращает любую CI-схему автоматически в безопасную.



### BuildKit и build cache

#### BuildKit

BuildKit — современный backend Docker build.

В современных Docker установках BuildKit — фактически базовый механизм сборки.

Он:

- строит dependency graph build steps;
- использует cache;
- поддерживает cache exporters/importers;
- умеет параллелить независимые операции;
- используется Buildx.



#### Локальный BuildKit cache

Обычный:

```bash
docker build -t app .
```

уже использует локальный build cache builder'а.

То есть:

```text
docker build
    ↓
BuildKit
    ↓
local cache
```

Никаких `--cache-from/--cache-to` для **локального** cache не требуется.

Если четыре persistent Runner:

```text
Runner A → cache A
Runner B → cache B
Runner C → cache C
Runner D → cache D
```

Это вполне разумная схема.



#### Почему cache полезен даже при частых commits

Хорошо построенный Dockerfile:

```dockerfile
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build ./...
```

Изменение исходников инвалидирует поздние steps, но dependency layer может жить долго, пока не меняются `go.mod/go.sum`.



#### Remote BuildKit cache

BuildKit умеет:

```text
cache-to
→ экспортировать cache

cache-from
→ импортировать cache
```

Registry может быть backend'ом cache.

Например GitLab Container Registry:

```text
GitLab Registry
├── application image
└── BuildKit registry cache
```

Но для remote cache параметры относятся к **конкретному build invocation**.

Нет простого глобального `daemon.json` параметра:

```text
для каждого docker build автоматически:
cache-from X
cache-to X
```

Поэтому если хочется сохранить UX:

```bash
docker build -t app .
```

самая простая модель — persistent local caches на Runner'ах.



#### BuildKit GC

Cache нельзя позволять расти бесконечно.

Для встроенного Docker builder GC настраивается через Docker daemon config.

Для standalone/отдельного `buildkitd` — через `buildkitd.toml`.

Концептуально:

```text
reservedSpace
→ объём cache, который хотим сохранить

maxUsedSpace
→ верхняя граница использования

minFreeSpace
→ сколько свободного диска стараемся оставить
```

Правильная модель:

```text
cache остаётся тёплым
+
GC ограничивает диск
```

а не:

```text
cron → rm -rf /var/lib/docker
```



### Docker Hub mirror

Это **отдельная проблема от build cache**.

Нам нужно решить две разные задачи:

```text
1. Не тянуть base images постоянно из Docker Hub
2. Переиспользовать результаты наших build steps
```

Это НЕ один cache.



#### Registry mirror / pull-through cache

Поднимаем CNCF Distribution Registry в pull-through cache mode.

Docker daemon:

```json
{
  "registry-mirrors": [
    "https://docker-mirror.company.local"
  ]
}
```

Тогда:

```dockerfile
FROM ubuntu:24.04
```

остаётся обычным.

Модель:

```text
dockerd
   ↓
local mirror
   ├── HIT → вернуть cached blobs
   └── MISS
         ↓
      Docker Hub
         ↓
      сохранить
         ↓
      вернуть
```

Это работает не только для Buildx, а для обычных pull'ов через настроенный daemon.



#### Отдельный BuildKit daemon

Если registry pull выполняет отдельный `buildkitd`, mirror настраивается уже в его config:

```toml
[registry."docker.io"]
  mirrors = ["docker-mirror.company.local"]
```

Всегда спрашиваем:

**кто именно выполняет pull? `dockerd` или отдельный `buildkitd`?**



#### GitLab Dependency Proxy

GitLab Dependency Proxy тоже умеет cache Docker Hub dependencies, но требует обращения через GitLab-specific image prefix.

Поэтому для требования:

```dockerfile
FROM ubuntu:24.04
```

без изменения Dockerfile удобнее настоящий Docker Registry mirror.

Итоговая схема:

```text
Docker Hub dependencies
→ transparent Distribution mirror

our build steps
→ local BuildKit cache × Runner
→ GC

final application images
→ GitLab Container Registry
```

Remote BuildKit registry cache добавлять только если реальные метрики покажут, что cross-runner misses стоят дорого.



## 8. Environments и Deployments

### Environment

**Environment** — логическое место/цель, куда развёрнуто приложение.

Например:

```text
development
staging
production
review/123
```

Environment — долгоживущая сущность GitLab, которая может иметь историю deployments.



### Deployment

**Deployment** — конкретный факт развёртывания конкретной версии в environment.

Модель:

```text
Environment: production

Deployment #101 → commit A
Deployment #102 → commit B
Deployment #103 → commit C
```

То есть:

```text
Environment
→ место / logical target

Deployment
→ событие изменения этого environment
```



### Job создаёт deployment через `environment`

```yaml
deploy:
  script:
    - ./deploy.sh
  environment:
    name: production
```

Когда job успешно выполняет deployment, GitLab регистрирует соответствующий deployment в environment.



### Dynamic environments и Review Apps

#### Dynamic environment

Environment name может вычисляться:

```yaml
environment:
  name: review/$CI_COMMIT_REF_SLUG
```

Получаем:

```text
review/feature-a
review/feature-b
review/feature-c
```



#### Review App

**Review App** — временно развёрнутая версия приложения для branch/Merge Request, построенная на dynamic environment.

Review App — не отдельный тип Linux/Kubernetes объекта.

Это GitLab-модель:

```text
MR/branch
   ↓
pipeline
   ↓
deployment
   ↓
dynamic environment
   ↓
URL для проверки
```



#### `on_stop`

Environment может указать stop job:

```yaml
environment:
  name: review/...
  on_stop: stop_review
```

Stop job:

```yaml
environment:
  name: review/...
  action: stop
```

Модель:

```text
deploy_review
   ↓
Environment ACTIVE
   ↓
stop_review
   ↓
cleanup resources
   ↓
Environment STOPPED
```



#### MR lifecycle

Современный GitLab умеет связать Review Environment с Merge Request lifecycle.

Ключевой момент:

**нам не обязательно создавать отдельный pipeline по событию `MR closed` через какую-то переменную состояния.**

Схема:

```text
workflow:rules
→ разрешает MR pipeline

MR pipeline
→ deploy Review App
→ environment:on_stop

MR merged / closed
→ GitLab environment lifecycle
→ запускает stop action
```

Это именно та механика, которой очень не хватало в старых GitLab CE-era схемах, из-за чего приходилось poll'ить GitLab API и самостоятельно искать умершие branch/MR environments.



#### `auto_stop_in`

Для временных environments полезен TTL:

```yaml
environment:
  auto_stop_in: 2 days
```

Это дополнительная страховка от забытых environments.



#### Reconciliation / garbage collector

Несмотря на event-driven cleanup, полезно помнить production-grade принцип:

```text
PRIMARY:
MR lifecycle → on_stop → cleanup

SAFETY NET:
periodic reconciler
→ сравнить desired state и actual resources
→ удалить orphan resources
```

Потому что stop job может упасть, Runner может быть недоступен и т.п.

Старый отдельный cleanup-controller, который через GitLab API находил закрытые MR и удалял containers/DB/buckets/directories, сегодня логичнее использовать именно как **reconciliation safety net**, а не основной lifecycle mechanism.



## 9. Организация `.gitlab-ci.yml`

### `include`

Позволяет подключать CI configuration из других файлов/проектов/источников.

Для shared templates:

```yaml
include:
  - project: devops/ci-templates
    ref: v2.4.1
    file: /go/service.yml
```

Очень важно фиксировать versioned `ref`, а не бесконтрольно зависеть от `main`.



### Hidden jobs

Job, имя которой начинается с точки:

```yaml
.go_build:
  script:
    - go build ./...
```

не запускается сама как обычная job и удобна как reusable configuration/template.



### `extends`

Позволяет job наследовать configuration другой job.

```yaml
.go_build:
  script:
    - go build ./...

build:
  extends: .go_build
```

Идея:

```text
template
→ базовое поведение

project job
→ extends
→ project-specific variables/rules/overrides
```



### Templates не должны превращаться в «весь мир»

После обсуждения deployment architecture пришли к важному разделению:

Shared CI templates хороши для повторяемых строительных блоков:

```text
.go_lint
.go_test
.docker_build
.trigger_review
.trigger_release
```

Но не стоит тащить в каждый microservice через include всю инфраструктурную оркестрацию компании.



### Permissions shared templates

Для private `include:project` GitLab проверяет доступ пользователя, который запускает pipeline, к проекту, откуда берётся include.

Отсюда неприятная operational dependency:

```text
новый developer
→ доступ к service есть
→ к ci-template project нет
→ pipeline ломается ещё при конфигурации
```

Для внутренней компании практичная модель:

```text
devops/ci-templates
visibility: Internal

DevOps
→ write/maintain

internal developers/QA
→ read
```

Тогда не нужно добавлять каждого нового человека в DevOps group только ради include.

Важно: `Internal` visibility — прежде всего Self-Managed модель; на GitLab.com её возможности отличаются.



### CI/CD Components и Catalog

#### Что такое Component

CI/CD Component — GitLab-native reusable CI unit.

Это не бинарный package и не ZIP.

Исходник всё равно живёт в GitLab project.

Жёсткая структура:

```text
component project
└── templates/
    ├── build.yml
    └── deploy/
        └── template.yml
```

`templates/` — часть формата component project.



#### `spec:inputs`

Component может иметь формальный интерфейс:

```yaml
spec:
  inputs:
    go-version:
      default: "1.26"

...
```

Consumer передаёт inputs при `include:component`.

Это делает reusable CI больше похожим на версионируемую библиотеку с контрактом, чем на произвольный YAML include.



#### CI/CD Catalog

Project можно пометить:

```text
CI/CD Catalog project = ON
```

После semver tag + GitLab Release версия компонентов публикуется в Catalog.

Важно:

```text
Catalog toggle
+
semver tag
+
GitLab Release
→ опубликованная версия
```

Никакой отдельной сборки/rsync/upload component package нет.

CI job с `release:` — лишь способ автоматизировать создание GitLab Release. Сам `script: echo ...` ничего «не публикует».



#### Components vs обычные templates

```text
include:project
→ просто reusable YAML из repository

component
→ reusable CI unit
→ inputs
→ versioning
→ Catalog
→ include:component
```

Для простой внутренней инфраструктуры мы решили, что обычный `Internal ci-templates project` вполне может быть достаточен.

Components становятся интереснее, когда нужен формализованный CI API/catalog/version lifecycle.



## 10. Debugging GitLab CI/CD

Наш порядок атаки:

```text
1. Pipeline graph
2. Job status
3. Job log
4. Почему job вообще не стартует?
5. Runner / tags / protected constraints
6. rules / workflow
7. artifacts / cache
8. CI Lint
```



### Pipeline graph

Сначала понять:

```text
Какие jobs вообще появились?
Какие skipped?
Какие blocked?
Какие manual?
Какие pending?
Как выглядит DAG?
```

Это часто сразу показывает, проблема в execution или ещё в pipeline construction.



### Job log

Если job реально запустилась — читаем log и отделяем:

```text
GitLab/Runner infrastructure error
от
ошибки пользовательского script
```



### Job не появился вообще

Смотрим:

```text
workflow:rules
job rules
only/except в старых конфигурациях
include resolution
YAML/config errors
```



### Job появился, но `pending`

Смотрим Runner:

```text
Runner online?
tags совпадают?
protected?
allowed project?
concurrency?
executor healthy?
```



### Artifact/cache problems

Спрашиваем:

```text
Файл вообще был создан?
Попал в artifacts:paths?
Artifact upload прошёл?
Следующая job его скачала?
Не перепутали artifact и cache?
Совпадает cache key?
```



### CI Lint

CI Lint — средство проверки/анализа GitLab CI configuration.

Современный UI GitLab перемещал/расширял его, поэтому не стоит привязывать знание к старому месту кнопки.

Что важно концептуально:

```text
CI Lint
→ проверить YAML/config
→ увидеть результат include/extends
→ проверить итоговую конфигурацию
→ диагностировать rules/configuration
```

На собеседовании важнее понимать назначение, чем помнить координаты кнопки в конкретной версии UI.



## 11. Git branches, tags и Merge Requests в CI-контексте

### Branch

Branch — подвижная Git reference на commit.

Push в branch может создать branch pipeline.



### Tag

Tag — Git reference, обычно используемая как именованная версия/release point.

Tag pipeline удобно использовать для release flow.

В CI:

```text
CI_COMMIT_TAG
```

присутствует в tag context.



### Merge Request

MR — GitLab-сущность вокруг предложения интегрировать изменения source branch в target branch.

Она даёт отдельный CI context.

MR pipeline важен потому, что pipeline знает:

```text
это не просто push в feature branch
это изменение, рассматриваемое как Merge Request
```

и получает MR-specific predefined variables.



### MR pipeline vs branch pipeline

Один и тот же commit потенциально может участвовать в разных pipeline contexts.

Поэтому `workflow:rules` нужен ещё и для того, чтобы не получить ненужные duplicate pipelines:

```text
push pipeline
+
MR pipeline
```

когда нужен только один из них.



## 12. Multi-project downstream pipelines

Это один из главных архитектурных выводов дня.

Раньше cross-project deployment часто приходилось делать через:

```text
curl
→ GitLab API
→ trigger token
→ вручную передавать variables
```

Современный GitLab имеет декларативный:

```yaml
trigger:
  project: platform/deployments
```

Это **multi-project downstream pipeline**.



### Общая схема

```text
SERVICE PROJECT
│
├── lint
├── unit tests
├── build image
├── push image
└── trigger
      │
      ▼
PLATFORM / DEPLOYMENT PROJECT
│
├── create environment
├── deploy dependencies
├── deploy candidate
├── DB / fixtures
├── integration orchestration
├── staging/prod deployment
└── cleanup
```

Это очень важная architectural boundary.



### Service repo выражает intent

Service pipeline не обязан знать:

```text
какой Kubernetes namespace
какой Helm chart implementation
сколько replicas
какие соседние services поднять
какую test DB создать
какой sanitized dump залить
какие buckets создать
как устроен ingress
```

Он передаёт контракт:

```text
SERVICE_NAME
IMAGE
SOURCE_SHA
SOURCE_BRANCH
MR_IID
ACTION
ENVIRONMENT_KIND
```

И говорит:

```text
"создай review environment для этого image"
```



### Platform project реализует environment

```text
intent
   ↓
platform/deployments
   ↓
actual infrastructure
```

Это исправляет плохую архитектуру:

```text
каждый microservice
→ включает внутрь себя весь общий infrastructure world
```

на:

```text
microservice
→ вызывает общий deployment control plane через стабильный контракт
```



### `strategy: mirror`

Trigger job может отражать состояние downstream pipeline.

Концептуально:

```text
upstream trigger job
      ↓
downstream pipeline
      ↓
failed
      ↓
upstream видит failure
```

То есть это не fire-and-forget webhook.



### Авторизация

Permissions полностью не исчезли.

Cross-project trigger всё равно имеет security model GitLab и требует, чтобы инициатор/контекст имел допустимые права на downstream project.

Практичная организационная модель:

```text
platform/deployments
visibility: Internal

Platform/DevOps
→ write

Developers / QA
→ read/use as internal clients
```

Тогда project можно воспринимать как внутренний platform service:

```text
Developers/QA
→ клиенты

Platform team
→ владельцы реализации
```



## 13. Где должен жить Environment при multi-project deployment

Мы рассмотрели две модели.

### Модель A

Service project владеет Environment:

```text
service
→ environment
→ on_stop job
→ trigger platform cleanup
```

Работает, но lifecycle размазан по двум проектам.

### Модель B — более чистая для нашей архитектуры

Deployment project владеет Environment:

```text
service
→ trigger intent

platform/deployments
→ deploy job
→ environment
→ deployment history
→ on_stop
→ cleanup
```

Тогда в одном месте находятся:

```text
Environment
Deployment
create implementation
stop implementation
infrastructure ownership
```

Service project знает только deployment intent.



## 14. Review App через downstream deployment project

Полная схема:

```text
Developer
   ↓
push / MR
   ↓
SERVICE MR PIPELINE
   │
   ├── lint
   ├── tests
   ├── build candidate image
   └── trigger platform
             │
             ▼
      PLATFORM PIPELINE
             │
             ├── environment:
             │     review/service/MR
             ├── create namespace/resources
             ├── deploy dependencies
             ├── deploy candidate image
             ├── seed DB
             └── run/trigger integration tests
```

При merge/close:

```text
MR closed/merged
    ↓
Review Environment lifecycle
    ↓
on_stop
    ↓
platform cleanup
    ↓
containers/namespaces/DB/buckets/etc deleted
```

Плюс periodic reconciler как safety net.



## 15. Integration tests: ownership

Не надо смешивать:

```text
КТО ПИШЕТ TEST
```

и:

```text
КТО СОЗДАЁТ СРЕДУ ДЛЯ TEST
```

Возможная декомпозиция:

```text
Service team
→ application + unit/component tests

Platform
→ environment orchestration

QA/Automation
→ integration/e2e test suite
```

Pipeline chain:

```text
service pipeline
    ↓
platform environment pipeline
    ↓
QA integration pipeline
```

Multi-project downstream pipelines позволяют связать эти ownership boundaries без копирования всей логики в каждый service repository.



## 16. Production deployment architecture

После merge:

```text
MR merged
   │
   ├── Review Environment
   │      ↓
   │    stop / cleanup
   │
   └── default branch pipeline
          │
          ├── tests
          ├── build immutable application image
          ├── push GitLab Registry
          └── trigger deployment control plane
                    │
                    ├── dev
                    ├── staging
                    └── production
```

Production authority не должна появляться в arbitrary feature branch pipeline только потому, что developer может менять `.gitlab-ci.yml`.

Главный security principle:

```text
не "спрятать deployment YAML от разработчика"

а

"не дать developer-controlled pipeline production authority"
```



## 17. Итоговая архитектура CI/CD

```text
                       GITLAB

                ┌──────────────────┐
                │   Service repo   │
                │                  │
                │ code             │
                │ Dockerfile       │
                │ small CI config  │
                └────────┬─────────┘
                         │
                         ▼
                 Service pipeline
                         │
              ┌──────────┼──────────┐
              │          │          │
             lint       test      build
                                    │
                                    ▼
                            GitLab Registry
                                    │
                                    ▼
                             trigger:project
                                    │
                                    ▼
                  ┌─────────────────────────┐
                  │ platform/deployments    │
                  │                         │
                  │ Environment lifecycle   │
                  │ Helm/K8s/etc            │
                  │ dependencies            │
                  │ DB/fixtures             │
                  │ Review Apps             │
                  │ stage/prod              │
                  │ cleanup                 │
                  └────────────┬────────────┘
                               │
                               ▼
                       integration / QA
```

Shared configuration:

```text
devops/ci-templates
visibility: Internal

├── lint templates
├── test templates
├── Docker build template
└── small trigger helpers
```

Runner infrastructure:

```text
Ansible / IaC
   ↓
persistent Runner fleet
   │
   ├── Docker configuration
   ├── BuildKit
   ├── local BuildKit cache + GC
   ├── Docker Hub registry mirror
   ├── Runner tags
   └── credentials / certificates
```

## 20. Ментальная модель

```text
Git event
→ workflow
→ pipeline
→ rules/DAG
→ jobs
→ runners
→ artifacts/images
→ downstream deployment intent
→ environment/deployment lifecycle
→ cleanup
```

И организационно:

```text
APPLICATION TEAM
→ что собрать

RUNNER/CI PLATFORM
→ где и как выполнить CI

PLATFORM/DEVOPS
→ где и как развернуть

QA
→ как проверить систему
```

### Архитектурный подход

**Не надо делать `.gitlab-ci.yml` каждого микросервиса моделью всей компании.**

Хорошая граница:

```text
service repository
→ code
→ lint/test/build
→ immutable image
→ deployment intent

platform repository
→ environments
→ deployments
→ infrastructure orchestration
→ Review App lifecycle
→ cleanup

shared CI templates
→ только повторяемые строительные блоки
```

Тогда GitLab CI перестаёт быть гигантским YAML-монолитом и становится связкой небольших декларативных контрактов между командами.
