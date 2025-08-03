# Helm テンプレート構文の説明

このドキュメントでは、deployment.yamlで使用されているHelmテンプレート構文について説明します。

## 基本的なテンプレート構文

### 1. `{{ include "testchart.fullname" . }}`
- **説明**: テンプレート関数を呼び出して値を挿入します
- **構成要素**:
  - `{{ }}`: テンプレートアクションの区切り文字
  - `include`: 定義済みのテンプレートを呼び出す関数
  - `"testchart.fullname"`: _helpers.tplで定義されたテンプレート名
  - `.`: 現在のコンテキスト（ルートコンテキスト）を渡す

### 2. `{{- include "testchart.labels" . | nindent 4 }}`
- **説明**: テンプレートを呼び出し、出力をインデントします
- **構成要素**:
  - `{{-`: 左側の空白を削除するテンプレート開始
  - `include "testchart.labels" .`: ラベルテンプレートを呼び出す
  - `|`: パイプ演算子（出力を次の関数に渡す）
  - `nindent 4`: 新しい行を追加して4スペースでインデント

### 3. `{{- if not .Values.autoscaling.enabled }}`
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
- **説明**: リリース情報
- **主なフィールド**:
  - `.Release.Name`: リリース名
  - `.Release.Namespace`: リリース先のnamespace
  - `.Release.IsUpgrade`: アップグレード操作の場合true
  - `.Release.IsInstall`: インストール操作の場合true
  - `.Release.Revision`: リビジョン番号
  - `.Release.Service`: リリースを管理しているサービス（通常は"Helm"）

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
- **説明**: 現在実行中のテンプレート情報
- **主なフィールド**:
  - `.Template.Name`: テンプレートファイル名
  - `.Template.BasePath`: テンプレートのベースパス

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