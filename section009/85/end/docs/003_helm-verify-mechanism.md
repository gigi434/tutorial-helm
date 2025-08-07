# Helm署名検証の内部メカニズム

## `helm install --verify`の動作原理

### 1. 検証プロセスの概要

`helm install --verify`コマンドを実行すると、Helmは以下の順序で署名検証を行います：

```
1. チャートファイル (.tgz) の存在確認
2. 署名ファイル (.tgz.prov) の自動検索
3. GPG公開鍵の検索
4. 署名の検証
5. ハッシュ値の照合
```

### 2. 公開鍵の検索場所と優先順位

Helmは以下の場所から公開鍵を検索します（優先順位順）：

#### 2.1 デフォルトのGPGキーリング

```bash
# Helmが最初に確認する場所
~/.gnupg/pubring.gpg     # GPG v1形式（レガシー）
~/.gnupg/pubring.kbx     # GPG v2形式（現在の標準）

# 確認コマンド
ls -la ~/.gnupg/
```

#### 2.2 環境変数による指定

```bash
# GNUPGHOMEの設定
export GNUPGHOME=/custom/path/.gnupg
helm install --verify myrelease ./testchart-0.1.0.tgz

# Helm特有の環境変数
export HELM_KEYRING=~/my-keys/pubring.gpg
helm install --verify myrelease ./testchart-0.1.0.tgz
```

#### 2.3 コマンドラインオプション

```bash
# --keyringオプションで明示的に指定
helm install --verify --keyring ~/.gnupg/pubring.gpg myrelease ./testchart-0.1.0.tgz

# 複数のキーリングを指定
helm install --verify \
  --keyring ~/.gnupg/pubring.gpg \
  --keyring /etc/helm/keys/pubring.gpg \
  myrelease ./testchart-0.1.0.tgz
```

### 3. 署名検証の詳細な流れ

#### Step 1: Provファイルの読み込み

```bash
# Helmは自動的に.provファイルを探す
testchart-0.1.0.tgz      # チャートファイル
testchart-0.1.0.tgz.prov  # 署名ファイル（自動検出）
```

#### Step 2: GPG検証の実行

内部的には以下のようなGPGコマンドが実行されます：

```bash
# Helmが内部で実行する検証（簡略化）
gpg --verify testchart-0.1.0.tgz.prov testchart-0.1.0.tgz

# 実際の検証プロセス
gpg --status-fd 1 \
    --keyring ~/.gnupg/pubring.gpg \
    --verify testchart-0.1.0.tgz.prov \
    testchart-0.1.0.tgz
```

#### Step 3: ハッシュ値の照合

```yaml
# .provファイル内のハッシュ値
files:
  testchart-0.1.0.tgz: sha256:abc123def456...

# Helmは以下を実行
# 1. 実際のファイルのSHA256を計算
# 2. .provファイル内のハッシュ値と比較
# 3. 一致しない場合はエラー
```

### 4. 公開鍵のインポートと管理

#### 4.1 基本的なインポート方法

```bash
# 公開鍵のインポート（GPGキーリングに追加）
gpg --import developer-public.key

# インポート結果の確認
gpg --list-keys
# pub   rsa3072 2025-08-07 [SC]
#       9A5EC7699281809357B474BE38CF4719788C9FB5
# uid   [ultimate] wagatsuma.yuichi <firefly.of.april@gmail.com>
```

#### 4.2 Helm専用キーリングの作成

```bash
# 専用キーリングディレクトリの作成
mkdir -p ~/.helm/keys

# Helm用の公開鍵リングを作成
gpg --no-default-keyring \
    --keyring ~/.helm/keys/pubring.gpg \
    --import developer-public.key

# 検証時に専用キーリングを使用
helm install --verify \
    --keyring ~/.helm/keys/pubring.gpg \
    myrelease ./testchart-0.1.0.tgz
```

### 5. 検証エラーのトラブルシューティング

#### 5.1 公開鍵が見つからない場合

```bash
# エラーメッセージ例
Error: failed to verify chart: openpgp: signature made by unknown entity

# 解決方法1: 公開鍵の確認
gpg --list-keys | grep -A2 "署名者のメールアドレス"

# 解決方法2: 公開鍵の再インポート
gpg --import chart-signer-public.key

# 解決方法3: キーリングパスの明示
helm install --verify \
    --keyring $(gpg --list-keys | grep "pubring" | head -1) \
    myrelease ./testchart-0.1.0.tgz
```

#### 5.2 GPG v1とv2の互換性問題

```bash
# GPG v2のキーリングをv1形式にエクスポート
gpg --export > ~/.gnupg/pubring.gpg

# 権限の設定
chmod 644 ~/.gnupg/pubring.gpg
```

### 6. 実践例：完全な検証フロー

```bash
# 1. 署名者の公開鍵を取得
curl -O https://example.com/keys/chart-signer.asc

# 2. 公開鍵をインポート
gpg --import chart-signer.asc

# 3. インポートされた鍵の確認
gpg --list-keys --keyid-format LONG

# 4. チャートと署名ファイルのダウンロード
helm pull stable/mysql --version 1.6.9
# mysql-1.6.9.tgz
# mysql-1.6.9.tgz.prov (自動でダウンロード)

# 5. 署名の検証付きインストール
helm install --verify my-mysql ./mysql-1.6.9.tgz

# 6. 検証の詳細ログを確認
helm install --verify --debug my-mysql ./mysql-1.6.9.tgz
```

### 7. プログラム的な検証の仕組み

Helmの内部では、以下のような処理が行われています：

```go
// 簡略化されたHelm内部の検証ロジック
func verifyChart(chartPath string) error {
    // 1. .provファイルの存在確認
    provFile := chartPath + ".prov"
    if !fileExists(provFile) {
        return errors.New("provenance file not found")
    }
    
    // 2. キーリングの検索
    keyring := findKeyring() // ~/.gnupg/pubring.gpg or $HELM_KEYRING
    
    // 3. GPG検証の実行
    cmd := exec.Command("gpg", 
        "--keyring", keyring,
        "--verify", provFile, chartPath)
    
    // 4. ハッシュ値の検証
    actualHash := calculateSHA256(chartPath)
    expectedHash := extractHashFromProv(provFile)
    
    if actualHash != expectedHash {
        return errors.New("hash mismatch")
    }
    
    return nil
}
```

### 8. ベストプラクティス

#### 8.1 組織での鍵管理

```bash
# 組織用のキーリングディレクトリ構造
/etc/helm/
├── keys/
│   ├── trusted-signers.gpg    # 信頼する署名者の公開鍵
│   ├── vendor-keys.gpg         # ベンダー提供の公開鍵
│   └── internal-keys.gpg       # 内部開発者の公開鍵

# 環境変数での設定
export HELM_KEYRING=/etc/helm/keys/trusted-signers.gpg
```

#### 8.2 CI/CDでの自動検証

```yaml
# GitHub Actions例
- name: Setup GPG keys
  run: |
    # 信頼する公開鍵をインポート
    echo "${{ secrets.TRUSTED_KEYS }}" | base64 -d | gpg --import
    
- name: Verify and install chart
  run: |
    # 検証付きインストール
    helm install --verify --wait \
      myapp ./charts/myapp-1.0.0.tgz
```

### 9. セキュリティ考慮事項

#### 9.1 鍵の信頼レベル

```bash
# 鍵の信頼レベルを設定
gpg --edit-key chart-signer@example.com
# gpg> trust
# Your decision? 4  # I trust fully
# gpg> save

# 信頼レベルの確認
gpg --list-keys --list-options show-uid-validity
```

#### 9.2 検証の強制

```bash
# Helmの設定で常に検証を要求
cat <<EOF > ~/.helm/config.yaml
verify: true
keyring: ~/.helm/keys/trusted.gpg
EOF

# エイリアスで検証を強制
alias helm-install='helm install --verify'
```

### まとめ

`helm install --verify`は以下の順序で動作します：

1. **自動検索**: `.tgz.prov`ファイルを自動的に探す
2. **鍵の探索**: `~/.gnupg/pubring.gpg`または`pubring.kbx`から公開鍵を検索
3. **GPG検証**: 署名の正当性を確認
4. **ハッシュ照合**: ファイルの完全性を確認

公開鍵が見つからない場合は、`--keyring`オプションで明示的に指定するか、事前に`gpg --import`で公開鍵をインポートする必要があります。