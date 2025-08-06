# Helm OCI Registry ガイド

## OCI（Open Container Initiative）レジストリとは

OCI（Open Container Initiative）は、コンテナ技術の標準化を推進する組織で、OCI準拠のレジストリは、Dockerイメージだけでなく、**Helmチャート**も保存できます。従来のHTTPベースのHelmリポジトリに加えて、OCI準拠のコンテナレジストリを使用してHelmチャートをホスティングすることが可能です。

## OCI対応レジストリ

### 主要なOCI対応レジストリ

1. **Docker Hub**
2. **Amazon Elastic Container Registry (ECR)**
3. **Google Container Registry (GCR) / Artifact Registry**
4. **Azure Container Registry (ACR)**
5. **GitHub Container Registry (GHCR)**
6. **Harbor**
7. **Quay.io**
8. **JFrog Artifactory**

## 従来のHelmリポジトリとの違い

| 項目 | 従来のHelmリポジトリ | OCI Registry |
|------|-------------------|--------------|
| プロトコル | HTTP/HTTPS | OCI Distribution API |
| 認証 | Basic認証/Token | Docker認証 |
| index.yaml | 必要 | 不要 |
| 管理 | `helm repo` コマンド | `helm registry` コマンド |
| URL形式 | `https://charts.example.com` | `oci://registry.example.com/charts` |

## 基本的な使い方

### 1. レジストリへの認証

```bash
# Docker Hubの場合
helm registry login registry-1.docker.io

# GitHub Container Registryの場合
echo $GITHUB_TOKEN | helm registry login ghcr.io --username your-username --password-stdin

# AWS ECRの場合（aws-cli必要）
aws ecr get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Azure Container Registryの場合
helm registry login myregistry.azurecr.io --username myuser --password mypassword
```

### 2. チャートのプッシュ

```bash
# 1. チャートをパッケージ化
helm package mychart

# 2. OCI registryにプッシュ
helm push mychart-0.1.0.tgz oci://registry-1.docker.io/myusername

# 3. 複数のチャートをプッシュ
helm push mychart-0.1.0.tgz oci://ghcr.io/myorg/charts
helm push anotherchart-1.2.3.tgz oci://ghcr.io/myorg/charts
```

### 3. チャートのプル

```bash
# OCI registryからチャートをpull
helm pull oci://registry-1.docker.io/myusername/mychart --version 0.1.0

# 展開してpull
helm pull oci://ghcr.io/myorg/charts/mychart --version 0.1.0 --untar
```

### 4. チャートのインストール

```bash
# OCI registryから直接インストール
helm install my-release oci://registry-1.docker.io/myusername/mychart --version 0.1.0

# GitHub Container Registryからインストール
helm install my-app oci://ghcr.io/myorg/charts/myapp --version 1.2.3
```

## 具体的な実装例

### Docker Hubを使用した例

```bash
# 1. Docker Hubにログイン
helm registry login registry-1.docker.io
Username: mydockerhubuser
Password: [パスワードまたはアクセストークン]

# 2. チャートの準備
helm create webapp
helm package webapp

# 3. Docker Hubにプッシュ
helm push webapp-0.1.0.tgz oci://registry-1.docker.io/mydockerhubuser

# 4. プッシュされたチャートを確認
helm show chart oci://registry-1.docker.io/mydockerhubuser/webapp --version 0.1.0

# 5. チャートをインストール
helm install my-webapp oci://registry-1.docker.io/mydockerhubuser/webapp --version 0.1.0
```

### GitHub Container Registry (GHCR) を使用した例

```bash
# 1. GitHubトークンを設定
export GITHUB_TOKEN="ghp_xxxxxxxxxxxxxxxxxxxx"

# 2. GHCRにログイン
echo $GITHUB_TOKEN | helm registry login ghcr.io --username myusername --password-stdin

# 3. チャートをプッシュ
helm package mychart
helm push mychart-0.1.0.tgz oci://ghcr.io/myusername/helm-charts

# 4. 組織のリポジトリにプッシュ
helm push mychart-0.1.0.tgz oci://ghcr.io/myorganization/charts

# 5. プライベートリポジトリからインストール
helm install my-app oci://ghcr.io/myorganization/charts/mychart --version 0.1.0
```

### AWS ECRを使用した例

```bash
# 1. ECRレジストリの作成
aws ecr create-repository --repository-name helm-charts --region us-east-1

# 2. ECRにログイン
aws ecr get-login-password --region us-east-1 | \
  helm registry login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# 3. チャートをプッシュ
helm package mychart
helm push mychart-0.1.0.tgz oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm-charts

# 4. ECRからインストール
helm install my-app oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/helm-charts/mychart --version 0.1.0
```

### Azure Container Registry (ACR) を使用した例

```bash
# 1. ACRレジストリの作成
az acr create --resource-group myResourceGroup --name myregistry --sku Basic

# 2. ACRにログイン
az acr login --name myregistry
# または
helm registry login myregistry.azurecr.io --username myuser --password mypassword

# 3. チャートをプッシュ
helm package mychart
helm push mychart-0.1.0.tgz oci://myregistry.azurecr.io/helm-charts

# 4. ACRからインストール
helm install my-app oci://myregistry.azurecr.io/helm-charts/mychart --version 0.1.0
```

## 利用可能なコマンド

### helm registryコマンド

```bash
# レジストリにログイン
helm registry login [OPTIONS] [HOSTNAME]

# レジストリからログアウト
helm registry logout [HOSTNAME]
```

### helm pushコマンド

```bash
# チャートをOCI registryにプッシュ
helm push [chart] [remote] [flags]

# 使用例
helm push mychart-0.1.0.tgz oci://registry.example.com/charts
```

### helm pullコマンド（OCI対応）

```bash
# OCI registryからチャートをpull
helm pull oci://registry.example.com/charts/mychart --version 0.1.0

# オプション
--version    # 特定バージョンを指定
--untar      # 展開して保存
--destination # 保存先ディレクトリ
```

### helm installコマンド（OCI対応）

```bash
# OCI registryから直接インストール
helm install [NAME] oci://[REGISTRY]/[REPOSITORY]/[CHART] [flags]

# 使用例
helm install myapp oci://registry.example.com/charts/webapp --version 1.0.0
```

## CI/CDでの活用

### GitHub Actionsの例

```yaml
name: Build and Push Helm Chart

on:
  push:
    paths:
    - 'charts/**'
    branches:
    - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v3
      
    - name: Install Helm
      uses: azure/setup-helm@v3
      with:
        version: '3.12.0'
        
    - name: Log in to GitHub Container Registry
      run: |
        echo ${{ secrets.GITHUB_TOKEN }} | helm registry login ghcr.io --username ${{ github.actor }} --password-stdin
        
    - name: Package and Push Chart
      run: |
        for chart in charts/*/; do
          chart_name=$(basename $chart)
          helm package $chart
          chart_version=$(helm show chart $chart | grep '^version:' | cut -d' ' -f2)
          helm push ${chart_name}-${chart_version}.tgz oci://ghcr.io/${{ github.repository_owner }}/charts
        done
```

### GitLab CIの例

```yaml
stages:
  - package
  - push

package-chart:
  stage: package
  image: alpine/helm:latest
  script:
    - helm package charts/mychart
  artifacts:
    paths:
      - "*.tgz"
    expire_in: 1 hour

push-chart:
  stage: push
  image: alpine/helm:latest
  dependencies:
    - package-chart
  script:
    - echo $CI_REGISTRY_PASSWORD | helm registry login $CI_REGISTRY --username $CI_REGISTRY_USER --password-stdin
    - helm push *.tgz oci://$CI_REGISTRY/$CI_PROJECT_PATH/charts
  only:
    - main
```

## メリットとデメリット

### メリット

1. **統一された管理**: コンテナイメージとHelmチャートを同じレジストリで管理
2. **セキュリティ**: 既存のレジストリのセキュリティ機能を活用
3. **認証統合**: 既存のDocker認証を利用可能
4. **index.yaml不要**: レジストリ側でメタデータ管理
5. **スケーラビリティ**: エンタープライズレベルのレジストリを活用

### デメリット

1. **HTTP repository との互換性**: 既存のHTTPリポジトリとの完全な互換性なし
2. **学習コスト**: 新しいコマンド体系に慣れが必要
3. **レジストリ依存**: OCI対応レジストリが必要
4. **検索機能**: `helm search`が使えない（レジストリの検索機能に依存）

## ベストプラクティス

### 1. 適切な命名規則

```bash
# 組織/プロジェクト別の命名
oci://registry.example.com/myorg/charts/webapp
oci://registry.example.com/myorg/charts/database
oci://registry.example.com/myorg/charts/monitoring

# 環境別の命名
oci://registry.example.com/charts/prod/webapp
oci://registry.example.com/charts/staging/webapp
oci://registry.example.com/charts/dev/webapp
```

### 2. バージョン管理

```bash
# セマンティックバージョニング
helm push webapp-1.0.0.tgz oci://registry.example.com/charts
helm push webapp-1.1.0.tgz oci://registry.example.com/charts
helm push webapp-2.0.0.tgz oci://registry.example.com/charts

# タグによる管理
helm push webapp-1.0.0.tgz oci://registry.example.com/charts/webapp:stable
helm push webapp-1.1.0-beta.1.tgz oci://registry.example.com/charts/webapp:beta
```

### 3. セキュリティ

```bash
# 認証情報の安全な管理
export REGISTRY_PASSWORD="$(cat ~/.docker/config.json | jq -r '.auths["registry.example.com"].auth' | base64 -d | cut -d: -f2)"
echo $REGISTRY_PASSWORD | helm registry login registry.example.com --username myuser --password-stdin

# プライベートレジストリの使用
helm push mychart-0.1.0.tgz oci://private-registry.company.com/charts
```

### 4. 自動化

```bash
#!/bin/bash
# push-charts.sh

REGISTRY="oci://ghcr.io/myorg/charts"
CHARTS_DIR="charts"

for chart_dir in $CHARTS_DIR/*/; do
  if [ -f "$chart_dir/Chart.yaml" ]; then
    chart_name=$(basename "$chart_dir")
    echo "Packaging $chart_name..."
    
    helm package "$chart_dir"
    
    chart_version=$(helm show chart "$chart_dir" | grep '^version:' | awk '{print $2}')
    chart_file="${chart_name}-${chart_version}.tgz"
    
    echo "Pushing $chart_file to $REGISTRY..."
    helm push "$chart_file" "$REGISTRY"
    
    rm "$chart_file"
  fi
done
```

## トラブルシューティング

### よくあるエラーと対処法

#### 1. 認証エラー

```bash
Error: failed to authorize: failed to fetch oauth token
```

**対処法**:
```bash
# 再度ログイン
helm registry logout registry.example.com
helm registry login registry.example.com

# 認証情報の確認
cat ~/.docker/config.json
```

#### 2. プッシュ権限エラー

```bash
Error: failed to push: insufficient_scope: authorization failed
```

**対処法**:
- レジストリでの権限設定を確認
- 適切なpush権限があるトークンを使用

#### 3. チャートが見つからない

```bash
Error: failed to download "oci://registry.example.com/charts/mychart"
```

**対処法**:
```bash
# チャートの存在確認（レジストリのWeb UIまたはAPI）
# 正確なURL・バージョンを確認

# 利用可能なタグの確認（レジストリ固有のコマンド）
docker run --rm -it alpine/helm:latest registry-cli list oci://registry.example.com/charts/mychart
```

## まとめ

OCI registryを使用したHelmチャートホスティングは、従来のHTTPリポジトリに比べて以下の特徴があります：

- **統合性**: コンテナイメージと同じインフラで管理
- **セキュリティ**: エンタープライズレベルの認証・認可
- **スケーラビリティ**: 大規模な組織での利用に適している
- **標準化**: OCI標準に準拠した管理方法

適切に活用することで、より安全で効率的なHelmチャート配布が実現できます。