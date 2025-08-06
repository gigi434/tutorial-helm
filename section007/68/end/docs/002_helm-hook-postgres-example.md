# Helm Hook Delete Policy 具体例：PostgreSQLのセットアップ

`hook-delete-policy`の動作を理解するため、PostgreSQLデータベースの初期化からシードデータ投入までの実例を示します。

## 完全な例：PostgreSQLアプリケーションのデプロイ

### 1. データベース作成Hook（pre-install）

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-db-create"
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-2"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      containers:
        - name: db-create
          image: postgres:15-alpine
          env:
            - name: PGPASSWORD
              value: "{{ .Values.postgresql.auth.password }}"
          command:
            - sh
            - -c
            - |
              echo "Creating database..."
              psql -h {{ .Values.postgresql.host }} -U postgres -tc "SELECT 1 FROM pg_database WHERE datname = 'myapp'" | grep -q 1 || \
              psql -h {{ .Values.postgresql.host }} -U postgres -c "CREATE DATABASE myapp"
      restartPolicy: OnFailure
```

**`before-hook-creation`の動作**：
- 次回のhelm upgrade時に、新しいJobを作成する前に古いJobを削除
- デバッグが不要で、常に最新の状態を保ちたい場合に適している

### 2. マイグレーションHook（pre-install/pre-upgrade）

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-db-migrate"
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-1"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: migrate/migrate:v4.15.2
          args:
            - "-path=/migrations"
            - "-database=postgres://{{ .Values.postgresql.auth.username }}:{{ .Values.postgresql.auth.password }}@{{ .Values.postgresql.host }}/myapp?sslmode=disable"
            - "up"
          volumeMounts:
            - name: migrations
              mountPath: /migrations
      volumes:
        - name: migrations
          configMap:
            name: "{{ .Release.Name }}-migrations"
      restartPolicy: OnFailure
```

**`hook-succeeded`の動作**：
- Jobが成功した場合のみ削除される
- 失敗した場合はJobが残るため、ログを確認してデバッグ可能

### 3. シードデータ投入Hook（post-install）

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-db-seed"
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": hook-failed
spec:
  template:
    spec:
      containers:
        - name: seed
          image: postgres:15-alpine
          env:
            - name: PGPASSWORD
              value: "{{ .Values.postgresql.auth.password }}"
          command:
            - sh
            - -c
            - |
              echo "Seeding database..."
              psql -h {{ .Values.postgresql.host }} -U {{ .Values.postgresql.auth.username }} -d myapp <<'EOF'
              INSERT INTO users (email, name) VALUES 
                ('admin@example.com', 'Admin User'),
                ('user@example.com', 'Regular User')
              ON CONFLICT (email) DO NOTHING;
              
              INSERT INTO settings (key, value) VALUES
                ('app_version', '{{ .Chart.Version }}'),
                ('initialized_at', NOW())
              ON CONFLICT (key) DO UPDATE SET value = EXCLUDED.value;
              EOF
      restartPolicy: Never
```

**`hook-failed`の動作**：
- Jobが失敗した場合のみ削除される
- 成功した場合はJobが残り、実行結果を確認できる
- シードデータの投入内容を後から確認したい場合に便利

### 4. 複数のdelete-policyを組み合わせた例

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-db-check"
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      containers:
        - name: db-test
          image: postgres:15-alpine
          env:
            - name: PGPASSWORD
              value: "{{ .Values.postgresql.auth.password }}"
          command:
            - sh
            - -c
            - |
              echo "Testing database connection and data..."
              psql -h {{ .Values.postgresql.host }} -U {{ .Values.postgresql.auth.username }} -d myapp -c "SELECT COUNT(*) FROM users"
      restartPolicy: Never
```

**複数ポリシーの動作**：
- `before-hook-creation`: 次回実行時に古いJobを削除
- `hook-succeeded`: 成功時にも削除
- つまり、失敗時のみJobが残る

## delete-policyごとの使い分けガイド

### 1. ポリシーなし（デフォルト）
```yaml
# annotations:
#   "helm.sh/hook-delete-policy": 設定なし
```
- **用途**: 開発・デバッグ時
- **メリット**: すべての実行履歴が残る
- **デメリット**: リソースが蓄積される

### 2. before-hook-creation
```yaml
"helm.sh/hook-delete-policy": before-hook-creation
```
- **用途**: 定期的に実行されるHook
- **メリット**: リソースがクリーンに保たれる
- **例**: ヘルスチェック、定期バックアップ

### 3. hook-succeeded
```yaml
"helm.sh/hook-delete-policy": hook-succeeded
```
- **用途**: 重要な処理で失敗時のデバッグが必要
- **メリット**: 失敗時にログが残る
- **例**: データベースマイグレーション、重要な初期化処理

### 4. hook-failed
```yaml
"helm.sh/hook-delete-policy": hook-failed
```
- **用途**: 成功の証跡を残したい処理
- **メリット**: 成功した処理の内容を確認可能
- **例**: 監査ログ、コンプライアンスチェック

## 実行順序の完全な例

```yaml
# values.yaml
postgresql:
  host: postgres-service
  auth:
    username: myapp
    password: secretpassword
```

実行順序：
1. **pre-install hooks** (weight順)
   - `-2`: db-create（データベース作成）
   - `-1`: db-migrate（スキーマ作成）
2. **メインリソース**: PostgreSQL Pod、Service、etc
3. **post-install hooks**
   - `0`: db-seed（初期データ投入）

## トラブルシューティング

```bash
# すべてのHook Jobを確認
kubectl get jobs -l "helm.sh/hook" --show-labels

# 特定のHookのログを確認
kubectl logs job/myapp-db-migrate

# 失敗したJobの詳細を確認
kubectl describe job myapp-db-seed

# Hook関連のリソースをすべて削除（手動クリーンアップ）
kubectl delete jobs -l "helm.sh/hook"
```