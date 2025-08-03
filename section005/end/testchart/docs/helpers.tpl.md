# helpers.tpl ファイルについて

## 概要
`helpers.tpl` は Helm チャートで使用される再利用可能なテンプレート関数を定義するファイルです。このファイルは `templates/` ディレクトリに配置され、他のテンプレートファイルから参照できる共通のヘルパー関数を含みます。

## 主な用途

### 1. 名前の生成
チャート名やリリース名を組み合わせて、Kubernetes リソースの名前を一貫して生成します。

```yaml
{{- define "testchart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

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

### 2. ラベルの定義
共通のラベルセットを定義して、すべてのリソースで一貫したラベリングを保証します。

```yaml
{{- define "testchart.labels" -}}
helm.sh/chart: {{ include "testchart.chart" . }}
{{ include "testchart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "testchart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "testchart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### 3. チャート情報の生成
チャートのバージョン情報を含む文字列を生成します。

```yaml
{{- define "testchart.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}
```

## 使用方法

他のテンプレートファイルから `include` 関数を使用してヘルパー関数を呼び出します：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "testchart.fullname" . }}
  labels:
    {{- include "testchart.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "testchart.selectorLabels" . | nindent 6 }}
```

## ベストプラクティス

1. **命名規則**: ヘルパー関数名は `<chartname>.<function>` の形式で定義する
2. **再利用性**: 複数の場所で使用される値や構造は必ずヘルパー関数として定義する
3. **一貫性**: すべてのリソースで同じヘルパー関数を使用して一貫性を保つ
4. **ドキュメント**: 複雑なヘルパー関数にはコメントで説明を追加する

## 注意点

- `helpers.tpl` ファイル自体は Kubernetes マニフェストを生成しません
- `{{- define ... -}}` で定義された関数のみが含まれます
- ファイル名は慣例的に `_helpers.tpl` とすることが多い（アンダースコアで始まる）