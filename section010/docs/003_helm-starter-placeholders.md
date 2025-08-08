# Helm Starter プレースホルダーの仕組み

## プレースホルダーとは

`<CHARTNAME>`のような`<>`で囲まれた文字列は、Helm Starterの**自動置換マーカー**です。`helm create`コマンド実行時に、Helmが自動的に適切な値に置き換えます。

## 置換の仕組み

### 1. コマンド実行時の流れ

```bash
# コマンド実行
helm create mychart --starter mycompany-starter

# 置換プロセス
1. Starterテンプレートをコピー
2. プレースホルダーを検索
3. 実際の値に置換
4. 新しいチャートとして生成
```

### 2. 値の参照元

| プレースホルダー | 値の参照元 | 実際の値（例） |
|-----------------|------------|---------------|
| `<CHARTNAME>` | `helm create`の第1引数 | `mychart` |
| `<YEAR>` | システムの現在年 | `2024` |
| `<AUTHOR>` | Gitのuser.name設定 | `John Doe` |
| `<EMAIL>` | Gitのuser.email設定 | `john@example.com` |

## 実例：置換前後の比較

### Starterテンプレート（置換前）

```yaml
# ~/.local/share/helm/starters/mycompany-web/Chart.yaml
apiVersion: v2
name: <CHARTNAME>
description: A Helm chart for <CHARTNAME>
type: application
version: 0.1.0
appVersion: "1.0.0"
maintainers:
  - name: <AUTHOR>
    email: <EMAIL>
```

```yaml
# ~/.local/share/helm/starters/mycompany-web/values.yaml
image:
  repository: myregistry.io/<CHARTNAME>
  tag: latest

app:
  name: <CHARTNAME>
```

```yaml
# ~/.local/share/helm/starters/mycompany-web/templates/_helpers.tpl
{{- define "<CHARTNAME>.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "<CHARTNAME>.fullname" -}}
{{- printf "%s-%s" .Release.Name "<CHARTNAME>" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "<CHARTNAME>.labels" -}}
app.kubernetes.io/name: {{ include "<CHARTNAME>.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### 生成されたチャート（置換後）

```bash
# 実行コマンド
helm create myapp --starter mycompany-web
```

```yaml
# myapp/Chart.yaml （置換後）
apiVersion: v2
name: myapp                    # <CHARTNAME> → myapp
description: A Helm chart for myapp  # <CHARTNAME> → myapp
type: application
version: 0.1.0
appVersion: "1.0.0"
maintainers:
  - name: John Doe              # <AUTHOR> → Git設定から
    email: john@example.com     # <EMAIL> → Git設定から
```

```yaml
# myapp/values.yaml （置換後）
image:
  repository: myregistry.io/myapp  # <CHARTNAME> → myapp
  tag: latest

app:
  name: myapp                      # <CHARTNAME> → myapp
```

```yaml
# myapp/templates/_helpers.tpl （置換後）
{{- define "myapp.name" -}}        # <CHARTNAME> → myapp
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "myapp.fullname" -}}    # <CHARTNAME> → myapp
{{- printf "%s-%s" .Release.Name "myapp" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "myapp.labels" -}}      # <CHARTNAME> → myapp
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

## Git設定の確認と設定

### 現在の設定確認

```bash
# Gitの設定を確認
git config --global user.name
git config --global user.email

# 出力例
John Doe
john@example.com
```

### 設定がない場合

```bash
# Git設定を追加
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# これらの値が<AUTHOR>と<EMAIL>に使用される
```

## カスタムプレースホルダーは作れない

**重要**: Helmがサポートするプレースホルダーは以下の4つのみです：

```yaml
# ✅ Helmがサポートする標準プレースホルダー
<CHARTNAME>   # チャート名
<YEAR>        # 現在の年
<AUTHOR>      # Git設定の作成者名
<EMAIL>       # Git設定のメールアドレス

# ❌ これらは置換されない（カスタムプレースホルダーは不可）
<COMPANY>     # 置換されない
<TEAM>        # 置換されない
<VERSION>     # 置換されない
```

## 実践例：複雑なStarterでの使用

### deployment.yaml テンプレート

```yaml
# Starter内のファイル
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "<CHARTNAME>.fullname" . }}
  labels:
    {{- include "<CHARTNAME>.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "<CHARTNAME>.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "<CHARTNAME>.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: <CHARTNAME>
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### 置換後（helm create webapp実行後）

```yaml
# webapp/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "webapp.fullname" . }}      # <CHARTNAME> → webapp
  labels:
    {{- include "webapp.labels" . | nindent 4 }}  # <CHARTNAME> → webapp
spec:
  selector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}  # <CHARTNAME> → webapp
  template:
    metadata:
      labels:
        {{- include "webapp.selectorLabels" . | nindent 8 }}  # <CHARTNAME> → webapp
    spec:
      containers:
      - name: webapp                            # <CHARTNAME> → webapp
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

## 動作確認の方法

### 1. Starterの作成とテスト

```bash
# Starterディレクトリの作成
mkdir -p ~/.local/share/helm/starters/test-starter

# Chart.yamlの作成
cat > ~/.local/share/helm/starters/test-starter/Chart.yaml <<EOF
apiVersion: v2
name: <CHARTNAME>
description: Created by <AUTHOR> (<EMAIL>) in <YEAR>
version: 0.1.0
EOF

# テスト実行
helm create testapp --starter test-starter

# 結果確認
cat testapp/Chart.yaml
# 出力例:
# apiVersion: v2
# name: testapp
# description: Created by John Doe (john@example.com) in 2024
# version: 0.1.0
```

### 2. デバッグ方法

```bash
# プレースホルダーが置換されない場合の確認

# 1. プレースホルダーの綴りを確認
grep -r "<CHARTNAME>" ~/.local/share/helm/starters/my-starter/

# 2. Git設定を確認（<AUTHOR>と<EMAIL>用）
git config --list | grep user

# 3. Helmのバージョン確認（古いバージョンはサポートが限定的）
helm version
```

## トラブルシューティング

### 問題1: プレースホルダーが置換されない

```yaml
# ❌ 間違った書き方
name: ${CHARTNAME}      # 置換されない
name: {{CHARTNAME}}     # 置換されない
name: $CHARTNAME        # 置換されない
name: <chartname>       # 大文字小文字が違う

# ✅ 正しい書き方
name: <CHARTNAME>       # 正確に大文字で
```

### 問題2: Git設定が反映されない

```bash
# グローバル設定の確認
git config --global user.name
git config --global user.email

# ローカル設定も確認（ローカルが優先される場合がある）
git config --local user.name
git config --local user.email

# 設定がない場合は追加
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

### 問題3: 部分的にしか置換されない

```yaml
# Starterファイル内で一貫性を保つ
# すべて同じ形式で記述する

# ✅ 一貫性のある記述
{{- define "<CHARTNAME>.name" -}}
{{- include "<CHARTNAME>.labels" . }}
app: <CHARTNAME>

# ❌ 混在した記述（一部が置換されない可能性）
{{- define "<CHARTNAME>.name" -}}
{{- include "<chartname>.labels" . }}  # 小文字が混在
app: ${CHARTNAME}  # 違う形式
```

## まとめ

- `<CHARTNAME>`などは**Helmの標準プレースホルダー**
- `helm create`実行時に**自動的に置換**される
- 値は**コマンド引数やシステム情報から取得**
- カスタムプレースホルダーは作成できない
- 正確な記述（`<>`と大文字）が必要