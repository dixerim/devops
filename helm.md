# Helm

## 1. Helm

**Helm** — инструмент, который собирает Kubernetes manifests из шаблонов и values, а затем устанавливает, обновляет и удаляет этот набор resources через Kubernetes API.

Helm не является Control Plane component.

### Chart

**Chart** — директория или `.tgz`-package с описанием того, какие Kubernetes manifests нужно сгенерировать.

Типичная директория chart:

```text
my-chart/
├── Chart.yaml
├── values.yaml
└── templates/
```

Что внутри:

```text
Chart.yaml  → metadata chart: name, version, type, dependencies
values.yaml → default values для шаблонов
templates/  → шаблоны, из которых Helm рендерит Kubernetes manifests
```

Chart сам не является Kubernetes object.

### Templates

`templates/` содержат text templates с Go-template expressions:

```text
{% raw %}{{ ... }}{% endraw %}
```

До rendering это не обязательно valid Kubernetes manifest.

### values.yaml

`values.yaml` — значения по умолчанию, которые Helm подставляет в шаблоны chart.

Итоговые values получаются из `values.yaml` и переопределений, переданных пользователем, например через `-f` или `--set`.

### Release

**Release** — конкретная установка chart в Kubernetes: с именем, выбранными values, сгенерированными manifests и историей revisions.

Helm revision ≠ Deployment revision.

## 2. Helm commands

### install

```bash
helm install accounts ./chart
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
helm upgrade accounts ./chart
```

Рендерит новый desired set manifests и применяет изменения через API.

Kubernetes controllers выполняют actual workload rollouts.

### rollback

```bash
helm rollback accounts 2
```

Старая Release configuration применяется заново как новая текущая revision.

Это не возвращает физический cluster назад во времени.

### uninstall

```bash
helm uninstall accounts
```

Удаляет Helm-managed resources согласно lifecycle/policies; persistent/external resources могут сохраняться в зависимости от configuration.

## 3. Helm validation и dry run

### helm lint

```bash
helm lint ./my-chart
```

Проверяет Chart structure/templates/schema и ловит часть ошибок до изменения cluster.

### helm template

```bash
helm template accounts ./my-chart -f values.yaml
```

Локально рендерит окончательные Kubernetes manifests и печатает их, ничего не устанавливая.

Это лучший ответ на вопрос:

```text
"во что именно сейчас собираются мои templates?"
```

### dry-run

```bash
helm install accounts ./chart --dry-run --debug
helm upgrade accounts ./chart --dry-run
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

### --wait

```bash
helm install accounts ./chart --wait --timeout 5m
helm upgrade accounts ./chart --wait --timeout 5m
```

`--wait` заставляет Helm не завершать команду сразу после отправки manifests в Kubernetes API.

Helm ждёт, пока основные resources станут ready/successful:

```text
Deployment/StatefulSet/ReplicaSet → нужное число ready Pods
Pod → Ready
Service → создан
PVC → bound
Job → completed, если включён wait for jobs
```

Если readiness не наступила до `--timeout`, команда завершается ошибкой.

Важно: `--wait` сам по себе не откатывает изменения.

### --atomic

```bash
helm install accounts ./chart --atomic --timeout 5m
helm upgrade accounts ./chart --atomic --timeout 5m
```

`--atomic` включает `--wait` автоматически.

При `helm install --atomic`:

```text
Helm создаёт resources
↓
ждёт ready/success до timeout
↓
если install failed
↓
Helm удаляет resources этого release
```

При `helm upgrade --atomic`:

```text
Helm применяет новую revision
↓
ждёт ready/success до timeout
↓
если upgrade failed
↓
Helm откатывает release на предыдущую revision
```

`--atomic` не превращает operation в настоящую database transaction.

Что уже могло случиться до rollback/cleanup:

```text
Pod успел стартовать
Job/hook успел выполнить side effect
external controller успел создать cloud resource
application успела выполнить migration/request
PVC/PV/external data могли сохраниться по policy
```

Итог:

```text
--wait   → ждать готовности
--atomic → ждать готовности и при ошибке cleanup/rollback
```

## 4. Пример Chart для accounts

Задача:

```text
example.com/api/v1/accounts → accounts Service → accounts Pods
```

Один chart описывает один application service: `Deployment`, `Service` и, если нужно, `Ingress`.

Структура:

```text
accounts-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

### Chart.yaml

```yaml
apiVersion: v2
name: accounts
version: 0.1.0
type: application
```

### values.yaml

```yaml
replicaCount: 2

image:
  repository: registry.example.com/accounts-api
  tag: "1.0.0"
  pullPolicy: IfNotPresent

containerPort: 8080

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  host: example.com
  path: /api/v1/accounts

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

`values.yaml` содержит различающиеся параметры: image, replicas, ports, resources и ingress route.

### templates/_helpers.tpl

```text
{% raw %}
{{- define "accounts.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "accounts.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
{% endraw %}
```

`_helpers.tpl` не создаёт Kubernetes object. Это набор маленьких template-функций, которые потом вызываются из других templates.

### templates/deployment.yaml

```yaml
{% raw %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "accounts.fullname" . }}
  labels:
    {{- include "accounts.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        {{- include "accounts.labels" . | nindent 8 }}
    spec:
      containers:
        - name: accounts
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.containerPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
{% endraw %}
```

### templates/service.yaml

```yaml
{% raw %}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "accounts.fullname" . }}
  labels:
    {{- include "accounts.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/instance: {{ .Release.Name }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.containerPort }}
{% endraw %}
```

### templates/ingress.yaml

```yaml
{% raw %}
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "accounts.fullname" . }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: Prefix
            backend:
              service:
                name: {{ include "accounts.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
{% endraw %}
```

Ingress template использует `ingress.host`, `ingress.path` и `service.port` из `values.yaml`.

```text
example.com/api/v1/accounts → prod-accounts:80
```

### Проверка результата

```bash
helm template prod ./accounts-chart -f values.yaml
```

Эта команда покажет обычные Kubernetes manifests, которые Helm отправил бы в API:

```text
Deployment/prod-accounts
Service/prod-accounts
Ingress/prod-accounts
```

Главная идея:

```text
values.yaml
→ параметры конкретного environment/release

templates/
→ общая форма Kubernetes objects

_helpers.tpl
→ переиспользуемые имена/labels/маленькие template-функции
```

Если нужно задеплоить `billing`, обычно делают отдельный chart/release с похожей структурой или выносят общие helper templates в library chart.
