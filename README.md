1. Install [metrics-server](https://github.com/kubernetes-sigs/metrics-server) by helm:
```shell
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
```
```shell
helm upgrade --install metrics-server metrics-server/metrics-server --set args={"--kubelet-insecure-tls"}
```
2. Copy and apply yaml:
```yaml
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: loki
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: loki
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
apiVersion: v1
kind: Service
metadata:
  name: loki
spec:
  selector:
    app.kubernetes.io/name: loki
  ports:
    - protocol: TCP
      port: 80
      targetPort: http
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loki
  labels:
    app.kubernetes.io/name: loki
    app.kubernetes.io/version: 2.6.1
    app.kubernetes.io/component: loki
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: loki
  template:
    metadata:
      labels:
        app.kubernetes.io/name: loki
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
                  - loki
              topologyKey: "topology.kubernetes.io/zone"
      terminationGracePeriodSeconds: 60
      containers:
      - name: loki
        image: grafana/loki:2.6.1
        ports:
        - name: http
          containerPort: 3100
        resources: 
          requests:
            memory: "150Mi"
            cpu: "150m"
          limits:
            memory: "150Mi"
        livenessProbe:
          httpGet:
            path: /
            port: 3100
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 3100
          initialDelaySeconds: 5
          periodSeconds: 5
---
```
