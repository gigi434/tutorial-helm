# Helm依存関係における@記法の使い方

## 概要
Helmの依存関係定義では、`repository`フィールドで`@`記法を使用してエイリアス（別名）を指定できます。これにより、事前に登録したリポジトリのエイリアスを使って、より簡潔に依存関係を定義できます。

## @記法の基本
### 通常の記法
```yaml
dependencies:
  - name: mysql
    version: ^13.0.0
    repository: https://charts.bitnami.com/bitnami
```

### @記法を使った記法
```yaml
dependencies:
  - name: mysql
    version: ^13.0.0
    repository: "@bitnami"
```

## 使い方

### 1. リポジトリの追加
まず、`helm repo add`コマンドでリポジトリをエイリアス付きで追加します：
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

### 2. リポジトリの確認
追加されたリポジトリを確認：
```bash
helm repo list
```

### 3. Chart.yamlでの使用
`Chart.yaml`の`dependencies`セクションで、`@`に続けてエイリアス名を指定：
```yaml
apiVersion: v2
name: testchart
version: 0.1.0
dependencies:
  - name: mysql
    version: ^13.0.0
    repository: "@bitnami"  # @記法でエイリアスを参照
```

### 4. 依存関係の更新
```bash
helm dependency update
```

## メリット

1. **簡潔性**: 長いURLを毎回書く必要がない
2. **一元管理**: リポジトリのURLが変更されても、`helm repo add`の更新だけで済む
3. **可読性**: どのリポジトリから取得しているかが明確
4. **標準化**: チーム内で共通のエイリアスを使用できる

## 注意点

- `@`記法を使用する前に、必ず`helm repo add`でリポジトリを追加しておく必要があります
- エイリアス名は大文字小文字を区別します
- リポジトリが追加されていない場合、`helm dependency update`はエラーになります

## 実例

### 複数の依存関係での使用
```yaml
dependencies:
  - name: mysql
    version: ^13.0.0
    repository: "@bitnami"
  - name: redis
    version: ~17.0.0
    repository: "@bitnami"
  - name: postgresql
    version: 12.x.x
    repository: "@bitnami"
```

このように、同じリポジトリから複数のチャートを使用する場合、@記法により定義がより簡潔になります。