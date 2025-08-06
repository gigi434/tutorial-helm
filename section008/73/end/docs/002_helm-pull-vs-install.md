# Helm Pull vs Install の違い

## 基本的な違い

| コマンド | 目的 | 動作 | 結果 |
|---------|------|------|------|
| `helm pull` | チャートのダウンロード | チャートパッケージを取得するだけ | ローカルにファイル保存 |
| `helm install` | アプリケーションのデプロイ | Kubernetesクラスターにリソース作成 | 実際のアプリケーションが動作 |

## helm pullとは

`helm pull`は、Helmチャートをローカルにダウンロードするだけのコマンドです。**Kubernetesクラスターには何もデプロイしません**。

### 基本的な使い方

```bash
# リポジトリからチャートをpull
helm pull bitnami/mysql

# 特定バージョンを指定してpull
helm pull bitnami/mysql --version 9.4.6

# 展開して保存
helm pull bitnami/mysql --untar

# カスタムディレクトリに保存
helm pull bitnami/mysql --destination ./charts/
```

### helm pullの結果

```bash
$ helm pull bitnami/mysql
$ ls -la
mysql-9.4.6.tgz  # チャートパッケージファイル

$ helm pull bitnami/mysql --untar
$ ls -la
mysql/  # 展開されたチャートディレクトリ
├── Chart.yaml
├── values.yaml
├── templates/
└── README.md
```

## helm installとは

`helm install`は、チャートを実際にKubernetesクラスターにデプロイし、アプリケーションを動作させるコマンドです。

### 基本的な使い方

```bash
# リポジトリから直接インストール
helm install my-mysql bitnami/mysql

# ローカルチャートからインストール
helm install my-mysql ./mysql

# values.yamlを指定してインストール
helm install my-mysql bitnami/mysql -f custom-values.yaml
```

### helm installの結果

```bash
$ helm install my-mysql bitnami/mysql
NAME: my-mysql
LAST DEPLOYED: Wed Aug  6 14:00:00 2025
NAMESPACE: default
STATUS: deployed

$ kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
my-mysql-0                 1/1     Running   0          2m

$ helm list
NAME     NAMESPACE  REVISION  UPDATED         STATUS    CHART         APP VERSION
my-mysql default   1         2025-08-06...   deployed  mysql-9.4.6   8.0.35
```

## 使用シナリオ別の違い

### 1. チャートの内容確認（helm pull使用）

```bash
# チャートの内容を確認したい場合
helm pull bitnami/mysql --untar

# values.yamlを確認
cat mysql/values.yaml

# テンプレートを確認
ls mysql/templates/

# カスタマイズしてから使用
cp mysql/values.yaml custom-values.yaml
# custom-values.yamlを編集

# 編集後にインストール
helm install my-mysql ./mysql -f custom-values.yaml
```

### 2. オフライン環境での使用

```bash
# オンライン環境でチャートをpull
helm pull bitnami/mysql
helm pull bitnami/postgresql
helm pull bitnami/redis

# オフライン環境に.tgzファイルを転送後
helm install my-mysql mysql-9.4.6.tgz
helm install my-postgres postgresql-12.1.2.tgz
```

### 3. CI/CDパイプラインでの使用

```bash
# パイプライン内でチャートを取得
helm pull myrepo/myapp --version $APP_VERSION

# チャートをカスタマイズ
helm template myapp myapp-$APP_VERSION.tgz -f production-values.yaml > manifests.yaml

# 検証後にデプロイ
helm install myapp-$BUILD_ID myapp-$APP_VERSION.tgz -f production-values.yaml
```

## 詳細な比較

### helm pull

**用途**:
- チャートの内容確認
- オフライン環境での使用
- チャートのカスタマイズ
- バックアップ目的
- CI/CDでの事前取得

**オプション**:
```bash
--version         # 特定バージョンを指定
--untar          # 展開して保存
--destination    # 保存先ディレクトリ
--verify         # GPG署名を検証
--repo           # リポジトリURL指定
--username       # 認証用ユーザー名
--password       # 認証用パスワード
```

**実行例**:
```bash
# 基本的なpull
helm pull stable/nginx-ingress

# バージョン指定でpull
helm pull stable/nginx-ingress --version 1.41.3

# 展開してpull
helm pull stable/nginx-ingress --untar --destination ./charts/

# プライベートリポジトリからpull
helm pull myrepo/private-chart --username user --password pass
```

### helm install

**用途**:
- アプリケーションの実際のデプロイ
- 本番環境での運用
- 開発・テスト環境構築

**オプション**:
```bash
--values/-f      # values.yamlファイル指定
--set            # 個別値の設定
--namespace      # デプロイ先namespace
--create-namespace # namespace自動作成
--wait           # リソース準備完了まで待機
--timeout        # タイムアウト設定
--dry-run        # 実際にはデプロイしない（確認用）
```

**実行例**:
```bash
# 基本的なインストール
helm install my-app bitnami/mysql

# カスタム設定でインストール
helm install my-app bitnami/mysql \
  --set auth.rootPassword=secret \
  --set primary.persistence.size=50Gi

# 別namespaceにインストール
helm install my-app bitnami/mysql \
  --namespace production \
  --create-namespace

# values.yamlを使用してインストール
helm install my-app bitnami/mysql -f production-values.yaml
```

## 実際のワークフロー例

### 開発フロー
```bash
# 1. チャートを取得して内容確認
helm pull bitnami/mysql --untar

# 2. values.yamlをカスタマイズ
cp mysql/values.yaml dev-values.yaml
# dev-values.yamlを編集

# 3. 開発環境にデプロイ
helm install dev-mysql ./mysql -f dev-values.yaml

# 4. 動作確認
kubectl get pods
kubectl logs dev-mysql-0

# 5. 本番用設定作成
cp dev-values.yaml prod-values.yaml
# prod-values.yamlを本番用に調整

# 6. 本番環境にデプロイ
helm install prod-mysql ./mysql -f prod-values.yaml --namespace production
```

### アップグレードフロー
```bash
# 1. 新しいバージョンをpull
helm pull bitnami/mysql --version 9.5.0 --untar

# 2. 既存の設定を確認
helm get values my-mysql > current-values.yaml

# 3. テンプレートで差分確認
helm template my-mysql ./mysql -f current-values.yaml > new-manifests.yaml
kubectl diff -f new-manifests.yaml

# 4. アップグレード実行
helm upgrade my-mysql ./mysql -f current-values.yaml
```

## まとめ

| 状況 | 推奨コマンド | 理由 |
|------|-------------|------|
| チャート内容を確認したい | `helm pull --untar` | ファイルを直接確認可能 |
| すぐにデプロイしたい | `helm install` | 最も効率的 |
| カスタマイズしてからデプロイ | `helm pull` → 編集 → `helm install` | 安全性が高い |
| オフライン環境 | `helm pull` → 転送 → `helm install` | 必須 |
| CI/CD | `helm pull` → 検証 → `helm install` | 確実性を重視 |

**重要**: `helm pull`はチャートを取得するだけ、`helm install`は実際にアプリケーションをデプロイします。目的に応じて適切に使い分けましょう。