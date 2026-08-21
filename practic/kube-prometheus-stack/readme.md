## Шаг 1. Установка `kube-prometheus-stack`

`kube-prometheus-stack` — это Helm-чарт, который разворачивает сразу весь стек: Prometheus, Alertmanager, Grafana, Node Exporter и `kube-state-metrics`.

Создайте файл `values-prometheus.yaml`:

```
prometheus:
  prometheusSpec:
    podMonitorSelector: {}
    podMonitorNamespaceSelector: {}
    ruleSelector: {}
    ruleNamespaceSelector: {}
```

Что означают эти настройки:

* `podMonitorSelector: {}` — Prometheus подхватывает **все** PodMonitor в кластере (пустой селектор = без фильтра).
* `podMonitorNamespaceSelector: {}` — искать PodMonitor во ​**всех неймспейсах**​, а не только там, где стоит Prometheus. Это критично: Prometheus будет в неймспейсе `prometheus`, а ваши сервисы — в других.
* `ruleSelector: {}` и `ruleNamespaceSelector: {}` — то же самое для PrometheusRule.

Установите стек через Helm:

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create ns prometheus

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n prometheus \
  -f values-prometheus.yaml

kubectl -n prometheus get pods -w
```

**Ожидаемый результат:** все поды переходят в статус `Running`. Вы увидите поды примерно такого вида:

```
kube-prometheus-stack-grafana-xxx                Running
kube-prometheus-stack-kube-state-metrics-xxx     Running
kube-prometheus-stack-operator-xxx               Running
kube-prometheus-stack-prometheus-node-exporter   Running
prometheus-kube-prometheus-stack-prometheus-0    Running
alertmanager-kube-prometheus-stack-alertmanager  Running
```

Что установилось:

* **`prometheus-operator`** — контроллер, следящий за CRD (PodMonitor, PrometheusRule и др.);
* **`prometheus`** — сервер метрик;
* **`alertmanager`** — маршрутизация алертов;
* **`grafana`** — визуализация;
* **`node-exporter`** — метрики узлов Kubernetes (CPU, память, диск);
* **`kube-state-metrics`** — метрики состояния K8s-объектов (поды, деплойменты, ноды).

## Шаг 2. Проверка автоматического обнаружения целей

Пробросьте порт Prometheus на локальную машину:

```
kubectl -n prometheus port-forward svc/kube-prometheus-stack-prometheus 9090:9090 &
```

Откройте в браузере `http://localhost:9090` и перейдите в ​**Status → Target Health**​.

Вы увидите, что Prometheus уже мониторит компоненты кластера — без какой-либо ручной настройки. Среди целей будут:

* `serviceMonitor/.../node-exporter` — метрики операционной системы на каждом узле;
* `serviceMonitor/.../kube-state-metrics` — состояние Kubernetes-объектов;
* `serviceMonitor/.../apiserver` — метрики API-сервера Kubernetes;
* `serviceMonitor/.../prometheus` — метрики самого Prometheus.

Это и есть Prometheus Operator в действии: при установке `kube-prometheus-stack` Operator автоматически создал ServiceMonitor для каждого компонента стека. Никакого ручного `prometheus.yml`.

Сравнение с классическим подходом

Классический способ мониторинга подов через аннотации `prometheus.io/scrape: "true"` работает только в том случае, если в конфигурации Prometheus явно настроен `kubernetes_sd_configs`. В `kube-prometheus-stack` такой механизм по умолчанию не используется, поэтому в современных Kubernetes-кластерах обычно работают через PodMonitor и ServiceMonitor.

## Шаг 3. Запросы к метрикам инфраструктуры

В интерфейсе Prometheus откройте раздел **Query** (в верхнем меню) — внутри него находится вкладка ​**Graph**​. Введите запрос и нажмите ​**Execute**​.

Посмотрите на метрики узла:

```
node_memory_MemAvailable_bytes
```

Вы увидите несколько временных рядов — по одному на каждый узел кластера. Обратите внимание на лейблы: `instance`, `job` и другие. Именно они позволяют отличить метрики одного узла от другого.

Попробуйте отфильтровать данные конкретного узла:

```
node_memory_MemAvailable_bytes{instance="<имя-вашего-узла>"}
```

Имя узла можно найти в лейблах, которые Prometheus показывает рядом с каждым значением.

Другие полезные метрики для изучения:

```
node_cpu_seconds_total{mode="idle"}
```

```
kube_pod_status_phase
```

```
kube_deployment_spec_replicas
```

Исследуйте лейблы у каждой метрики. Это поможет понять, как писать запросы к метрикам ваших сервисов.

## Шаг 4. Деплой `podinfo` и проверка метрик

Теперь подключим к мониторингу реальное приложение. Мы используем **`podinfo`** — небольшой тестовый сервис с готовыми Prometheus-метриками.

```
kubectl create ns podinfo

helm repo add podinfo https://stefanprodan.github.io/podinfo
helm repo update

helm install podinfo podinfo/podinfo -n podinfo

kubectl -n podinfo get pods
```

**Ожидаемый результат:** под `podinfo-xxx` в статусе `Running`.

Прежде чем настраивать PodMonitor, убедитесь, что приложение действительно отдаёт метрики:

```
kubectl -n podinfo port-forward svc/podinfo 9898:9898 &

curl localhost:9898/metrics | head -30
```

Вы увидите метрики в формате Prometheus: `http_requests_total`, `process_cpu_seconds_total` и другие. Запомните их имена — они понадобятся на следующем шаге.

## Шаг 5. Создать PodMonitor для `podinfo`

Сейчас Prometheus не знает о `podinfo` — он не появится в Targets. Создайте файл `podinfo-podmonitor.yaml`:

```
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: podinfo
  namespace: podinfo
  labels:
    release: kube-prometheus-stack  # Этот лейбл должен совпадать с podMonitorSelector
spec:
  namespaceSelector:
    matchNames:
      - podinfo                     # Неймспейс, в котором ищем поды
  selector:
    matchLabels:
      app.kubernetes.io/name: podinfo  # Лейблы подов, которые нужно мониторить
  podMetricsEndpoints:
    - port: http                    # Имя порта (из spec.containers[].ports в деплойменте)
      path: /metrics
```

Примените манифест:

```
kubectl apply -f podinfo-podmonitor.yaml
```

Перейдите в Prometheus → **Status → Targets** и найдите новую цель:

```
podMonitor/podinfo/podinfo/0   UP
```

**Ожидаемый результат:** цель появилась со статусом `UP` — то есть Prometheus Operator увидел PodMonitor, добавил под в конфигурацию, и Prometheus успешно получает метрики.

Что делать, если цель не появилась?

* Убедитесь, что лейбл `release: kube-prometheus-stack` на PodMonitor совпадает с `podMonitorSelector` вашего Prometheus. Проверить можно командой: `kubectl -n prometheus get prometheus -o jsonpath='{.items[0].spec.podMonitorSelector}'`.
* Убедитесь, что лейблы в `selector.matchLabels` совпадают с лейблами пода: `kubectl -n podinfo get pods --show-labels`.
* Убедитесь, что имя порта в `podMetricsEndpoints.port` совпадает с именем порта в деплойменте: `kubectl -n podinfo get pod <имя-пода> -o jsonpath='{.spec.containers[0].ports}'`.

## Шаг 6. Запросы к метрикам `podinfo`

Prometheus теперь собирает метрики `podinfo`. Перейдите в **Query → Graph** и попробуйте несколько запросов.

Посмотрите на все HTTP-запросы к `podinfo`:

```
http_requests_total{namespace="podinfo"}
```

Вы увидите временные ряды с лейблами `pod`, `method`, `status_code` и другими. Обратите внимание: фильтр `namespace="podinfo"` важен, чтобы отделить метрики `podinfo` от метрик других компонентов кластера, которые могут отдавать метрику с таким же именем.

Использование памяти процессом:

```
process_resident_memory_bytes{namespace="podinfo"}
```

Потребление CPU:

```
process_cpu_seconds_total{namespace="podinfo"}
```

На этом этапе вы видите сырые метрики. В уроке по PromQL вы научитесь трансформировать их в полезные показатели (скорость запросов, перцентили задержки и т. д.).

## Шаг 7. Создать PrometheusRule для `podinfo`

Теперь создадим правило алертинга. Оно будет срабатывать, если Prometheus перестанет получать метрики с пода (то есть если под недоступен).

Создайте файл `podinfo-rule.yaml`:

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: podinfo-alerts
  namespace: podinfo
  labels:
    release: kube-prometheus-stack   # Лейбл для ruleSelector
spec:
  groups:
    - name: podinfo.rules
      rules:
        - alert: PodInfoDown
          expr: absent(up{namespace="podinfo"})
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "podinfo недоступен в namespace podinfo"
            description: "Prometheus не получает метрики с подов podinfo — поды отсутствуют или недоступны"
```

Разберём ключевые поля:

* **`expr`** — PromQL-запрос. Метрика `up` — специальная: Prometheus выставляет `1`, если последний scrape успешен, `0` — если нет. `absent()` возвращает 1, если метрика с такими лейблами полностью отсутствует — то есть нет ни одного пода, с которого Prometheus мог бы получить `up`. В реальных алертах `expr` бывает сложнее — с `rate()`, `histogram_quantile()` и другими функциями. Синтаксис ничем не отличается: всё, что работает в Prometheus → Graph, работает и здесь.
* **`for: 1m`** — условие должно выполняться непрерывно одну минуту перед переходом в `firing`.
* **`labels.severity`** — кастомный лейбл для маршрутизации в Alertmanager.

PrometheusRule подхватывается через `ruleSelector: {}`. Эту настройку мы задали ещё в Шаге 1 — по той же логике, по которой `podMonitorSelector: {}` позволяет Prometheus видеть PodMonitor.

Примените:

```
kubectl apply -f podinfo-rule.yaml
```

Перейдите в Prometheus → ​**Alerts**​. Правило `PodInfoDown` должно появиться в статусе `inactive` (условие не выполняется, под работает).

**Ожидаемый результат:** алерт виден в списке, статус `inactive`.

## Самостоятельное задание 1

Найдите в Prometheus метрику `http_requests_total` для неймспейса `podinfo`. Изучите все лейблы, которые есть у этой метрики.

**Задача:** напишите запрос, который показывает метрику `http_requests_total` только для конкретного пода `podinfo` (не для всех подов).

Имя пода узнайте командой:

```
kubectl -n podinfo get pods
```

Когда будете готовы, сравните своё решение с авторским

## Самостоятельное задание 2

**Задача:** создайте PrometheusRule, который срабатывает, если под `podinfo` не отвечает дольше двух минут.

Требования:

* метрика `up` с фильтром по namespace `podinfo`,
* `for: 2m`,
* лейбл `severity: critical`

Проверьте: правило появилось в Prometheus → Alerts.

Когда будете готовы, сравните своё решение с авторским

## Итоги

Вы прошли полный цикл настройки мониторинга в Kubernetes с помощью Prometheus Operator:

* Установили `kube-prometheus-stack` с правильными настройками для PodMonitor и PrometheusRule.
* Убедились, что Prometheus автоматически обнаруживает компоненты кластера через ServiceMonitor.
* Задеплоили приложение, проверили его метрики и подключили к мониторингу через PodMonitor.
* Написали запросы к метрикам и разобрались со структурой лейблов.
* Создали PrometheusRule и увидели алерт в Prometheus → Alerts.

В этом уроке вы на практике научились доставать сырые значения из приложения и увидели их в интерфейсе Prometheus. В следующем уроке мы разберём типы метрик Prometheus и научимся выбирать подходящий тип метрики под конкретную задачу мониторинга.

