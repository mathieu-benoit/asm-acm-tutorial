Concepts leveraged/illustrated:
- Config Sync (all Kubernetes manifests deployed by GitOps)
- Policy Controller (4 `Constraints`)
- Multi-repos: `RootSync` and `RepoSync`
- Unstructured repo + `Kustomize`
- Referential constraints
- ASM MCP
- `STRICT` Mesh mTLS
- `AuthorizationPolicy`

TODO:
- Take some wording on the approach from [this existing tutorial](https://cloud.google.com/anthos-config-management/docs/config-sync-quickstart). Something around _"Imagine that your compliance team is responsible for making sure that everyone in your organization is following internal rules. To enforce these rules, the compliance team has created configs, which they have added to the samples repository."_ 

Questions:
- Diagram? CS+PoCo+Namespaces+GH-repos

## Init variables

```
WORK_DIR=~/asm-acm-tutorial-dir
mkdir $WORK_DIR
```

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
    --machine-type=e2-standard-4 \
    --num-nodes 4 \
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

Potential things we should show/explain/illustrate/callout in this section:
- ASM MCP via Fleet API: https://cloud.google.com/service-mesh/docs/managed/auto-control-plane-with-fleet
- Multi-repo with Config Sync: https://cloud.google.com/anthos-config-management/docs/how-to/namespace-repositories
- Default policies library: https://cloud.google.com/anthos-config-management/docs/reference/constraint-template-library

```
cat <<EOF > $WORK_DIR/acm-config.yaml
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
    --config $WORK_DIR/acm-config.yaml
```

"Show in GitHub" snippet:
- root-sync/init/repo-sync/reposync.yaml

Checks:
```
gcloud beta container hub config-management status
gcloud alpha container hub mesh describe
gcloud alpha anthos config sync repo list
gcloud alpha anthos config sync repo describe \
    --managed-resources all \
    --sync-name root-sync \
    --sync-namespace config-management-system
```
Outputs:
```
Name               Status         Last_Synced_Token  Sync_Branch  Last_Synced_Time      Policy_Controller  Hierarchy_Controller
asm-acm-cluster-2  SYNCED         f9969f0            main         2022-03-22T13:03:21Z  INSTALLED          PENDING
...
createTime: '2022-03-19T15:20:16.653593277Z'
membershipStates:
  projects/454838580268/locations/global/memberships/asm-acm-tutorial-3:
    servicemesh:
      controlPlaneManagement:
        state: DISABLED
    state:
      code: OK
      description: 'Revision(s) ready for use: asm-managed.'
      updateTime: '2022-03-23T16:30:47.216630490Z'
name: projects/mabenoit-asm-acm/locations/global/features/servicemesh
resourceState:
  state: ACTIVE
spec: {}
state:
  servicemesh: {}
  state: {}
updateTime: '2022-03-23T16:30:53.359173436Z'
...
getting 2 RepoSync and RootSync from asm-acm-tutorial-3
┌───────────────────────────────────────────────────────────────────────────────────┬───────┬────────┬─────────┬───────┬─────────┬─────────────┐
│                                       SOURCE                                      │ TOTAL │ SYNCED │ PENDING │ ERROR │ STALLED │ RECONCILING │
├───────────────────────────────────────────────────────────────────────────────────┼───────┼────────┼─────────┼───────┼─────────┼─────────────┤
│ https://github.com/mathieu-benoit/asm-acm-tutorial//root-sync/init@main           │ 1     │ 1      │ 0       │ 0     │ 0       │ 0           │
│ https://github.com/mathieu-benoit/asm-acm-tutorial//onlineboutique/init@main:HEAD │ 1     │ 1      │ 0       │ 0     │ 0       │ 0           │
└───────────────────────────────────────────────────────────────────────────────────┴───────┴────────┴─────────┴───────┴─────────┴─────────────┘
...
getting 1 RepoSync and RootSync from asm-acm-tutorial-3
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                             managed_resources                                             │
├───────────────────────────┬──────────────────────┬────────────────┬────────────────┬─────────┬────────────┤
│           GROUP           │         KIND         │      NAME      │   NAMESPACE    │  STATUS │ CONDITIONS │
├───────────────────────────┼──────────────────────┼────────────────┼────────────────┼─────────┼────────────┤
│                           │ Namespace            │ istio-system   │                │ Current │            │
│                           │ Namespace            │ onlineboutique │                │ Current │            │
│ mesh.cloud.google.com     │ ControlPlaneRevision │ asm-managed    │ istio-system   │ Current │            │
│ configsync.gke.io         │ RepoSync             │ repo-sync      │ onlineboutique │ Current │            │
│ rbac.authorization.k8s.io │ RoleBinding          │ repo-sync      │ onlineboutique │ Current │            │
└───────────────────────────┴──────────────────────┴────────────────┴────────────────┴─────────┴────────────┘
```

## Deploy Ingress Gateway and OnlineBoutique apps

Potential things we should show/explain/illustrate/callout in this section:
- Ingress Gateway: https://cloud.google.com/service-mesh/docs/gateways
- OnlineBoutique: https://github.com/GoogleCloudPlatform/microservices-demo

```
sed -i "s,root-sync/init,root-sync/deployments,g" $WORK_DIR/acm-config.yaml
gcloud beta container hub config-management apply \
    --membership ${CLUSTER} \
    --config $WORK_DIR/acm-config.yaml
```

Checks:
```
gcloud alpha anthos config sync repo describe \
    --managed-resources all \
    --sync-name root-sync \
    --sync-namespace config-management-system
gcloud alpha anthos config sync repo describe \
    --managed-resources all \
    --sync-name repo-sync \
    --sync-namespace onlineboutique
```
Outputs:
```
...
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                               managed_resources                                               │
├───────────────────────────┬──────────────────────┬────────────────────┬────────────────┬─────────┬────────────┤
│           GROUP           │         KIND         │        NAME        │   NAMESPACE    │  STATUS │ CONDITIONS │
├───────────────────────────┼──────────────────────┼────────────────────┼────────────────┼─────────┼────────────┤
│                           │ Namespace            │ asm-ingress        │                │ Current │            │
│                           │ Namespace            │ istio-system       │                │ Current │            │
│                           │ Namespace            │ onlineboutique     │                │ Current │            │
│                           │ Service              │ asm-ingressgateway │ asm-ingress    │ Current │            │
│ apps                      │ Deployment           │ asm-ingressgateway │ asm-ingress    │ Current │            │
│ networking.istio.io       │ Gateway              │ asm-ingressgateway │ asm-ingress    │ Current │            │
│ mesh.cloud.google.com     │ ControlPlaneRevision │ asm-managed        │ istio-system   │ Current │            │
│ configsync.gke.io         │ RepoSync             │ repo-sync          │ onlineboutique │ Current │            │
│ rbac.authorization.k8s.io │ RoleBinding          │ repo-sync          │ onlineboutique │ Current │            │
└───────────────────────────┴──────────────────────┴────────────────────┴────────────────┴─────────┴────────────┘
...
┌──────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                          managed_resources                                           │
├─────────────────────┬────────────────┬───────────────────────┬────────────────┬─────────┬────────────┤
│        GROUP        │      KIND      │          NAME         │   NAMESPACE    │  STATUS │ CONDITIONS │
├─────────────────────┼────────────────┼───────────────────────┼────────────────┼─────────┼────────────┤
│                     │ Service        │ adservice             │ onlineboutique │ Current │            │
│                     │ Service        │ cartservice           │ onlineboutique │ Current │            │
│                     │ Service        │ checkoutservice       │ onlineboutique │ Current │            │
│                     │ Service        │ currencyservice       │ onlineboutique │ Current │            │
│                     │ Service        │ emailservice          │ onlineboutique │ Current │            │
│                     │ Service        │ frontend              │ onlineboutique │ Current │            │
│                     │ Service        │ paymentservice        │ onlineboutique │ Current │            │
│                     │ Service        │ productcatalogservice │ onlineboutique │ Current │            │
│                     │ Service        │ recommendationservice │ onlineboutique │ Current │            │
│                     │ Service        │ redis-cart            │ onlineboutique │ Current │            │
│                     │ Service        │ shippingservice       │ onlineboutique │ Current │            │
│ apps                │ Deployment     │ adservice             │ onlineboutique │ Current │            │
│ apps                │ Deployment     │ cartservice           │ onlineboutique │ Current │            │
│ apps                │ Deployment     │ checkoutservice       │ onlineboutique │ Current │            │
│ apps                │ Deployment     │ currencyservice       │ onlineboutique │ Current │            │
│ apps                │ Deployment     │ emailservice          │ onlineboutique │ Current │            │
│ apps                │ Deployment     │ frontend              │ onlineboutique │ Current │            │
│ apps                │ Deployment     │ loadgenerator         │ onlineboutique │ Current │            │
│ apps                │ Deployment     │ paymentservice        │ onlineboutique │ Current │            │
│ apps                │ Deployment     │ productcatalogservice │ onlineboutique │ Current │            │
│ apps                │ Deployment     │ recommendationservice │ onlineboutique │ Current │            │
│ apps                │ Deployment     │ redis-cart            │ onlineboutique │ Current │            │
│ apps                │ Deployment     │ shippingservice       │ onlineboutique │ Current │            │
│ networking.istio.io │ VirtualService │ frontend              │ onlineboutique │ Current │            │
└─────────────────────┴────────────────┴───────────────────────┴────────────────┴─────────┴────────────┘
```

From here, you could now browse the OnlineBoutique website by hitting the Ingress Gateway's public IP address:
```
kubectl get svc asm-ingressgateway -n asm-ingress -o jsonpath="{.status.loadBalancer.ingress[*].ip}"
```

## Enforce ASM sidecar injection

Potential things we should show/explain/illustrate/callout in this section:
- ASM sidecar injection: https://cloud.google.com/service-mesh/docs/anthos-service-mesh-proxy-injection
- ASM Control Plane revisions: https://cloud.google.com/service-mesh/docs/revisions-overview
- Policy Controller: https://cloud.google.com/anthos-config-management/docs/concepts/policy-controller
- Enforcing policies by creating `Constraints`: https://cloud.google.com/anthos-config-management/docs/how-to/creating-constraints

```
sed -i "s,root-sync/deployments,root-sync/enforce-sidecar-injection,g" $WORK_DIR/acm-config.yaml
gcloud beta container hub config-management apply \
    --membership ${CLUSTER} \
    --config $WORK_DIR/acm-config.yaml
```

"Show in GitHub" snippet:
- root-sync/enforce-sidecar-injection/namespace-sidecar-injection-label.yaml
- root-sync/enforce-sidecar-injection/pod-sidecar-injection-annotation.yaml

Checks:
```
gcloud alpha anthos config sync repo describe \
    --managed-resources all \
    --sync-name root-sync \
    --sync-namespace config-management-system
kubectl get constraints
```
Outputs:
```
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                           managed_resources                                                           │
├───────────────────────────┬───────────────────────────────┬───────────────────────────────────┬────────────────┬─────────┬────────────┤
│           GROUP           │              KIND             │                NAME               │   NAMESPACE    │  STATUS │ CONDITIONS │
├───────────────────────────┼───────────────────────────────┼───────────────────────────────────┼────────────────┼─────────┼────────────┤
│                           │ Namespace                     │ asm-ingress                       │                │ Current │            │
│                           │ Namespace                     │ istio-system                      │                │ Current │            │
│                           │ Namespace                     │ onlineboutique                    │                │ Current │            │
│ constraints.gatekeeper.sh │ K8sRequiredLabels             │ namespace-sidecar-injection-label │                │ Current │            │
│ constraints.gatekeeper.sh │ PodSidecarInjectionAnnotation │ pod-sidecar-injection-annotation  │                │ Current │            │
│ templates.gatekeeper.sh   │ ConstraintTemplate            │ podsidecarinjectionannotation     │                │ Current │            │
│                           │ Service                       │ asm-ingressgateway                │ asm-ingress    │ Current │            │
│ apps                      │ Deployment                    │ asm-ingressgateway                │ asm-ingress    │ Current │            │
│ networking.istio.io       │ Gateway                       │ asm-ingressgateway                │ asm-ingress    │ Current │            │
│ mesh.cloud.google.com     │ ControlPlaneRevision          │ asm-managed                       │ istio-system   │ Current │            │
│ configsync.gke.io         │ RepoSync                      │ repo-sync                         │ onlineboutique │ Current │            │
│ rbac.authorization.k8s.io │ RoleBinding                   │ repo-sync                         │ onlineboutique │ Current │            │
└───────────────────────────┴───────────────────────────────┴───────────────────────────────────┴────────────────┴─────────┴────────────┘
...
NAME                                                                                       ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
podsidecarinjectionannotation.constraints.gatekeeper.sh/pod-sidecar-injection-annotation   deny                 0

NAME                                                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8srequiredlabels.constraints.gatekeeper.sh/namespace-sidecar-injection-label   deny                 0
```

We could see that for the 2 `Constraint` resources deployed we have 0 `TOTAL-VIOLATIONS`.

## Enforce STRICT mTLS in the Mesh

Potential things we should show/explain/illustrate/callout in this section:
- mTLS STRICT: https://cloud.google.com/service-mesh/docs/security/security-overview#mutual_tls
- Referrential constraints: https://cloud.google.com/anthos-config-management/docs/how-to/creating-constraints#gatekeeper-config

```
sed -i "s,root-sync/enforce-sidecar-injection,root-sync/enforce-strict-mtls,g" $WORK_DIR/acm-config.yaml
gcloud beta container hub config-management apply \
    --membership ${CLUSTER} \
    --config $WORK_DIR/acm-config.yaml
```

"Show in GitHub" snippet:
- root-sync/enforce-strict-mtls/gatekeeper-system/config-referential-constraints.yaml
- root-sync/enforce-strict-mtls/policies/mesh-level-strict-mtls.yaml

Because we haven't set up yet mTLS `STRICT` in our Mesh, running the command below will tell us about this violation:
```
kubectl get meshlevelstrictmtls.constraints.gatekeeper.sh/mesh-level-strict-mtls -ojsonpath='{.status.violations}'  | jq
```
Output:
```
[
  {
    "enforcementAction": "deny",
    "kind": "MeshLevelStrictMtls",
    "message": "Root namespace <istio-system> does not have a strict mTLS PeerAuthentication",
    "name": "mesh-level-strict-mtls"
  }
]
```

Let's fix this issue by actually deploying a `PeerAuthentication` in the `istio-system` in order to give mTLS `STRICT` to the entire Mesh:
```
sed -i "s,root-sync/enforce-strict-mtls,root-sync/fix-strict-mtls,g" $WORK_DIR/acm-config.yaml
gcloud beta container hub config-management apply \
    --membership ${CLUSTER} \
    --config $WORK_DIR/acm-config.yaml
```

"Show in GitHub" snippet:
- istio-system/PeerAuthentication

To complete this `MeshLevelStrictMtls` `Constraint` just deployed and in order to make sure no one in your Mesh overrides this mTLS `STRICT` setup, two more `Constraints` have been deployed too:
"Show in GitHub" snippet:
- root-sync/enforce-strict-mtls/policies/destinationrule-tls-enabled.yaml
- root-sync/enforce-strict-mtls/policies/peerauthentication-strict-mtls.yaml

Checks:
```
gcloud alpha anthos config sync repo describe \
    --managed-resources all \
    --sync-name root-sync \
    --sync-namespace config-management-system
kubectl get constraints
```
Outputs:
```
┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                            managed_resources                                                             │
├───────────────────────────┬───────────────────────────────┬───────────────────────────────────┬───────────────────┬─────────┬────────────┤
│           GROUP           │              KIND             │                NAME               │     NAMESPACE     │  STATUS │ CONDITIONS │
├───────────────────────────┼───────────────────────────────┼───────────────────────────────────┼───────────────────┼─────────┼────────────┤
│                           │ Namespace                     │ asm-ingress                       │                   │ Current │            │
│                           │ Namespace                     │ gatekeeper-system                 │                   │ Current │            │
│                           │ Namespace                     │ istio-system                      │                   │ Current │            │
│                           │ Namespace                     │ onlineboutique                    │                   │ Current │            │
│ constraints.gatekeeper.sh │ DestinationRuleTlsEnabled     │ destination-rule-tls-enabled      │                   │ Current │            │
│ constraints.gatekeeper.sh │ K8sRequiredLabels             │ namespace-sidecar-injection-label │                   │ Current │            │
│ constraints.gatekeeper.sh │ MeshLevelStrictMtls           │ mesh-level-strict-mtls            │                   │ Current │            │
│ constraints.gatekeeper.sh │ PeerAuthenticationStrictMtls  │ peerauthentication-strict-mtls    │                   │ Current │            │
│ constraints.gatekeeper.sh │ PodSidecarInjectionAnnotation │ pod-sidecar-injection-annotation  │                   │ Current │            │
│ templates.gatekeeper.sh   │ ConstraintTemplate            │ destinationruletlsenabled         │                   │ Current │            │
│ templates.gatekeeper.sh   │ ConstraintTemplate            │ meshlevelstrictmtls               │                   │ Current │            │
│ templates.gatekeeper.sh   │ ConstraintTemplate            │ peerauthenticationstrictmtls      │                   │ Current │            │
│ templates.gatekeeper.sh   │ ConstraintTemplate            │ podsidecarinjectionannotation     │                   │ Current │            │
│                           │ Service                       │ asm-ingressgateway                │ asm-ingress       │ Current │            │
│ apps                      │ Deployment                    │ asm-ingressgateway                │ asm-ingress       │ Current │            │
│ networking.istio.io       │ Gateway                       │ asm-ingressgateway                │ asm-ingress       │ Current │            │
│ config.gatekeeper.sh      │ Config                        │ config                            │ gatekeeper-system │ Current │            │
│ mesh.cloud.google.com     │ ControlPlaneRevision          │ asm-managed                       │ istio-system      │ Current │            │
│ security.istio.io         │ PeerAuthentication            │ default                           │ istio-system      │ Current │            │
│ configsync.gke.io         │ RepoSync                      │ repo-sync                         │ onlineboutique    │ Current │            │
│ rbac.authorization.k8s.io │ RoleBinding                   │ repo-sync                         │ onlineboutique    │ Current │            │
└───────────────────────────┴───────────────────────────────┴───────────────────────────────────┴───────────────────┴─────────┴────────────┘
...
NAME                                                                               ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
destinationruletlsenabled.constraints.gatekeeper.sh/destination-rule-tls-enabled   deny                 
NAME                                                                   ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
meshlevelstrictmtls.constraints.gatekeeper.sh/mesh-level-strict-mtls   deny                 0
NAME                                                                                    ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
peerauthenticationstrictmtls.constraints.gatekeeper.sh/peerauthentication-strict-mtls   deny                 0
NAME                                                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8srequiredlabels.constraints.gatekeeper.sh/namespace-sidecar-injection-label   deny                 0
NAME                                                                                       ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
podsidecarinjectionannotation.constraints.gatekeeper.sh/pod-sidecar-injection-annotation   deny                 0
```

## Enforce AuthorizationPolicies

FIXME