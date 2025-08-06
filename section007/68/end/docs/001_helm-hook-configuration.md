# Helm Hook設定の詳細説明

Helm Hookを使用する際の重要な設定項目について説明します。

## Helm Hook アノテーション

### helm.sh/hook
Hookのタイミングを指定します。

利用可能な値：
- `pre-install`: インストール前に実行
- `post-install`: インストール後に実行
- `pre-delete`: 削除前に実行
- `post-delete`: 削除後に実行
- `pre-upgrade`: アップグレード前に実行
- `post-upgrade`: アップグレード後に実行
- `pre-rollback`: ロールバック前に実行
- `post-rollback`: ロールバック後に実行
- `test`: helm testコマンド実行時に実行

### helm.sh/hook-weight
複数のHookがある場合の実行順序を制御します。
- 数値で指定（文字列として記述）
- 小さい数値から順に実行される
- デフォルトは "0"
- 負の値も使用可能（例："-5", "1", "10"）

### helm.sh/hook-delete-policy
Hook実行後のリソースの削除ポリシーを指定します。

利用可能な値：
- `before-hook-creation`: 新しいHook実行前に前回のリソースを削除（デフォルト）
- `hook-succeeded`: Hook成功時にリソースを削除
- `hook-failed`: Hook失敗時にリソースを削除

複数指定も可能：
```yaml
"helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

## Pod仕様の重要な設定

### restartPolicy
PodまたはJobの再起動ポリシーを指定します。

利用可能な値：
- `Always`: 常に再起動（通常のPodのデフォルト）
- `OnFailure`: 失敗時のみ再起動（JobやHookで推奠）
- `Never`: 再起動しない

Hookでは通常 `OnFailure` または `Never` を使用します。

### imagePullPolicy
コンテナイメージの取得ポリシーを指定します。

利用可能な値：
- `Always`: 常に最新のイメージをプルする
- `IfNotPresent`: ローカルにイメージがない場合のみプル（デフォルト）
- `Never`: ローカルのイメージのみを使用、プルしない

#### 各ポリシーの使い分け：

**Always**:
- 本番環境で`latest`タグを使用する場合
- 頻繁に更新されるイメージを使用する場合
- セキュリティパッチを確実に適用したい場合

**IfNotPresent**:
- 特定のバージョンタグ（例：`v1.2.3`）を使用する場合
- ネットワーク帯域を節約したい場合
- 最も一般的な設定

**Never**:
- オフライン環境での実行
- 事前にイメージをプリロードしている場合
- ローカル開発環境でのテスト

### 使用例：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "testchart.fullname" . }}-pre-install-hook"
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
        - name: pre-install
          image: busybox
          imagePullPolicy: IfNotPresent
          command: ["sh", "-c", "echo 'Running pre-install hook' && sleep 5"]
      restartPolicy: OnFailure
```

## ベストプラクティス

### 1. restartPolicyの選択
- **データベースマイグレーション**: `OnFailure` - 失敗時に再試行可能
- **一度だけ実行すべきタスク**: `Never` - 重複実行を防ぐ
- **クリーンアップタスク**: `OnFailure` - 確実に実行される必要がある

### 2. hook-delete-policyの選択
- **デバッグが必要な場合**: ポリシーを設定しない（リソースが残る）
- **本番環境**: `hook-succeeded` - 成功時のみクリーンアップ
- **テスト環境**: `before-hook-creation,hook-succeeded` - 常にクリーンな状態を保つ

### 3. imagePullPolicyの選択
- **開発環境**: `IfNotPresent` - 高速なデプロイが可能
- **本番環境（バージョンタグ使用）**: `IfNotPresent` - 安定性重視
- **本番環境（latestタグ使用）**: `Always` - 最新の修正を反映（非推奨）
- **エアギャップ環境**: `Never` - 外部ネットワークアクセスなし

### 4. hook-weightの使用
```yaml
# 例：データベース作成 → マイグレーション → 初期データ投入
db-create:     "helm.sh/hook-weight": "-1"
db-migrate:    "helm.sh/hook-weight": "0"
db-seed:       "helm.sh/hook-weight": "1"
```

## トラブルシューティング

### Hook実行の確認
```bash
# Hook Jobの状態を確認
kubectl get jobs -l "helm.sh/hook"

# Podのログを確認
kubectl logs -l job-name=<hook-job-name>
```

### よくある問題
1. **restartPolicy: Always を使用した場合**: JobやCronJobでエラーになる
2. **hook-delete-policy未設定**: デバッグには便利だが、リソースが蓄積される
3. **hook-weight未設定**: 実行順序が保証されない
4. **imagePullPolicy: Always でlatestタグ**: ネットワーク遅延やレート制限の問題
5. **imagePullPolicy: Never で新規イメージ**: イメージが見つからずエラー