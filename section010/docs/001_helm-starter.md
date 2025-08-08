# Helm Starter - カスタムチャートテンプレート

## Helm Starterとは

Helm Starterは、`helm create`コマンドで新しいチャートを作成する際の**カスタムテンプレート（雛形）**です。組織やプロジェクト固有の規約やベストプラクティスを事前定義したチャートテンプレートを作成・共有できます。

### 通常のhelm createとの違い

```bash
# デフォルトテンプレート
helm create mychart
# → Helmビルトインのテンプレートを使用

# Starterを使用
helm create mychart --starter mycompany-starter
# → カスタムテンプレートを使用
```

## Starterの仕組み

### 1. Starterの保存場所

```bash
# Starterチャートの格納ディレクトリ
$HOME/.local/share/helm/starters/     # Linux/WSL
$HOME/Library/helm/starters/           # macOS
%APPDATA%\helm\starters\               # Windows

# 環境変数で変更可能
export HELM_DATA_HOME=/custom/path
# → /custom/path/helm/starters/
```

### 2. ディレクトリ構造

```
~/.local/share/helm/starters/
├── mycompany-web/           # Webアプリ用Starter
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── charts/
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       └── _helpers.tpl
├── mycompany-api/           # API用Starter
│   └── ...
└── mycompany-cronjob/       # CronJob用Starter
    └── ...
```

## Starterの作成

### 1. 基本的なStarterチャート

```bash
# Starterディレクトリの作成
mkdir -p ~/.local/share/helm/starters/mycompany-web

# 基本構造の作成
cd ~/.local/share/helm/starters/mycompany-web
```

#### Chart.yaml

```yaml
apiVersion: v2
name: <CHARTNAME>  # プレースホルダー
description: A Helm chart for Kubernetes web applications
type: application
version: 0.1.0
appVersion: "1.0.0"
keywords:
  - web
  - <CHARTNAME>
maintainers:
  - name: <AUTHOR>
    email: <EMAIL>
```

#### values.yaml

```yaml
# 組織標準の設定値
replicaCount: 2  # 本番環境のデフォルト

image:
  repository: <REGISTRY>/<CHARTNAME>
  pullPolicy: IfNotPresent
  tag: ""  # Overrides the image tag

# 組織のセキュリティポリシー
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

# 組織標準のリソース設定
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

# 組織標準のProbe設定
livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5

# 自動スケーリング設定
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

# 組織標準のラベル
labels:
  team: <TEAM>
  environment: <ENVIRONMENT>
  managed-by: helm
```

#### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "<CHARTNAME>.fullname" . }}
  labels:
    {{- include "<CHARTNAME>.labels" . | nindent 4 }}
    {{- with .Values.labels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "<CHARTNAME>.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        # 組織標準のアノテーション
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
      labels:
        {{- include "<CHARTNAME>.selectorLabels" . | nindent 8 }}
    spec:
      # セキュリティ設定
      {{- with .Values.podSecurityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      
      # init containerのテンプレート
      initContainers:
      - name: wait-for-deps
        image: busybox:1.35
        command: ['sh', '-c', 'echo "Waiting for dependencies..."; sleep 2']
      
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        
        # セキュリティコンテキスト
        {{- with .Values.securityContext }}
        securityContext:
          {{- toYaml . | nindent 12 }}
        {{- end }}
        
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        - name: metrics
          containerPort: 9090
          protocol: TCP
        
        # ヘルスチェック
        {{- with .Values.livenessProbe }}
        livenessProbe:
          {{- toYaml . | nindent 12 }}
        {{- end }}
        {{- with .Values.readinessProbe }}
        readinessProbe:
          {{- toYaml . | nindent 12 }}
        {{- end }}
        
        # リソース制限
        {{- with .Values.resources }}
        resources:
          {{- toYaml . | nindent 12 }}
        {{- end }}
        
        # 環境変数
        env:
        - name: APP_NAME
          value: {{ include "<CHARTNAME>.fullname" . }}
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        
        # ボリュームマウント
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
      
      # ボリューム定義
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
```

#### templates/_helpers.tpl

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "<CHARTNAME>.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "<CHARTNAME>.fullname" -}}
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
Create chart name and version as used by the chart label.
*/}}
{{- define "<CHARTNAME>.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels - 組織標準のラベル
*/}}
{{- define "<CHARTNAME>.labels" -}}
helm.sh/chart: {{ include "<CHARTNAME>.chart" . }}
{{ include "<CHARTNAME>.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
# 組織固有のラベル
organization: mycompany
cost-center: engineering
{{- end }}

{{/*
Selector labels
*/}}
{{- define "<CHARTNAME>.selectorLabels" -}}
app.kubernetes.io/name: {{ include "<CHARTNAME>.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### 2. プレースホルダーの置換

Starterでは以下のプレースホルダーが自動置換されます：

| プレースホルダー | 置換内容 | 使用例 |
|-----------------|----------|--------|
| `<CHARTNAME>` | チャート名 | `helm create mychart` → `mychart` |
| `<YEAR>` | 現在の年 | `2024` |
| `<AUTHOR>` | Git設定の作成者 | `user.name` from git config |
| `<EMAIL>` | Git設定のメール | `user.email` from git config |

## 実践的なStarterパターン

### 1. マイクロサービス用Starter

```bash
mkdir -p ~/.local/share/helm/starters/microservice
```

```yaml
# microservice/values.yaml
service:
  type: ClusterIP
  port: 80
  targetPort: 8080

# サービスメッシュ設定
istio:
  enabled: true
  virtualService:
    enabled: true
    timeout: 30s
    retries:
      attempts: 3
      perTryTimeout: 10s

# 分散トレーシング
tracing:
  enabled: true
  jaeger:
    agent:
      host: jaeger-agent.istio-system
      port: 6831

# メトリクス
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    interval: 30s
```

### 2. バッチJob用Starter

```bash
mkdir -p ~/.local/share/helm/starters/batch-job
```

```yaml
# batch-job/templates/cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ include "<CHARTNAME>.fullname" . }}
spec:
  schedule: {{ .Values.schedule | quote }}
  concurrencyPolicy: {{ .Values.concurrencyPolicy }}
  successfulJobsHistoryLimit: {{ .Values.historyLimit.successful }}
  failedJobsHistoryLimit: {{ .Values.historyLimit.failed }}
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: {{ .Chart.Name }}
            image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
            command: {{ .Values.command }}
            args: {{ .Values.args }}
            resources:
              {{- toYaml .Values.resources | nindent 14 }}
```

### 3. StatefulSet用Starter

```bash
mkdir -p ~/.local/share/helm/starters/stateful-app
```

```yaml
# stateful-app/templates/statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "<CHARTNAME>.fullname" . }}
spec:
  serviceName: {{ include "<CHARTNAME>.fullname" . }}-headless
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "<CHARTNAME>.selectorLabels" . | nindent 6 }}
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: {{ .Values.persistence.storageClass }}
      resources:
        requests:
          storage: {{ .Values.persistence.size }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        volumeMounts:
        - name: data
          mountPath: /data
```

## Starterの使用

### 1. Starterのインストール

```bash
# GitリポジトリからStarter取得
git clone https://github.com/mycompany/helm-starters
cp -r helm-starters/* ~/.local/share/helm/starters/

# 利用可能なStarterの確認
ls ~/.local/share/helm/starters/
```

### 2. Starterを使用したチャート作成

```bash
# Webアプリケーション用
helm create mywebapp --starter mycompany-web

# APIサービス用
helm create myapi --starter mycompany-api

# バッチジョブ用
helm create mybatch --starter batch-job

# 作成されたチャートの確認
tree mywebapp/
```

### 3. 作成後のカスタマイズ

```bash
cd mywebapp

# values.yamlの編集
cat > values.yaml <<EOF
replicaCount: 3

image:
  repository: myregistry.io/mywebapp
  tag: "1.0.0"

labels:
  team: platform
  environment: production
EOF

# デプロイ
helm install mywebapp .
```

## Starterの管理

### 1. 組織でのStarter共有

```yaml
# starter-repo/install.sh
#!/bin/bash
STARTER_DIR="${HELM_DATA_HOME:-$HOME/.local/share}/helm/starters"

echo "Installing company starters to $STARTER_DIR"
mkdir -p "$STARTER_DIR"

for starter in */Chart.yaml; do
  dir=$(dirname "$starter")
  echo "Installing $dir starter..."
  cp -r "$dir" "$STARTER_DIR/"
done

echo "Available starters:"
ls -la "$STARTER_DIR"
```

### 2. CI/CDでのStarter利用

```yaml
# .github/workflows/create-chart.yml
name: Create Chart from Starter

on:
  workflow_dispatch:
    inputs:
      chart_name:
        description: 'Chart name'
        required: true
      starter:
        description: 'Starter template'
        required: true
        default: 'mycompany-web'

jobs:
  create:
    runs-on: ubuntu-latest
    steps:
    - name: Setup Helm
      uses: azure/setup-helm@v3
      
    - name: Install Starters
      run: |
        git clone https://github.com/${{ github.repository_owner }}/helm-starters
        mkdir -p ~/.local/share/helm/starters
        cp -r helm-starters/* ~/.local/share/helm/starters/
    
    - name: Create Chart
      run: |
        helm create ${{ github.event.inputs.chart_name }} \
          --starter ${{ github.event.inputs.starter }}
    
    - name: Commit Chart
      run: |
        git add ${{ github.event.inputs.chart_name }}
        git commit -m "Create ${{ github.event.inputs.chart_name }} from ${{ github.event.inputs.starter }}"
        git push
```

## ベストプラクティス

### 1. Starter設計の原則

```yaml
# 原則1: 最小限の必須設定
# BAD - 多すぎる設定
resources:
  limits:
    cpu: 2000m
    memory: 4Gi
    ephemeral-storage: 10Gi
  requests:
    cpu: 1000m
    memory: 2Gi
    ephemeral-storage: 5Gi

# GOOD - 適切なデフォルト
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

### 2. セキュリティの標準化

```yaml
# templates/networkpolicy.yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "<CHARTNAME>.fullname" . }}
spec:
  podSelector:
    matchLabels:
      {{- include "<CHARTNAME>.selectorLabels" . | nindent 6 }}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: nginx-ingress
  egress:
  - to:
    - podSelector: {}
{{- end }}
```

### 3. 監視設定の標準化

```yaml
# templates/servicemonitor.yaml
{{- if .Values.metrics.serviceMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "<CHARTNAME>.fullname" . }}
spec:
  selector:
    matchLabels:
      {{- include "<CHARTNAME>.selectorLabels" . | nindent 6 }}
  endpoints:
  - port: metrics
    interval: {{ .Values.metrics.serviceMonitor.interval }}
    path: {{ .Values.metrics.serviceMonitor.path }}
{{- end }}
```

## トラブルシューティング

### 1. Starterが見つからない

```bash
# Starterディレクトリの確認
echo $HELM_DATA_HOME
ls -la ~/.local/share/helm/starters/

# 権限の確認
chmod -R 755 ~/.local/share/helm/starters/
```

### 2. プレースホルダーが置換されない

```bash
# 正しいプレースホルダーを使用
# ✅ 正しい
<CHARTNAME>
<YEAR>

# ❌ 間違い
${CHARTNAME}
{{CHARTNAME}}
```

### 3. Git情報が取得できない

```bash
# Git設定の確認
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

## まとめ

Helm Starterは以下の利点を提供します：

1. **標準化**: 組織のベストプラクティスを強制
2. **効率化**: 繰り返し作業の削減
3. **品質向上**: セキュリティ設定の標準化
4. **学習曲線の緩和**: 新規メンバーの立ち上げ支援

適切に設計されたStarterにより、一貫性のある高品質なHelmチャートを効率的に作成できます。