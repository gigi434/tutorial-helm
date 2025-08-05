# AutoscalingとReplicasの設計パターン

## なぜautoscaling有効時にreplicasを設定しないのか？

deployment.yamlには以下のような条件分岐があります：

```yaml
{{- if not .Values.autoscaling.enabled }}
replicas: {{ .Values.replicaCount }}
{{- end }}
```

この設計は、KubernetesのHorizontal Pod Autoscaler (HPA)の仕様に基づいています。

## HPAとDeploymentの競合問題

### 1. 制御の競合
HPA有効時には、HPAがPod数を直接管理します。もしDeploymentとHPAが同時にPod数を制御しようとすると：
- 意図しない動作が発生
- Pod数が不安定になる
- 期待したスケーリングが機能しない

### 2. 設定の上書き
Deploymentを更新するたびに、HPAが動的に設定したPod数がDeploymentの`replicas`値に戻されてしまいます。

例：
1. HPA が負荷に基づいてPod数を5に増やす
2. 開発者がDeploymentを更新（例：イメージタグの変更）
3. Deploymentの`replicas: 3`により、Pod数が3に戻ってしまう
4. HPAが再度5に増やすまでの間、サービスが劣化する可能性

## 正しい設計パターン

Helmチャートでは、以下の二者択一の設計を採用しています：

### 手動スケーリングモード
```yaml
autoscaling:
  enabled: false
replicaCount: 3
```
- Deploymentで`replicas`を設定
- Pod数は常に固定値
- 手動でのスケーリングが必要

### 自動スケーリングモード
```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```
- HPAリソースを作成
- Deploymentの`replicas`フィールドは設定しない
- HPAがPod数を動的に管理

## 実装例

### deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "testchart.fullname" . }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  # 以下、他の設定...
```

### hpa.yaml
```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "testchart.fullname" . }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "testchart.fullname" . }}
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
{{- end }}
```

## リソースの作成パターン

### 重要な誤解の訂正

DeploymentとHPAは排他的ではありません。実際の動作は以下の通りです：

#### autoscaling.enabled: false の場合
- **Deployment**: 作成される（`replicas`フィールドあり）
- **HPA**: 作成されない

#### autoscaling.enabled: true の場合
- **Deployment**: 作成される（`replicas`フィールドなし）
- **HPA**: 作成される

### 分岐処理の実装場所

#### 1. HPA リソースの条件付き作成（hpa.yaml）
```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
# ... HPAの定義 ...
{{- end }}
```
- ファイル全体が`if`ブロックで囲まれている
- `autoscaling.enabled: true`の場合のみHPAリソースが作成される

#### 2. Deployment の replicas フィールドの条件付き設定（deployment.yaml）
```yaml
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
```
- Deployment自体は常に作成される
- `replicas`フィールドのみが条件付きで設定される

### HPAとDeploymentの関係

HPAはDeploymentを「参照」して動作します：

```yaml
# hpa.yaml より
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "testchart.fullname" . }}
```

- HPAは独立したリソースとして作成される
- `scaleTargetRef`でDeploymentを指定
- HPAがDeploymentのPod数を外部から制御

## まとめ

この設計により：
1. **設定の競合を防ぐ**: HPAとDeploymentが同じPod数を制御しない
2. **明確な責任分離**: スケーリングの責任が明確（手動 or 自動）
3. **予測可能な動作**: 設定に基づいて期待通りの動作を保証

開発者は`autoscaling.enabled`の値を設定するだけで、適切なスケーリング戦略を選択できます。