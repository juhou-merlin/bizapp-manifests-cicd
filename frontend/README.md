# フロントエンド CI/CD パイプライン（演習用）

## 概要

このディレクトリには、フロントエンド（Next.js）用の CI/CD パイプラインを
作成するための演習ガイドが含まれています。

バックエンドの CI/CD パイプライン（`../backend/`）を参考に、
フロントエンド用のパイプラインを自分で作成してください。

## 作成するファイル

バックエンドのファイル構成を参考に、以下のファイルを作成してください：

### 1. `pipeline.yaml`

以下のリソースを定義します：

#### カスタム Task: `npm-build-test`
- **参考**: バックエンドの `maven-build-test` Task
- **変更点**:
  - イメージ: `registry.access.redhat.com/ubi9/nodejs-20:latest`
    （Node.js 24が利用可能になったらそちらに変更）
  - スクリプト: `npm ci && npm run build && npm run test`
  - ワークスペース: `npm-cache`（npm パッケージキャッシュ用）

#### カスタム Task: `s2i-nodejs-build`
- **参考**: バックエンドの `s2i-java-build` Task
- **変更点**:
  - ビルダーイメージ: `registry.access.redhat.com/ubi9/nodejs-20-minimal:latest`
  - Dockerfile の COPY 対象:
    ```
    COPY .next/standalone/ /opt/app-root/src/
    COPY .next/static/ /opt/app-root/src/.next/static/
    COPY public/ /opt/app-root/src/public/
    ```
  - CMD: `["node", "server.js"]`

#### Pipeline: `frontend-pipeline`
- **参考**: バックエンドの `backend-pipeline`
- **変更点**:
  - `maven-build-test` → `npm-build-test` に置き換え
  - `s2i-java-build` → `s2i-nodejs-build` に置き換え
  - `maven-cache` → `npm-cache` ワークスペースに変更
  - `git-update-manifest` はそのまま流用（APP_NAME を "frontend" に）

### 2. `eventlistener.yaml`
- **参考**: バックエンドの `eventlistener.yaml`
- **変更点**:
  - 各リソース名を frontend 用に変更
  - PipelineRun が `frontend-pipeline` を参照するように変更
  - `npm-cache-pvc` をワークスペースに指定

### 3. `pvc.yaml`
- **参考**: バックエンドの `pvc.yaml`
- **変更点**:
  - 名前を `npm-cache-pvc` に変更
  - npm パッケージキャッシュ用に使用

### 4. `kustomization.yaml`
- **参考**: バックエンドの `kustomization.yaml`
- **変更点**:
  - フロントエンド用のリソースを参照

## 注意事項

- **rbac.yaml は不要**: バックエンドで定義した `pipeline-sa` をそのまま共用できます
- **git-update-manifest Task は共用**: バックエンドで定義済みのため、再定義は不要です
- **git-clone ClusterTask も共用**: OpenShift 内蔵の ClusterTask をそのまま使用します

## ヒント

- パイプラインの `when` 式によるブランチ戦略はバックエンドと同じ方式を使ってください
- npm キャッシュは `$(workspaces.npm-cache.path)/.npm` に配置すると効果的です
- Next.js の standalone モード（`output: 'standalone'`）を使うと S2I デプロイが容易です
