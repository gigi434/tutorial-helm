# Helm Hooks 完全ガイド

## Helm Hookとは

Helm Hookは、Helmのリリースライフサイクルの特定のタイミングで実行されるKubernetesリソースです。通常のリソースとは異なり、特定のイベントが発生したときにのみ作成・実行されます。

## なぜHelm Hookを使うのか

### 主な用途
1. **データベースの初期化**: アプリケーション起動前にスキーマやデータを準備
2. **バックアップ**: アップグレード前に既存データをバックアップ
3. **検証**: デプロイ後にアプリケーションの動作を確認
4. **クリーンアップ**: 削除時に外部リソースを適切に処理
5. **設定の準備**: ConfigMapやSecretの動的生成

### 通常のリソースとの違い（公式ドキュメントより）

#### 1. **リソース管理の違い**
- **通常のリソース**: Helmによって完全に管理される（作成・更新・削除が自動）
- **Hook**: Helmはhookの実行を確認した後は管理しない。`helm uninstall`でも自動削除されない

#### 2. **実行タイミングとブロッキング**
- **通常のリソース**: `helm install/upgrade`時に並列でデプロイされる
- **Hook**: 特定のライフサイクルポイントで順次実行される。Job/Podの場合は完了まで待機（ブロッキング動作）

#### 3. **失敗時の動作**
- **通常のリソース**: 一部が失敗してもデプロイは続行される場合がある
- **Hook**: 失敗するとリリース全体が失敗する

#### 4. **定義方法**
- **通常のリソース**: templatesディレクトリ内の通常のマニフェスト
- **Hook**: `"helm.sh/hook"`アノテーションを持つマニフェスト

```yaml
# 通常のリソース
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config

# Hook
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  annotations:
    "helm.sh/hook": pre-install  # この行がHookにする
```

#### 5. **実行順序の制御**
- **通常のリソース**: Kubernetesが決定する順序で作成
- **Hook**: weight、リソースタイプ、名前の順で厳密に制御される

#### 6. **削除時の挙動**
- **通常のリソース**: `helm uninstall`で自動的に削除
- **Hook**: 明示的な削除ポリシーが必要。設定しない限り残り続ける

## Hook実行可能なリソースタイプ

Helm Hookとして使用できるKubernetesリソース：
- **Job**: 最も一般的、一回限りのタスク実行
- **Pod**: シンプルなタスク（非推奨、Jobを推奨）
- **ConfigMap/Secret**: 動的な設定生成
- **PersistentVolumeClaim**: ストレージの事前準備
- **ServiceAccount/Role/RoleBinding**: 権限の設定

## Hookのライフサイクルイベント

### インストール関連
```yaml
"helm.sh/hook": pre-install   # helm install実行前
"helm.sh/hook": post-install  # helm install成功後
```

### アップグレード関連
```yaml
"helm.sh/hook": pre-upgrade   # helm upgrade実行前
"helm.sh/hook": post-upgrade  # helm upgrade成功後
```

### 削除関連
```yaml
"helm.sh/hook": pre-delete    # helm uninstall実行前
"helm.sh/hook": post-delete   # helm uninstall成功後
```

### ロールバック関連
```yaml
"helm.sh/hook": pre-rollback  # helm rollback実行前
"helm.sh/hook": post-rollback # helm rollback成功後
```

### テスト
```yaml
"helm.sh/hook": test          # helm test実行時
```

## 実践的な例

### 1. データベースマイグレーション（pre-upgrade + pre-install）
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-migrate"
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: migrate/migrate:latest
        command: ["migrate", "-path", "/migrations", "-database", "$(DATABASE_URL)", "up"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
```

### 2. バックアップJob（pre-upgrade）
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-backup"
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: backup
        image: postgres:15
        command:
        - sh
        - -c
        - |
          BACKUP_NAME="{{ .Release.Name }}-$(date +%Y%m%d-%H%M%S).sql"
          pg_dump $DATABASE_URL > /backup/$BACKUP_NAME
          echo "Backup completed: $BACKUP_NAME"
        volumeMounts:
        - name: backup
          mountPath: /backup
      volumes:
      - name: backup
        persistentVolumeClaim:
          claimName: backup-pvc
```

### 3. ヘルスチェック（post-install, post-upgrade）
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-health-check"
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: health-check
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          echo "Waiting for application to be ready..."
          for i in $(seq 1 30); do
            if curl -f http://{{ .Release.Name }}-service/health; then
              echo "Application is healthy!"
              exit 0
            fi
            sleep 10
          done
          echo "Health check failed!"
          exit 1
```

### 4. クリーンアップ（pre-delete）
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-cleanup"
  annotations:
    "helm.sh/hook": pre-delete
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: cleanup
        image: amazon/aws-cli:latest
        command:
        - sh
        - -c
        - |
          # S3バケットのクリーンアップ
          aws s3 rm s3://{{ .Values.bucket.name }}/{{ .Release.Name }}/ --recursive
          # CloudWatchロググループの削除
          aws logs delete-log-group --log-group-name /aws/eks/{{ .Release.Name }}
```

### 5. テストHook
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-test"
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: test
        image: postman/newman:latest
        command: ["newman", "run", "/tests/api-tests.json", "--env-var", "BASE_URL=http://{{ .Release.Name }}-service"]
        volumeMounts:
        - name: tests
          mountPath: /tests
      volumes:
      - name: tests
        configMap:
          name: "{{ .Release.Name }}-api-tests"
```

## Hookの実行順序制御

### hook-weightによる順序制御
```yaml
# 実行順序: backup → migrate → validate
backup:   "helm.sh/hook-weight": "-10"
migrate:  "helm.sh/hook-weight": "0"
validate: "helm.sh/hook-weight": "10"
```

### 複雑な依存関係の例
```yaml
# 1. 外部リソースチェック（weight: -20）
# 2. バックアップ（weight: -10）
# 3. データベース停止（weight: -5）
# 4. スキーマ更新（weight: 0）
# 5. データ変換（weight: 5）
# 6. データベース起動（weight: 10）
# 7. 検証（weight: 20）
```

## ベストプラクティス

### 1. 冪等性の確保
```yaml
# 良い例：既存チェックを含む
command:
- sh
- -c
- |
  if kubectl get secret my-secret 2>/dev/null; then
    echo "Secret already exists"
  else
    kubectl create secret generic my-secret --from-literal=key=value
  fi
```

### 2. タイムアウトの設定
```yaml
spec:
  activeDeadlineSeconds: 300  # 5分でタイムアウト
  template:
    spec:
      containers:
      - name: task
        image: busybox
        command: ["sh", "-c", "timeout 300 ./long-running-task.sh"]
```

### 3. エラーハンドリング
```yaml
command:
- sh
- -c
- |
  set -e  # エラー時に即座に終了
  echo "Starting task..."
  
  # クリーンアップ関数
  cleanup() {
    echo "Cleaning up..."
    # クリーンアップ処理
  }
  trap cleanup EXIT
  
  # メイン処理
  perform_task || {
    echo "Task failed!"
    exit 1
  }
```

### 4. リソース制限
```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

## トラブルシューティング

### Hookの実行状況確認
```bash
# 全Hookの確認
kubectl get jobs,pods -l "helm.sh/hook" --all-namespaces

# 特定リリースのHook
kubectl get jobs -l "helm.sh/hook,app.kubernetes.io/instance=myrelease"

# Hook実行ログ
kubectl logs job/myrelease-migrate
```

### よくある問題と解決策

1. **Hookが実行されない**
   - アノテーションのスペルミスを確認
   - YAMLのインデントを確認

2. **Hookが失敗する**
   - リソース不足（limits/requests確認）
   - 権限不足（ServiceAccount/RBAC確認）
   - 依存リソースの準備不足

3. **Hookが削除されない**
   - hook-delete-policyの設定確認
   - 手動削除: `kubectl delete job -l "helm.sh/hook"`

4. **実行順序が意図と異なる**
   - hook-weightの値を確認
   - 同じweightの場合は名前順

## まとめ

Helm Hookは、Kubernetesアプリケーションのデプロイメントをより柔軟かつ安全にする強力な機能です。適切に使用することで、複雑なデプロイメントシナリオも確実に実行できます。ただし、過度に複雑なHookは管理が困難になるため、シンプルさを保つことが重要です。