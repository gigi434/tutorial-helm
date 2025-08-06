# Helm Test ガイド

## Helm Testとは

Helm Testは、デプロイされたHelmリリースが正しく動作しているかを検証するための機能です。`helm test`コマンドを実行することで、事前に定義されたテスト用のPodやJobを実行し、アプリケーションの動作確認を自動化できます。

## 基本的な使い方

### テストの定義

テストは通常のKubernetesマニフェストに`helm.sh/hook: test`アノテーションを追加することで定義します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "myapp.fullname" . }}-test"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: test
      image: busybox
      command: ['wget', '-q', 'http://myservice:80']
  restartPolicy: Never
```

### テストの実行

```bash
# リリースのテストを実行
helm test <release-name>

# タイムアウトを指定してテスト実行（デフォルト: 300秒）
helm test <release-name> --timeout 600s

# ログを表示しながらテスト実行
helm test <release-name> --logs
```

## テストの種類と例

### 1. 接続性テスト

サービスへの基本的な接続確認を行います。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "chart.fullname" . }}-test-connection"
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  containers:
    - name: wget
      image: busybox:1.36
      command: 
        - sh
        - -c
        - |
          echo "Testing connection to {{ include "chart.fullname" . }}"
          if wget -q -O- http://{{ include "chart.fullname" . }}:{{ .Values.service.port }}; then
            echo "Connection successful"
          else
            echo "Connection failed"
            exit 1
          fi
  restartPolicy: Never
```

### 2. データベース接続テスト

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-db-test"
  annotations:
    "helm.sh/hook": test
spec:
  template:
    spec:
      containers:
        - name: db-test
          image: postgres:15-alpine
          env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-secret
                  key: password
          command:
            - sh
            - -c
            - |
              echo "Testing database connection..."
              if psql -h {{ .Values.postgresql.host }} -U {{ .Values.postgresql.user }} -d {{ .Values.postgresql.database }} -c "SELECT 1"; then
                echo "Database connection successful"
              else
                echo "Database connection failed"
                exit 1
              fi
      restartPolicy: Never
```

### 3. API エンドポイントテスト

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-api-test"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: api-test
      image: curlimages/curl:latest
      command:
        - sh
        - -c
        - |
          # ヘルスチェックエンドポイント
          echo "Testing health endpoint..."
          HEALTH=$(curl -s http://{{ include "chart.fullname" . }}:{{ .Values.service.port }}/health)
          if [ "$HEALTH" = '{"status":"ok"}' ]; then
            echo "✓ Health check passed"
          else
            echo "✗ Health check failed: $HEALTH"
            exit 1
          fi
          
          # APIバージョン確認
          echo "Testing API version..."
          VERSION=$(curl -s http://{{ include "chart.fullname" . }}:{{ .Values.service.port }}/api/version)
          echo "API Version: $VERSION"
  restartPolicy: Never
```

### 4. 複数のテストを含む包括的なテスト

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-comprehensive-test"
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-weight": "1"
spec:
  containers:
    - name: test
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Starting comprehensive tests..."
          
          # Test 1: DNS解決
          echo "Test 1: DNS Resolution"
          if nslookup {{ include "chart.fullname" . }} >/dev/null 2>&1; then
            echo "✓ DNS resolution successful"
          else
            echo "✗ DNS resolution failed"
            exit 1
          fi
          
          # Test 2: ポート接続性
          echo "Test 2: Port Connectivity"
          if nc -z {{ include "chart.fullname" . }} {{ .Values.service.port }}; then
            echo "✓ Port {{ .Values.service.port }} is open"
          else
            echo "✗ Port {{ .Values.service.port }} is closed"
            exit 1
          fi
          
          # Test 3: HTTPレスポンス
          echo "Test 3: HTTP Response"
          HTTP_CODE=$(wget -S -O /dev/null http://{{ include "chart.fullname" . }}:{{ .Values.service.port }} 2>&1 | grep "HTTP/" | awk '{print $2}')
          if [ "$HTTP_CODE" = "200" ]; then
            echo "✓ HTTP 200 OK"
          else
            echo "✗ HTTP response code: $HTTP_CODE"
            exit 1
          fi
          
          echo "All tests passed!"
  restartPolicy: Never
```

## Hook削除ポリシー

テストPodの削除タイミングを制御できます。

```yaml
annotations:
  "helm.sh/hook": test
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

利用可能なポリシー：
- `before-hook-creation`: 新しいテスト実行前に前回のPodを削除
- `hook-succeeded`: テスト成功時にPodを削除
- `hook-failed`: テスト失敗時にPodを削除

## テストの実行順序制御

複数のテストがある場合、`helm.sh/hook-weight`で実行順序を制御できます。

```yaml
# 最初に実行されるテスト
metadata:
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-weight": "-1"

# 次に実行されるテスト
metadata:
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-weight": "0"

# 最後に実行されるテスト
metadata:
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-weight": "1"
```

## ベストプラクティス

### 1. 適切なタイムアウト設定

```yaml
spec:
  activeDeadlineSeconds: 60  # 60秒でタイムアウト
```

### 2. リソース制限の設定

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "100m"
  limits:
    memory: "128Mi"
    cpu: "200m"
```

### 3. エラーハンドリング

```bash
# エラー時の詳細情報を含むテスト
command:
  - sh
  - -c
  - |
    set -e
    trap 'echo "Error on line $LINENO"' ERR
    
    # テストコード
    wget -q -O- http://myservice || {
      echo "Failed to connect to service"
      echo "Debug information:"
      nslookup myservice
      exit 1
    }
```

### 4. 環境別のテスト

```yaml
{{- if .Values.tests.enabled }}
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: test
      image: {{ .Values.tests.image }}
      command: {{ .Values.tests.command }}
  restartPolicy: Never
{{- end }}
```

## トラブルシューティング

### テストが失敗する場合

```bash
# テストPodの状態確認
kubectl get pods -l "helm.sh/hook=test"

# テストログの確認
kubectl logs <test-pod-name>

# テストPodの詳細情報
kubectl describe pod <test-pod-name>
```

### よくある問題

1. **テストPodが残る**: `hook-delete-policy`を設定する
2. **タイムアウト**: `activeDeadlineSeconds`を増やす
3. **権限エラー**: ServiceAccountとRBACを確認
4. **ネットワークエラー**: Service名とポートを確認

## まとめ

Helm Testは、デプロイされたアプリケーションの健全性を自動的に検証する強力な機能です。適切なテストを実装することで、リリースの品質を向上させ、問題を早期に発見できます。CI/CDパイプラインに組み込むことで、継続的な品質保証が可能になります。