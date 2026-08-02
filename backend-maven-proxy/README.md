# Maven Proxy Solution for Tekton

このディレクトリには、ROSA / OpenShift Pipelines 上の Maven Task から
HTTP Proxy を経由して外部 Maven Repository に接続するための設定を格納する。

## 結論

ROSA の Cluster-wide Proxy は Tekton Step に `HTTP_PROXY`、`HTTPS_PROXY`、
`NO_PROXY` を注入するが、Maven 用の `settings.xml` は生成しない。

検証した Maven 3.9.9 環境では、標準 Proxy 環境変数および
`MAVEN_OPTS=-Dhttp.proxyHost=...` だけでは Maven Central に接続できなかった。
`settings.xml` の `<proxies>` を使用した場合は接続に成功した。

## ファイル

- `maven-proxy-settings.yaml`: HTTP Proxy を定義する ConfigMap
- `maven-proxy-settings-sample.yaml`: HTTPS・認証付き Proxy 用 Secret テンプレート
- `maven-build-test-with-proxy.yaml`: 既存 Task と置き換え可能な Maven Task
- `backend-pipeline-with-maven-proxy.yaml`: Proxy 対応 Task を参照する独立した Pipeline 定義
- `kustomization.yaml`: ConfigMap、Proxy 対応 Task、Pipeline のインストール

## 実環境の Proxy に合わせて変更する

適用前に `maven-proxy-settings.yaml` を開き、以下の値を利用する会社・環境の
Proxy 設定に置き換える。

| 項目 | サンプル値 | 変更内容 |
| --- | --- | --- |
| `host` | `proxy.example.com` | 会社の Proxy FQDN または IP アドレス |
| `port` | `8080` | 会社の Proxy 待受ポート |
| `protocol` | `http` | Proxy 自体への接続方式。一般的な CONNECT Proxy は `http` |
| `nonProxyHosts` | `localhost\|127.0.0.1\|*.svc\|*.cluster.local\|*.example.com` | Proxy を経由しない内部ホストとドメイン |

例えば会社の Proxy が `corporate-proxy.example.net:3128`、内部ドメインが
`*.internal.example.net` の場合は、次のように変更する。

```xml
<settings>
  <proxies>
    <proxy>
      <id>outbound-proxy</id>
      <active>true</active>
      <protocol>http</protocol>
      <host>corporate-proxy.example.net</host>
      <port>3128</port>
      <nonProxyHosts>localhost|127.0.0.1|*.svc|*.cluster.local|*.internal.example.net</nonProxyHosts>
    </proxy>
  </proxies>
</settings>
```

Maven Central など接続先が HTTPS でも、HTTP CONNECT Proxy を使用する場合は
`protocol` を `http` にする。これは Maven Repository の URL ではなく、
Proxy 自体への接続方式を示す。

変更後、Pod から次の条件を満たすことも確認する。

1. Proxy のホスト名を DNS 解決できる。
2. NetworkPolicy、Security Group、Firewall で Proxy のホストとポートへの通信が許可されている。
3. Maven Repository のホスト名が誤って `nonProxyHosts` に含まれていない。

### Protocol と認証方式の選択

`protocol` と認証情報は、Proxy サーバーの仕様に合わせて次のように設定する。

| Proxy 自体への接続 | Proxy 認証 | `protocol` | `username` / `password` |
| --- | --- | --- | --- |
| HTTP | なし | `http` | 記述しない |
| HTTP | あり | `http` | 両方を記述する |
| HTTPS | なし | `https` | 記述しない |
| HTTPS | あり | `https` | 両方を記述する |

認証を使用しない場合、`username` と `password` は空文字で残さず、要素自体を
削除する。

#### HTTP Proxy・認証なし

一般的な HTTP CONNECT Proxy の設定。HTTPS の Maven Repository に接続する
場合でも、Proxy 自体が HTTP であればこの設定を使用する。

```xml
<proxy>
  <id>outbound-proxy</id>
  <active>true</active>
  <protocol>http</protocol>
  <host>proxy.example.com</host>
  <port>8080</port>
</proxy>
```

#### HTTP Proxy・認証あり

```xml
<proxy>
  <id>authenticated-http-proxy</id>
  <active>true</active>
  <protocol>http</protocol>
  <host>proxy.example.com</host>
  <port>8080</port>
  <username>REPLACE_WITH_PROXY_USERNAME</username>
  <password>REPLACE_WITH_PROXY_PASSWORD</password>
</proxy>
```

#### HTTPS Proxy・認証なし

Proxy サーバー自体への接続に TLS を使用し、認証が不要な場合の設定。

```xml
<proxy>
  <id>outbound-https-proxy</id>
  <active>true</active>
  <protocol>https</protocol>
  <host>secure-proxy.example.com</host>
  <port>8443</port>
</proxy>
```

#### HTTPS Proxy・認証あり

`maven-proxy-settings-sample.yaml` は、この組み合わせの Secret テンプレートで
ある。

```xml
<proxy>
  <id>authenticated-https-proxy</id>
  <active>true</active>
  <protocol>https</protocol>
  <host>secure-proxy.example.com</host>
  <port>8443</port>
  <username>REPLACE_WITH_PROXY_USERNAME</username>
  <password>REPLACE_WITH_PROXY_PASSWORD</password>
</proxy>
```

`protocol` は Maven Repository の URL ではなく、Proxy サーバー自体への接続
方式を示す。HTTPS Proxy を使用する場合は、Proxy のサーバー証明書をコンテナが
信頼できることも確認する。

認証情報を含む `settings.xml` は Git や ConfigMap に保存しない。リポジトリ内
の `maven-proxy-settings-sample.yaml` に記載されたユーザー名とパスワードは、
書式を示すための架空のサンプル値であり、実際の認証情報ではない。実際の値への
置き換えは Git 管理外で行い、そのファイルを Kubernetes Secret として適用する。
実際の認証情報へ置き換えたファイルはコミットしない。

Secret を使用する場合は、`maven-build-test-with-proxy.yaml` の volume 定義を
`configMap` から次の `secret` 定義に変更する。

```yaml
volumes:
  - name: maven-proxy-settings
    secret:
      secretName: maven-proxy-settings
      items:
        - key: settings.xml
          path: settings.xml
```

## インストール

`bizapp` namespace に ConfigMap、Proxy 対応 Task、Pipeline を作成する。

```bash
oc apply -k manifests-cicd/backend-maven-proxy -n bizapp
```

この Kustomize 設定は、既存の `backend-pipeline` を変更せず、Proxy 対応済みの
`backend-pipeline-proxy-fixed` を別の Pipeline として作成する。

PipelineRun 側に Proxy 設定を追加する必要はない。Task が ConfigMap を
`/etc/maven-proxy/settings.xml` に read-only でマウントし、Maven を
次の形式で実行する。このコマンドは Task 内に定義済みのため、利用者が
手動で実行する必要はない。

```bash
mvn -s /etc/maven-proxy/settings.xml clean package
```

## 注意事項

- Proxy のホスト名を Pod から DNS 解決できることを確認する。
- `proxy.example.com:8080` はサンプル値のため、環境の Proxy アドレスに変更する。
- Pod から設定した Proxy ホストとポートへの通信を許可する。
- Proxy 認証が必要な場合、ユーザー名とパスワードを ConfigMap に保存しない。
  Secret を使用して `settings.xml` を提供する。
- 内部ドメインを Proxy 経由にしない場合は、対象ドメインを `nonProxyHosts`
  に追加する。
