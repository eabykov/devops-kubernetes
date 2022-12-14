### Установка Kubernetes локально

1. Ставим себе Docker Desktop https://www.docker.com/products/docker-desktop/
2. Ставим себе WSL (аналог виртуальной машины Linux на Windows) https://aka.ms/wslinstall
3. Ставим себе Ubuntu в WSL из Microsoft Store https://apps.microsoft.com/store/search?publisher=Canonical%20Group%20Limited
4. Перезапускаем компьютер
5. Запускам Ubuntu и ждем завершение установки
6. Запускаем Docker Desktop
7. Подключаем Ubuntu к Docker Desctop. Для этого в его настройках переходим в раздел `Resorces` и там в `WSL Integration` включаем интеграцию с Ubuntu
8. Включаем Kubernetes в настройках Docker Desctop (если хотим смотреть за прогрессом загрузки выходим из настроек и смотрим как появляются новые Images)
9. Когда у нас иконка Kubernetes (штурвал от корабля) внизу станет зеленой это значит что мы можем зайти в Ubuntu и выполнить команду `kubectl version --short` которая должна вывести версию Kubernetes

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
spec:
  replicas: 3 # можно удалить если используем HPA который сам будет следить за числом реплик
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

##### Сетевые функции

- `service` одно DNS имя которым обьеденен набор pod (используя лейблы на pod) https://kubernetes.io/docs/concepts/services-networking/service/

<details>
  <summary>Пример обьекта Service</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <insert-service-name-here> # одно DNS имя которым обьеденен набор pod (используя лейблы на pod в selector ниже)
spec:
  selector:
    app.kubernetes.io/name: <insert-lable-name-here> # выбирает pod по лейблу
  ports:
    - protocol: TCP
      port: 80
      targetPort: http
```

</details>


- `ingress` управляет внешним доступом к службам в кластере https://kubernetes.io/docs/concepts/services-networking/ingress/

##### Диски, конфигурация и секреты

- persistent volumes https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- configmap https://kubernetes.io/docs/concepts/configuration/configmap/
- secret https://kubernetes.io/docs/concepts/configuration/secret/

<details>
  <summary>Пример обьектов Kubenretes</summary>

```yaml
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler # автоматическое увеличение или уменьшение количества реплик приложения
metadata:
  name: nginx-echo-headers
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-echo-headers
  minReplicas: 3
  maxReplicas: 4
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-echo-headers
  labels: # используются для идентификации и выбора объектов
    app.kubernetes.io/name: nginx-echo-headers
    app.kubernetes.io/version: latest
    app.kubernetes.io/component: nginx-echo-headers
spec:
#  replicas: 3 # можно удалить т.к. мы используем HPA который сам будет следить за числом реплик
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx-echo-headers
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx-echo-headers
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution: 
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app.kubernetes.io/name
                  operator: In
                  values:
                  - nginx-echo-headers
              topologyKey: "topology.kubernetes.io/zone"
      terminationGracePeriodSeconds: 60
      containers:
      - name: nginx-echo-headers
        image: brndnmtthws/nginx-echo-headers:latest
        ports:
        - name: http
          containerPort: 8080
        resources: 
          requests:
            memory: "150Mi"
            cpu: "150m"
          limits:
            memory: "150Mi"
        livenessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
```

</details>
