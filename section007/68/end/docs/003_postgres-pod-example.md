# PostgreSQL Pod設定例（Hookと連携）

前述のHookと連携して動作するPostgreSQLのPod設定例を示します。

## 1. PostgreSQL StatefulSet

```yaml
# templates/postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "myapp.fullname" . }}-postgres
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
    app.kubernetes.io/component: database
spec:
  serviceName: {{ include "myapp.fullname" . }}-postgres-headless
  replicas: 1
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: database
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: database
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "myapp.fullname" . }}-postgres-secret
              key: postgres-password
        - name: POSTGRES_DB
          value: postgres  # デフォルトDB（myappはHookで作成）
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U postgres
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U postgres
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          {{- toYaml .Values.postgresql.resources | nindent 10 }}
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: {{ .Values.postgresql.persistence.size }}
```

## 2. PostgreSQL Service

```yaml
# templates/postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}-postgres
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
    app.kubernetes.io/component: database
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: postgres
    protocol: TCP
    name: postgres
  selector:
    {{- include "myapp.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: database
---
# Headless Service for StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}-postgres-headless
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
    app.kubernetes.io/component: database
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - port: 5432
    targetPort: postgres
    protocol: TCP
    name: postgres
  selector:
    {{- include "myapp.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: database
```

## 3. PostgreSQL Secret

```yaml
# templates/postgres-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}-postgres-secret
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
type: Opaque
data:
  postgres-password: {{ .Values.postgresql.auth.postgresPassword | b64enc | quote }}
  password: {{ .Values.postgresql.auth.password | b64enc | quote }}
  username: {{ .Values.postgresql.auth.username | b64enc | quote }}
```

## 4. ConfigMap（マイグレーション用）

```yaml
# templates/migrations-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-migrations
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
data:
  001_create_users_table.up.sql: |
    CREATE TABLE IF NOT EXISTS users (
      id SERIAL PRIMARY KEY,
      email VARCHAR(255) UNIQUE NOT NULL,
      name VARCHAR(255) NOT NULL,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    CREATE INDEX idx_users_email ON users(email);
  
  002_create_settings_table.up.sql: |
    CREATE TABLE IF NOT EXISTS settings (
      key VARCHAR(255) PRIMARY KEY,
      value TEXT NOT NULL,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
```

## 5. アプリケーションPod（PostgreSQLを使用）

```yaml
# templates/app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}-app
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
    app.kubernetes.io/component: app
spec:
  replicas: {{ .Values.app.replicaCount }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: app
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: app
    spec:
      initContainers:
      # データベースが準備できるまで待機
      - name: wait-for-db
        image: postgres:15-alpine
        command:
        - sh
        - -c
        - |
          until pg_isready -h {{ include "myapp.fullname" . }}-postgres -p 5432 -U {{ .Values.postgresql.auth.username }}; do
            echo "Waiting for database..."
            sleep 2
          done
        env:
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "myapp.fullname" . }}-postgres-secret
              key: password
      containers:
      - name: app
        image: "{{ .Values.app.image.repository }}:{{ .Values.app.image.tag }}"
        imagePullPolicy: {{ .Values.app.image.pullPolicy }}
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: DATABASE_URL
          value: "postgres://{{ .Values.postgresql.auth.username }}:$(DB_PASSWORD)@{{ include "myapp.fullname" . }}-postgres:5432/myapp?sslmode=disable"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "myapp.fullname" . }}-postgres-secret
              key: password
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          {{- toYaml .Values.app.resources | nindent 10 }}
```

## 6. values.yaml

```yaml
# values.yaml
postgresql:
  auth:
    postgresPassword: "postgres-admin-password"
    username: "myapp"
    password: "myapp-password"
  persistence:
    size: 10Gi
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"

app:
  replicaCount: 2
  image:
    repository: myapp
    tag: "1.0.0"
    pullPolicy: IfNotPresent
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "200m"
```

## 7. 完全なデプロイフロー

1. **helm install時の実行順序**：
   ```
   1. Secret作成（postgres-secret）
   2. ConfigMap作成（migrations）
   3. Hook実行：
      a. db-create Job（weight: -2）- データベース"myapp"を作成
      b. db-migrate Job（weight: -1）- テーブル作成
   4. メインリソース：
      a. PostgreSQL StatefulSet
      b. PostgreSQL Services
      c. アプリケーションDeployment
   5. Hook実行：
      a. db-seed Job（weight: 0）- 初期データ投入
   ```

2. **コマンド例**：
   ```bash
   # インストール
   helm install myapp ./myapp-chart
   
   # 状態確認
   kubectl get pods,jobs,pvc
   
   # データベース接続テスト
   kubectl exec -it myapp-postgres-0 -- psql -U myapp -d myapp -c "\dt"
   ```

## 8. トラブルシューティング

### Hookが失敗する場合
```bash
# Hook Jobのログ確認
kubectl logs job/myapp-db-create
kubectl logs job/myapp-db-migrate

# PostgreSQL Podのログ確認
kubectl logs myapp-postgres-0
```

### アプリケーションがDBに接続できない場合
```bash
# Service確認
kubectl get svc myapp-postgres

# Secret確認
kubectl get secret myapp-postgres-secret -o yaml

# アプリケーションPodからの接続テスト
kubectl exec -it deployment/myapp-app -- sh
# Pod内で
pg_isready -h myapp-postgres -p 5432
```

## 注意点

1. **StatefulSetを使用する理由**：
   - データの永続性が保証される
   - Pod名が固定される（myapp-postgres-0）
   - 順序立てた起動・終了が可能

2. **initContainerの重要性**：
   - アプリケーションがDBより先に起動しても安全
   - 接続エラーを防ぐ

3. **リソース制限**：
   - 本番環境では適切なリソース設定が必須
   - PostgreSQLは特にメモリ使用量に注意