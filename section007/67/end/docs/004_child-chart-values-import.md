# Helmの子チャートから値をインポートする2つの方法

Helmでは、親チャートから子チャートの値を参照・インポートする方法が2つあります。

## 1. import-values を使用する方法

`Chart.yaml`の`dependencies`セクションで`import-values`を使用して、子チャートの値を親チャートにインポートできます。

### 例：

```yaml
# Chart.yaml
dependencies:
  - name: subchart
    version: "1.0.0"
    repository: "https://example.com/charts"
    import-values:
      - child: mysql.port
        parent: database.port
      - child: mysql.host
        parent: database.host
```

この設定により、子チャートの`mysql.port`と`mysql.host`の値が、親チャートの`database.port`と`database.host`として利用可能になります。

### import-valuesの書き方のバリエーション：

```yaml
import-values:
  # 子チャートのexportsセクションをすべてインポート
  - exports
  
  # 特定の値をマッピング
  - child: specific.value
    parent: new.location
```

## 2. グローバル値（global values）を使用する方法

`values.yaml`で`global`セクションを定義することで、すべての子チャートと共有される値を設定できます。

### 例：

```yaml
# 親チャートのvalues.yaml
global:
  database:
    host: "mysql.example.com"
    port: 3306
    username: "admin"
  
# 子チャート固有の値
subchart:
  replicaCount: 2
```

子チャートのテンプレート内では、以下のように参照できます：

```yaml
# 子チャートのテンプレート
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  db-host: {{ .Values.global.database.host }}
  db-port: {{ .Values.global.database.port | quote }}
```

## 使い分けのポイント

### import-valuesを使用する場合：
- 子チャートの特定の値を親チャートの名前空間に取り込みたい場合
- 子チャートの値を親チャートで異なる名前で参照したい場合
- 子チャートがexportsセクションを提供している場合

### globalを使用する場合：
- 複数の子チャート間で共通の設定を共有したい場合
- データベース接続情報やAPIエンドポイントなど、アプリケーション全体で使用される設定
- 環境ごとの設定（dev/staging/prod）を統一的に管理したい場合

## 注意点

1. **優先順位**: 値の優先順位は以下の通りです：
   - ユーザーが提供した値（--set または -f）
   - 親チャートの values.yaml
   - import-values でインポートされた値
   - 子チャートのデフォルト値

2. **名前の衝突**: import-valuesを使用する際は、親チャートの既存の値と名前が衝突しないよう注意が必要です。

3. **グローバル値の影響範囲**: globalセクションの値はすべての子チャートに影響するため、慎重に設計する必要があります。