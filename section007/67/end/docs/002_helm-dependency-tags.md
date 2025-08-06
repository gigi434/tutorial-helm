# Helm依存関係のtagsによる条件制御

## 概要
Helmの`tags`機能を使うと、複数の依存関係を一括で有効/無効にできます。これは、関連する複数のチャートをグループとして管理したい場合に便利です。

## tagsの基本的な使い方

### 1. Chart.yamlでの定義
依存関係に`tags`フィールドを追加してタグを指定します：

```yaml
dependencies:
  - name: mysql
    version: ^14.0.0
    repository: https://charts.bitnami.com/bitnami
    tags:
      - enabled
  - name: apache
    version: ^12.0.0
    repository: https://charts.bitnami.com/bitnami
    tags:
      - enabled
```

### 2. values.yamlでの制御
`tags`セクションでタグの有効/無効を設定します：

```yaml
tags:
  enabled: false  # false = mysql と apache 両方が無効になる
```

## conditionとtagsの違い

### condition（個別制御）
各依存関係を個別に制御する場合：

```yaml
# Chart.yaml
dependencies:
  - name: mysql
    version: ^14.0.0
    repository: https://charts.bitnami.com/bitnami
    condition: mysql.enabled  # mysql.enabled の値で制御

# values.yaml
mysql:
  enabled: true  # mysqlのみ有効
apache:
  enabled: false # apacheは無効
```

### tags（グループ制御）
複数の依存関係をグループとして制御する場合：

```yaml
# Chart.yaml
dependencies:
  - name: mysql
    version: ^14.0.0
    repository: https://charts.bitnami.com/bitnami
    tags:
      - database
  - name: postgresql
    version: ^12.0.0
    repository: https://charts.bitnami.com/bitnami
    tags:
      - database

# values.yaml
tags:
  database: false  # database タグを持つ全ての依存関係が無効
```

## 実践的な使用例

### 例1: 環境別の制御
開発環境と本番環境で異なるコンポーネントを使用：

```yaml
# Chart.yaml
dependencies:
  - name: mysql
    version: ^14.0.0
    repository: "@bitnami"
    tags:
      - production
      - database
  - name: postgresql
    version: ^12.0.0
    repository: "@bitnami"
    tags:
      - development
      - database
  - name: redis
    version: ^17.0.0
    repository: "@bitnami"
    tags:
      - cache
      - production
      - development

# values-dev.yaml
tags:
  development: true
  production: false
  cache: true

# values-prod.yaml
tags:
  development: false
  production: true
  cache: true
```

### 例2: 機能別のグループ化
機能単位でサービスをグループ化：

```yaml
# Chart.yaml
dependencies:
  - name: mysql
    version: ^14.0.0
    repository: "@bitnami"
    tags:
      - backend
      - database
  - name: postgresql
    version: ^12.0.0
    repository: "@bitnami"
    tags:
      - backend
      - database
  - name: nginx
    version: ^14.0.0
    repository: "@bitnami"
    tags:
      - frontend
      - web
  - name: apache
    version: ^12.0.0
    repository: "@bitnami"
    tags:
      - frontend
      - web

# values.yaml
tags:
  backend: true    # バックエンド関連を有効
  frontend: false  # フロントエンド関連を無効
  database: true   # データベース関連を有効
  web: false      # Webサーバー関連を無効
```

## 複数タグの動作

依存関係が複数のタグを持つ場合、**いずれか一つでも**`true`なら有効になります：

```yaml
# Chart.yaml
dependencies:
  - name: redis
    tags:
      - cache     # false
      - backend   # true
      # → redisは有効になる（OR条件）

# values.yaml
tags:
  cache: false
  backend: true
```

## conditionとtagsの併用

`condition`と`tags`を両方指定した場合、`condition`が優先されます：

```yaml
# Chart.yaml
dependencies:
  - name: mysql
    condition: mysql.enabled  # これが優先
    tags:
      - database

# values.yaml
mysql:
  enabled: false  # conditionでfalse
tags:
  database: true  # tagsでtrue
# → mysqlは無効（conditionが優先）
```

## ベストプラクティス

1. **論理的なグループ化**: 関連するサービスには同じタグを付ける
2. **環境別タグ**: `development`, `staging`, `production`など
3. **機能別タグ**: `database`, `cache`, `monitoring`, `logging`など
4. **タグの命名規則**: わかりやすく一貫性のある名前を使用

## まとめ

- `tags`は複数の依存関係を一括制御するための機能
- `condition`は個別制御、`tags`はグループ制御
- 複数タグはOR条件で動作
- `condition`と`tags`を併用した場合は`condition`が優先

Helmチャートの tags（タグ）を使う具体的なシチュエーションは、「環境や条件によって、一部のKubernetesリソースやサブチャートだけをデプロイしたい／したくない場合」に役立ちます。
以下に典型的な利用例を示します。

1. 開発・本番環境で違う構成を簡単に切り替えたいとき
例：開発環境ではダミーデータ投入バッチを動かしたいが、本番では不要

チャート内に job.yaml（ダミーデータJob）を用意し、{{- if .Values.tags.withSampleData }} で条件分岐。

values-dev.yaml では tags: { withSampleData: true }、values-prod.yaml では tags: { withSampleData: false }。

インストール時に --values values-dev.yaml で開発用ジョブを有効化、本番では省略。

2. サブコンポーネント（例：MySQL、Redisなど）を任意で組み合わせ可能にしたい
例：メインアプリだけでなく、任意のDBやキャッシュを同時にデプロイしたい／不要なとき外したい

mysql.enabled、redis.enabled のような値とtags.database（まとめタグ）を併用し、

依存チャートに condition や tags を指定。

helm install myapp --set tags.database=true でDB/Redisも一括有効化できる。

3. テスト用リソースだけを一括で有効・無効にしたい
例：インテグレーションTesting用リソース（テスト用PodやService）を本番では絶対無効化したい

tags: { test: true } のような値を使い、if .Values.tags.test でテスト用リソースの生成を切り替える。

4. サブチャートが増えた場合の管理がシンプルになる
多くのサブチャートを含むシステムで、タグ一つでグルーピングしてデプロイ内容をコントロールできるため、

「APIのみ」「API＋バッチ」「全サービス」など切り替えが容易。

まとめ
tagsはテンプレート内の条件分岐を柔軟にし、チャート利用者が「必要なときだけリソースを追加、除外できる」仕組みです。
例：「--set tags.debug=true」でデバッグ用の全部署リソースを即時追加、本番では外す…などDevOps現場でよく使われます。

さらに踏み込んだシナリオや、具体的なYAMLサンプルが必要でしたら追加でお伝えできます。