# Helm テンプレート構文の説明

このドキュメントでは、deployment.yamlで使用されているHelmテンプレート構文について説明します。

## 基本的なテンプレート構文

### 1. `include`関数
`include`は、定義済みのテンプレート（主に_helpers.tplで定義）を呼び出して、その結果を現在のテンプレートに挿入する関数です。

#### 基本構文
```yaml
{{ include "テンプレート名" コンテキスト }}
```

#### 具体例と説明
```yaml
{{ include "testchart.fullname" . }}
```
- `include`: テンプレートを呼び出す関数
- `"testchart.fullname"`: _helpers.tplで定義されたテンプレート名
- `.`: 現在のコンテキスト（ルートコンテキスト）を渡す

#### `include`と`template`の違い
- **`include`**: 結果を文字列として返し、パイプで他の関数に渡せる
- **`template`**: 結果を直接出力し、パイプが使えない

#### `include`の使用例

1. **基本的な使用**:
   ```yaml
   name: {{ include "testchart.fullname" . }}
   ```

2. **パイプとの組み合わせ**:
   ```yaml
   labels:
     {{- include "testchart.labels" . | nindent 4 }}
   ```

3. **ネストした呼び出し**:
   ```yaml
   {{- include "testchart.selectorLabels" . | nindent 6 }}
   ```
   これは_helpers.tplで定義された`testchart.selectorLabels`テンプレートを呼び出し、結果を6スペースでインデントします。

#### _helpers.tplでの定義例
```yaml
{{- define "testchart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "testchart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

このテンプレートが`include`で呼ばれると：
1. `testchart.name`テンプレートを呼び出す
2. リリース名を追加
3. 2つのラベルを返す

### 2. `with`ブロック
`with`は、指定した値が存在する場合のみブロック内のコードを実行し、コンテキストを変更する制御構造です。

#### 基本構文
```yaml
{{- with .Values.someValue }}
  # ブロック内では . が .Values.someValue を指す
{{- end }}
```

#### deployment.yamlでの使用例

1. **podAnnotationsの条件付き追加**:
   ```yaml
   {{- with .Values.podAnnotations }}
   annotations:
     {{- toYaml . | nindent 8 }}
   {{- end }}
   ```
   - `.Values.podAnnotations`が存在する場合のみannotationsセクションを作成
   - ブロック内の`.`は`.Values.podAnnotations`の値を指す

2. **ネストしたwithの使用**:
   ```yaml
   {{- with .Values.podLabels }}
   {{- toYaml . | nindent 8 }}
   {{- end }}
   ```

#### `with`の利点
- nilや空の値の場合、ブロック全体がスキップされる
- YAMLの構造を保ちながら、オプション項目を条件付きで追加できる
- コンテキストが変わるため、深い階層の値へのアクセスが簡潔になる

### 3. `toYaml`関数
`toYaml`は、Go言語のデータ構造をYAML形式の文字列に変換する関数です。

#### 基本構文
```yaml
{{- toYaml .someValue }}
```

#### deployment.yamlでの使用例
```yaml
{{- with .Values.resources }}
resources:
  {{- toYaml . | nindent 12 }}
{{- end }}
```

#### 動作例
values.yamlに以下がある場合：
```yaml
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

生成されるYAML：
```yaml
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

### 4. `nindent`関数
`nindent`は、新しい行を追加してから指定されたスペース数でインデントする関数です。

#### 基本構文
```yaml
{{- someContent | nindent N }}
```

#### `indent`との違い
- `indent`: 最初の行からインデント
- `nindent`: 新しい行を追加してからインデント

#### deployment.yamlでの使用例
```yaml
labels:
  {{- include "testchart.labels" . | nindent 4 }}
```

### 5. `default`関数
`default`は、値が空の場合にデフォルト値を返す関数です。

#### 基本構文
```yaml
{{ .Values.someValue | default "defaultValue" }}
```

#### deployment.yamlでの使用例
```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```
- `.Values.image.tag`が空の場合、`.Chart.AppVersion`を使用

### 6. `upper`関数
`upper`は、文字列を大文字に変換する関数です。

#### deployment.yamlでの使用例
```yaml
{{ .Values.my.custom.values | default "test-my-data" | upper | quote }}
```
このパイプラインでは：
1. `.Values.my.custom.values`を取得
2. 空の場合は`"test-my-data"`を使用
3. 大文字に変換
4. 引用符で囲む

### 7. `quote`関数
`quote`は、値を二重引用符で囲む関数です。

#### 使用場面
- 文字列値を確実に引用符で囲みたい場合
- 数値や真偽値を文字列として扱いたい場合

### 8. `if`条件分岐
`if`は、条件に基づいてコンテンツの表示/非表示を制御します。

#### deployment.yamlでの使用例
```yaml
{{- if not .Values.autoscaling.enabled }}
replicas: {{ .Values.replicaCount }}
{{- end }}
```
- `not`で否定条件
- autoscalingが無効の場合のみreplicasを設定

#### なぜautoscaling有効時にreplicasを設定しないのか？

これはKubernetesのHorizontal Pod Autoscaler (HPA)の仕様に基づいた設計です。

**HPAとDeploymentの競合問題**:
1. **制御の競合**: HPAが有効な場合、HPAがPod数を直接管理します。DeploymentとHPAが同時にPod数を制御すると、意図しない動作が発生します
2. **設定の上書き**: Deploymentを更新するたびに、HPAが設定したPod数がDeploymentの`replicas`値に戻されてしまいます

**正しい設計パターン**:
- `autoscaling.enabled: false` → Deploymentで`replicas`を設定（手動スケーリング）
- `autoscaling.enabled: true` → HPAリソースを作成し、min/maxレプリカ数を設定（自動スケーリング）

この二者択一の設計により、設定の競合を防ぎ、意図通りの動作を保証しています。

### 9. `{{-` と `-}}`（空白制御）
テンプレートの前後の空白を制御します。

#### 空白制御の基本ルール
- `{{-`: 左側（前）の空白・改行を削除
- `-}}`: 右側（後）の空白・改行を削除
- `{{`: 左側の空白・改行をそのまま保持
- `}}`: 右側の空白・改行をそのまま保持

#### なぜ使い分けるのか？

**1. インライン値の場合（`-`を使わない）**
```yaml
serviceAccountName: {{ include "testchart.serviceAccountName" . }}
```
- 値を直接挿入する場合は`-`不要
- YAMLの`key: value`形式を保つため

**2. ブロック制御構文の場合（`{{-`を使う）**
```yaml
{{- with .Values.podSecurityContext }}
securityContext:
  {{- toYaml . | nindent 8 }}
{{- end }}
```
- `with`や`if`などの制御構文は余分な改行を生成する
- `{{-`で前の改行を削除し、YAMLの構造を保つ

#### 具体例での比較

**`-`を使わない場合（問題のある出力）**:
```yaml
# テンプレート
{{ with .Values.podSecurityContext }}
securityContext:
  {{ toYaml . | nindent 8 }}
{{ end }}

# 生成される出力（余分な空行が入る）

securityContext:
  runAsUser: 1000
  runAsGroup: 3000

```

**`{{-`を使う場合（正しい出力）**:
```yaml
# テンプレート
{{- with .Values.podSecurityContext }}
securityContext:
  {{- toYaml . | nindent 8 }}
{{- end }}

# 生成される出力（きれいなYAML）
securityContext:
  runAsUser: 1000
  runAsGroup: 3000
```

#### deployment.yamlでのパターン

1. **値の直接挿入** - `{{`を使用
   ```yaml
   name: {{ .Chart.Name }}
   containerPort: {{ .Values.service.port }}
   ```

2. **条件付きブロック** - `{{-`を使用
   ```yaml
   {{- if not .Values.autoscaling.enabled }}
   replicas: {{ .Values.replicaCount }}
   {{- end }}
   ```

3. **with文での値挿入** - 内部で`{{-`を使用
   ```yaml
   {{- with .Values.resources }}
   resources:
     {{- toYaml . | nindent 12 }}
   {{- end }}
   ```

#### ベストプラクティス
- **インライン値**: 通常は`{{`と`}}`を使用
- **制御構文**: `{{-`と`-}}`を使用して余分な空白を削除
- **nindentと組み合わせ**: `{{-`を使って適切なインデントを保証
- **デバッグ時**: まず`-`なしで試し、出力を確認してから調整

### 10. パイプ演算子 `|`
複数の関数を連鎖させて処理を行います。

#### deployment.yamlでの使用例
```yaml
{{ .Values.my.custom.values | default "test-my-data" | upper | quote }}
```
処理の流れ：
1. 値を取得
2. デフォルト値を適用
3. 大文字に変換
4. 引用符で囲む

## deployment.yamlで使用されている全テンプレート構文のまとめ

### 基本的な値の参照
- `{{ .Values.replicaCount }}`: values.yamlの値を直接参照
- `{{ .Chart.Name }}`: Chart.yamlの値を参照
- `{{ .Chart.Version }}`: チャートのバージョンを参照
- `{{ .Values.service.port }}`: ネストした値の参照

### 関数の使用例
- `{{ include "testchart.fullname" . }}`: テンプレートを呼び出す
- `{{ .Values.image.tag | default .Chart.AppVersion }}`: デフォルト値の設定
- `{{ .Values.my.custom.values | default "test-my-data" | upper | quote }}`: 複数関数のチェーン

### 制御構造
- `{{- if not .Values.autoscaling.enabled }}...{{- end }}`: 条件分岐
- `{{- with .Values.podAnnotations }}...{{- end }}`: コンテキスト変更と条件実行

### フォーマット関数
- `{{- toYaml . | nindent 8 }}`: YAML変換とインデント
- `{{- include "testchart.labels" . | nindent 4 }}`: テンプレート結果のインデント

### 文字列操作
- `"{{ .Values.image.repository }}:{{ .Values.image.tag }}"`: 文字列の連結
- `{{- "helm templating is" -}} , {{- "Cool" -}}`: 空白制御付き文字列

---

## 以前の説明セクション（参考用）

### 以前の番号付きセクション

#### 3. `{{- if not .Values.autoscaling.enabled }}`
- **説明**: 条件分岐でコンテンツの表示/非表示を制御
- **構成要素**:
  - `if`: 条件分岐の開始
  - `not`: 否定演算子
  - `.Values.autoscaling.enabled`: values.yamlの値を参照
- **使用例**: autoscalingが無効の場合のみreplicasを設定

### 4. `{{ .Values.replicaCount }}`
- **説明**: values.yamlから値を直接挿入
- **パス**: `.Values`はvalues.yamlのルートを指す

### 5. `{{- end }}`
- **説明**: 制御構造（if、with、range）の終了タグ

### 6. `{{- with .Values.podAnnotations }}`
- **説明**: コンテキストを変更してブロックを実行
- **動作**: 
  - 値が存在する場合のみブロック内を実行
  - ブロック内では`.`が`.Values.podAnnotations`を指す

### 7. `{{- toYaml . | nindent 8 }}`
- **説明**: 値をYAML形式に変換してインデント
- **構成要素**:
  - `toYaml`: オブジェクトをYAML形式に変換
  - `.`: 現在のコンテキスト（withブロック内の値）
  - `nindent 8`: 8スペースでインデント

### 8. `{{- .Chart.Version -}}`
- **説明**: Chart.yamlから情報を参照
- **構成要素**:
  - `.Chart`: Chart.yamlの内容を参照する組み込みオブジェクト
  - `.Version`: チャートのバージョン
  - `{{-` と `-}}`: 前後の空白を削除
- **その他のChartフィールド**:
  - `.Chart.Name`: チャート名
  - `.Chart.Description`: チャートの説明
  - `.Chart.AppVersion`: アプリケーションのバージョン
  - `.Chart.Home`: プロジェクトのホームページ
  - `.Chart.Keywords`: キーワードのリスト
  - `.Chart.Maintainers`: メンテナーのリスト

## 組み込みオブジェクト

Helmには以下の組み込みオブジェクトがあります：

### 1. `.Values`
- **説明**: values.yamlとコマンドラインで指定された値
- **例**: `.Values.replicaCount`, `.Values.image.tag`

### 2. `.Chart`
- **説明**: Chart.yamlの内容
- **例**: `.Chart.Name`, `.Chart.Version`

### 3. `.Release`
- **説明**: Helmリリースに関する情報を提供する組み込みオブジェクト
- **主なフィールド**:
  - `.Release.Name`: リリース名（helm install/upgradeで指定した名前）
  - `.Release.Namespace`: リリース先のKubernetes namespace
  - `.Release.IsUpgrade`: アップグレード操作の場合true
  - `.Release.IsInstall`: 新規インストール操作の場合true
  - `.Release.Revision`: リビジョン番号（アップグレードごとに増加）
  - `.Release.Service`: リリースを管理しているサービス（通常は"Helm"）

#### `.Release`の使用例

1. **リリース名を使った動的な命名**:
   ```yaml
   metadata:
     name: {{ .Release.Name }}-configmap
   ```

2. **namespace固有のラベル付け**:
   ```yaml
   labels:
     release: {{ .Release.Name }}
     namespace: {{ .Release.Namespace }}
   ```

3. **条件付きリソース作成**:
   ```yaml
   {{- if .Release.IsInstall }}
   # 初回インストール時のみ作成されるリソース
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: {{ .Release.Name }}-initial-config
   data:
     message: "Welcome to {{ .Release.Name }}!"
   {{- end }}
   ```

4. **アップグレード時の特別な処理**:
   ```yaml
   {{- if .Release.IsUpgrade }}
   annotations:
     helm.sh/hook: pre-upgrade
     revision: "{{ .Release.Revision }}"
   {{- end }}
   ```

5. **リビジョン情報の埋め込み**:
   ```yaml
   metadata:
     annotations:
       deployment.kubernetes.io/revision: "{{ .Release.Revision }}"
   ```

#### `.Release`の実用的な使用シナリオ

1. **マルチテナント環境での分離**:
   - `.Release.Name`を使って同じチャートから複数のインスタンスをデプロイ
   - 各インスタンスが独立したリソース名を持つ

2. **環境別の設定**:
   - namespace名から環境を判断して設定を変更
   ```yaml
   {{- if eq .Release.Namespace "production" }}
   replicas: 5
   {{- else }}
   replicas: 2
   {{- end }}
   ```

3. **デバッグ情報の追加**:
   ```yaml
   annotations:
     helm.sh/release-name: {{ .Release.Name }}
     helm.sh/release-namespace: {{ .Release.Namespace }}
     helm.sh/revision: "{{ .Release.Revision }}"
   ```

### 4. `.Files`
- **説明**: チャート内のファイルへのアクセス
- **メソッド**:
  - `.Files.Get`: ファイルの内容を取得
  - `.Files.GetBytes`: バイト配列として取得
  - `.Files.Glob`: パターンマッチでファイルを取得
  - `.Files.AsConfig`: ConfigMapデータとして取得
  - `.Files.AsSecrets`: Secretデータとして取得

### 5. `.Capabilities`
- **説明**: Kubernetesクラスターの機能情報
- **主なフィールド**:
  - `.Capabilities.APIVersions`: 利用可能なAPIバージョン
  - `.Capabilities.KubeVersion`: Kubernetesのバージョン情報
  - `.Capabilities.KubeVersion.Major`: メジャーバージョン
  - `.Capabilities.KubeVersion.Minor`: マイナーバージョン

### 6. `.Template`
- **説明**: 現在処理中のテンプレートファイルに関する情報を提供する組み込みオブジェクト
- **主なフィールド**:
  - `.Template.Name`: テンプレートファイルの完全パス（例: `testchart/templates/deployment.yaml`）
  - `.Template.BasePath`: テンプレートのベースパス（例: `testchart/templates`）

#### `.Template`の使用例

1. **デバッグ用のテンプレート名表示**:
   ```yaml
   metadata:
     annotations:
       generated-from: {{ .Template.Name }}
   ```

2. **条件付き処理（特定のテンプレートでのみ実行）**:
   ```yaml
   {{- if eq .Template.Name "testchart/templates/deployment.yaml" }}
   # deployment.yaml専用の処理
   spec:
     replicas: {{ .Values.specialReplicaCount | default 3 }}
   {{- end }}
   ```

3. **テンプレートファイル名に基づく動的な命名**:
   ```yaml
   {{- $templateName := .Template.Name | base | trimSuffix ".yaml" }}
   metadata:
     name: {{ include "testchart.fullname" . }}-{{ $templateName }}
   ```

4. **開発環境でのトラブルシューティング**:
   ```yaml
   {{- if .Values.debug }}
   # デバッグ情報の追加
   metadata:
     annotations:
       debug/template-name: {{ .Template.Name }}
       debug/template-base: {{ .Template.BasePath }}
   {{- end }}
   ```

5. **テンプレートパスに基づく条件分岐**:
   ```yaml
   {{- if contains "cronjob" .Template.Name }}
   # CronJob用の特別な設定
   successfulJobsHistoryLimit: 3
   failedJobsHistoryLimit: 1
   {{- end }}
   ```

#### `.Template`の実用的な使用シナリオ

1. **自動生成されたマニフェストの追跡**:
   - どのテンプレートファイルから生成されたかを記録
   - 問題が発生した際のデバッグを容易に

2. **テンプレートファイル名による動的な振る舞い**:
   - ファイル名に基づいて異なる設定を適用
   - 例: `deployment-frontend.yaml`と`deployment-backend.yaml`で異なる設定

3. **ヘルパーテンプレートでの使用**:
   ```yaml
   {{- define "testchart.resourceName" -}}
   {{- $name := include "testchart.fullname" . -}}
   {{- $template := .Template.Name | base | trimSuffix ".yaml" -}}
   {{ printf "%s-%s" $name $template }}
   {{- end }}
   ```

#### 注意事項

- `.Template`オブジェクトは、`_helpers.tpl`などの部分テンプレート内では使用できません
- `include`や`template`で呼び出されたテンプレート内では、呼び出し元のテンプレート情報が保持されます
- テンプレート名はチャートのルートディレクトリからの相対パスで表されます

## テンプレート関数の種類

### 文字列関数
- `quote`: 値を引用符で囲む
- `upper`: 大文字に変換
- `lower`: 小文字に変換
- `trim`: 前後の空白を削除

### 論理関数
- `and`: 論理AND
- `or`: 論理OR
- `not`: 否定
- `eq`: 等価比較

### デフォルト値関数
- `default`: 値が空の場合のデフォルト値を設定
  ```yaml
  {{ .Values.image.tag | default .Chart.AppVersion }}
  ```

### リスト/辞書関数
- `list`: リストを作成
- `dict`: 辞書を作成
- `hasKey`: キーの存在確認

## 空白制御

- `{{-`: 左側の空白を削除
- `-}}`: 右側の空白を削除
- 例: `{{- "helm templating is" -}} , {{- "Cool" -}}`
  - 結果: `helm templating is,Cool`（空白が削除される）

## ベストプラクティス

1. **インデントの一貫性**: `nindent`を使用して適切なYAML構造を維持
2. **条件分岐の使用**: オプション機能には`if`や`with`を使用
3. **ヘルパーテンプレート**: 再利用可能なロジックは_helpers.tplに定義
4. **デフォルト値**: `default`関数で適切なフォールバック値を設定
5. **空白制御**: `-`を使用して不要な空白や改行を削除

## 実例での使用パターン

deployment.yamlでは以下のパターンが使用されています：

1. **メタデータの設定**:
   ```yaml
   metadata:
     name: {{ include "testchart.fullname" . }}
     labels:
       {{- include "testchart.labels" . | nindent 4 }}
   ```

2. **条件付きリソース設定**:
   ```yaml
   {{- if not .Values.autoscaling.enabled }}
   replicas: {{ .Values.replicaCount }}
   {{- end }}
   ```

3. **オプション設定の適用**:
   ```yaml
   {{- with .Values.podAnnotations }}
   annotations:
     {{- toYaml . | nindent 8 }}
   {{- end }}
   ```

これらのテンプレート構文により、values.yamlの設定に基づいて動的にKubernetesマニフェストを生成できます。