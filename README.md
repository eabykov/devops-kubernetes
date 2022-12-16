<details>
  <summary>Основные цели использования Kubernetes и какие проблемы он решает</summary>

Kubernetes нужен для управления контейнеризованными рабочими нагрузками и сервисами
- при правильной настройке гарантирует отсутствие простоев
- упрощает мониторинг сервисов
- автоматическое распределение нагрузки на имеющихся ресурсах в зависимости от требований
- автоматизированные развертывание (деплой сервисов), откаты при ошибках
- самоконтроль, перезапустит упавшие контейнеры, проверит что контейнеры работают как "нужно" и многое другое

>  Оркестрация — это выполнение определенного рабочего процесса: сначала сделай A, затем B, затем C. Напротив, Kubernetes содержит набор независимых, компонуемых процессов управления, которые непрерывно переводит текущее состояние к предполагаемому состоянию. Неважно, как добраться от А до С. Это делает систему более простой в использовании, более мощной, надежной, устойчивой и расширяемой

</details>

### Составляющие Kubernetes кластера

<details>
  <summary>master node (управляющий уровень)</summary>

https://kubernetes.io/ru/docs/concepts/overview/components/#плоскость-управления-компонентами

- `kube-apiserver` - клиентская часть панели управления Kubernetes
- `etcd` - высоконадёжное хранилище данных в формате "ключ-значение", основное хранилище всех данных кластера в Kubernetes
- `kube-scheduler` - отслеживает созданные поды без привязанного узла и выбирает узел, на котором они должны работать
- `kube-controller-manager` - уведомляет и реагирует на сбои node, поддерживает правильное количество pod для каждого объекта контроллера репликации в системе,  связывает Services и Pods, создает стандартные учетные записи и токены доступа API для новых namespace

</details>

<details>
  <summary>worker node (исполнительный уровень)</summary>

https://kubernetes.io/ru/docs/concepts/overview/components/#компоненты-узла

- `kubelet` - агент который следит за тем, чтобы контейнеры были запущены в поде
- `kube-proxy` - конфигурирует правила сети на узлах (service), при помощи которых разрешаются сетевые подключения к вашими подам изнутри и снаружи кластера
- `Среда выполнения контейнера` - это программа, предназначенная для выполнения контейнеров (Docker, containerd, CRI-O)

</details>

### Структура объектов и пространства имен

<details>
  <summary>Kubernetes использует объекты для представления состояния кластера</summary>

https://kubernetes.io/ru/docs/concepts/overview/working-with-objects/kubernetes-objects/

> Почти в каждом объекте Kubernetes есть два вложенных поля-объекта, которые управляют конфигурацией объекта:
> - `spec` - требуемое состояние (описание характеристик, которые должны быть у объекта)
> - `status` - текущее состояние

</details>

- `namespace` (пространство имен) - виртуальные кластера в одном физическом кластере Kubernetes, нужны чтобы изолировать группы обьектов в одном кластере (Имена ресурсов должны быть уникальными в пределах одного и того же namespace) 

<details>
  <summary>Пример обьекта Namespace</summary>

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <insert-namespace-name-here> # имя namespace
```

</details>

### Запуск приложений, заданий и управление ими

> - labels (метки, этикетки) которые используются для идентификации и выбора объектов https://kubernetes.io/ru/docs/concepts/overview/working-with-objects/labels/
> - annotations (аннотации) похожи на лейблы, но не используются для идентификации и выбора объектов https://kubernetes.io/ru/docs/concepts/overview/working-with-objects/annotations/

- `ReplicaSet` - создает набор одинаковых pod и работает с ними, как с единой сущностью. Поддерживает нужное количество реплик, при необходимости создавая новые pod или убивая старые

<details>
  <summary>Deployment - контролирует обновления ReplicaSet который является набором pod</summary>

https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

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

</details>

- statefulset https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- daemonset https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/
- `job` (задания) https://kubernetes.io/docs/concepts/workloads/controllers/job/
- `cronjob` (задание по расписанию) https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/

<details>
  <summary>HorizontalPodAutoscaler автоматическое увеличение или уменьшение количества реплик приложения</summary>

https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

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

</details>

### Сетевые функции

<details>
  <summary>Service - для связи приложений внутри кластера, является DNS именем которым обьеденен набор pod (используя лейблы на pod)</summary>

https://kubernetes.io/docs/concepts/services-networking/service/

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

</details>

<details>
  <summary>Ingress - для внешнего доступа (из интернета, сети офиса и тд) к приложениям в кластере</summary>

https://kubernetes.io/docs/concepts/services-networking/ingress/

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

</details>

### Диски, конфигурация и секреты

<details>
  <summary>PersistentVolume - запрос пользователя на храненилище данных (виртуальный диск для pod)</summary>

https://kubernetes.io/docs/concepts/storage/persistent-volumes/

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

</details>

<details>
  <summary>ConfigMap - для хранения неконфиденциальных данных в парах ключ-значение</summary>

https://kubernetes.io/docs/concepts/configuration/configmap/

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

</details>

<details>
  <summary>Secret - небольшой объем конфиденциальных данных, таких как пароль, токен или ключ</summary>

https://kubernetes.io/docs/concepts/configuration/secret/

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

</details>

### Полезные ссылки

1. [Установка Kubernetes локально](docs/LOCAL_INSTALL.md) на Windows с помощью WSL и Docker Desktop
2. Хорошее начальное руководство по Kubernetes: https://habr.com/ru/users/Anshelen/posts/
