# Helm Hook と通常のリソースの違い（公式ドキュメント準拠）

## 根本的な違い

公式ドキュメントには重要な記述があります：
> "Hook resources are not managed with corresponding releases"

これが最も重要な違いです。以下、具体的に説明します。

## 1. リソース管理の違い

### 通常のリソース
- Helmが完全に管理
- `helm install`で作成
- `helm upgrade`で更新
- `helm uninstall`で削除
- `helm list`で確認可能

### Helm Hook
- 実行後はHelmの管理外
- `helm uninstall`でも削除されない
- 手動削除または`hook-delete-policy`が必要
- リリースの一部として扱われない

## 2. 実行タイミングとブロッキング動作

### 通常のリソース
```yaml
# すべて並列で作成される
apiVersion: v1
kind: Service
---
apiVersion: apps/v1
kind: Deployment
---
apiVersion: v1
kind: ConfigMap
```

### Helm Hook
```yaml
# 順次実行され、完了を待つ
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-5"
```

**重要**: Job/Podタイプのhookは完了まで待機（ブロッキング）します。

## 3. 失敗時の動作の違い

### 通常のリソース
- 一部のリソースが失敗してもデプロイ続行
- 失敗したリソースは再試行される
- 全体のリリースは「部分的成功」になる可能性

### Helm Hook
- **Hookが失敗 = リリース全体が失敗**
- 後続の処理は実行されない
- ロールバックが必要

## 4. 具体例で見る違い

### シナリオ: データベース初期化を含むアプリケーションデプロイ

```yaml
# ❌ 通常のリソースで実装（問題あり）
apiVersion: batch/v1
kind: Job
metadata:
  name: db-init
spec:
  template:
    spec:
      containers:
      - name: init
        image: postgres
        command: ["psql", "-c", "CREATE DATABASE myapp;"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
# 問題: JobとDeploymentが同時に実行され、DBが準備できていない
```

```yaml
# ✅ Hookで実装（正しい）
apiVersion: batch/v1
kind: Job
metadata:
  name: db-init
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: init
        image: postgres
        command: ["psql", "-c", "CREATE DATABASE myapp;"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
# 正解: Hookが完了してからDeploymentが作成される
```

## 5. 削除時の挙動の違い

### 通常のリソース削除時
```bash
$ helm uninstall myapp
# すべてのリソースが自動削除される
release "myapp" uninstalled
```

### Hook付きリソース削除時
```bash
$ helm uninstall myapp
release "myapp" uninstalled

$ kubectl get jobs
NAME          COMPLETIONS   AGE
myapp-db-init 1/1           5m    # Hookは残っている！
```

## 6. 実行順序の制御

### 通常のリソース
- Kubernetesが内部的に決定
- 依存関係は`initContainers`や`readinessProbe`で制御

### Helm Hook
```yaml
# 厳密な順序制御が可能
hook1: "helm.sh/hook-weight": "-10"  # 最初
hook2: "helm.sh/hook-weight": "0"    # 次
hook3: "helm.sh/hook-weight": "10"   # 最後
```

## 7. `--no-hooks`フラグの影響

```bash
# 通常のリソース: 影響なし
helm upgrade myapp ./chart --no-hooks

# Hook: スキップされる
helm upgrade myapp ./chart --no-hooks  # pre/post-upgrade hookは実行されない
```

## まとめ：使い分けの指針

### 通常のリソースを使うべき場合
- アプリケーションの主要コンポーネント
- 継続的に動作するサービス
- Helmで管理したいリソース

### Hookを使うべき場合
- 一時的な初期化処理
- 順序制御が重要な処理
- リリースのライフサイクル外で実行したい処理
- 失敗時にリリース全体を止めたい処理

## 注意事項

1. **Hookは残る**: `hook-delete-policy`を設定しないと永続的に残る
2. **デバッグが難しい**: 通常のリソースと別管理のため
3. **リソース競合**: 同名のリソースがあると問題になる