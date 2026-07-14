# 業務管理システム — CI/CD 用マニフェスト

## 概要

業務管理システムの CI/CD を OpenShift Pipelines（Tekton）で実行するためのマニフェストです。
バックエンド（Spring Boot）とフロントエンド（Next.js）の両方の
Pipeline、Task、Trigger、EventListener、RBAC、PVC を管理します。

このリポジトリは ArgoCD によって監視され、CI/CD 基盤そのものを GitOps で同期します。

## 技術スタック

| 技術 | 用途 |
|------|------|
| OpenShift Pipelines | Pipeline / Task / PipelineRun |
| Tekton Triggers | GitHub Webhook 受信と PipelineRun 生成 |
| Buildah | S2I 方式のコンテナイメージ作成 |
| OpenShift Route | EventListener の外部公開 |
| ArgoCD | CI/CD リソースの GitOps 同期 |

## ディレクトリ構成

```
manifests-cicd/
├── backend/
│   ├── pipeline.yaml       ← Task（3つ）+ Pipeline 定義
│   ├── eventlistener.yaml  ← TriggerBinding / TriggerTemplate / EventListener / Route
│   ├── rbac.yaml           ← Pipeline 実行用 ServiceAccount と権限
│   ├── pvc.yaml            ← Maven キャッシュ兼ワークスペース PVC
│   └── kustomization.yaml  ← backend CI/CD リソース一覧
│
└── frontend/
    ├── pipeline.yaml       ← Task（2つ）+ Pipeline 定義
    ├── eventlistener.yaml  ← TriggerBinding / TriggerTemplate / EventListener / Route
    ├── pvc.yaml            ← npm キャッシュ兼ワークスペース PVC
    ├── kustomization.yaml  ← frontend CI/CD リソース一覧
    └── README.md           ← フロントエンド CI/CD の解説
```

## バックエンド Pipeline

`backend-pipeline` は main ブランチへの push を対象に、以下の流れで実行されます。

```
git-clone → maven-build-test → s2i-java-build → git-update-manifest
```

| Task | 種別 | 説明 |
|------|------|------|
| `git-clone` | OpenShift 内蔵 | バックエンドソースコードを取得 |
| `maven-build-test` | カスタム | `mvn clean package` によるビルドとユニットテスト |
| `s2i-java-build` | カスタム | Buildah で OpenJDK ランタイムイメージを作成し内部レジストリへ push |
| `git-update-manifest` | カスタム（共用） | `bizapp-manifests-app` の `.argocd-source.yaml` を更新して push |

main 以外のブランチではビルドとテストのみを実行し、イメージ作成とデプロイは行いません（`when` 式で制御）。

## フロントエンド Pipeline

`frontend-pipeline` はバックエンドと同じ構成で、以下の流れで実行されます。

```
git-clone → npm-build-test → s2i-nodejs-build → git-update-manifest
```

| Task | 種別 | 説明 |
|------|------|------|
| `git-clone` | OpenShift 内蔵 | フロントエンドソースコードを取得 |
| `npm-build-test` | カスタム | `npm ci` / `npm run build` / `npm run test` |
| `s2i-nodejs-build` | カスタム | Buildah で Node.js ランタイムイメージを作成し内部レジストリへ push |
| `git-update-manifest` | バックエンドと共用 | `bizapp-manifests-app` の `.argocd-source.yaml` を更新して push |

### バックエンドとの共用リソース

フロントエンドは以下のリソースをバックエンド側の定義と共用するため、再定義していません。

| リソース | 定義場所 |
|---------|---------|
| `git-update-manifest` Task | `backend/pipeline.yaml` |
| `pipeline-sa` ServiceAccount | `backend/rbac.yaml` |
| ClusterRole / ClusterRoleBinding | `backend/rbac.yaml` |

## GitHub Webhook

バックエンドとフロントエンドそれぞれの Route URL を GitHub Webhook の Payload URL に設定します。

```bash
# バックエンド
oc get route backend-github-listener -n bizapp -o jsonpath='https://{.spec.host}{"\n"}'

# フロントエンド
oc get route frontend-github-listener -n bizapp -o jsonpath='https://{.spec.host}{"\n"}'
```

GitHub 側の設定：

| 項目 | 値 |
|------|----|
| Content type | `application/json` |
| Events | `Just the push event` |

## 必要な Secret

Pipeline がアプリ用マニフェストリポジトリへ push するため、`bizapp` Namespace に `github-credentials` Secret が必要です。

```bash
oc create secret generic github-credentials -n bizapp \
  --from-literal=username=<GitHubユーザー名> \
  --from-literal=token=<GitHub Token>
```

## RBAC

Pipeline 実行用 ServiceAccount `pipeline-sa` に以下の権限を付与します（`backend/rbac.yaml` で定義）。
フロントエンドパイプラインでも同じ ServiceAccount を共用します。

| 権限 | 用途 |
|------|------|
| `admin` | Namespace 内リソースの作成・更新 |
| `system:image-builder` | 内部レジストリへのイメージ push |
| `system:openshift:scc:privileged` | Buildah と PVC 書き込みの実行要件 |
| `bizapp-interceptor-reader` | CEL ClusterInterceptor の参照 |

## PVC とスケジューリング

バックエンド・フロントエンドそれぞれにキャッシュ兼ワークスペース PVC を定義しています。
OpenShift Pipelines の RWO PVC 制約を避けるため、PipelineRun では単一 PVC を共有し、サブディレクトリで用途を分けます。

### バックエンド（`maven-cache-pvc`）

| パス | 用途 |
|------|------|
| `source/` | ソースコードとビルド成果物 |
| `.m2/` | Maven 依存キャッシュ |
| `manifest/` | アプリ用マニフェスト更新作業 |

### フロントエンド（`npm-cache-pvc`）

| パス | 用途 |
|------|------|
| `source/` | ソースコードとビルド成果物 |
| `.npm/` | npm パッケージキャッシュ |
| `manifest/` | アプリ用マニフェスト更新作業 |

## ローカル検証

### Kustomize 出力確認

```bash
# バックエンド
oc kustomize backend

# フロントエンド
oc kustomize frontend
```

### OpenShift API での dry-run

```bash
# バックエンド
oc apply --dry-run=server -k backend -n bizapp

# フロントエンド
oc apply --dry-run=server -k frontend -n bizapp
```

## 関連リポジトリ

| リポジトリ | 用途 |
|-----------|------|
| [bizapp-backend](https://github.com/juhou-merlin/bizapp-backend) | バックエンドソースコード（Spring Boot） |
| [bizapp-frontend](https://github.com/juhou-merlin/bizapp-frontend) | フロントエンドソースコード（Next.js） |
| [bizapp-manifests-app](https://github.com/juhou-merlin/bizapp-manifests-app) | アプリ用マニフェスト（Kustomize） |
| [bizapp-bootstrap](https://github.com/juhou-merlin/bizapp-bootstrap) | 環境セットアップ（AppProject / Application） |
