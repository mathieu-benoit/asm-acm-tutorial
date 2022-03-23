Concepts leveraged/illustrated:
- Config Sync (all Kubernetes manifests deployed by GitOps)
- Policy Controller (4 `Constraints`)
- Multi-repos: `RootSync` and `RepoSync`
- Unstructured repo + `Kustomize`
- Referential constraints
- ASM MCP
- `STRICT` Mesh mTLS
- `AuthorizationPolicy`

Questions:
- Diagram? CS+PoCo+Namespaces+GH-repos

## Init variables

```
PROJECT_ID=FIXME
gcloud config set project $PROJECT_ID
CLUSTER=asm-acm-tutorial
CLUSTER_ZONE=us-east4-a
PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='get(projectNumber)')
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
    --labels mesh_id=proj-${PROJECT_NUMBER}
```

## Add cluster in Fleet

```
gcloud container hub memberships register ${CLUSTER} \
    --gke-cluster ${CLUSTER_ZONE}/${CLUSTER} \
    --enable-workload-identity
```

## Enable ASM

```
gcloud beta container hub mesh enable
gcloud alpha container hub mesh describe
```

## Enable ACM

```
gcloud beta container hub config-management enable
gcloud beta container hub config-management describe
```

## Initialize PoCo & ConfigSync (RootSync + RepoSync) repos

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
    policyDir: root-sync/init
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

Show 1 or 2 `Constraint` resources ("View on GitHub") and provide some explanations.

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

## Set ASM sidecar injection

Show the snippet of the `label` and provide some explanations.

```
sed -i "s/deploy-policies/set-asm-injection/g" acm-config.yaml
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
asm-acm-cluster-2  SYNCED         bade531            main         2022-03-22T13:13:32Z  INSTALLED          PENDING
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                         managed_resources                                                         │
├───────────────────────────┬───────────────────────────┬───────────────────────────────────┬────────────────┬─────────┬────────────┤
│           GROUP           │            KIND           │                NAME               │   NAMESPACE    │  STATUS │ CONDITIONS │
├───────────────────────────┼───────────────────────────┼───────────────────────────────────┼────────────────┼─────────┼────────────┤
│                           │ Namespace                 │ onlineboutique                    │                │ Current │            │
│ constraints.gatekeeper.sh │ AllowedServicePortName    │ port-name-constraint              │                │ Current │            │
│ constraints.gatekeeper.sh │ DestinationRuleTLSEnabled │ destination-rules-tls-enabled     │                │ Current │            │
│ constraints.gatekeeper.sh │ K8sRequiredLabels         │ must-have-istio-sidecar-injection │                │ Current │            │
│ constraints.gatekeeper.sh │ PolicyStrictOnly          │ policy-strict-only                │                │ Current │            │
│ configsync.gke.io         │ RepoSync                  │ repo-sync                         │ onlineboutique │ Current │            │
│ rbac.authorization.k8s.io │ RoleBinding               │ repo-sync                         │ onlineboutique │ Current │            │
└───────────────────────────┴───────────────────────────┴───────────────────────────────────┴────────────────┴─────────┴────────────┘
```

## Next deployments

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