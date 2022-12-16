- [Установка Kubernetes локально](docs/LOCAL_INSTALL.md)

### Kubernetes нужен для управления контейнеризованными рабочими нагрузками и сервисами

>  Оркестрация — это выполнение определенного рабочего процесса: сначала сделай A, затем B, затем C. Напротив, Kubernetes содержит набор независимых, компонуемых процессов управления, которые непрерывно переводит текущее состояние к предполагаемому состоянию. Неважно, как добраться от А до С. Это делает систему более простой в использовании, более мощной, надежной, устойчивой и расширяемой

- при правильной настройке гарантирует отсутствие простоев
- упрощает мониторинг сервисов
- автоматическое распределение нагрузки на имеющихся ресурсах в зависимости от требований
- автоматизированные развертывание (деплой сервисов), откаты при ошибках
- самоконтроль, перезапустит упавшие контейнеры, проверит что контейнеры работают как "нужно" и многое другое

### Составляющие Kubernetes кластера

##### Из чего состоит (компоненты, составляющие) Kubernetes

master node (управляющий уровень) https://kubernetes.io/ru/docs/concepts/overview/components/#плоскость-управления-компонентами
- `kube-apiserver` - клиентская часть панели управления Kubernetes
- `etcd` - высоконадёжное хранилище данных в формате "ключ-значение", основное хранилище всех данных кластера в Kubernetes
- `kube-scheduler` - отслеживает созданные поды без привязанного узла и выбирает узел, на котором они должны работать
- `kube-controller-manager` - уведомляет и реагирует на сбои node, поддерживает правильное количество pod для каждого объекта контроллера репликации в системе,  связывает Services и Pods, создает стандартные учетные записи и токены доступа API для новых namespace

worker node (исполнительный уровень) https://kubernetes.io/ru/docs/concepts/overview/components/#компоненты-узла
- `kubelet` - агент который следит за тем, чтобы контейнеры были запущены в поде
- `kube-proxy` - конфигурирует правила сети на узлах (service), при помощи которых разрешаются сетевые подключения к вашими подам изнутри и снаружи кластера
- `Среда выполнения контейнера` - это программа, предназначенная для выполнения контейнеров (Docker, containerd, CRI-O)

##### Структура объектов и пространства имен

- Kubernetes использует объекты для представления состояния кластера https://kubernetes.io/ru/docs/concepts/overview/working-with-objects/kubernetes-objects/

> Почти в каждом объекте Kubernetes есть два вложенных поля-объекта, которые управляют конфигурацией объекта:
> - `spec` - требуемое состояние (описание характеристик, которые должны быть у объекта)
> - `status` - текущее состояние

### Основные обьекты

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

##### Запуск приложений, заданий и управление ими

> - labels (метки, этикетки) которые используются для идентификации и выбора объектов https://kubernetes.io/ru/docs/concepts/overview/working-with-objects/labels/
> - annotations (аннотации) похожи на лейблы, но не используются для идентификации и выбора объектов https://kubernetes.io/ru/docs/concepts/overview/working-with-objects/annotations/

- deployment https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

<details>
  <summary>Пример обьекта Deployment</summary>

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
- HPA автоматическое увеличение или уменьшение количества реплик приложения https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

<details>
  <summary>Пример обьектов HorizontalPodAutoscaler</summary>

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

##### Сетевые функции

- `service` одно DNS имя которым обьеденен набор pod (используя лейблы на pod) для связи приложений внутри кластера https://kubernetes.io/docs/concepts/services-networking/service/

<details>
  <summary>Пример обьекта Service</summary>

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

- `ingress` для внешнего доступа (из интернета, сети офиса и тд) к приложениям в кластере https://kubernetes.io/docs/concepts/services-networking/ingress/

<details>
  <summary>Пример обьекта Ingress</summary>

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

##### Диски, конфигурация и секреты

- persistent volumes https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- configmap https://kubernetes.io/docs/concepts/configuration/configmap/

<details>
  <summary>Пример обьекта ConfigMap</summary>

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

- `secret` - небольшой объем конфиденциальных данных, таких как пароль, токен или ключ https://kubernetes.io/docs/concepts/configuration/secret/

<details>
  <summary>Пример обьекта Secret</summary>

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
