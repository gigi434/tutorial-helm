# _helpers.tpl ファイルの説明

## _helpers.tpl とは

`_helpers.tpl`は、Helmチャート全体で再利用可能なテンプレート（ヘルパー関数）を定義するファイルです。アンダースコア（`_`）で始まるファイルは、Kubernetesマニフェストとしては出力されず、他のテンプレートから呼び出して使用されます。

## ファイルの特徴

1. **再利用可能なコード**: 複数のテンプレートファイルで共通して使用する処理を定義
2. **命名規則の統一**: リソース名やラベルの生成ロジックを一元管理
3. **DRY原則**: Don't Repeat Yourselfの原則に従い、コードの重複を防ぐ

## 定義されているヘルパー関数

### 1. `testchart.name`

```yaml
{{- define "testchart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}
```

**機能**: チャートの名前を生成
- `nameOverride`が設定されていればそれを使用
- なければ`Chart.Name`を使用
- 63文字で切り詰め（Kubernetesの制限）
- 末尾のハイフンを削除

### 2. `testchart.fullname`

```yaml
{{- define "testchart.fullname" -}}
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
```

**機能**: 完全修飾アプリケーション名を生成
- `fullnameOverride`があれば優先使用
- リリース名にチャート名が含まれている場合は、リリース名のみ使用
- そうでなければ「リリース名-チャート名」の形式で生成
- DNS命名規則に従い63文字で切り詰め

**使用例**:
- リリース名: `my-app`
- チャート名: `testchart`
- 結果: `my-app-testchart`

### 3. `testchart.chart`

```yaml
{{- define "testchart.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}
```

**機能**: チャート名とバージョンのラベル値を生成
- 形式: `チャート名-バージョン`
- `+`記号を`_`に置換（ラベル値の制限）
- 例: `testchart-0.1.0`

### 4. `testchart.labels`

```yaml
{{- define "testchart.labels" -}}
helm.sh/chart: {{ include "testchart.chart" . }}
{{ include "testchart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

**機能**: 共通ラベルのセットを生成
- `helm.sh/chart`: チャート名とバージョン
- セレクターラベル（下記参照）
- `app.kubernetes.io/version`: アプリケーションバージョン
- `app.kubernetes.io/managed-by`: 管理ツール（通常は"Helm"）

### 5. `testchart.selectorLabels`

```yaml
{{- define "testchart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "testchart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

**機能**: セレクター用の基本ラベルを生成
- `app.kubernetes.io/name`: アプリケーション名
- `app.kubernetes.io/instance`: リリースインスタンス名

これらはServiceのセレクターやDeploymentのmatchLabelsで使用されます。

### 6. `testchart.serviceAccountName`

```yaml
{{- define "testchart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "testchart.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

**機能**: 使用するServiceAccount名を決定
- ServiceAccountを作成する場合: 指定された名前か、fullnameを使用
- 作成しない場合: 指定された名前か、"default"を使用

## 使用方法

他のテンプレートファイルから`include`関数で呼び出します：

```yaml
# deployment.yaml での使用例
metadata:
  name: {{ include "testchart.fullname" . }}
  labels:
    {{- include "testchart.labels" . | nindent 4 }}
```

## なぜ重要なのか

1. **一貫性**: すべてのリソースで同じ命名規則とラベルを使用
2. **保守性**: 命名ロジックを一箇所で管理
3. **柔軟性**: values.yamlで簡単にカスタマイズ可能
4. **Kubernetes準拠**: DNS命名規則や推奨ラベルに準拠

## ベストプラクティス

1. チャート固有のプレフィックスを使用（例: `testchart.`）
2. Kubernetesの命名制限を考慮（63文字制限など）
3. 推奨ラベル（`app.kubernetes.io/*`）を使用
4. コメントで各ヘルパーの目的を明記