# Helm Chart の version と appVersion の違い

## 概要

Helm Chartの`Chart.yaml`には2つの異なるバージョン番号があります：
- `version`: Helm Chart自体のバージョン
- `appVersion`: デプロイされるアプリケーションのバージョン

これらは異なる目的を持ち、独立して管理されます。

## version（チャートバージョン）

### 定義
`version`は**Helm Chart自体のバージョン**を表します。これはチャートのパッケージング、テンプレート、設定の変更を追跡するためのものです。

### 特徴
- **Semantic Versioning（セマンティックバージョニング）**に従う必要があります
- フォーマット: `MAJOR.MINOR.PATCH` (例: `1.2.3`)
- チャートリポジトリで一意である必要があります
- 一度公開されたバージョンは変更してはいけません

### いつ更新するか
以下の変更を行った場合に更新します：
- テンプレートファイルの変更
- `values.yaml`のデフォルト値の変更
- 新しいKubernetesリソースの追加
- ヘルパー関数の変更
- 依存関係の変更

### バージョン番号の増やし方
```
MAJOR: 後方互換性のない変更
MINOR: 後方互換性のある新機能の追加
PATCH: バグ修正
```

## appVersion（アプリケーションバージョン）

### 定義
`appVersion`は**デプロイされるアプリケーション自体のバージョン**を表します。通常、Dockerイメージのタグと一致します。

### 特徴
- Semantic Versioningに従う必要はありません
- アプリケーションの実際のバージョンを反映すべきです
- クォートで囲むことが推奨されます（`"1.16.0"`）
- デフォルトのイメージタグとして使用されることが多い

### いつ更新するか
- アプリケーションの新しいバージョンがリリースされた時
- Dockerイメージの新しいタグが作成された時
- アプリケーションのコードが更新された時

## 実例

### 例1: アプリケーションのみ更新
アプリケーションのバグ修正をリリースした場合：

```yaml
# 変更前
version: 0.1.0
appVersion: "1.16.0"

# 変更後
version: 0.1.1  # チャートも更新（appVersionの変更はチャートの変更）
appVersion: "1.16.1"  # アプリケーションのバージョンを更新
```

### 例2: チャートのみ更新
チャートのテンプレートにバグ修正を加えた場合：

```yaml
# 変更前
version: 0.1.0
appVersion: "1.16.0"

# 変更後
version: 0.1.1  # チャートのバージョンを更新
appVersion: "1.16.0"  # アプリケーションは変更なし
```

### 例3: 両方更新
新機能追加とチャートの大幅な変更を行った場合：

```yaml
# 変更前
version: 0.1.0
appVersion: "1.16.0"

# 変更後
version: 1.0.0  # メジャーバージョンアップ（破壊的変更）
appVersion: "2.0.0"  # アプリケーションもメジャーアップデート
```

## ベストプラクティス

### 1. バージョン管理の独立性
- チャートとアプリケーションのバージョンは独立して管理する
- アプリケーションが更新されていなくても、チャートの改善があればチャートバージョンを更新

### 2. イメージタグとの連携
`values.yaml`でappVersionをデフォルトのイメージタグとして使用：

```yaml
# values.yaml
image:
  repository: nginx
  tag: ""  # 空の場合、Chart.yamlのappVersionが使用される
```

```yaml
# templates/deployment.yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

### 3. 変更履歴の記録
チャートバージョンごとの変更内容を記録：

```markdown
# CHANGELOG.md
## [0.2.0] - 2025-01-15
### Added
- Ingress サポートを追加
### Changed
- appVersion を 1.16.0 から 1.17.0 に更新

## [0.1.1] - 2025-01-10
### Fixed
- Service のラベルセレクターを修正
```

### 4. リリース前のチェック
```bash
# チャートの検証
helm lint ./mychart

# バージョンの確認
helm show chart ./mychart
```

## まとめ

| 項目 | version | appVersion |
|------|---------|------------|
| 対象 | Helm Chart | アプリケーション |
| 更新タイミング | チャートの変更時 | アプリの更新時 |
| バージョニング | Semantic Versioning必須 | 自由（SemVer推奨） |
| 影響範囲 | チャートのインストール/アップグレード | デプロイされるアプリのバージョン |
| 使用例 | `helm install mychart-0.1.0.tgz` | `image: nginx:1.16.0` |

これらを適切に管理することで、Helm Chartとアプリケーションの両方のライフサイクルを明確に追跡できます。

## GitHub Actions による自動更新

### 1. 基本的な自動更新ワークフロー

```yaml
name: Update Chart Versions

on:
  push:
    branches: [ main ]
    paths:
      - 'src/**'  # アプリケーションコードの変更を検知
  workflow_dispatch:
    inputs:
      version_type:
        description: 'Version bump type (patch, minor, major)'
        required: true
        default: 'patch'

jobs:
  update-versions:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Get current versions
        id: current
        run: |
          CHART_VERSION=$(grep '^version:' Chart.yaml | awk '{print $2}')
          APP_VERSION=$(grep '^appVersion:' Chart.yaml | awk '{print $2}' | tr -d '"')
          echo "chart_version=$CHART_VERSION" >> $GITHUB_OUTPUT
          echo "app_version=$APP_VERSION" >> $GITHUB_OUTPUT
      
      - name: Bump versions
        id: bump
        run: |
          # セマンティックバージョニングに基づいて更新
          npm install -g semver
          
          # Chart version の更新
          NEW_CHART_VERSION=$(semver -i ${{ github.event.inputs.version_type || 'patch' }} ${{ steps.current.outputs.chart_version }})
          
          # App version の更新（Git SHA または タグを使用）
          NEW_APP_VERSION="${{ github.sha }}"
          # または、タグベースの場合
          # NEW_APP_VERSION=$(git describe --tags --abbrev=0)
          
          echo "new_chart_version=$NEW_CHART_VERSION" >> $GITHUB_OUTPUT
          echo "new_app_version=$NEW_APP_VERSION" >> $GITHUB_OUTPUT
      
      - name: Update Chart.yaml
        run: |
          sed -i "s/^version:.*/version: ${{ steps.bump.outputs.new_chart_version }}/" Chart.yaml
          sed -i "s/^appVersion:.*/appVersion: \"${{ steps.bump.outputs.new_app_version }}\"/" Chart.yaml
      
      - name: Commit and push changes
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add Chart.yaml
          git commit -m "chore: bump chart version to ${{ steps.bump.outputs.new_chart_version }}"
          git push
```

### 2. Docker イメージと連動した更新

```yaml
name: Build and Update Chart

on:
  push:
    branches: [ main ]

jobs:
  build-and-update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Generate version
        id: version
        run: |
          # タイムスタンプベースのバージョン
          VERSION=$(date +%Y%m%d.%H%M%S)
          echo "version=$VERSION" >> $GITHUB_OUTPUT
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            myapp/image:${{ steps.version.outputs.version }}
            myapp/image:latest
      
      - name: Update Helm Chart appVersion
        run: |
          # Chart.yaml の appVersion を更新
          sed -i "s/^appVersion:.*/appVersion: \"${{ steps.version.outputs.version }}\"/" charts/myapp/Chart.yaml
          
          # values.yaml のイメージタグも更新
          sed -i "s/tag:.*/tag: ${{ steps.version.outputs.version }}/" charts/myapp/values.yaml
          
          # Chart version も自動インクリメント
          CURRENT_VERSION=$(grep '^version:' charts/myapp/Chart.yaml | awk '{print $2}')
          NEW_VERSION=$(echo $CURRENT_VERSION | awk -F. '{$NF = $NF + 1;} 1' | sed 's/ /./g')
          sed -i "s/^version:.*/version: $NEW_VERSION/" charts/myapp/Chart.yaml
      
      - name: Commit and push
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add charts/myapp/Chart.yaml charts/myapp/values.yaml
          git commit -m "chore: update app version to ${{ steps.version.outputs.version }}"
          git push
```

### 3. リリースタグと連動した更新

```yaml
name: Release Chart

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Get tag version
        id: tag
        run: |
          TAG=${GITHUB_REF#refs/tags/}
          VERSION=${TAG#v}
          echo "version=$VERSION" >> $GITHUB_OUTPUT
      
      - name: Update Chart versions
        run: |
          # タグからバージョンを設定
          sed -i "s/^version:.*/version: ${{ steps.tag.outputs.version }}/" Chart.yaml
          sed -i "s/^appVersion:.*/appVersion: \"${{ steps.tag.outputs.version }}\"/" Chart.yaml
      
      - name: Package Helm chart
        run: |
          helm package .
      
      - name: Upload to chart repository
        run: |
          # Chart Museum や GitHub Pages などにアップロード
          curl --data-binary "@myapp-${{ steps.tag.outputs.version }}.tgz" \
            http://chartmuseum.example.com/api/charts
```

### 4. 高度な自動化例

```yaml
name: Advanced Version Management

on:
  pull_request:
    types: [closed]
    branches: [main]

jobs:
  update-versions:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Determine version bump
        id: bump_type
        run: |
          # PR ラベルに基づいてバージョンアップの種類を決定
          if ${{ contains(github.event.pull_request.labels.*.name, 'major') }}; then
            echo "type=major" >> $GITHUB_OUTPUT
          elif ${{ contains(github.event.pull_request.labels.*.name, 'minor') }}; then
            echo "type=minor" >> $GITHUB_OUTPUT
          else
            echo "type=patch" >> $GITHUB_OUTPUT
          fi
      
      - name: Update versions
        run: |
          # 現在のバージョンを取得
          CURRENT_VERSION=$(grep '^version:' Chart.yaml | awk '{print $2}')
          
          # semver ツールを使用してバージョンを更新
          npm install -g semver
          NEW_VERSION=$(semver -i ${{ steps.bump_type.outputs.type }} $CURRENT_VERSION)
          
          # Chart.yaml を更新
          sed -i "s/^version:.*/version: $NEW_VERSION/" Chart.yaml
          
          # appVersion も更新（短縮コミットハッシュを使用）
          SHORT_SHA=$(git rev-parse --short HEAD)
          sed -i "s/^appVersion:.*/appVersion: \"$SHORT_SHA\"/" Chart.yaml
      
      - name: Create release commit
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add Chart.yaml
          git commit -m "chore(release): bump version to $NEW_VERSION [skip ci]"
          git push
```

## 自動更新のベストプラクティス

1. **バージョニング戦略の明確化**
   - Chart バージョン: SemVer に従う
   - App バージョン: Git SHA、タグ、またはタイムスタンプ

2. **自動化のトリガー**
   - main ブランチへのマージ時
   - タグのプッシュ時
   - 手動トリガー（workflow_dispatch）

3. **変更の追跡**
   - コミットメッセージに [skip ci] を含めて無限ループを防ぐ
   - CHANGELOG.md の自動生成も検討

4. **検証の追加**
   ```yaml
   - name: Validate Chart
     run: |
       helm lint .
       helm template . > /dev/null
   ```

5. **ブランチ保護**
   - main ブランチへの直接プッシュを制限
   - PR 経由でのみ更新を許可