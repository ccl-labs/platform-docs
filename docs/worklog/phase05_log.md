# Phase 5 作業ログ

## Step 1: `platform-charts` リポジトリの作成

```bash
cd ~
mkdir platform-charts
cd platform-charts
git init
git branch -m main
mkdir -p charts/common-app/templates
```

### .envrc / mise.toml のセットアップ（トラブルシュート）

新規リポジトリでは以下の対応が必要。既存リポジトリからコピーするのが確実。

```bash
cp ~/platform-infra/.mise.toml ~/platform-charts/.mise.toml
cp ~/platform-infra/.envrc ~/platform-charts/.envrc
direnv allow ~/platform-charts
mise trust ~/platform-charts
```

---

## Step 2: `common-app` Library Chart の作成

```bash
cat > charts/common-app/Chart.yaml << 'EOF'
apiVersion: v2
name: common-app
description: プラットフォーム共通テンプレート（Library Chart）
type: library
version: 0.1.0
EOF
```

```bash
cat > charts/common-app/values.yaml << 'EOF'
app:
  name: ""
  version: "latest"

image:
  repository: ""
  tag: ""
  pullPolicy: IfNotPresent

replicaCount: 1

containerPort: 8080

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

probes:
  liveness:
    path: /healthz
    initialDelaySeconds: 10
    periodSeconds: 10
  readiness:
    path: /readyz
    initialDelaySeconds: 5
    periodSeconds: 5

ingress:
  enabled: false
  host: ""

hpa:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 70

serviceMonitor:
  enabled: false
  path: /metrics
  interval: 30s
EOF
```

---

## Step 3: テンプレートの実装

Library Chart のテンプレートは `_helpers.tpl` に `define` でまとめて定義する。
（個別の `.yaml` ファイルに分割すると Helm がテンプレートを認識しない問題があったため）

```bash
cat > charts/common-app/templates/_helpers.tpl << 'EOF'
{{- define "common-app.deployment" -}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.app.name }}
  labels:
    app.kubernetes.io/name: {{ .Values.app.name }}
    app.kubernetes.io/version: {{ .Values.app.version }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Values.app.name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ .Values.app.name }}
        app.kubernetes.io/version: {{ .Values.app.version }}
    spec:
      containers:
        - name: {{ .Values.app.name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.containerPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: {{ .Values.containerPort }}
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: {{ .Values.containerPort }}
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
{{- end }}

{{- define "common-app.service" -}}
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.app.name }}
  labels:
    app.kubernetes.io/name: {{ .Values.app.name }}
spec:
  selector:
    app.kubernetes.io/name: {{ .Values.app.name }}
  ports:
    - protocol: TCP
      port: 80
      targetPort: {{ .Values.containerPort }}
{{- end }}

{{- define "common-app.ingress" -}}
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Values.app.name }}
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-issuer
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - {{ .Values.ingress.host }}
      secretName: {{ .Values.app.name }}-tls
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Values.app.name }}
                port:
                  number: 80
{{- end }}
{{- end }}

{{- define "common-app.hpa" -}}
{{- if .Values.hpa.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ .Values.app.name }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ .Values.app.name }}
  minReplicas: {{ .Values.hpa.minReplicas }}
  maxReplicas: {{ .Values.hpa.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.hpa.targetCPUUtilizationPercentage }}
{{- end }}
{{- end }}

{{- define "common-app.serviceMonitor" -}}
{{- if .Values.serviceMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ .Values.app.name }}
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Values.app.name }}
  endpoints:
    - port: http
      path: {{ .Values.serviceMonitor.path }}
      interval: {{ .Values.serviceMonitor.interval }}
{{- end }}
{{- end }}
EOF
```

---

## Step 4: `sample-service` Chart の作成

```bash
mkdir -p charts/sample-service/templates

cat > charts/sample-service/Chart.yaml << 'EOF'
apiVersion: v2
name: sample-service
description: サンプルアプリ（common-appライブラリを利用）
type: application
version: 0.1.0
dependencies:
  - name: common-app
    version: "0.1.0"
    repository: "file://../common-app"
EOF
```

```bash
cat > charts/sample-service/values.yaml << 'EOF'
app:
  name: sample-service
  version: "1.0.0"

image:
  repository: nginx
  tag: "alpine"
  pullPolicy: IfNotPresent

replicaCount: 1

containerPort: 80

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

probes:
  liveness:
    path: /
    initialDelaySeconds: 10
    periodSeconds: 10
  readiness:
    path: /
    initialDelaySeconds: 5
    periodSeconds: 5

ingress:
  enabled: true
  host: sample-service.localhost

hpa:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 70

serviceMonitor:
  enabled: true
  path: /metrics
  interval: 30s
EOF
```

```bash
cat > charts/sample-service/templates/all.yaml << 'EOF'
{{ include "common-app.deployment" . }}
---
{{ include "common-app.service" . }}
---
{{ include "common-app.ingress" . }}
---
{{ include "common-app.hpa" . }}
---
{{ include "common-app.serviceMonitor" . }}
EOF
```

依存関係の解決とテンプレート展開確認。

```bash
cd charts/sample-service
helm dependency build
helm template .
```

---

## Step 5: `platform-charts` を GitHub にpush

```bash
cd ~/platform-charts

cat > .gitignore << 'EOF'
charts/*/charts/*.tgz
EOF

git add .
git commit -m "feat: add common-app library chart and sample-service"
git remote add origin git@github.com:okccl/platform-charts.git
git push -u origin main
```

---

## Step 6: ArgoCD への登録

`platform-charts` リポジトリをArgoCDに登録。

```bash
argocd repo add git@github.com:okccl/platform-charts.git \
  --ssh-private-key-path ~/.ssh/id_ed25519
```

`platform-gitops` に Application マニフェストと values ファイルを追加。

```bash
# apps/sample-service.yaml
cat > apps/sample-service.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-service
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: git@github.com:okccl/platform-charts.git
      targetRevision: main
      path: charts/sample-service
      helm:
        valueFiles:
          - $values/apps/sample-service/values.yaml
    - repoURL: git@github.com:okccl/platform-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: sample-service
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

mkdir -p apps/sample-service
# apps/sample-service/values.yaml は sample-service の values.yaml と同内容を配置

git add .
git commit -m "feat: add sample-service argocd application"
git push
```

ArgoCDを手動syncしてApplicationを反映。

```bash
argocd app sync apps-root
```

---

## 動作確認

```bash
curl -k https://sample-service.localhost
# nginx のデフォルトページが返ることを確認
```
