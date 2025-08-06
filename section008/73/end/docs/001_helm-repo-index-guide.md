# Helm Repo Index ガイド

## helm repo indexとは

`helm repo index`は、Helmチャートリポジトリのインデックスファイル（`index.yaml`）を生成・更新するコマンドです。このインデックスファイルは、リポジトリ内で利用可能なすべてのチャートとそのメタデータを記録し、Helmクライアントがチャートを検索・取得するために使用されます。

## 基本的な使い方

### コマンド構文

```bash
helm repo index <directory> [flags]
```

### 基本例

```bash
# カレントディレクトリのindex.yamlを生成
helm repo index .

# 指定ディレクトリのindex.yamlを生成
helm repo index ./chartrepo

# URLを指定してindex.yamlを生成
helm repo index . --url https://example.com/charts
```

## index.yamlファイルの構造

生成されるindex.yamlは以下のような構造になります：

```yaml
apiVersion: v1
entries:
  chartname:
  - apiVersion: v2
    appVersion: 1.16.0
    created: "2025-08-06T13:45:39.554977307+09:00"
    description: A Helm chart for Kubernetes
    digest: 0b882f2c79cc9ea2edb530cf5790571e8c1b7a7b7287bfb85fb5a1105d11d89d
    name: chartname
    type: application
    urls:
    - chartname-0.1.0.tgz
    version: 0.1.0
generated: "2025-08-06T13:45:39.553515309+09:00"
```

### 各フィールドの説明

- **apiVersion**: インデックスファイルのAPIバージョン
- **entries**: チャートのエントリ一覧
- **created**: チャートがパッケージ化された日時
- **description**: チャートの説明
- **digest**: チャートパッケージのSHA256ハッシュ
- **name**: チャート名
- **type**: チャートタイプ（application/library）
- **urls**: チャートパッケージのダウンロードURL
- **version**: チャートバージョン
- **generated**: インデックスファイルの生成日時

## 正しい手順（重要）

チャートリポジトリを作成する際の**正しい順序**：

```bash
# 1. チャートディレクトリを作成
mkdir chartrepo

# 2. チャートをパッケージ化（必須：index.yamlより先に実行）
helm package mychart -d chartrepo/

# 3. index.yamlを生成・更新
helm repo index chartrepo/
```

### ❌ 間違った順序

```bash
# 先にindex.yamlを作成（空のファイルになる）
helm repo index chartrepo/

# 後からパッケージ化（index.yamlは更新されない）
helm package mychart -d chartrepo/
```

この順序では、パッケージファイルが存在しない状態でindex.yamlが生成されるため、空のインデックスファイルができてしまいます。

## コマンドオプション

### --url オプション

リモートリポジトリのベースURLを指定します。

```bash
# GitHubページでホストする場合
helm repo index . --url https://username.github.io/helm-charts/

# 独自ドメインでホストする場合
helm repo index . --url https://charts.example.com/
```

URLを指定した場合のindex.yaml：

```yaml
entries:
  mychart:
  - urls:
    - https://charts.example.com/mychart-0.1.0.tgz  # フルURL
```

### --merge オプション

既存のindex.yamlファイルと新しいチャートをマージします。

```bash
# 既存のindex.yamlと新しいチャートをマージ
helm repo index . --merge existing-index.yaml
```

### --json オプション

JSON形式で出力します。

```bash
helm repo index . --json
```

## 実践例

### 1. 基本的なローカルリポジトリの作成

```bash
# ディレクトリ構造の準備
mkdir my-helm-repo
cd my-helm-repo

# チャートの作成
helm create webapp
helm create api

# チャートのパッケージ化
helm package webapp
helm package api

# index.yamlの生成
helm repo index .

# 結果確認
ls -la
# webapp-0.1.0.tgz
# api-0.1.0.tgz
# index.yaml
```

### 2. GitHub Pagesでのホスティング

```bash
# gh-pagesブランチ用のディレクトリ
mkdir gh-pages
cd gh-pages

# チャートのパッケージ化
helm package ../mychart -d .

# GitHub PagesのURLを指定してindex.yaml生成
helm repo index . --url https://username.github.io/helm-charts/

# GitHubにプッシュ
git add .
git commit -m "Add helm chart"
git push origin gh-pages
```

### 3. 複数バージョンの管理

```bash
# 異なるバージョンのチャートをパッケージ化
helm package mychart-v1.0.0 -d repo/
helm package mychart-v1.1.0 -d repo/
helm package mychart-v2.0.0 -d repo/

# すべてのバージョンを含むindex.yamlを生成
helm repo index repo/
```

生成されるindex.yaml：

```yaml
entries:
  mychart:
  - version: 2.0.0
    urls: [mychart-2.0.0.tgz]
  - version: 1.1.0
    urls: [mychart-1.1.0.tgz]
  - version: 1.0.0
    urls: [mychart-1.0.0.tgz]
```

## トラブルシューティング

### よくあるエラーと対処法

#### 1. "no such file or directory" エラー

```bash
Error: open /path/to/chartrepo/index.yaml557198640: no such file or directory
```

**原因**: 指定したディレクトリが存在しない、または権限がない

**対処法**:
```bash
# ディレクトリの存在確認
ls -la chartrepo/

# ディレクトリの作成
mkdir -p chartrepo

# 権限の確認・修正
chmod 755 chartrepo/
```

#### 2. 空のindex.yamlファイル

```yaml
apiVersion: v1
entries: {}
generated: "2025-08-06T13:45:39.553515309+09:00"
```

**原因**: パッケージファイル（.tgz）が存在しない状態でindex.yamlを生成

**対処法**:
```bash
# 正しい順序で実行
helm package mychart -d chartrepo/
helm repo index chartrepo/
```

#### 3. URL設定の間違い

**問題**: 相対パスとURLが混在している

**対処法**:
```bash
# ローカル使用の場合（URLオプションなし）
helm repo index .

# リモートホスティングの場合
helm repo index . --url https://example.com/charts/
```

## ベストプラクティス

### 1. 自動化スクリプト

```bash
#!/bin/bash
# update-repo.sh

CHART_DIR="charts"
REPO_DIR="docs"  # GitHub Pages用

# チャートのパッケージ化
for chart in $CHART_DIR/*/; do
  helm package "$chart" -d "$REPO_DIR/"
done

# index.yamlの更新
helm repo index "$REPO_DIR/" --url "https://username.github.io/helm-charts/"

echo "Repository updated successfully"
```

### 2. CI/CDパイプライン（GitHub Actions）

```yaml
name: Update Helm Repository
on:
  push:
    paths:
    - 'charts/**'

jobs:
  update-repo:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Install Helm
      run: |
        curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    
    - name: Package and Index
      run: |
        helm package charts/* -d docs/
        helm repo index docs/ --url https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}/
    
    - name: Deploy to GitHub Pages
      uses: peaceiris/actions-gh-pages@v3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir: ./docs
```

### 3. バージョン管理

- セマンティックバージョニングを使用
- `Chart.yaml`の`version`フィールドを適切に更新
- 古いバージョンも保持してdowngradeに対応

```yaml
# Chart.yaml
version: 1.2.3  # パッチ更新
version: 1.3.0  # マイナー更新
version: 2.0.0  # メジャー更新
```

## まとめ

`helm repo index`は、Helmチャートリポジトリの中核となるインデックスファイルを管理する重要なコマンドです。正しい順序（パッケージ化→インデックス生成）を守り、適切なオプションを使用することで、効率的なチャート配布が可能になります。CI/CDパイプラインに組み込むことで、チャートの更新を自動化し、安定したリポジトリ運用が実現できます。