# DeploymentとHPAのリソース作成パターン

## リソース作成の実際の動作

DeploymentとHPAは排他的ではありません。実際の動作は以下の通りです：

### autoscaling.enabled: false の場合
- **Deployment**: 作成される（`replicas`フィールドあり）
- **HPA**: 作成されない

### autoscaling.enabled: true の場合
- **Deployment**: 作成される（`replicas`フィールドなし）
- **HPA**: 作成される

## 分岐処理の実装場所

### 1. HPA リソースの条件付き作成（hpa.yaml）

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "testchart.fullname" . }}
  labels:
    {{- include "testchart.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "testchart.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  # ... 以下、メトリクス設定 ...
{{- end }}
```

**重要なポイント**:
- ファイル全体が`if`ブロックで囲まれている
- `autoscaling.enabled: true`の場合のみHPAリソースが作成される

### 2. Deployment の replicas フィールドの条件付き設定（deployment.yaml）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "testchart.fullname" . }}
  labels:
    {{- include "testchart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "testchart.selectorLabels" . | nindent 6 }}
  # ... 以下、他の設定 ...
```

**重要なポイント**:
- Deployment自体は常に作成される
- `replicas`フィールドのみが条件付きで設定される
- `autoscaling.enabled: false`の場合のみ`replicas`が設定される

## HPAとDeploymentの関係

HPAはDeploymentを「参照」して動作します：

```yaml
# hpa.yaml より
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "testchart.fullname" . }}
```

### 動作の流れ

1. **autoscaling無効時**:
   - Deploymentが`replicas: 3`などの固定値で作成される
   - HPAは作成されない
   - Pod数は常に固定

2. **autoscaling有効時**:
   - Deploymentが`replicas`フィールドなしで作成される
   - HPAが作成され、Deploymentを参照
   - HPAがPod数を動的に制御（min/max範囲内）

### なぜこの設計なのか

- HPAは独立したリソースとして作成される
- `scaleTargetRef`でDeploymentを指定
- HPAがDeploymentのPod数を外部から制御
- これにより、DeploymentとHPAの責任が明確に分離される

## まとめ

- **Deploymentは常に存在**する（autoscalingの有無に関わらず）
- **HPAは条件付き**で作成される（autoscaling有効時のみ）
- 分岐処理は2箇所：
  1. `hpa.yaml`の1行目でHPAの作成可否を制御
  2. `deployment.yaml`の`spec`セクションで`replicas`フィールドの有無を制御