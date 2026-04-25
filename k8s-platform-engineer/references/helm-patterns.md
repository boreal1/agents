# Helm Chart Patterns

Detailed patterns for writing production Helm charts. Read this when you're actively
authoring or reviewing a chart and need the specifics.

## Table of Contents

1. [Chart.yaml](#chartyaml)
2. [_helpers.tpl Named Templates](#helperstpl-named-templates)
3. [Deployment Template](#deployment-template)
4. [Service Template](#service-template)
5. [Ingress / HTTPRoute Template](#ingress--httproute-template)
6. [HPA Template](#hpa-template)
7. [PDB Template](#pdb-template)
8. [NetworkPolicy Template](#networkpolicy-template)
9. [ServiceAccount Template](#serviceaccount-template)
10. [ConfigMap and Secrets](#configmap-and-secrets)
11. [NOTES.txt](#notestxt)
12. [Multi-Environment Values](#multi-environment-values)
13. [Chart Dependencies](#chart-dependencies)

## Chart.yaml

```yaml
apiVersion: v2
name: myapp
description: A Helm chart for deploying myapp on Kubernetes
type: application
version: 0.1.0
appVersion: "1.0.0"
kubeVersion: ">=1.27.0-0"
home: https://github.com/myorg/myapp
sources:
  - https://github.com/myorg/myapp
maintainers:
  - name: platform-team
    email: platform@myorg.com
keywords:
  - api
  - microservice
annotations:
  category: Application
```

Key points:
- `apiVersion: v2` is required for Helm 3 charts
- `version` follows SemVer 2 and is the *chart* version, not the app version
- `appVersion` is the application version — templates reference `.Chart.AppVersion`
- `kubeVersion` prevents installation on unsupported clusters

## _helpers.tpl Named Templates

```gotemplate
{{/*
Expand the name of the chart.
*/}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels — include these on every resource.
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{ include "myapp.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels — use these in spec.selector.matchLabels and svc.selector.
These must be stable across chart upgrades (don't include version).
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Service account name.
*/}}
{{- define "myapp.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "myapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

## Deployment Template

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- with .Values.deploymentAnnotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  strategy:
    {{- if .Values.strategy }}
    {{- toYaml .Values.strategy | nindent 4 }}
    {{- else }}
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
    {{- end }}
  template:
    metadata:
      labels:
        {{- include "myapp.labels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            {{- toYaml .Values.ports | nindent 12 }}
          {{- with .Values.env }}
          env:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.envFrom }}
          envFrom:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.volumeMounts }}
          volumeMounts:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          {{- with .Values.startupProbe }}
          startupProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.topologySpreadConstraints }}
      topologySpreadConstraints:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

Key points about the checksum annotation:
- `checksum/config` forces a rollout when the ConfigMap changes, since Kubernetes
  doesn't automatically restart pods on ConfigMap updates. This is a common gotcha.

## Service Template

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  {{- if and .Values.service.clusterIP (eq .Values.service.type "ClusterIP") }}
  clusterIP: {{ .Values.service.clusterIP }}
  {{- end }}
  {{- if .Values.service.externalTrafficPolicy }}
  externalTrafficPolicy: {{ .Values.service.externalTrafficPolicy }}
  {{- end }}
  ports:
    {{- toYaml .Values.service.ports | nindent 4 }}
  selector:
    {{- include "myapp.selectorLabels" . | nindent 4 }}
```

## Ingress / HTTPRoute Template

For Ingress (traditional):

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType | default "Prefix" }}
            backend:
              service:
                name: {{ include "myapp.fullname" $ }}
                port:
                  number: {{ .port }}
          {{- end }}
    {{- end }}
{{- end }}
```

For Gateway API (recommended for new deployments):

```yaml
{{- if .Values.gateway.enabled }}
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  parentRefs:
    {{- toYaml .Values.gateway.parentRefs | nindent 4 }}
  hostnames:
    {{- toYaml .Values.gateway.hostnames | nindent 4 }}
  rules:
    {{- range .Values.gateway.rules }}
    - backendRefs:
        {{- range .backendRefs }}
        - name: {{ include "myapp.fullname" $ }}
          port: {{ .port }}
          weight: {{ .weight | default 100 }}
        {{- end }}
      {{- with .filters }}
      filters:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .matches }}
      matches:
        {{- toYaml . | nindent 8 }}
      {{- end }}
    {{- end }}
{{- end }}
```

## HPA Template

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "myapp.fullname" . }}
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
{{- end }}
```

## PDB Template

```yaml
{{- if .Values.pdb.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  {{- if .Values.pdb.minAvailable }}
  minAvailable: {{ .Values.pdb.minAvailable }}
  {{- end }}
  {{- if .Values.pdb.maxUnavailable }}
  maxUnavailable: {{ .Values.pdb.maxUnavailable }}
  {{- end }}
{{- end }}
```

## NetworkPolicy Template

```yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  podSelector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              {{- toYaml .Values.networkPolicy.ingress.from | nindent 14 }}
      ports:
        {{- toYaml .Values.networkPolicy.ingress.ports | nindent 8 }}
  egress:
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    {{- with .Values.networkPolicy.egress }}
    - to:
        {{- toYaml .to | nindent 8 }}
      ports:
        {{- toYaml .ports | nindent 8 }}
    {{- end }}
{{- end }}
```

The egress rule always allows DNS resolution — this is the most common footgun with
NetworkPolicy (forgetting to allow DNS breaks everything).

## ServiceAccount Template

```yaml
{{- if .Values.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "myapp.serviceAccountName" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
automountServiceAccountToken: {{ .Values.serviceAccount.automount | default true }}
{{- end }}
```

For AKS Workload Identity, add this annotation:

```yaml
annotations:
  azure.workload.identity/client-id: "{{ .Values.serviceAccount.azureClientId }}"
```

For EKS IRSA, add this annotation:

```yaml
annotations:
  eks.amazonaws.com/role-arn: "{{ .Values.serviceAccount.awsRoleArn }}"
```

## ConfigMap and Secrets

**ConfigMap** — for non-sensitive configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
data:
  {{- toYaml .Values.config | nindent 2 }}
```

**Secrets** — avoid templating sensitive values. Instead, recommend external-secrets
and provide a placeholder Secret for development:

```yaml
{{- if .Values.secret.create }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
type: Opaque
stringData:
  {{- toYaml .Values.secret.data | nindent 4 }}
{{- end }}
```

In values.yaml, add a comment steering users toward external-secrets:

```yaml
secret:
  # -- Create a Secret with these values (for dev only; use external-secrets in production)
  create: false
  # -- Secret data — DO NOT put real secrets here in production
  data: {}
```

## NOTES.txt

```gotemplate
{{ include "myapp.fullname" . }} has been deployed!

{{- if .Values.ingress.enabled }}
URL: http{{ if .Values.ingress.tls }}s{{ end }}://{{ .Values.ingress.hosts[0].host }}
{{- else if .Values.gateway.enabled }}
URL: Check the HTTPRoute for host configuration
{{- else }}
Port: {{ .Values.service.ports[0].port }}
{{- end }}

{{- if not .Values.autoscaling.enabled }}
Replicas: {{ .Values.replicaCount }}
{{- else }}
Replicas: {{ .Values.autoscaling.minReplicas }}-{{ .Values.autoscaling.maxReplicas }} (HPA)
{{- end }}
```

## Multi-Environment Values

Structure environment overrides as separate files:

```
myapp/
├── values.yaml              # base defaults
├── values-dev.yaml          # dev overrides (lower resources, 1 replica)
├── values-staging.yaml      # staging overrides (production-like but smaller)
└── values-prod.yaml          # production overrides (full resources, PDB, HPA)
```

Typical `values-dev.yaml`:

```yaml
replicaCount: 1

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

autoscaling:
  enabled: false

pdb:
  enabled: false

networkPolicy:
  enabled: false
```

Typical `values-prod.yaml`:

```yaml
replicaCount: 3

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: 1

networkPolicy:
  enabled: true

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: {{ include "myapp.name" . }}
```

Deploy with:

```bash
helm install myapp ./myapp -f values.yaml -f values-prod.yaml
```

Later files override earlier ones, so `-f values-prod.yaml` overrides the base.

## Chart Dependencies

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "16.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
    alias: postgresql

  - name: redis
    version: "19.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```yaml
# values.yaml
postgresql:
  enabled: false
  auth:
    username: myapp
    database: myapp

redis:
  enabled: false
  architecture: standalone
```

After adding dependencies, always run:

```bash
helm dependency update ./mychart
```
