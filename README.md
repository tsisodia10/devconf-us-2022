# Devconf-us-2022 : Let's Talk about Kubernetes Cluster Monitoring

## STEPS

1. Install Observability operator
```
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  annotations:
  name: observability-operator
  namespace: openshift-marketplace
spec:
  displayName: Observability Operator - Test
  icon:
    base64data: ""
    mediatype: ""
  image: quay.io/rhobs/observability-operator-catalog:latest
  publisher: Sunil Thaha
  sourceType: grpc
  updateStrategy:
    registryPoll:
      interval: 1m0s
```
```
oc apply -f catalogSource.yaml
```

2. Install the Observability Operator on Operator Hub

3. Deploy Blue app
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: blue
    version: v1
  name: blue
  namespace: blue
spec:
  ports:
    - port: 9000
      name: http
      nodePort: 32003
  selector:
    app: blue
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: blue
    version: v1
  name: blue
  namespace: blue
spec:
  selector:
    matchLabels:
      app: blue
      version: v1
  replicas: 1
  template:
    metadata:
      labels:
        app: blue
        version: v1
    spec:
      serviceAccountName: blue
      containers:
        - image: docker.io/cmwylie19/go-metrics-ex
          name: blue
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
          readinessProbe:
            failureThreshold: 3
            initialDelaySeconds: 10
            successThreshold: 1
            periodSeconds: 10
            httpGet:
              path: /
              port: 9000
          livenessProbe:
            initialDelaySeconds: 10
            periodSeconds: 10
            httpGet:
              path: /
              port: 9000
          ports:
            - containerPort: 9000
              name: http
          imagePullPolicy: Always
      restartPolicy: Always
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: blue
  namespace: blue
```
```
oc apply -f app.yaml
```

4. Create Monitoring stack instance in `ns=monitor`(this will create empty prometheus and alertmanager instances)
```
apiVersion: monitoring.rhobs/v1alpha1
kind: MonitoringStack
metadata:
  name: blue
  labels:
    mso: blue
  namespace: monitor
spec:
  logLevel: debug
  prometheusConfig:
    replicas: 1
  retention: 1d
```
```
oc apply -f mso.yaml
```

5. Create servicemonitor
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: blue
  namespace: monitor
  labels:
    app: blue
spec:
  selector:
    matchLabels:
      app: blue
  namespaceSelector:
    any: true
    matchNames:
    - blue    
  endpoints:
  - port: http
```
```
oc apply -f servicemonitor.yaml
```

6. Create PrometheusRules
```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule 
metadata:
  creationTimestamp: null
  labels:
    prometheus: blue
    role: alert-rules
  name: blue-rules
  namespace: monitor-1
spec:
  groups:
  - name: recording_rules
    interval: 2s
    rules: 
    - record: blue_requests_per_minute
      expr: increase(http_requests_total{container="blue"}[1m])
  - name: LoadRules
    rules: 
    - alert: HighLoadBlue
      expr: blue_requests_per_minute >= 10 
      labels:
        severity: page # or critical 
      annotations:
        summary: "high load average"
        description: "high load average"
    - alert: MediumLoadBlue
      expr: blue_requests_per_minute >= 5 
      labels:
        severity: warn 
      annotations:
        summary: "medium load average"
        description: "medium load average" 
    - alert: LowLoadBlue
      expr: blue_requests_per_minute >= 1 
      labels:
        severity: acknowledged
      annotations:
        summary: "low load average"
        description: "low load average" 
```
```
oc apply -f prometheusRules.yaml
```

7. Create alertmanager secret
```
oc create secret generic alertmanager-blue --from-file=alertmanager.yaml=alertmanager.yaml -n monitor
```

8. Create RBAC rules
```
oc create clusterrole blue-view --verb=get,list,watch --resource=pods,pods/status,services,endpoints 

oc create clusterrolebinding blue-crb --clusterrole=blue-view --serviceaccount=monitor:blue-prometheus
```

9. Port forward prometheus and see the alerts, targets and rules
```
oc port-forward pod/prometheus-blue-0 9090 -n monitor
```

10. Port forward alertmanager and see the alerts
```
oc port-forward pod/alertmanager-blue-0 9093 -n monitor
```

11. Trigger alerts
```
oc run curler --image=nginx:alpine --port=80 --expose -n blue
```

Curl the blue app 15 times:
```
for z in $(seq 15); do oc exec -it pod/curler -n blue -- curl blue:9000/; done
```

12. Install Grafana operator

13. Create grafana instance
```
apiVersion: integreatly.org/v1alpha1
kind: Grafana
metadata:
  name: grafana
  namespace: grafana
spec:
  client:
    preferService: true
  config:
    log:
      mode: console
      level: info
    security:
      admin_user: "admin"
      admin_password: "admin"
    auth.anonymous:
      enabled: True
    users:
      viewers_can_edit: true  
  service:
    name: "grafana-operator-controller-manager-metrics-service"
    labels:
      app: "grafana"
      type: "grafana-operator-controller-manager-metrics-service"    
  deployment:
    envFrom:
      - secretRef:
          name: external-credentials    
  dashboardLabelSelector:
    - matchExpressions:
        - { key: app, operator: In, values: [grafana] }
  resources:
    # Optionally specify container resources
    limits:
      cpu: 200m
      memory: 200Mi
    requests:
      cpu: 100m
      memory: 100Mi  
```
```
oc apply -f grafana.yaml
```
