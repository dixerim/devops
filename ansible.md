# Ansible

## 1. Что такое Ansible и как он работает

**Ansible** — инструмент автоматизации управления удалёнными машинами.

Он нужен, чтобы не заходить вручную по SSH на каждый сервер и не выполнять одни и те же действия руками. Через Ansible можно централизованно:

- устанавливать пакеты;
- создавать пользователей;
- копировать и генерировать конфиги;
- управлять сервисами;
- менять права на файлы;
- выполнять команды;
- приводить серверы к нужному состоянию.

### Control node

**Control node** — машина, с которой запускается Ansible.

На ней находится Ansible, inventory, playbook'и и остальные файлы автоматизации.

### Managed host

**Managed host** — машина, которой Ansible управляет.

Обычно Ansible подключается к managed host по SSH.

Упрощённая схема:

```text
Control node -> SSH -> Managed host
```

### Agentless

Ansible использует **agentless-модель**.

Это означает, что на управляемом сервере не требуется постоянно работающий специальный Ansible-agent.

Обычно Ansible:

1. запускается на control node;
2. определяет целевые managed hosts;
3. подключается к ним по SSH;
4. выполняет нужные действия;
5. получает результат;
6. завершает соединение.

Упрощённая модель:

```text
Пользователь -> Ansible -> SSH -> Linux host -> изменение системы
```

## 2. Inventory: hosts, groups, variables

### Inventory

**Inventory** — описание машин, которыми Ansible может управлять.

Он содержит hosts, groups и связанные с ними variables.

Пример:

```ini
[web]
web1.example.com
web2.example.com

[db]
db1.example.com
```

### Host

**Host** — конкретная управляемая машина в inventory.

Например:

```text
web1.example.com
```

### Group

**Group** — логическая группа hosts.

Например:

```ini
[web]
web1.example.com
web2.example.com
```

Группы нужны, чтобы выполнять действия сразу на наборе машин.

Один host может входить сразу в несколько groups, например:

```text
web
production
eu
```

### Host variables

**Host variables** — переменные, относящиеся к конкретному host.

Пример:

```ini
[web]
web1.example.com http_port=8080
web2.example.com http_port=8081
```

### Group variables

**Group variables** — переменные, относящиеся ко всей группе.

Пример:

```ini
[web:vars]
app_env=production
```

Все hosts группы `web` получают эту переменную.

Главная модель:

```text
Inventory = hosts + groups + связанные variables
```

## 3. Ad-hoc команды и modules

### Ad-hoc command

**Ad-hoc command** — разовая команда Ansible, запускаемая напрямую из CLI без playbook.

Она нужна для быстрых единичных действий или диагностики.

Общая форма:

```bash
ansible <hosts> -m <module> -a "<arguments>"
```

Где:

- `ansible` — CLI-программа;
- `<hosts>` — целевые hosts или groups;
- `-m` — module;
- `-a` — аргументы module.

Пример:

```bash
ansible web -m ping
```

### Module

**Module** — готовая операция, которую Ansible умеет выполнять на managed host.

Часто используемые modules:

- `ping` — проверить, что Ansible может подключиться к host и выполнить код;
- `package` — управлять пакетами;
- `user` — управлять пользователями;
- `file` — управлять файлами, каталогами и правами;
- `copy` — копировать файлы;
- `service` / `systemd` — управлять сервисами;
- `command` — выполнить программу;
- `shell` — выполнить команду через shell.

Специализированные modules обычно предпочтительнее `shell`, потому что они часто умеют определять текущее состояние системы и не делать лишних изменений.

Главная модель:

```text
Ad-hoc command = разовый вызов module на выбранных hosts
```

## 4. Playbook: plays, tasks, changed / ok / failed

### Playbook

**Playbook** — YAML-файл, описывающий последовательность действий Ansible.

Пример:

```yaml
- name: Configure web servers
  hosts: web

  tasks:
    - name: Install nginx
      package:
        name: nginx
        state: present

    - name: Start nginx
      service:
        name: nginx
        state: started
```

### Play

**Play** — блок playbook, который связывает набор hosts с набором tasks.

Например:

```yaml
hosts: web
```

означает, что play выполняется на группе `web`.

### Task

**Task** — одно действие внутри play.

Обычно task вызывает один module с нужными параметрами.

Например:

```yaml
- name: Install nginx
  package:
    name: nginx
    state: present
```

### Структура

```text
Playbook
  -> Plays
      -> Tasks
          -> Modules
```

### ok

**ok** — task выполнен успешно, но состояние менять не пришлось.

### changed

**changed** — task выполнен успешно и Ansible реально изменил состояние host.

### failed

**failed** — task завершился ошибкой.

### skipped

**skipped** — task был сознательно пропущен, например из-за условия.

Пример:

```text
TASK [Install nginx]
web1 : ok
web2 : changed
web3 : failed
```

## 5. Idempotency и desired state

### Desired state

**Desired state** — желаемое состояние системы.

Например:

```yaml
package:
  name: nginx
  state: present
```

Здесь описано не действие «выполни apt install nginx», а состояние:

```text
nginx должен быть установлен
```

### Idempotency

**Idempotency** — свойство операции, при котором повторный запуск приводит к тому же конечному состоянию и не делает лишних изменений.

Первый запуск:

```text
nginx не установлен
-> Ansible устанавливает nginx
-> changed
```

Повторный запуск:

```text
nginx уже установлен
-> менять ничего не нужно
-> ok
```

Упрощённая модель:

```text
current state != desired state
        ->
      changed

current state == desired state
        ->
        ok
```

Идемпотентность не гарантируется автоматически для любого действия.

Например:

```yaml
shell: echo hello >> /tmp/test.txt
```

будет дописывать строку при каждом запуске.

Главная идея:

> Ansible старается приводить систему к описанному состоянию, а не просто бездумно выполнять команды.

## 6. Variables, facts, templates, conditions, loops

### Variables

**Variables** — значения, которые можно подставлять в tasks и другие части Ansible-конфигурации вместо захардкоженных данных.

Пример:

```yaml
vars:
  app_port: 8080
```

Использование:

```yaml
{% raw %}"{{ app_port }}"{% endraw %}
```

Variables позволяют использовать один playbook для разных hosts и окружений.

### Facts

**Facts** — информация о managed host, которую Ansible может собрать автоматически.

Например:

- hostname;
- IP-адреса;
- операционная система;
- архитектура;
- CPU;
- память.

Facts можно воспринимать как автоматически полученные variables о текущем host.

### Templates

**Templates** — файлы-шаблоны, обычно использующие Jinja2.

В шаблон можно подставлять variables.

Пример:

```jinja2
{% raw %}listen_port = {{ app_port }}{% endraw %}
```

Из одного template для разных hosts могут получаться разные конфигурационные файлы.

### Conditions

**Conditions** — условия, определяющие, должен ли выполняться task.

Для этого используется `when`.

Пример:

```yaml
when: ansible_os_family == "Debian"
```

Task выполняется только если условие истинно.

### Loops

**Loops** — повторение одного task для нескольких значений.

Пример:

```yaml
loop:
  - nginx
  - curl
  - git
```

Один task будет последовательно обработан для каждого элемента списка.

Главная модель:

```text
variables  -> наши данные
facts      -> данные о host
templates  -> генерация файлов из данных
conditions -> решают, выполнять ли task
loops      -> повторяют task для нескольких значений
```

## 7. Handlers, become, roles, Vault

### Handler

**Handler** — специальный task, который запускается только после уведомления от другого task.

Уведомление выполняется через `notify`.

Типичный случай:

```text
конфиг изменился -> нужно перезапустить сервис
```

Пример:

```yaml
tasks:
  - name: Update nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: restart nginx

handlers:
  - name: restart nginx
    service:
      name: nginx
      state: restarted
```

Если конфиг не изменился и task вернул `ok`, handler обычно не запускается.

Если task вернул `changed`, handler получает уведомление.

### notify

**notify** — указание Ansible уведомить определённый handler после изменения состояния.

### become

**become** — механизм выполнения действий с повышенными привилегиями.

Часто используется вместе с `sudo`.

Пример:

```yaml
become: true
```

Нужен для действий, требующих административных прав:

- установка пакетов;
- изменение `/etc`;
- управление systemd;
- изменение системных пользователей.

### Role

**Role** — способ структурировать и переиспользовать Ansible-код как отдельный логический компонент.

Например роль `nginx` может содержать:

- tasks;
- handlers;
- templates;
- variables;
- files.

Roles помогают не превращать playbook в огромный монолитный YAML-файл.

### Vault

**Ansible Vault** — механизм шифрования секретных данных.

Через Vault можно защищать:

- пароли;
- токены;
- API keys;
- приватные variables.

Секреты можно хранить рядом с Ansible-кодом, но не в открытом виде.

Главная модель:

```text
handlers -> реакция на изменения
become   -> повышение привилегий
roles    -> организация и переиспользование кода
Vault    -> защита секретов
```

## 8. Место Ansible среди инфраструктурных инструментов

### Configuration management

**Configuration management** — управление состоянием уже существующих систем.

Это основная область применения Ansible.

Например:

- нужные пакеты установлены;
- нужные пользователи существуют;
- нужные конфиги присутствуют;
- сервисы находятся в нужном состоянии;
- права на файлы настроены.

### Provisioning

**Provisioning** — создание инфраструктурных ресурсов.

Например:

- virtual machine;
- cloud instance;
- network;
- load balancer.

Ansible умеет участвовать в provisioning, но это не его основная специализация.

### Orchestration

**Orchestration** — координация нескольких действий и систем в заданном порядке.

Например:

```text
создать сервер
-> настроить сервер
-> задеплоить приложение
-> перезапустить сервис
```

Ansible хорошо подходит для такой координации.

### Ansible и Terraform

**Terraform** в первую очередь используется для описания и создания инфраструктуры.

Пример желаемого состояния:

```text
VM должна существовать
network должна существовать
load balancer должен существовать
```

Ansible чаще используется для конфигурации уже существующих машин:

```text
nginx должен быть установлен
конфиг должен быть таким
service должен быть запущен
```

Типичная связка:

```text
Terraform создаёт инфраструктуру
-> Ansible настраивает её
```


## 9. Итоговая модель Ansible

```text
Inventory
  -> определяет hosts и groups

Playbook
  -> содержит plays

Play
  -> выбирает hosts
  -> содержит tasks

Task
  -> вызывает module

Module
  -> проверяет / изменяет состояние host

Variables / Facts
  -> дают данные

Templates
  -> создают конфиги из данных

Conditions / Loops
  -> управляют выполнением tasks

Handlers
  -> реагируют на изменения

Become
  -> даёт повышенные привилегии

Roles
  -> структурируют и переиспользуют Ansible-код

Vault
  -> защищает секреты
```

Главная идея Ansible:

```text
описать желаемое состояние
-> применить его к выбранным hosts
-> менять только то, что действительно требует изменения
```

## 10. Пример: установка PostgreSQL для test и production

Ниже — учебный пример для Debian/Ubuntu. Он устанавливает PostgreSQL на два сервера,
применяет к ним разные параметры и кладёт конфигурацию через роль `postgresql`.

Это не готовая production-схема высокой доступности: здесь один сервер для test и
один для production, без репликации, backup-процесса и failover. Эти части нужно
проектировать отдельно.

### Структура проекта

```text
ansible/
├── inventory/
│   ├── hosts.ini
│   └── group_vars/
│       ├── postgres_test.yml
│       └── postgres_prod.yml
├── playbooks/
│   └── postgresql.yml
└── roles/
    └── postgresql/
        ├── tasks/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        └── templates/
            ├── 20-ansible.conf.j2
            └── pg_hba.conf.j2
```

В этом разделе файлы показаны как образец, но создавать их в проекте не нужно.

### Inventory

`inventory/hosts.ini` описывает отдельные группы для окружений и общую группу
`postgres`.

```ini
[postgres_test]
pg-test-01 ansible_host=10.20.10.10

[postgres_prod]
pg-prod-01 ansible_host=10.30.10.10

[postgres:children]
postgres_test
postgres_prod
```

```text
postgres_test / postgres_prod -> задают environment-specific variables
postgres                      -> позволяет применить одну роль к обеим машинам
```

Не стоит помещать пароли в inventory. Для секретов, например пароля роли
приложения или ключа шифрования backup, следует использовать Ansible Vault.

### Переменные окружений

`inventory/group_vars/postgres_test.yml` содержит менее ресурсоёмкие значения для
тестовой базы и разрешает подключения только из подсети test-приложений:

```yaml
postgresql_version: 16
postgresql_listen_addresses: "10.20.10.10"
postgresql_port: 5432
postgresql_max_connections: 50
postgresql_shared_buffers: "512MB"
postgresql_allowed_cidrs:
  - "10.20.20.0/24"
```

`inventory/group_vars/postgres_prod.yml` использует production-адрес, больший
лимит и отдельную подсеть приложений:

```yaml
postgresql_version: 16
postgresql_listen_addresses: "10.30.10.10"
postgresql_port: 5432
postgresql_max_connections: 200
postgresql_shared_buffers: "4GB"
postgresql_allowed_cidrs:
  - "10.30.20.0/24"
```

Значения `max_connections` и `shared_buffers` — примеры, а не универсальная
настройка. Их выбирают по доступной памяти, характеру нагрузки, пулу соединений
и результатам мониторинга. Для production полезно использовать connection pooler,
например PgBouncer, вместо безусловного роста `max_connections`.

### Playbook

`playbooks/postgresql.yml` выбирает общую группу `postgres`, получает переменные
из её дочерней группы и применяет роль. `become: true` нужен, потому что роль
устанавливает пакет и меняет файлы в `/etc`.

```yaml
- name: Configure PostgreSQL servers
  hosts: postgres
  become: true
  gather_facts: true

  roles:
    - role: postgresql
```

Запуск с явным inventory:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/postgresql.yml
```

Для предварительной проверки удобно использовать:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/postgresql.yml --check --diff
```

`--check` не заменяет тестовый запуск: некоторые задачи и модули не могут
полностью предсказать результат без реального выполнения.

### Задачи роли

`roles/postgresql/tasks/main.yml` предполагает стандартную структуру пакета
PostgreSQL в Debian/Ubuntu: кластер называется `main`, а конфигурация лежит в
`/etc/postgresql/<version>/main`. Для RHEL-подобных систем пути, имя сервиса и
пакеты будут другими.

Чтобы версия `postgresql_version` не зависела от версии PostgreSQL в базовом
репозитории ОС, роль сначала подключает официальный PostgreSQL APT Repository
(PGDG). Он поддерживает несколько версий PostgreSQL одновременно, поэтому далее
устанавливается именно пакет `{% raw %}postgresql-{{ postgresql_version }}{% endraw %}`, а не
изменяемый meta-пакет `postgresql`.

```yaml
{% raw %}
- name: Install PGDG repository prerequisites
  ansible.builtin.apt:
    name:
      - ca-certificates
      - postgresql-common
      - python3-debian
    state: present
    update_cache: true

- name: Ensure PGDG key directory exists
  ansible.builtin.file:
    path: /usr/share/postgresql-common/pgdg
    state: directory
    owner: root
    group: root
    mode: "0755"

- name: Download PGDG signing key
  ansible.builtin.get_url:
    url: https://www.postgresql.org/media/keys/ACCC4CF8.asc
    dest: /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc
    owner: root
    group: root
    mode: "0644"

- name: Configure PostgreSQL APT repository
  ansible.builtin.deb822_repository:
    name: pgdg
    types: [deb]
    uris: https://apt.postgresql.org/pub/repos/apt
    suites: "{{ ansible_distribution_release }}-pgdg"
    components: [main]
    signed_by: /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc
    state: present

- name: Update APT cache after adding PGDG repository
  ansible.builtin.apt:
    update_cache: true

- name: Install PostgreSQL packages
  ansible.builtin.apt:
    name:
      - "postgresql-{{ postgresql_version }}"
      - "postgresql-contrib-{{ postgresql_version }}"
    state: present
{% endraw %}
```

`ansible_distribution_release` берётся из facts: например, на Ubuntu 24.04 это
`noble`, поэтому source получает suite `noble-pgdg`. `signed_by` привязывает
ключ только к репозиторию PGDG и не делает его доверенным для всех APT sources.
`deb822_repository` создаёт `/etc/apt/sources.list.d/pgdg.sources` в современном
формате Deb822.

Остальная часть `roles/postgresql/tasks/main.yml`:

```yaml
{% raw %}
- name: Ensure PostgreSQL configuration directory exists
  ansible.builtin.file:
    path: "/etc/postgresql/{{ postgresql_version }}/main/conf.d"
    state: directory
    owner: postgres
    group: postgres
    mode: "0750"

- name: Install Ansible-managed PostgreSQL settings
  ansible.builtin.template:
    src: 20-ansible.conf.j2
    dest: "/etc/postgresql/{{ postgresql_version }}/main/conf.d/20-ansible.conf"
    owner: postgres
    group: postgres
    mode: "0640"
  notify: Restart PostgreSQL

- name: Install client authentication rules
  ansible.builtin.template:
    src: pg_hba.conf.j2
    dest: "/etc/postgresql/{{ postgresql_version }}/main/pg_hba.conf"
    owner: postgres
    group: postgres
    mode: "0640"
  notify: Restart PostgreSQL

- name: Ensure PostgreSQL is enabled and running
  ansible.builtin.service:
    name: postgresql
    state: started
    enabled: true
{% endraw %}
```

Роль меняет только выделенный файл `conf.d/20-ansible.conf`, а не полностью
перезаписывает основной `postgresql.conf`, который поддерживает пакетный менеджер.
Это уменьшает риск случайно потерять дистрибутивные параметры. Перед применением
нужно убедиться, что основной конфиг содержит `include_dir = 'conf.d'` — это
типичная настройка Debian/Ubuntu.

### Шаблон параметров PostgreSQL

Содержимое `roles/postgresql/templates/20-ansible.conf.j2`:

```conf
{% raw %}
# Managed by Ansible. Do not edit manually.

# Слушаем конкретный приватный адрес, а не все интерфейсы ('*').
listen_addresses = '{{ postgresql_listen_addresses }}'
port = {{ postgresql_port }}

# Значения различаются между test и production через group_vars.
max_connections = {{ postgresql_max_connections }}
shared_buffers = '{{ postgresql_shared_buffers }}'

# Новые пароли сохраняются в современном SCRAM-формате.
password_encryption = 'scram-sha-256'

# Логи с датой, пользователем, базой и удалённым адресом удобнее для аудита.
logging_collector = on
log_line_prefix = '%m [%p] user=%u,db=%d,app=%a,client=%h '
log_timezone = 'UTC'
timezone = 'UTC'
{% endraw %}
```

`listen_addresses` ограничивает интерфейс, на котором PostgreSQL принимает
подключения. Это не замена firewall: доступ также должен быть ограничен security
groups или правилами firewall. TLS стоит включать отдельной задачей вместе с
доставкой и ротацией сертификатов; не следует включать его, не определив этот
процесс.

### Шаблон правил доступа

Содержимое `roles/postgresql/templates/pg_hba.conf.j2`:

```conf
{% raw %}
# Managed by Ansible. Do not edit manually.

# Локальные подключения ОС-пользователя postgres.
local   all             postgres                                peer

# Остальные локальные клиенты проходят аутентификацию по паролю.
local   all             all                                     scram-sha-256

# Разрешаем только подсети приложений конкретного environment.
{% for cidr in postgresql_allowed_cidrs %}
host    all             all             {{ cidr }}              scram-sha-256
{% endfor %}

# Другие подключения не разрешены: PostgreSQL использует первое совпавшее правило,
# а при отсутствии совпадения отклоняет соединение.
{% endraw %}
```

Правила `pg_hba.conf` обрабатываются сверху вниз, поэтому широкое разрешающее
правило в начале файла может случайно открыть базу. Не используйте `trust` для
сетевых подключений и не заменяйте список CIDR на `0.0.0.0/0` без явно
спроектированной защиты.

### Handler роли

`roles/postgresql/handlers/main.yml` перезапускает сервис только если один из
шаблонов изменился. Даже если оба task отправят `notify`, handler выполнится один
раз в конце play.

```yaml
- name: Restart PostgreSQL
  ansible.builtin.service:
    name: postgresql
    state: restarted
```

Итоговый поток выглядит так:

```text
inventory groups
-> group_vars выбирают test/prod параметры
-> playbook применяет роль postgresql к группе postgres
-> role устанавливает пакеты и рендерит templates
-> изменившиеся конфиги уведомляют handler
-> PostgreSQL перезапускается только при необходимости
```
