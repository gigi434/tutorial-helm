# CLAUDE.md

このファイルは、Claude Code (claude.ai/code) がこのリポジトリで作業する際のガイダンスを提供します。

## プロジェクト概要

このリポジトリは、Helmチャート開発のさまざまな側面を示す複数のセクションを含むHelmチュートリアルリポジトリです。基本から高度なHelmの概念まで、段階的な例を含む学習パスとして構成されています。

## 主要コマンド

### Helmチャート開発

```bash
# 新しいチャートの作成
helm create <chartname>

# チャートのリント（構文チェック）
helm lint ./testchart

# チャートのパッケージ化
helm package ./testchart
helm package ./testchart -d ./chartrepo/  # 特定ディレクトリにパッケージ

# インストールせずにテンプレートをテスト
helm template ./testchart
helm install myrelease ./testchart --dry-run --debug

# チャートのインストール/アップグレード
helm install myrelease ./testchart
helm upgrade myrelease ./testchart

# テストの実行
helm test myrelease
```

### リポジトリ管理

```bash
# リポジトリインデックスの作成/更新（必ず先にチャートをパッケージ化すること）
helm repo index ./chartrepo/
helm repo index ./chartrepo/ --url https://example.com/charts

# OCIレジストリ操作
helm registry login <registry>
helm push <chart.tgz> oci://<registry>/<namespace>
```

### WSL/Minikube特有のコマンド

```bash
# WSLでサービスにアクセス
minikube service <service-name> --url
kubectl port-forward svc/<service-name> 8080:80
```

## リポジトリ構造

このリポジトリは、各セクションが特定のHelm機能を示すセクションベースの構造に従っています：

- **section004**: 基本的なHelmオプション（atomic、cleanup-on-fail）
- **section005**: チャートの基本、バージョニング、ヘルパー
- **section006**: テンプレート化、オートスケーリング、リソース管理
- **section007**: 依存関係、フック、テスト
  - **/67**: チャートの依存関係とタグ
  - **/68**: Helmフックの設定
  - **/71**: Helmテストの実装
- **section008**: リポジトリ管理、OCIレジストリ
- **section009**: 高度なトピック

各セクションには通常以下が含まれます：
- `end/testchart/`: 動作する例のチャート
- `docs/`: 日本語での詳細なドキュメント
- チャートコンポーネントは標準的なHelm構造に従う

## 重要なパターン

### フック設定
Helmフックを実装する際は、必ず以下を指定：
- `helm.sh/hook`: フックのタイミング（pre-install、post-installなど）
- `helm.sh/hook-delete-policy`: クリーンアップポリシー
- `restartPolicy`: JobsにはNeverまたはOnFailure

### テスト実装
テストPodは`helm.sh/hook: test`アノテーションを使用し、適切なエラーハンドリングを含める。WSL互換性のため、`nslookup`の代わりに`wget --spider`を使用。

### リポジトリインデックス管理
必ずこの順序に従う：
1. 最初にチャートをパッケージ化: `helm package <chart>`
2. その後インデックスを更新: `helm repo index <directory>`

パッケージ化前に`helm repo index`を実行すると、index.yamlが空になります。

### バージョン管理
このリポジトリはNode.jsバージョン管理にmise（旧rtx）を使用。設定は`mise.toml`にあり、セマンティックバージョニングをサポート。

## 環境に関する考慮事項

- 開発はWSL（Windows Subsystem for Linux）で行われる
- KubernetesクラスターはMinikube
- 一部のネットワークコマンドはWSLで異なる動作をする可能性がある
- より良いWSL互換性のため、`curl`や`nslookup`の代わりに`wget`を使用

## 言語

ドキュメントは主に日本語です。新しいドキュメントファイルを作成する際は、日本語で記述し、実践的な例を提供することで一貫性を保ってください。