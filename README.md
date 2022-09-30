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
apiVersion: v1
kind: Service
metadata:
  name: nginx-echo-headers
spec:
  selector:
    app.kubernetes.io/name: nginx-echo-headers
  ports:
    - protocol: TCP
      port: 80
      targetPort: http
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-echo-headers
  labels:
    app.kubernetes.io/name: nginx-echo-headers
    app.kubernetes.io/version: latest
    app.kubernetes.io/component: nginx-echo-headers
spec:
  replicas: 3
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
---
```
