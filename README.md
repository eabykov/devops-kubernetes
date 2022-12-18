Оркестрация — это выполнение определенного рабочего процесса: сначала сделай A, затем B, затем C. Напротив, Kubernetes содержит набор независимых, компонуемых процессов управления, которые `непрерывно переводит текущее состояние к предполагаемому состоянию` и `неважно, как добраться от А до С`. Это делает систему более простой в использовании, более мощной, надежной, устойчивой и расширяемой

- при правильной настройке гарантирует отсутствие простоев
- упрощает мониторинг сервисов
- автоматическое распределение нагрузки на имеющихся ресурсах в зависимости от требований
- автоматизированные развертывание (деплой сервисов), откаты при ошибках
- самоконтроль, перезапустит упавшие контейнеры, проверит что контейнеры работают как "нужно" и многое другое

Kubernetes кластер - обьединение виртуальных (или нет) машины `worker node` на которых будут работать наши контенйнеры управляемых несколькими `master node`

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

</details>

### Структура объектов и пространства имен

<details>
  <summary>Kubernetes использует объекты в формате YAML для представления состояния кластера</summary>

> Почти в каждом объекте Kubernetes есть два вложенных поля-объекта, которые управляют конфигурацией объекта:
> - `spec` - требуемое состояние (описание характеристик, которые должны быть у объекта)
> - `status` - текущее состояние

https://kubernetes.io/ru/docs/concepts/overview/working-with-objects/kubernetes-objects/

</details>

<details>
  <summary>Namespace (пространство имен) - виртуальные кластера в одном физическом кластере Kubernetes</summary>

Нужны чтобы изолировать группы обьектов в одном кластере (Имена ресурсов должны быть уникальными в пределах одного и того же namespace) 

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <insert-namespace-name-here> # имя namespace
```

</details>

### Запуск приложений, заданий и управление ими

> - labels (метки, этикетки) которые используются для идентификации и выбора группы объектов. Рекомендуемые лейблы https://kubernetes.io/ru/docs/concepts/overview/working-with-objects/common-labels/
> - annotations (аннотации) похожи на лейблы, но не используются для идентификации и выбора объектов https://kubernetes.io/ru/docs/concepts/overview/working-with-objects/annotations/

`Pod` - один или нескольких контейнеров с общим хранилищем и сетевыми ресурсами, а также описывает спецификацию для запуска контейнеров
> Pod, обычно, не создаются напрямую и создаются с использованием других ресурсов котоыре описаны ниже

<details>
  <summary>Deployment - контролирует обновления ReplicaSet который является набором pod</summary>

Deployment создает `ReplicaSet`, который в свою очередь создает набор одинаковых pod и работает с ними, как с единой сущностью. Поддерживает нужное количество реплик, при необходимости создавая новые pod или убивая старые

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels: # используются для идентификации и выбора объектов
    app.kubernetes.io/name: nginx-deployment
    app.kubernetes.io/version: latest
    app.kubernetes.io/component: nginx-deployment
  annotations:
    imageregistry: "https://hub.docker.com/"
spec:
  replicas: 3 # можно удалить если используем HPA который сам будет следить за числом реплик (описание и пример ниже)
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx-deployment
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx-deployment
    spec:
      affinity:
        podAntiAffinity: # анти зависимость чтобы реплики pod разьехались по разным node
          preferredDuringSchedulingIgnoredDuringExecution: 
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app.kubernetes.io/name
                  operator: In
                  values:
                  - nginx-deployment # создавать на node где нет pod с лейблом app.kubernetes.io/name: nginx-deployment
              topologyKey: "topology.kubernetes.io/zone" # стараться размещать pod в разных зонах доступности
      terminationGracePeriodSeconds: 60 # после отправки приложению сигнала 'Заверши работу' даем ему 60 сек закончить свою работу и умереть, иначе убиваем
      containers:
      - name: nginx
        image: nginx:latest
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
  <summary>StatefulSet - контролирует обновления ReplicaSet который является набором pod</summary>

StatefulSet в отличии от Deployment не создает ReplicaSet, а сам контролирует, обновляет и создает pod

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  selector:
    matchLabels:
      app: postgress # должно совпадать с .spec.template.metadata.labels
  serviceName: "postgres"
  replicas: 3 # по умолчанию '1'
  minReadySeconds: 10 # по умолчанию '0'
  template:
    metadata:
      labels:
        app: postgres # должно совпадать с .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: postgres
        image: postgres:14.6-alpine
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
  <summary>DaemonSet - гарантирует, что все (или некоторые) noвуускают копию модуля</summary>

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # these tolerations are to have the daemonset runnable on control plane nodes
      # remove them if your control plane nodes should not run pods
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
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
  name: nginx-busybox-job
spec:
  template:
    spec:
      containers:
      - name: busybox
        image: busybox:1.35.0
        command: ["/bin/sleep", "10"] # спать 10 секунд, а потом завершить работу
      restartPolicy: Never # при ошибке не перезапускать
  backoffLimit: 4
```

https://kubernetes.io/docs/concepts/workloads/controllers/job/

</details>

<details>
  <summary>CronJob - задание по расписанию, то же что и Job только по расписанию</summary>

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ngixn-cronjob
spec:
  schedule: "* * * * *" # расписание, в данном случае каждую минуту https://crontab.guru/
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: busybox
            image: busybox:1.35.0
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
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 3
  maxReplicas: 4
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
  <summary>Service - для связи приложений внутри кластера, является DNS именем которым обьеденен набор pod (используя лейблы на pod)</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service # одно DNS имя которым обьеденен набор pod (используя лейблы на pod в selector ниже)
spec:
  selector:
    app.kubernetes.io/name: nginx-deployment # выбирает pod по лейблу
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
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

https://kubernetes.io/docs/concepts/services-networking/ingress/

</details>

### Диски, конфигурация и секреты

<details>
  <summary>PersistentVolume - запрос пользователя на храненилище данных (виртуальный диск для pod)</summary>

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pvc
spec:
  capacity:
    storage: 5Gi # обьем запрашиваемого диска
  accessModes:
    - ReadWriteOnce # режим доступа который разрешает нескольким pod получать доступ к pvc, когда pod запущены на одной node
  storageClassName: nginx-storageclass # имя обьекта 'storageClass' который хранит параметры подключения к системе хранения данных (дисковым массивам и тд)
```

https://kubernetes.io/docs/concepts/storage/persistent-volumes/

</details>

<details>
  <summary>ConfigMap - если нам нужно хранить какие-то данные в виде текстового файла</summary>

По возможности лучше настраивать приложение через переменные среды в `env` как в примере с Deployment выше (это просто удобнее), а ConfigMap использовать если нужно настроить что-то сложное или приложение умеет работать только с конфигфайлом

```yaml
apiVersion: v1
kind: ConfigMap # имя конфигмапа по котрому мы будем его добавлять в pod
metadata:
  name: nginx-configmap
data:
  # настройки вида ключ=знач; у каждого ключа есть свое значение
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"
  # запись в виде файла game.properties который можно будет использовать в контейнерах
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
```

https://kubernetes.io/docs/concepts/configuration/configmap/

</details>

<details>
  <summary>Secret - небольшой объем конфиденциальных данных, таких как пароль, токен или ключ</summary>

Нужен чтобы удобно хранить секретные данные внутри Kubenrtes, пароли, логины, номера счетов и тд.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-secret # имя секрета по котрому мы его будем подключать в наши pod
type: Opaque # тип: произвольные пользовательские данные (все типы https://kubernetes.io/docs/concepts/configuration/secret/#secret-types )
data:
  USER_NAME: aDm1n
  PASSWORD: myStr0ngPa5SworD
```

https://kubernetes.io/docs/concepts/configuration/secret/

</details>

### Полезные ссылки

1. [Установка Kubernetes локально](docs/LOCAL_INSTALL.md) на Windows с помощью WSL и Docker Desktop
2. От Docker контейнера до автоматического деплоя в Kubernetes: https://habr.com/ru/users/Anshelen/posts/
