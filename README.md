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
  name: grafana
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: grafana
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
  name: grafana
spec:
  selector:
    app.kubernetes.io/name: grafana
  ports:
    - protocol: TCP
      port: 80
      targetPort: http
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  labels:
    app.kubernetes.io/name: grafana
    app.kubernetes.io/version: 9.1.5
    app.kubernetes.io/component: grafana
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: grafana
  template:
    metadata:
      labels:
        app.kubernetes.io/name: grafana
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
                  - grafana
              topologyKey: "topology.kubernetes.io/zone"
      terminationGracePeriodSeconds: 60
      containers:
      - name: grafana
        image: grafana/grafana::9.1.5
        ports:
        - name: http
          containerPort: 3000
        resources: 
          requests:
            memory: "150Mi"
            cpu: "150m"
          limits:
            memory: "150Mi"
        livenessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
```
