### Базовые сведения о Kubernetes
Kubernetes содержит набор независимых, объединённых вместе процессов управления, которые `постоянно стараются привести текущее состояние контейнеризированного приложения к заданному состоянию` основываясь на декларативном подходе: когда мы описываем требуемое состояние (декларируем), а не действия, которые надо предпринять, и `неважно, как добраться от А до С, K8s сделает это за вас`

Целевая область деятельности Kubernetes - это автоматизации развёртывания контейнезированных приложений, их масштабирования и координации в условиях кластера (группа компьютеров, объединённых высокоскоростными каналами связи, представляющая с точки зрения пользователя единый аппаратный ресурс)

Это делает выполнение приложений более простым в использовании, более мощным, надежным, устойчивым и расширяемым:

- при правильной настройке гарантирует отсутствие простоев
- упрощает наблюдаемость (мониторинг, логирование и тд)
- позволяет обеспечить автоматическое распределение нагрузки на имеющихся ресурсах в зависимости от требований
- автоматизированные развертывание (деплой сервисов), откаты при ошибках
- самоконтроль, перезапустит упавшие контейнеры, проверит что контейнеры работают как "нужно" и многое другое

Kubernetes кластер - объединение виртуальных (или нет) машин `worker node` (на которых будут работать наши контейнеры), управляемых несколькими `master node`

<details>
  <summary>Составляющие master node (управляющий уровень)</summary>

https://kubernetes.io/ru/docs/concepts/overview/components/#плоскость-управления-компонентами

- `kube-apiserver` - клиентская часть панели управления Kubernetes
- `etcd` - высоконадёжное хранилище данных в формате "ключ-значение", основное хранилище всех данных кластера в Kubernetes
- `kube-scheduler` - отслеживает созданные поды без привязанного узла и выбирает узел, на котором они должны работать
- `kube-controller-manager` - уведомляет и реагирует на сбои node, поддерживает правильное количество pod для каждого объекта контроллера репликации в системе,  связывает Services и Pods, создает стандартные учетные записи и токены доступа API для новых namespace

</details>

<details>
  <summary>Составляющие worker node (исполнительный уровень)</summary>

https://kubernetes.io/ru/docs/concepts/overview/components/#компоненты-узла

- `kubelet` - агент который следит за тем, чтобы контейнеры были запущены в поде
- `kube-proxy` - конфигурирует правила сети на узлах (service), при помощи которых разрешаются сетевые подключения к вашими подам изнутри и снаружи кластера
- `Среда выполнения контейнера` - это программа, предназначенная для выполнения контейнеров (Docker, containerd, CRI-O)
  CRI-O - Это альтернатива containerd, которая также позволяет загружать образы контейнеров из репозиториев, управлять ими и запускать Container Runtime нижнего уровня для запуска процессов контейнера
  > Docker — это лишь часть всей экосистемы контейнеров. Существуют открытые стандарты: CRI и OCI, и несколько Container Runtime с поддержкой CRI: containerd, runc, CRI-O и, конечно, сам Docker

</details>

### Структура объектов и пространства имен

<details>
  <summary>Kubernetes использует объекты в формате YAML для представления состояния кластера. Описание требуемого состояния объектов называется манифестом</summary>

> Почти в каждом объекте Kubernetes есть два вложенных поля-объекта, которые управляют конфигурацией объекта:
> - `spec` - требуемое состояние (описание характеристик, которые должны быть у объекта)
> - `status` - текущее состояние

https://kubernetes.io/ru/docs/concepts/overview/working-with-objects/kubernetes-objects/

</details>

<details>
  <summary>Namespace (пространство имен) - виртуальные кластера в одном физическом кластере Kubernetes</summary>

Нужны чтобы отделять группы обьектов (контейнеров и их сетевые или любые другие настройки) в одном кластере

> Имена ресурсов должны быть уникальными в пределах одного и того же namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <insert-namespace-name-here> # имя namespace
```

</details>

### Запуск приложений, заданий и управление ими

> - labels (лейблы, метки, этикетки) - используются для идентификации и выбора группы объектов. Рекомендуемые лейблы: https://kubernetes.io/ru/docs/concepts/overview/working-with-objects/common-labels/
> - annotations (аннотации) похожи на лейблы, но не используются для идентификации и выбора объектов https://kubernetes.io/ru/docs/concepts/overview/working-with-objects/annotations/

`Pod` - один или несколько контейнеров с общим хранилищем и сетевыми ресурсами, а также описывает спецификацию для запуска контейнеров
> Pod, обычно, создаются с использованием других ресурсов которые описаны ниже

<details>
  <summary>Deployment - контролирует обновления ReplicaSet который является набором pod</summary>

Deployment создает `ReplicaSet`, который в свою очередь создает набор одинаковых pod и работает с ними, как с единой сущностью. Поддерживает нужное количество реплик, при необходимости создавая новые pod или убивая старые

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels: # используются для идентификации и выбора объектов
    app.kubernetes.io/name: nginx
    app.kubernetes.io/version: latest
    app.kubernetes.io/component: nginx
  annotations:
    imageregistry: "https://hub.docker.com/"
spec:
  replicas: 3 # можно удалить если используем HPA который сам будет следить за числом реплик (описание и пример ниже)
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx
    spec:
      affinity:
        podAntiAffinity: # анти зависимость чтобы реплики pod разъехались по разным node
          preferredDuringSchedulingIgnoredDuringExecution: 
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app.kubernetes.io/name
                  operator: In
                  values:
                  - nginx # создавать на node где нет pod с лейблом app.kubernetes.io/name: nginx-deployment
              topologyKey: "topology.kubernetes.io/zone" # стараться размещать pod в разных зонах доступности
      terminationGracePeriodSeconds: 30 # после отправки приложению сигнала 'Заверши работу' даем ему 30 сек закончить свою работу и умереть, иначе убиваем
      containers:
      - name: nginx
        image: nginx:1.24-alpine-slim
        ports:
        - name: http
          containerPort: 8080
        resources:
          requests: # запросы ресурсов по которым kube-scheduler ищет на какую node разместить pod
            memory: "150Mi"
            cpu: "250m"
          limits:
            memory: "150Mi" # если приложение попытается использовать больше памяти, Kubernetes убьет его
            cpu: "500m" # если приложение попытается использовать больше ресурсов CPU поставит его в очередь xD
        livenessProbe: # проверяет 'живо ли приложение' и если нет, перезапускает его
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe: # проверяет 'может ли приложение принимать запросы'
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
```

https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

</details>

<details>
  <summary>StatefulSet - в отличии от Deployment не создает ReplicaSet, а сам контролирует, обновляет и создает набор pod</summary>

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  labels: # используются для идентификации и выбора объектов
    app.kubernetes.io/name: postgres
    app.kubernetes.io/version: 14.7-alpine
    app.kubernetes.io/component: postgres
spec:
  selector:
    matchLabels:
      app: postgress # должно совпадать с .spec.template.metadata.labels
  serviceName: "postgres"
  replicas: 3
  template:
    metadata:
      labels:
        app: postgres # должно совпадать с .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: postgres
        image: postgres:14.7-alpine
        ports:
        - containerPort: 5432
          name: dbport
        volumeMounts:
        - name: default-database
          mountPath: /usr/share/postgres/database
  volumeClaimTemplates:
  - metadata:
      name: default-database
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class-name"
      resources:
        requests:
          storage: 5Gi
```

https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

</details>

<details>
  <summary>DaemonSet - гарантирует, что все (или некоторые) node имеют копию pod (используют обычно для мониторинга или сбора логов)</summary>

Ниже пример запуска `Fluentd` который будет собирать логи контейнеров (и не только) и отправлять их в централизованный `Elasticsearch` для дальнейшего хранения, обработки и просмотра

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels: # используются для идентификации и выбора объектов
    app.kubernetes.io/name: fluentd
    app.kubernetes.io/version: v1-debian-elasticsearch
    app.kubernetes.io/component: fluentd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: fluentd # должно совпадать с .spec.template.metadata.labels (ниже)
  template:
    metadata:
      labels:
        app.kubernetes.io/name: fluentd # должно совпадать с .spec.selector.matchLabels (выше)
    spec:
      tolerations:
      # Эти tolerations (допуски) предназначены для того, чтобы набор демонов (pod) мог выполняться на master node
      # Если мы не хотим запускать демонов (pod) на master node, то нужно удалить tolerations
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
        env:
          - name: FLUENT_ELASTICSEARCH_HOST
            value: "elasticsearch.monitoring" # имя service и namespace в которых установлен Elasticsearch
          - name: FLUENT_ELASTICSEARCH_PORT
            value: "9200" # порт Elasticsearch service на который отправлять логи
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "http" # по какому протоколу отправлять логи в Elasticsearch
        resources:
          limits:
            cpu: 250m
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/

</details>

<details>
  <summary>Job - задание, запуск pod для выполнения единоразовой задачи, например резервное сохранение данных перед обновлением</summary>

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: busybox-sleep
  labels: # используются для идентификации и выбора объектов
    app.kubernetes.io/name: busybox
    app.kubernetes.io/version: 1.36.0
    app.kubernetes.io/component: busybox
spec:
  ttlSecondsAfterFinished: 300 # автоматически удалить Job после ее завершения через 300 сек 
  template:
    spec:
      containers:
      - name: busybox
        image: busybox:1.36.0
        command: ["/bin/sleep", "10"] # спать 10 секунд, а потом завершить работу
      restartPolicy: Never # при ошибке не перезапускать
  backoffLimit: 4 # количество повторных попыток прежде чем job упадет
```

https://kubernetes.io/docs/concepts/workloads/controllers/job/

</details>

<details>
  <summary>CronJob - задание по расписанию, то же что и Job только по расписанию</summary>

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: busybox-date
  labels: # используются для идентификации и выбора объектов
    app.kubernetes.io/name: busybox
    app.kubernetes.io/version: 1.36.0
    app.kubernetes.io/component: busybox
spec:
  schedule: "* * * * *" # расписание, в данном случае каждую минуту https://crontab.guru/
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: busybox
            image: busybox:1.36.0
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure # попробовать еще раз если Job упадет
```

https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/

</details>

<details>
  <summary>HorizontalPodAutoscaler - автоматическое увеличение или уменьшение количества реплик приложения</summary>

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler # автоматическое увеличение или уменьшение количества реплик приложения
metadata:
  name: nginx
  labels: # используются для идентификации и выбора объектов
    app.kubernetes.io/name: nginx
    app.kubernetes.io/version: 1.24-alpine-slim
    app.kubernetes.io/component: nginx
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 3
  maxReplicas: 12
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

</details>

### Сетевые функции

<details>
  <summary>Service - для связи приложений внутри кластера, является DNS именем которым объединён набор pod (используя лейблы на pod)</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx # одно DNS имя которым объединён набор pod (используя лейблы на pod в selector ниже)
  labels: # используются для идентификации и выбора объектов
    app.kubernetes.io/name: nginx
    app.kubernetes.io/version: 1.24-alpine-slim
    app.kubernetes.io/component: nginx
spec:
  selector:
    app.kubernetes.io/name: nginx # выбирает pod по лейблу
  ports:
    - protocol: TCP
      port: 80
      targetPort: http
```

https://kubernetes.io/docs/concepts/services-networking/service/

</details>

<details>
  <summary>Ingress - для внешнего доступа (из интернета, сети офиса и тд) к приложениям в кластере</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  labels: # используются для идентификации и выбора объектов
    app.kubernetes.io/name: nginx
    app.kubernetes.io/version: 1.24-alpine-slim
    app.kubernetes.io/component: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-external
  rules:
  - http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```

https://kubernetes.io/docs/concepts/services-networking/ingress/

</details>

### Диски, конфигурация и секреты

<details>
  <summary>PersistentVolume - запрос пользователя на хранилище данных (виртуальный диск для pod)</summary>

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres
  labels: # используются для идентификации и выбора объектов
    app.kubernetes.io/name: postgres
    app.kubernetes.io/version: 14.7-alpine
    app.kubernetes.io/component: postgres
spec:
  capacity:
    storage: 5Gi # объем запрашиваемого диска
  accessModes:
    - ReadWriteOnce # режим доступа который разрешает нескольким pod получать доступ к pvc, когда pod запущены на одной node
  storageClassName: postgres-ssd # имя объекта 'storageClass' который хранит параметры подключения к системе хранения данных (дисковым массивам и тд)
```

https://kubernetes.io/docs/concepts/storage/persistent-volumes/

</details>

<details>
  <summary>ConfigMap - если нам нужно хранить какие-то данные в виде текстового файла</summary>

По возможности лучше настраивать приложение через переменные среды в `env` как в примере с Deployment выше (это просто удобнее), а ConfigMap использовать если нужно настроить что-то сложное или приложение умеет работать только с конфигурационным файлом

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configmap # имя конфигмапа по которому мы будем его добавлять в pod
  labels: # используются для идентификации и выбора объектов
    app.kubernetes.io/name: nginx
    app.kubernetes.io/version: 1.24-alpine-slim
    app.kubernetes.io/component: nginx
data:
  nginx.conf: |
    server {
      listen       80;
      server_name  localhost;
      location / {
          root   /usr/share/nginx/html;
          index  index.html index.htm;
      }
    }
```

https://kubernetes.io/docs/concepts/configuration/configmap/

</details>

<details>
  <summary>Secret - небольшой объем конфиденциальных данных, таких как пароль, токен или ключ</summary>

Нужен чтобы удобно хранить секретные данные внутри Kubernetes, пароли, логины, номера счетов и тд.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx # имя секрета по которому мы его будем подключать в наши pod
  labels: # используются для идентификации и выбора объектов
    app.kubernetes.io/name: nginx
    app.kubernetes.io/version: 1.24-alpine-slim
    app.kubernetes.io/component: nginx
type: Opaque # тип: произвольные пользовательские данные (все типы https://kubernetes.io/docs/concepts/configuration/secret/#secret-types )
data:
  USER_NAME: aDm1n
  PASSWORD: myStr0ngPa5SworD
```

https://kubernetes.io/docs/concepts/configuration/secret/

</details>

### Helm (штурвал) - управляет установкой, обновлением и настройкой приложений с помощью charts в Kubernetes

Helm Charts (схемы, чарты) - помогут вам настроить, установить и обновить приложение Kubernetes, -  это пакеты Helm, состоящие из файлов и шаблонов YAML, которые при развертывании (deployment) преобразуются в файлы манифестов Kubernetes

При развертывании приложения с помощью Helm Chart вы должны только передать значения для переменных, определенных в шаблоне

Чарты нужны чтобы было легко обьединить группу всех ресурсов, таких как Deployment, Service, Configmap и прочие, необходимую для работы приложения, версионировать (обновлять), публиковать (как один файл) и делиться ими
> В пункте 2.2 ниже вы можете посмотреть пример чарта с помощью которого можно установить Jira в k8s кластере одной командой и также потом обновлять

1. Пройти задание https://helm.sh/ru/docs/intro/quickstart/
2. Установить Jira через helm в свой Kubernetes https://artifacthub.io/packages/helm/atlassian-data-center/jira
   1. Найти как из service или из pod пробросить порт и открыть Jira в браузере
   2. Посмотреть из каких файлов состоит чарт: https://github.com/atlassian/data-center-helm-charts/tree/main/src/main/charts/jira
3. Встроенные переменные в helm: https://github.com/helm/helm-www/blob/main/content/en/docs/chart_template_guide/builtin_objects.md
   > Например проверить есть ли api `apps/v1` для ресурса `Deployment`: `{{- if .Capabilities.APIVersions.Has "apps/v1/Deployment" }}`

### Смена пользователя с root на appuser

1. Изменения в Dockerfile:
```Dockerfile
FROM python:3.11-slim-buster

WORKDIR /opt/apple/myapp

### Создаем пользователя, должно быть ниже чем WORKDIR ###
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "$(pwd)" \
    --no-create-home \
    --uid 1995 \
    appuser

  ### В инструкции COPY добавить --chown=1995:1995 ###
COPY --chown=1995:1995 requirements.txt .
RUN pip install --no-cache-dir --requirement requirements.txt \
    && rm -rfv requirements.txt
COPY --chown=1995:1995 app.py entrypoint.sh ./
COPY --chown=1995:1995 static ./

### Изменяем пользователя с root на appuser ###
RUN chown 1995:1995 -R /opt
USER appuser

ENTRYPOINT ["/bin/sh", "-c", "/entrypoint.sh"]
CMD ["-s", "all"]
```  
2. Для pod в Kubernetes задаем `securityContext`:
```yaml
securityContext:
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  runAsUser: 1995
  runAsGroup: 1995
  fsGroup: 1995
```
3. Протестированно в образах:
   - alpine:3.18
   - node:16.20.0-alpine
   - golang:alpine3.18
   - python:3.11
   - python:3.11-slim-buster
   - nginx:1.25
   - golang:1.20.5

### Нулевое время простоя при обновлении Deployment (zero downtime Deployment)

1. Чтобы при обновлении всегда было минимальное количество pod такое же как в `spec.replicas` нужно выставить: `spec.strategy.rollingUpdate.maxUnavailable: 0`
2. Настройте Readiness и Startup пробы чтобы избежать ошибок таких как отправка трафика на pod приложение в котором еще не запущенно (Liveness нужно ставить только в том случае если лучше что можно сделать это убить контейнер)
3. Для того чтобы k8s успел убрать pod и endpoint для service перед отправкой SIGTERM сигнала необходимо добавить sleep перед остановкой (который значительно меньше `terminationGracePeriodSeconds`):
    ```yaml
    spec:
      template:
        spec:
          terminationGracePeriodSeconds: 30
          containers:
            lifecycle:
              preStop:
                exec:
                  command:
                    - /bin/sh
                    - '-c'
                    - sleep 15
    ```
4. Не забывайте выставлять limits и requests для ваших контейнеров в pod чтобы k8s использовал requests для планирования, а limits для ограничения конкуренции за ресурсов на node (простой способ как подобрать написано выше в Deployment) 
5. Настройте HPA для вашего Deployment с запасом 2х от текущей нагрузки (по результатам нагрузочного тестирования)
   > С пиковой дневной нагрузкой за 7 дней справляются 6 pod, выставляем `maxReplicas: 12` (смотрим чтобы не было блокеров, например чтобы не закончились подключения к БД и тд)
6. Настройте в вашем service mesh политику повторных запросов при ошибке от нижестоящего сервиса чтобы избежать ошибки в случаях:
   - когда HPA уменьшает количество pod 
   - удалении старых pod при обновлении
   > Как настроить в istio https://istio.io/latest/docs/concepts/traffic-management/#retries или в linkerd https://linkerd.io/2.13/tasks/configuring-retries/
