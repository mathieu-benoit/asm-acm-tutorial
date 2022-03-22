```
export PROJECT_ID=$(gcloud info --format='value(config.project)')
export CLUSTER=asm-acm-cluster-2
export CLUSTER_ZONE=us-west2-a
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='get(projectNumber)')
```

## Enable APIs

```
gcloud services enable \
    container.googleapis.com \
    gkehub.googleapis.com \
    mesh.googleapis.com \
    anthos.googleapis.com
```

## Create cluster

```
gcloud container clusters create ${CLUSTER} \
  --zone ${CLUSTER_ZONE} \
  --machine-type=e2-standard-2 \
  --workload-pool ${PROJECT_ID}.svc.id.goog \
  --labels mesh_id=proj-${PROJECT_NUMBER} \
  --enable-dataplane-v2
```

## Register cluster to the Fleet

```
gcloud container hub memberships register ${CLUSTER} \
    --gke-cluster ${CLUSTER_ZONE}/${CLUSTER} \
    --enable-workload-identity
```

## Enable ASM and ACM

```
gcloud beta container hub mesh enable
gcloud beta container hub config-management enable
```

## Configure ASM MCP

```
gcloud alpha container hub mesh update \
    --control-plane automatic \
    --membership ${CLUSTER}
```

## Initialize RootSync + RepoSync repos

```
cat <<EOF > acm-config.yaml
applySpecVersion: 1
spec:
  policyController:
    enabled: true
    templateLibraryInstalled: true
    referentialRulesEnabled: true
  configSync:
    enabled: true
    sourceFormat: unstructured
    syncRepo: https://github.com/mathieu-benoit/asm-acm-tutorial
    syncBranch: main
    secretType: none
    policyDir: root-sync/deployments/init-reposync
EOF
gcloud beta container hub config-management apply \
    --membership ${CLUSTER} \
    --config acm-config.yaml
```

Checks:
```
gcloud beta container hub config-management status
gcloud alpha anthos config sync repo describe \
    --managed-resources all \
    --sync-name root-sync \
    --sync-namespace config-management-system
```
```
Name               Status         Last_Synced_Token  Sync_Branch  Last_Synced_Time      Policy_Controller  Hierarchy_Controller
asm-acm-cluster-2  SYNCED         f9969f0            main         2022-03-22T13:03:21Z  INSTALLED          PENDING
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                        managed_resources                                         │
├───────────────────────────┬─────────────┬────────────────┬────────────────┬─────────┬────────────┤
│           GROUP           │     KIND    │      NAME      │   NAMESPACE    │  STATUS │ CONDITIONS │
├───────────────────────────┼─────────────┼────────────────┼────────────────┼─────────┼────────────┤
│                           │ Namespace   │ onlineboutique │                │ Current │            │
│ configsync.gke.io         │ RepoSync    │ repo-sync      │ onlineboutique │ Current │            │
│ rbac.authorization.k8s.io │ RoleBinding │ repo-sync      │ onlineboutique │ Current │            │
└───────────────────────────┴─────────────┴────────────────┴────────────────┴─────────┴────────────┘
```

## Deploy Policies

```
sed -i "s/init-reposync/deploy-policies/g" acm-config.yaml
gcloud beta container hub config-management apply \
    --membership ${CLUSTER} \
    --config acm-config.yaml
```

Checks:
```
gcloud beta container hub config-management status
```
```
Name               Status         Last_Synced_Token  Sync_Branch  Last_Synced_Time      Policy_Controller  Hierarchy_Controller
asm-acm-cluster-2  ERROR          f9969f0            main         2022-03-22T13:07:43Z  INSTALLED          PENDING
 - cluster: asm-acm-cluster-2
   error: KNV2009: failed to apply Namespace, /onlineboutique: admission webhook "validation.gatekeeper.sh" denied the request: [must-have-istio-sidecar-injection] you must provide labels: {"istio.io/rev"}
```

## Next deployments

- ASM
- Policies
- Ingress Gateway
- mTLS STRICT
- OnlineBoutique deployments
- OnlineBoutique sidecar injection label
- OnlineBoutique authzpol

```
sed -i "s/init-reposync/deploy-policies/g" acm-config.yaml
gcloud beta container hub config-management apply \
    --membership ${CLUSTER} \
    --config acm-config.yaml
```

```
gcloud alpha anthos config sync repo describe \
    --managed-resources all \
    --sync-name repo-sync \
    --sync-namespace onlineboutique
```

## TODOs/FIXME

- Install ASM MCP via `mesh.cloud.google.com/v1beta1`/`ControlPlaneRevision` resource
- Update ASM Policies as soon as they are available
- KCC tab for gcloud commands?