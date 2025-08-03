# Minikube Tunnel と WSL環境でのサービスアクセス

## Minikube Tunnelとは

`minikube tunnel`は、Minikubeで実行されているKubernetesクラスタ内のサービスに外部からアクセスできるようにするためのコマンドです。特に以下のような場合に必要となります：

- LoadBalancerタイプのサービスを使用する場合
- NodePortサービスにローカルホストからアクセスする場合（Dockerドライバー使用時）
- ClusterIPサービスに外部からアクセスする場合

## WSL環境での課題

WSL（Windows Subsystem for Linux）環境でMinikubeを使用する際、以下のような課題があります：

1. **ネットワークの分離**: WSL2は独自の仮想ネットワークを持つため、Windows側とLinux側でネットワークが分離されています

2. **DockerドライバーでのNodePort制限**: MinikubeがDockerドライバーを使用している場合、NodePortへの直接アクセスに制限があります

3. **IPアドレスの不一致**: MinikubeのノードIP（例：192.168.49.2）は、Windows側から直接アクセスできません

## Minikube Tunnelの動作原理

`minikube tunnel`コマンドは以下のような動作をします：

1. **プロキシの作成**: ローカルホスト（127.0.0.1）とMinikubeクラスタ間にプロキシを作成します

2. **ポートフォワーディング**: サービスのポートをローカルホストのポートにフォワーディングします

3. **継続的な接続維持**: コマンドを実行している間、接続を維持し続けます

## 使用方法

### 1. LoadBalancerサービスの場合

```bash
# ターミナル1: トンネルを開始（管理者権限が必要な場合があります）
minikube tunnel

# ターミナル2: サービスの状態を確認
kubectl get svc
# EXTERNAL-IPが割り当てられていることを確認
```

### 2. NodePortサービスの場合（Dockerドライバー使用時）

```bash
# サービスのURLを取得して自動的にブラウザを開く
minikube service <service-name>

# URLのみを取得
minikube service <service-name> --url
```

### 3. 特定のサービスへのアクセス

```bash
# 例：testwebserver-testchartサービスにアクセス
minikube service testwebserver-testchart --url
```

このコマンドを実行すると、以下のような出力が得られます：
```
http://127.0.0.1:XXXXX
```

このURLをWindows側のブラウザで開くことで、サービスにアクセスできます。

## WSL環境での推奨設定

### 推奨: ClusterIPサービスを維持したままアクセスする方法

元のコードを変更せずに、ClusterIPタイプのサービスにWindows側からアクセスする方法を推奨します。

#### 方法1: kubectl port-forward を使用（最も推奨）

`kubectl port-forward`コマンドは、ローカルポートをKubernetesサービスまたはPodに転送します。

```bash
# サービスへのポートフォワード
kubectl port-forward svc/testwebserver-testchart 8080:80

# または特定のPodへのポートフォワード
kubectl port-forward pod/testwebserver-testchart-xxxxx 8080:80
```

アクセス方法：
- Windows側のブラウザで `http://localhost:8080` にアクセス
- Ctrl+C でポートフォワードを停止

#### 方法2: minikube service コマンドを使用

MinikubeはClusterIPサービスでも自動的にポートフォワードを設定できます：

```bash
# URLを取得して表示
minikube service testwebserver-testchart --url

# ブラウザを自動的に開く
minikube service testwebserver-testchart
```

### 具体的な実行例

1. **Helm chartのデプロイ**:
   ```bash
   cd /home/developer/tutorial-helm/section005/end/testchart
   helm install testwebserver .
   ```

2. **デプロイ状態の確認**:
   ```bash
   # Podの状態確認
   kubectl get pods -l app.kubernetes.io/instance=testwebserver
   
   # サービスの確認
   kubectl get svc testwebserver-testchart
   ```

3. **アクセス方法の選択**:

   **オプションA - kubectl port-forward（推奨）**:
   ```bash
   # ターミナル1で実行（実行したままにする）
   kubectl port-forward svc/testwebserver-testchart 8080:80
   ```
   
   別のターミナルまたはWindows側のブラウザで:
   ```
   http://localhost:8080
   ```

   **オプションB - minikube service**:
   ```bash
   # URLを取得
   minikube service testwebserver-testchart --url
   # 出力例: http://127.0.0.1:35791
   ```
   
   表示されたURLをWindows側のブラウザで開く

### NodePortを使用する場合（代替案）

コードを変更してNodePortを使用したい場合の設定：

`values.yaml`:
```yaml
service:
  type: NodePort
  port: 80
  nodePort: 30080  # 30000-32767の範囲で指定
```

`templates/service.yaml`:
```yaml
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
      {{- if and (eq .Values.service.type "NodePort") .Values.service.nodePort }}
      nodePort: {{ .Values.service.nodePort }}
      {{- end }}
```

その後：
```bash
helm upgrade testwebserver .
minikube service testwebserver-testchart --url
```

## 注意事項

1. **セッションの維持**: `minikube tunnel`や`minikube service`コマンドは、実行中のターミナルを開いたままにする必要があります

2. **管理者権限**: `minikube tunnel`は、ネットワーク設定を変更するため、管理者権限が必要な場合があります

3. **ポートの競合**: 既に使用されているポートとの競合に注意してください

4. **一時的なURL**: `minikube service --url`で生成されるURLは一時的なもので、セッションごとに変わります

## トラブルシューティング

### 接続が拒否される場合

1. Minikubeが起動していることを確認:
   ```bash
   minikube status
   ```

2. サービスとPodが正常に動作していることを確認:
   ```bash
   kubectl get pods
   kubectl get svc
   ```

3. サービスのエンドポイントを確認:
   ```bash
   kubectl get endpoints
   ```

### Windows Defenderファイアウォールの設定

Windows側でファイアウォールがブロックしている場合は、WSL2の通信を許可する必要があります。

## まとめ

WSL環境でMinikubeを使用する際は、`minikube service <service-name> --url`コマンドを使用することで、Windows側のブラウザからKubernetesサービスにアクセスできます。このコマンドは内部的にプロキシを設定し、ローカルホストからサービスへのアクセスを可能にします。