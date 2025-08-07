# Helmチャートのセキュリティ - PGPによる署名と検証

## 概要

Helmでは、チャートの配布時にセキュリティを確保するため、以下の2つの重要な要素を検証する仕組みを提供しています：

1. **チャートプロバイダーの正当性** - チャートが信頼できる作成者からのものか
2. **チャートの完全性** - チャートが改ざんされていないか

これらの検証には、**PGP（Pretty Good Privacy）** を使用した公開鍵暗号方式が一般的に採用されています。

## PGPとは

### 基本概念

**PGP (Pretty Good Privacy)** は、電子メールやファイルの暗号化・デジタル署名を行うためのソフトウェアおよびプロトコルです。

#### 主な特徴

- **ハイブリッド暗号方式**: 公開鍵暗号と共通鍵暗号を組み合わせた方式
- **機密性**: 受信者の公開鍵でメッセージを暗号化、受信者は自身の秘密鍵で復号
- **認証性**: 送信者の秘密鍵でデジタル署名、受信者は送信者の公開鍵で署名を検証

### PGPの規格と実装

- **OpenPGP**: PGPの仕様を標準化した規格（RFC 4880）
- **GnuPG (GPG)**: GNU Privacy Guard - OpenPGPのフリーソフトウェア実装
- Helmでは主にGPGを使用してチャートの署名・検証を行います

### 利用可能な暗号アルゴリズム

PGPでは、RSA以外にも様々な公開鍵暗号方式が利用可能です：

| アルゴリズム | 用途 | 特徴 |
|------------|------|------|
| **RSA** | 暗号化・署名 | 最も広く利用される標準的な方式 |
| **ElGamal** | 暗号化 | 楕円曲線暗号の一種 |
| **DSA** | 署名専用 | デジタル署名に特化 |
| **ECDSA** | 署名 | 楕円曲線DSA、より短い鍵長で同等の強度 |
| **ECDH** | 鍵共有 | 楕円曲線Diffie-Hellman |

## HelmにおけるPGP署名の仕組み

### 署名の流れ

```
1. チャート作成者
   ├── チャートをパッケージ化 (.tgz)
   ├── 秘密鍵で署名を生成
   └── 署名ファイル (.prov) を作成

2. チャート配布
   ├── チャートファイル (.tgz)
   ├── 署名ファイル (.prov)
   └── 公開鍵の公開

3. チャート利用者
   ├── 公開鍵を取得・インポート
   ├── チャートと署名をダウンロード
   └── 公開鍵で署名を検証
```

### Helmでの実装

#### 1. GPG鍵の準備

```bash
# GPG鍵ペアの生成
gpg --gen-key

# 公開鍵のエクスポート
gpg --export -a "Your Name" > public.key

# 秘密鍵のエクスポート（バックアップ用）
gpg --export-secret-keys -a "Your Name" > private.key
```

#### 2. チャートの署名

```bash
# チャートのパッケージ化と同時に署名
helm package --sign --key "Your Name" ./mychart

# または既存のパッケージに署名
helm gpg sign mychart-0.1.0.tgz
```

これにより以下のファイルが生成されます：
- `mychart-0.1.0.tgz` - チャートパッケージ
- `mychart-0.1.0.tgz.prov` - 署名ファイル（provenance file）

#### 3. 署名の検証

```bash
# 公開鍵のインポート
gpg --import public.key

# 署名付きチャートのインストール（自動検証）
helm install --verify myrelease ./mychart-0.1.0.tgz

# チャートのダウンロード時に検証
helm pull --verify stable/mysql

# 手動での署名検証
helm verify mychart-0.1.0.tgz
```

### .provファイルの構造

`.prov`ファイルは、チャートのメタデータとPGP署名を含むYAML形式のファイルです：

```yaml
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA256

apiVersion: v2
description: A Helm chart for Kubernetes
name: mychart
version: 0.1.0
...

files:
  mychart-0.1.0.tgz: sha256:1234567890abcdef...

-----BEGIN PGP SIGNATURE-----

iQEzBAEBCAAdFiEE...
...
-----END PGP SIGNATURE-----
```

## セキュリティのベストプラクティス

### 1. 鍵管理

- **秘密鍵の保護**: 秘密鍵は厳重に管理し、パスフレーズで保護
- **鍵の有効期限**: 定期的な鍵の更新（推奨：1-2年）
- **鍵の失効**: 漏洩時の失効証明書の準備

### 2. 公開鍵の配布

```bash
# Keybaseを使用した公開鍵の管理
keybase pgp export > public.key

# 鍵サーバーへの登録
gpg --keyserver keys.openpgp.org --send-keys YOUR_KEY_ID
```

### 3. リポジトリレベルのセキュリティ

```yaml
# index.yamlに公開鍵を含める
apiVersion: v1
entries:
  mychart:
  - apiVersion: v2
    name: mychart
    version: 0.1.0
    digest: sha256:...
    urls:
    - https://example.com/charts/mychart-0.1.0.tgz
    keyring: https://example.com/pubring.gpg
```

### 4. CI/CDでの自動署名

```yaml
# GitHub Actionsの例
- name: Sign Helm Chart
  run: |
    echo "${{ secrets.GPG_PRIVATE_KEY }}" | gpg --import
    helm package --sign --key "${{ secrets.GPG_KEY_NAME }}" ./chart
```

## 新しいセキュリティ技術の動向

### Sigstore/Rekor

PGP以外の新しいアプローチとして、sigstoreプロジェクトが注目されています：

```bash
# Helmプラグインのインストール
helm plugin install https://github.com/sigstore/helm-sigstore

# sigstoreを使用した署名
helm sigstore sign mychart-0.1.0.tgz

# sigstoreを使用した検証
helm sigstore verify mychart-0.1.0.tgz
```

### OCI Registryでの署名

OCI（Open Container Initiative）レジストリを使用する場合：

```bash
# Cosignを使用した署名
cosign sign oci://registry.example.com/charts/mychart:0.1.0

# 検証
cosign verify oci://registry.example.com/charts/mychart:0.1.0
```

## PGPの課題と対策

### 課題

1. **鍵管理の複雑さ**: 秘密鍵の安全な管理が難しい
2. **利用の煩雑さ**: 初心者にとって設定や操作が複雑
3. **Web of Trust**: 信頼の連鎖の構築が困難

### 対策

1. **自動化**: CI/CDパイプラインでの自動署名
2. **ドキュメント化**: 組織内での鍵管理手順の明文化
3. **トレーニング**: チームメンバーへのセキュリティ教育

## まとめ

Helmチャートのセキュリティにおいて、PGPによる署名と検証は現在も主流の方法です。これにより：

- チャートの作成者を確実に特定できる
- チャートが改ざんされていないことを保証できる
- 信頼できるチャートのみをクラスタにデプロイできる

適切な鍵管理と運用プロセスを確立することで、Helmチャートの配布と利用における高いセキュリティレベルを維持できます。

## 参考リンク

- [Helm Chart Signing and Verification](https://helm.sh/docs/topics/provenance/)
- [OpenPGP RFC 4880](https://tools.ietf.org/html/rfc4880)
- [GnuPG Documentation](https://gnupg.org/documentation/)
- [Sigstore Project](https://www.sigstore.dev/)