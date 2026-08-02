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

Proxy 認証が必要な場合は、次の要素も必要になる。

```xml
<username>proxy-user</username>
<password>proxy-password</password>
```

認証情報を含む `settings.xml` は Git や ConfigMap に保存しない。Kubernetes
Secret として作成し、`maven-build-test-with-proxy.yaml` の volume 定義を
`configMap` から `secret` に変更する。

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
