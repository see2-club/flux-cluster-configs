# Configurations of Kubernetes Clusters in FluxCD

本 repository 存放基本的 Kubernetes cluster 設定檔，
並使用 FluxCD 進行建置。

## Multiple Tenants Structure

此專案存放大多數 cluster 會部屬的 resources，
其他各 cluster 專用的 resources 會放置在另外的 repository 中。

這個 repository 會把對應的 tenant repository 加到對應的 cluster 中，
這樣 cluster 就可以吃到其他 repository 的 manifests。

## Project Structure

- `clusters`: 存放各個 cluster 的設定，每個 cluster 應當要對應到一個目錄。
  - `flux-system`: FluxCD 的相關 manifests，除非更新或特殊情況，不然不要亂動。
  - `infrastructures.yaml`: 用來指定 infrastructure 的 repository manifest
    - 可以指定 commit 以鎖版
  - `tenant.yaml`: 用來指定 tenant 的 repository manifest
    - 可以指定 commit 以鎖版
- `tenants`: 負責每個 cluster 的 tenant repository。

## Manually Deploy

### Prerequisite

需要先在 `clusters/<cluster_name>` 下準備 FluxCD 及 cluster 用的 manifests

### Procedure


1. 在 Github 建立 PAT 並設置以下權限

- Administration -> Access: Read-only
- Contents -> Access: Read and write
- Metadata -> Access: Read-only

2. `kubectl create namespace flux-system`
3. `kubectl -n flux-system create secret generic flux-system --from-literal=username=flux --from-literal=password=<deploy_token>`
4. `cd clusters/<cluster_name>` 並遵循後續步驟，建立檔案
5. 建立 `gotk-components.yaml`，兩種方式則一
    - 使用 Flux CLI (建議): `flux install --export > gotk-components.yaml`
    - 不使用 Flux CLI: `curl -sL https://github.com/fluxcd/flux2/releases/latest/download/install.yaml > gotk-components.yaml`
6. 建立 `gotk-sync.yaml`

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  url: https://github.com/see2-club/flux-cluster-configs.git
  ref:
    branch: main
  interval: 1m0s
  timeout: 60s
  secretRef:
    name: flux-system
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./clusters/<cluster_name>
  force: false
  prune: true
  interval: 10m0s
```

7. `git commit`
8. `kubectl apply -f gotk-components.yaml`
9. 待部屬完成
10. `kubectl apply -f gotk-sync.yaml`
11. 確認部屬的 Kustomization Ready

## Upgrade

### Procedure

1. 升級 Flux CLI
2. 執行 `flux install --export > ./clusters/<cluster_name>/flux-system/gotk-components.yaml`
3. 注意升級須知及事項
4. commit & push 讓 FluxCD 部屬新的版本

## Deploy with Flux CLI

注意，此方法需要先使用具有寫入權限的 deploy token，才能 bootstrap，
目前不建議使用此方法安裝。

### Prerequisite

請先在電腦上安裝 flux-cli v2.0 以上

### Procedure

1. 在 Github 建立 PAT 並設置以下權限

- Administration -> Access: Read-only
- Contents -> Access: Read and write
- Metadata -> Access: Read-only
  
2. 將 kubectl 的 context 設定為要部屬的 cluster

3. 執行 `flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --token-auth \
  --owner=see2-club \
  --repository=flux-cluster-configs \
  --branch=main \
  --path=clusters/gke-prod`
