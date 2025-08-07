# Helm チャート署名の実践ガイド - GPGコマンド

## 環境準備

### GPGバージョン確認

```bash
gpg --version
# gpg (GnuPG) 2.2.27
# libgcrypt 1.9.4
```

## 1. GPG鍵ペアの生成

### 対話式での鍵生成

```bash
gpg --full-generate-key
```

設定例：
- 鍵の種類: `1` (RSA and RSA)
- 鍵長: `3072` ビット
- 有効期限: `0` (無期限)
- ユーザー情報:
  - Real name: `wagatsuma.yuichi`
  - Email: `firefly.of.april@gmail.com`
  - Comment: `Windows11_WSL_Ubuntu22.04`

生成結果：
```
pub   rsa3072 2025-08-07 [SC]
      9A5EC7699281809357B474BE38CF4719788C9FB5
uid   wagatsuma.yuichi (Windows11_WSL_Ubuntu22.04) <firefly.of.april@gmail.com>
sub   rsa3072 2025-08-07 [E]
```

### 秘密鍵のエクスポート（レガシー形式）

```bash
# Helm 2.x互換のsecring.gpgファイル作成
gpg --export-secret-keys > ~/.gnupg/secring.gpg
```

## 2. 鍵の管理操作

### 鍵の一覧表示

```bash
# 公開鍵の一覧
gpg --list-keys

# 秘密鍵の一覧
gpg --list-secret-keys

# 鍵IDの短縮形式で表示
gpg --list-keys --keyid-format SHORT
```

### 公開鍵のエクスポート

```bash
# ASCII形式でエクスポート
gpg --armor --export wagatsuma.yuichi > public.key

# バイナリ形式でエクスポート
gpg --export wagatsuma.yuichi > public.gpg

# 鍵IDを使用してエクスポート
gpg --armor --export 9A5EC7699281809357B474BE38CF4719788C9FB5 > public.key
```

### 秘密鍵のエクスポート（バックアップ用）

```bash
# ASCII形式でエクスポート
gpg --armor --export-secret-keys wagatsuma.yuichi > private.key

# パスフレーズ付きでエクスポート
gpg --armor --export-secret-keys --cipher-algo AES256 wagatsuma.yuichi > private-encrypted.key
```

## 3. Helmチャートの署名

### チャートの作成とパッケージング

```bash
# チャート作成
helm create testchart

# チャートのパッケージング（署名なし）
helm package testchart/

# 署名付きパッケージング
helm package --sign --key wagatsuma.yuichi testchart/
```

### 既存パッケージへの署名追加

```bash
# provファイルの手動生成
helm package testchart/
gpg --armor --detach-sign testchart-0.1.0.tgz

# provファイルの確認
cat testchart-0.1.0.tgz.asc
```

### Helm 3での署名（推奨方法）

```bash
# helm-gpgプラグインのインストール
helm plugin install https://github.com/technosophos/helm-gpg

# プラグインを使用した署名
helm gpg sign testchart-0.1.0.tgz

# 署名の確認
helm gpg verify testchart-0.1.0.tgz
```

## 4. 署名の検証

### 公開鍵のインポート

```bash
# 公開鍵のインポート
gpg --import public.key

# インポートされた鍵の確認
gpg --list-keys

# 鍵の信頼レベル設定
gpg --edit-key wagatsuma.yuichi
# > trust
# > 5 (ultimate trust)
# > quit
```

### Helmでの署名検証

```bash
# インストール時の検証
helm install --verify myrelease ./testchart-0.1.0.tgz

# pullコマンドでの検証
helm pull --verify stable/mysql

# 手動検証
helm verify testchart-0.1.0.tgz

# GPGでの直接検証
gpg --verify testchart-0.1.0.tgz.prov testchart-0.1.0.tgz
```

## 5. 鍵サーバーの利用

### 公開鍵のアップロード

```bash
# keys.openpgp.orgへのアップロード
gpg --keyserver keys.openpgp.org --send-keys 9A5EC7699281809357B474BE38CF4719788C9FB5

# Ubuntu keyserverへのアップロード
gpg --keyserver keyserver.ubuntu.com --send-keys 9A5EC7699281809357B474BE38CF4719788C9FB5
```

### 公開鍵の検索とダウンロード

```bash
# メールアドレスで検索
gpg --keyserver keys.openpgp.org --search-keys firefly.of.april@gmail.com

# 鍵IDでダウンロード
gpg --keyserver keys.openpgp.org --recv-keys 9A5EC7699281809357B474BE38CF4719788C9FB5
```

## 6. provファイルの構造

### provファイルの手動作成

```bash
# チャートのメタデータ生成
cat <<EOF > testchart.yaml
apiVersion: v2
description: A Helm chart for Kubernetes
name: testchart
version: 0.1.0
EOF

# SHA256ハッシュの生成
sha256sum testchart-0.1.0.tgz

# provファイルの作成
cat testchart.yaml > testchart-0.1.0.tgz.prov
echo "" >> testchart-0.1.0.tgz.prov
echo "files:" >> testchart-0.1.0.tgz.prov
echo "  testchart-0.1.0.tgz: $(sha256sum testchart-0.1.0.tgz | cut -d' ' -f1)" >> testchart-0.1.0.tgz.prov

# 署名の追加
gpg --clearsign testchart-0.1.0.tgz.prov
mv testchart-0.1.0.tgz.prov.asc testchart-0.1.0.tgz.prov
```

## 7. トラブルシューティング

### 鍵が見つからない場合

```bash
# GPGホームディレクトリの確認
echo $GNUPGHOME

# デフォルトディレクトリの使用
export GNUPGHOME=~/.gnupg

# 鍵リングの再構築
gpg --rebuild-keydb-caches
```

### secring.gpgの問題（Helm 2.x互換）

```bash
# レガシー形式での秘密鍵エクスポート
gpg --export-secret-keys > ~/.gnupg/secring.gpg

# 権限の設定
chmod 600 ~/.gnupg/secring.gpg
```

### 署名検証エラー

```bash
# GPGエージェントの再起動
gpgconf --kill gpg-agent
gpg-agent --daemon

# キャッシュのクリア
rm -rf ~/.gnupg/*.d/S.gpg-agent*
```

## 8. CI/CDでの自動化

### GitHub Actionsでの署名

```yaml
name: Sign Helm Chart

on:
  push:
    tags:
      - 'v*'

jobs:
  sign:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Import GPG key
        run: |
          echo "${{ secrets.GPG_PRIVATE_KEY }}" | base64 -d | gpg --import
          echo "${{ secrets.GPG_PASSPHRASE }}" | gpg --batch --passphrase-fd 0 --pinentry-mode loopback --import
      
      - name: Package and sign chart
        run: |
          helm package --sign --key "${{ secrets.GPG_KEY_NAME }}" ./testchart
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: signed-chart
          path: |
            *.tgz
            *.tgz.prov
```

### Jenkins Pipeline

```groovy
pipeline {
    agent any
    
    stages {
        stage('Sign Chart') {
            steps {
                withCredentials([string(credentialsId: 'gpg-key', variable: 'GPG_KEY')]) {
                    sh '''
                        echo "${GPG_KEY}" | base64 -d | gpg --import
                        helm package --sign --key "wagatsuma.yuichi" ./testchart
                    '''
                }
            }
        }
    }
}
```

## 9. セキュリティのベストプラクティス

### 鍵の保護

```bash
# パスフレーズの変更
gpg --edit-key wagatsuma.yuichi
# > passwd
# > save

# 鍵の失効証明書生成
gpg --gen-revoke wagatsuma.yuichi > revoke-cert.asc

# サブキーの追加（署名専用）
gpg --edit-key wagatsuma.yuichi
# > addkey
# > 4 (RSA sign only)
# > save
```

### 鍵の定期更新

```bash
# 有効期限の設定
gpg --edit-key wagatsuma.yuichi
# > expire
# > 1y
# > save

# 新しい鍵への移行
gpg --full-generate-key
# 新しい鍵でチャートを再署名
```

## まとめ

このガイドでは、実際のGPGコマンドを使用したHelmチャートの署名と検証の手順を説明しました。重要なポイント：

1. **鍵の生成**: `gpg --full-generate-key`で3072ビット以上のRSA鍵を作成
2. **署名**: `helm package --sign`またはhelm-gpgプラグインを使用
3. **検証**: `helm install --verify`で署名を自動検証
4. **鍵管理**: 秘密鍵は厳重に保護し、公開鍵は鍵サーバーで配布

適切な鍵管理とPGP署名により、Helmチャートの信頼性と完全性を確保できます。