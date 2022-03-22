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