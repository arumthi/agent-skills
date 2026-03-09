# OpenShift/Kubernetes Deployment: Helm Charts

## Overview

This guide provides production-ready Helm charts and Kubernetes manifests for deploying microservices on Red Hat OpenShift Container Platform (OCP).

## Helm Chart Structure

```
helm-charts/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── ingress.yaml (or route.yaml for OpenShift)
│   ├── hpa.yaml (horizontal pod autoscaler)
│   ├── networkpolicy.yaml
│   └── NOTES.txt
└── values-prod.yaml
```

## Step 1: Create Helm Chart

### Chart.yaml

```yaml
apiVersion: v2
name: order-service
description: A Helm chart for Order Service microservice
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - microservices
  - order-management
  - spring-boot
maintainers:
  - name: Platform Team
    email: platform@example.com
dependencies:
  - name: postgresql
    version: "13.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

### values.yaml (Default Values)

```yaml
# Default values for order service
replicaCount: 2

image:
  repository: my-registry.azurecr.io/order-service
  pullPolicy: IfNotPresent
  tag: "1.0.0"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/actuator/prometheus"

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080
  annotations: {}

# OpenShift Route (instead of Ingress)
route:
  enabled: false  # Set to true for OpenShift
  host: order-service.example.com
  tls:
    enabled: true
    termination: edge

# Kubernetes Ingress (for cloud platforms)
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: order-service.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: order-service-tls
      hosts:
        - order-service.example.com

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 250m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

nodeSelector: {}

tolerations: []

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - order-service
          topologyKey: kubernetes.io/hostname
        weight: 100

# Database configuration
database:
  host: "oracle-db.default.svc.cluster.local"
  port: 1521
  sid: "XEPDB1"
  username: "order_service"
  # Password should be in secrets, not here
  poolSize: 20
  maxWaitMillis: 30000

# Environment variables
env:
  SPRING_PROFILES_ACTIVE: "production"
  JAVA_OPTS: "-Xmx768m -Xms256m -XX:+UseG1GC -XX:MaxGCPauseMillis=200"

# Secrets management
secrets:
  create: true
  # These should come from sealed-secrets or external vault
  database:
    password: "" # Set via values-secrets.yaml
  jwt:
    secret: ""
    issuerUri: "http://auth-service:8080"

# Health checks
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3

# Logging
logging:
  level: INFO
  format: json

# Monitoring
monitoring:
  enabled: true
  serviceMonitor:
    enabled: false  # Enable if using Prometheus Operator
    interval: 30s
```

### values-prod.yaml (Production Overrides)

```yaml
replicaCount: 3

image:
  tag: "1.0.0-prod"

resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 500m
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 75

env:
  SPRING_PROFILES_ACTIVE: "production"
  JAVA_OPTS: "-Xmx1536m -Xms512m -XX:+UseG1GC -XX:MaxGCPauseMillis=100"

database:
  poolSize: 50
  host: "oracle-prod.database.svc.cluster.local"

route:
  enabled: true
  host: "order-service-prod.apps.example.com"
  tls:
    enabled: true
    termination: reencrypt
```

## Step 2: Create Deployment Template

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-service.fullname" . }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "order-service.selectorLabels" . | nindent 8 }}
        version: {{ .Values.image.tag | quote }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "order-service.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      
      # Init container for database migrations
      initContainers:
      - name: db-migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        command: ["sh", "-c"]
        args:
          - |
            java -cp /app/libs/* \
            org.flywaydb.commandline.Main \
            -url="jdbc:oracle:thin:@//{{ .Values.database.host }}:{{ .Values.database.port }}/{{ .Values.database.sid }}" \
            -user="{{ .Values.database.username }}" \
            -password="${DB_PASSWORD}" \
            -locations=classpath:db/migration \
            migrate
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "order-service.fullname" . }}
              key: database-password
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      
      containers:
      - name: {{ .Chart.Name }}
        securityContext:
          {{- toYaml .Values.securityContext | nindent 12 }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: {{ .Values.env.SPRING_PROFILES_ACTIVE | quote }}
        - name: JAVA_OPTS
          value: {{ .Values.env.JAVA_OPTS | quote }}
        - name: SPRING_DATASOURCE_URL
          value: "jdbc:oracle:thin:@//{{ .Values.database.host }}:{{ .Values.database.port }}/{{ .Values.database.sid }}"
        - name: SPRING_DATASOURCE_USERNAME
          value: {{ .Values.database.username | quote }}
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "order-service.fullname" . }}
              key: database-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: {{ include "order-service.fullname" . }}
              key: jwt-secret
        - name: JWT_ISSUER_URI
          value: {{ .Values.secrets.jwt.issuerUri | quote }}
        
        {{- range $key, $value := .Values.env }}
        {{- if ne $key "SPRING_PROFILES_ACTIVE" }}
        {{- if ne $key "JAVA_OPTS" }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        {{- end }}
        {{- end }}
        
        livenessProbe:
          {{- toYaml .Values.livenessProbe | nindent 12 }}
        
        readinessProbe:
          {{- toYaml .Values.readinessProbe | nindent 12 }}
        
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
        
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir:
          sizeLimit: 1Gi
      
      terminationGracePeriodSeconds: 30
      
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

## Step 3: Create Service Template

### templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "order-service.fullname" . }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
  {{- with .Values.service.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "order-service.selectorLabels" . | nindent 4 }}
```

## Step 4: Create OpenShift Route

### templates/route.yaml (OpenShift-specific)

```yaml
{{- if .Values.route.enabled }}
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: {{ include "order-service.fullname" . }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
spec:
  host: {{ .Values.route.host }}
  to:
    kind: Service
    name: {{ include "order-service.fullname" . }}
    weight: 100
  port:
    targetPort: http
  {{- if .Values.route.tls.enabled }}
  tls:
    termination: {{ .Values.route.tls.termination }}
    insecureEdgeTerminationPolicy: Redirect
  {{- end }}
{{- end }}
```

## Step 5: Create ConfigMap and Secrets

### templates/configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "order-service.fullname" . }}-config
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
data:
  application.yml: |
    spring:
      application:
        name: {{ include "order-service.fullname" . }}
      profiles:
        active: {{ .Values.env.SPRING_PROFILES_ACTIVE }}
      jpa:
        show-sql: false
        hibernate:
          ddl-auto: validate
    
    logging:
      level:
        root: {{ .Values.logging.level }}
      format: {{ .Values.logging.format }}
```

### templates/secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "order-service.fullname" . }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
type: Opaque
data:
  database-password: {{ .Values.secrets.database.password | b64enc | quote }}
  jwt-secret: {{ .Values.secrets.jwt.secret | b64enc | quote }}
```

## Step 6: Create Horizontal Pod Autoscaler

### templates/hpa.yaml

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "order-service.fullname" . }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "order-service.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
  {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
  {{- end }}
  {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
  {{- end }}
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
    scaleUp:
      stabilizationWindowSeconds: 0
{{- end }}
```

## Step 7: Deployment Commands

```bash
# Create namespace
oc new-project order-service --display-name="Order Service"

# Create image pull secret (if using private registry)
oc create secret docker-registry regcred \
  --docker-server=my-registry.azurecr.io \
  --docker-username=<username> \
  --docker-password=<password> \
  -n order-service

# Deploy using Helm (development)
helm install order-service ./helm-charts \
  --namespace=order-service \
  --values values.yaml \
  --values values-dev.yaml

# Deploy using Helm (production)
helm install order-service ./helm-charts \
  --namespace=order-service-prod \
  --values values.yaml \
  --values values-prod.yaml \
  --set secrets.database.password=$(oc get secret oracle-db -o jsonpath='{.data.password}' | base64 -d) \
  --set secrets.jwt.secret=$(openssl rand -hex 32)

# Upgrade deployment
helm upgrade order-service ./helm-charts \
  --namespace=order-service \
  --values values.yaml

# Verify deployment
oc get pods -n order-service
oc describe deployment order-service -n order-service
oc logs -f deployment/order-service -n order-service

# Access service
oc port-forward svc/order-service 8080:8080 -n order-service
curl http://localhost:8080/actuator/health
```

## Step 8: Network Policies

### templates/networkpolicy.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "order-service.fullname" . }}
spec:
  podSelector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: order-DB
    ports:
    - protocol: TCP
      port: 1521
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 53  # DNS
    - protocol: UDP
      port: 53
```

## Success Criteria

✅ Helm chart generated and templates validated with `helm lint`  
✅ Deployment successful to dev OCP cluster  
✅ Service accessible via route  
✅ Database migrations executed automatically  
✅ Health probes responding  
✅ HPA pod scaling based on metrics  
✅ Logs accessible via `oc logs`  
✅ Multi-replica high availability working
