# testchart

Kubernetes上でシンプルなWebアプリケーションをデプロイするためのHelm chartです。

## 概要

このchartは、設定可能なリソース、スケーリング、Ingressオプションを備えた基本的なWebサーバー（nginx）をデプロイします。Helm chart開発の学習例として設計されており、より複雑なアプリケーションを作成するためのテンプレートとして使用できます。

## 前提条件

- Kubernetes 1.19+
- Helm 3.0+
- 基盤となるインフラストラクチャでのPVプロビジョナーサポート（永続化が必要な場合）

## インストール

### クイックスタート

```bash
# デフォルト設定でインストール
helm install my-release ./testchart

# カスタム値でインストール
helm install my-release ./testchart --set service.type=NodePort
```

### リポジトリからのインストール

```bash
# リポジトリを追加（公開されている場合）
helm repo add myrepo https://example.com/charts
helm repo update

# chartをインストール
helm install my-release myrepo/testchart
```

## アンインストール

```bash
helm uninstall my-release
```

## 設定

以下の表は、testchartの設定可能なパラメータとそのデフォルト値を示しています。

| パラメータ | 説明 | デフォルト値 |
|-----------|------|-------------|
| `replicaCount` | レプリカ数 | `1` |
| `image.repository` | コンテナイメージリポジトリ | `nginx` |
| `image.pullPolicy` | コンテナイメージのプルポリシー | `IfNotPresent` |
| `image.tag` | コンテナイメージタグ（chartのappVersionを上書き） | `""` |
| `imagePullSecrets` | イメージプルシークレット | `[]` |
| `nameOverride` | chart名の上書き | `""` |
| `fullnameOverride` | フルネームの上書き | `""` |
| `serviceAccount.create` | サービスアカウントの作成 | `true` |
| `serviceAccount.automount` | サービスアカウントトークンの自動マウント | `true` |
| `serviceAccount.annotations` | サービスアカウントのアノテーション | `{}` |
| `serviceAccount.name` | サービスアカウント名 | `""` |
| `podAnnotations` | Podアノテーション | `{}` |
| `podLabels` | Podラベル | `{}` |
| `podSecurityContext` | Podセキュリティコンテキスト | `{}` |
| `securityContext` | コンテナセキュリティコンテキスト | `{}` |
| `service.type` | サービスタイプ | `ClusterIP` |
| `service.port` | サービスポート | `80` |
| `ingress.enabled` | Ingressを有効化 | `false` |
| `ingress.className` | Ingressクラス名 | `""` |
| `ingress.annotations` | Ingressアノテーション | `{}` |
| `ingress.hosts` | Ingressホスト設定 | values.yaml参照 |
| `ingress.tls` | Ingress TLS設定 | `[]` |
| `resources` | リソースの制限とリクエスト | `{}` |
| `livenessProbe` | Livenessプローブ設定 | values.yaml参照 |
| `readinessProbe` | Readinessプローブ設定 | values.yaml参照 |
| `autoscaling.enabled` | Horizontal Pod Autoscalerを有効化 | `false` |
| `autoscaling.minReplicas` | 最小レプリカ数 | `1` |
| `autoscaling.maxReplicas` | 最大レプリカ数 | `100` |
| `autoscaling.targetCPUUtilizationPercentage` | 目標CPU使用率 | `80` |
| `volumes` | 追加ボリューム | `[]` |
| `volumeMounts` | 追加ボリュームマウント | `[]` |
| `nodeSelector` | ノードセレクター | `{}` |
| `tolerations` | トレラレーション | `[]` |
| `affinity` | アフィニティルール | `{}` |

### 値の指定

`helm install`の引数として`--set key=value[,key=value]`を使用して各パラメータを指定します。例：

```bash
helm install my-release ./testchart \
  --set replicaCount=2 \
  --set service.type=NodePort
```

または、パラメータ値を指定したYAMLファイルを使用してchartをインストールすることもできます：

```bash
helm install my-release ./testchart -f values.yaml
```

## 使用例

### NodePort経由での公開

```bash
helm install my-release ./testchart --set service.type=NodePort
```

### Ingressの有効化

```bash
helm install my-release ./testchart \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=example.local \
  --set ingress.hosts[0].paths[0].path=/ \
  --set ingress.hosts[0].paths[0].pathType=Prefix
```

### オートスケーリングの有効化

```bash
helm install my-release ./testchart \
  --set autoscaling.enabled=true \
  --set autoscaling.minReplicas=2 \
  --set autoscaling.maxReplicas=10
```

### カスタムイメージ

```bash
helm install my-release ./testchart \
  --set image.repository=myregistry/myapp \
  --set image.tag=1.0.0
```

## アプリケーションへのアクセス

### ClusterIPサービスの場合（デフォルト）

```bash
# kubectl port-forwardを使用
kubectl port-forward svc/my-release-testchart 8080:80

# http://localhost:8080 でアクセス
```

### NodePortサービスの場合

```bash
# NodePortを取得
kubectl get svc my-release-testchart

# http://<ノードIP>:<ノードポート> でアクセス
```

### WSL/Minikube環境の場合

```bash
# minikube serviceコマンドを使用
minikube service my-release-testchart --url
```

## 開発

### テスト

```bash
# chartのリント
helm lint ./testchart

# ドライランインストール
helm install my-release ./testchart --dry-run --debug

# テストの実行
helm test my-release
```

### パッケージング

```bash
# chartのパッケージ化
helm package ./testchart

# chartリポジトリ用のインデックスファイル作成
helm repo index .
```

## トラブルシューティング

### Podが起動しない場合

Podのステータスとログを確認：
```bash
kubectl get pods -l app.kubernetes.io/instance=my-release
kubectl describe pod <pod名>
kubectl logs <pod名>
```

### サービスにアクセスできない場合

サービスとエンドポイントを確認：
```bash
kubectl get svc my-release-testchart
kubectl get endpoints my-release-testchart
```

## ライセンス

このchartは教育およびテスト目的で現状のまま提供されています。