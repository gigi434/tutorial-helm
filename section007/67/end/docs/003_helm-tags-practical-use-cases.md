# Helmチャートのtagsを使う実践的なシチュエーション

Helmチャートの**tags（タグ）**を使う具体的なシチュエーションは、「環境や条件によって、一部のKubernetesリソースやサブチャートだけをデプロイしたい／したくない場合」に役立ちます。

以下に典型的な利用例を示します。

## 1. 開発・本番環境で違う構成を簡単に切り替えたいとき

**例：開発環境ではダミーデータ投入バッチを動かしたいが、本番では不要**

チャート内に`job.yaml`（ダミーデータJob）を用意し、`{{- if .Values.tags.withSampleData }}`で条件分岐：

```yaml
# templates/job.yaml
{{- if .Values.tags.withSampleData }}
apiVersion: batch/v1
kind: Job
metadata:
  name: sample-data-loader
spec:
  template:
    spec:
      containers:
      - name: loader
        image: myapp:latest
        command: ["sh", "-c", "load-sample-data.sh"]
      restartPolicy: OnFailure
{{- end }}
```

```yaml
# values-dev.yaml
tags:
  withSampleData: true

# values-prod.yaml
tags:
  withSampleData: false
```

インストール時に`--values values-dev.yaml`で開発用ジョブを有効化、本番では省略。

## 2. サブコンポーネント（例：MySQL、Redisなど）を任意で組み合わせ可能にしたい

**例：メインアプリだけでなく、任意のDBやキャッシュを同時にデプロイしたい／不要なとき外したい**

`mysql.enabled`、`redis.enabled`のような値と`tags.database`（まとめタグ）を併用し、依存チャートに`condition`や`tags`を指定：

```yaml
# Chart.yaml
dependencies:
  - name: mysql
    version: ^14.0.0
    repository: "@bitnami"
    condition: mysql.enabled
    tags:
      - database
  - name: redis
    version: ^17.0.0
    repository: "@bitnami"
    condition: redis.enabled
    tags:
      - database
      - cache
```

```bash
helm install myapp --set tags.database=true
```

でDB/Redisも一括有効化できる。

## 3. テスト用リソースだけを一括で有効・無効にしたい

**例：インテグレーションTesting用リソース（テスト用PodやService）を本番では絶対無効化したい**

`tags: { test: true }`のような値を使い、`if .Values.tags.test`でテスト用リソースの生成を切り替える：

```yaml
# templates/test-service.yaml
{{- if .Values.tags.test }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}-test
spec:
  type: ClusterIP
  ports:
    - port: 9090
      targetPort: test
  selector:
    app: {{ include "myapp.name" . }}-test
{{- end }}
```

## 4. サブチャートが増えた場合の管理がシンプルになる

多くのサブチャートを含むシステムで、タグ一つでグルーピングしてデプロイ内容をコントロールできるため、「APIのみ」「API＋バッチ」「全サービス」など切り替えが容易：

```yaml
# Chart.yaml
dependencies:
  - name: api-service
    tags:
      - api
      - core
  - name: batch-processor
    tags:
      - batch
      - backend
  - name: web-frontend
    tags:
      - frontend
      - ui
```

```bash
# APIのみデプロイ
helm install myapp . --set tags.api=true,tags.batch=false,tags.frontend=false

# 全サービスデプロイ
helm install myapp . --set tags.core=true,tags.backend=true,tags.ui=true
```

## まとめ

**tags**はテンプレート内の条件分岐を柔軟にし、チャート利用者が「必要なときだけリソースを追加、除外できる」仕組みです。

例：`--set tags.debug=true`でデバッグ用の全部署リソースを即時追加、本番では外す…などDevOps現場でよく使われます。

### 実践的な活用パターン

- **環境別制御**: `development`, `staging`, `production`
- **機能別制御**: `monitoring`, `logging`, `tracing`
- **デバッグ制御**: `debug`, `verbose`, `test`
- **セキュリティ制御**: `security`, `rbac`, `networkpolicy`

これらのタグを組み合わせることで、柔軟で管理しやすいHelmチャートを構築できます。