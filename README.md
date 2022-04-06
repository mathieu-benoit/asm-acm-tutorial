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

Question, what about a diagram to show an overview of the setup and the components? CS+PoCo+Namespaces+GH-repos, something like this (very roughtly):
![Tutorial overview](imgs/overview.png)

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

## Enable ASM

```
gcloud beta container hub mesh enable
gcloud alpha container hub mesh describe
```

## Enable ACM

```
gcloud beta container hub config-management enable
gcloud beta container hub config-management status
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
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                             managed_resources                                                             │
├───────────────────────────┬──────────────────────┬────────────────────────────────┬────────────────┬─────────┬────────────┤
│           GROUP           │         KIND         │      NAME                      │   NAMESPACE    │  STATUS │ CONDITIONS │
├───────────────────────────┼──────────────────────┼────────────────────────────────┼────────────────┼─────────┼────────────┤
│                           │ Namespace            │ istio-system                   │                │ Current │            │
│                           │ Namespace            │ onlineboutique                 │                │ Current │            │
│ rbac.authorization.k8s.io │ ClusterRole          │ custom:aggregate-to-edit:istio │                │ Current │            │
│ mesh.cloud.google.com     │ ControlPlaneRevision │ asm-managed                    │ istio-system   │ Current │            │
│ configsync.gke.io         │ RepoSync             │ repo-sync                      │ onlineboutique │ Current │            │
│ rbac.authorization.k8s.io │ RoleBinding          │ repo-sync                      │ onlineboutique │ Current │            │
└───────────────────────────┴──────────────────────┴────────────────────────────────┴────────────────┴─────────┴────────────┘
```

As a best practice a new `ClusterRole` has been aggregated to the default [`edit` user-facing role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles) in order to be able to managed Istio resources like `AuthorizationPolicy` and `VirtualService` within the `onlineboutique` namespace while meeting with the least-privilege requirement. --> Show snippet on GitHub (root-sync/init/istio-clusterrole.yaml).

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

If you go through the Anthos > Service Mesh > Topology page in the Google Cloud console, you will see this:
![ASM Topology with Ingress Gateway and OnlineBoutique apps](imgs/asm-topology.png)

## Enforce ASM sidecar injection

As an admin, I should be able to enforce the policy that all workloads in the mesh should all have sidecar injected.

As an admin, I should be able to enforce the policy that all workloads in the mesh should not bypass sidecar.

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
│ constraints.gatekeeper.sh │ AsmSidecarInjection           │ pod-sidecar-injection-annotation  │                │ Current │            │
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

As an admin, I can enforce that mesh level mTLS is enforced to ensure all traffic within the mesh is mTLS.

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
kubectl get asmpeerauthnmeshstrictmtls.constraints.gatekeeper.sh/mesh-level-strict-mtls -ojsonpath='{.status.violations}'  | jq
```
Output:
```
[
  {
    "enforcementAction": "deny",
    "kind": "AsmPeerAuthnMeshStrictMtls",
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
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                           managed_resources                                                           │
├───────────────────────────┬────────────────────────────┬───────────────────────────────────┬───────────────────┬─────────┬────────────┤
│           GROUP           │            KIND            │                NAME               │     NAMESPACE     │  STATUS │ CONDITIONS │
├───────────────────────────┼────────────────────────────┼───────────────────────────────────┼───────────────────┼─────────┼────────────┤
│                           │ Namespace                  │ asm-ingress                       │                   │ Current │            │
│                           │ Namespace                  │ gatekeeper-system                 │                   │ Current │            │
│                           │ Namespace                  │ istio-system                      │                   │ Current │            │
│                           │ Namespace                  │ onlineboutique                    │                   │ Current │            │
│ constraints.gatekeeper.sh │ AsmPeerAuthnMeshStrictMtls │ mesh-level-strict-mtls            │                   │ Current │            │
│ constraints.gatekeeper.sh │ AsmPeerAuthnStrictMtls     │ peerauthentication-strict-mtls    │                   │ Current │            │
│ constraints.gatekeeper.sh │ AsmSidecarInjection        │ pod-sidecar-injection-annotation  │                   │ Current │            │
│ constraints.gatekeeper.sh │ DestinationRuleTLSEnabled  │ destination-rule-tls-enabled      │                   │ Current │            │
│ constraints.gatekeeper.sh │ K8sRequiredLabels          │ namespace-sidecar-injection-label │                   │ Current │            │
│ rbac.authorization.k8s.io │ ClusterRole                │ custom:aggregate-to-edit:istio    │                   │ Current │            │
│                           │ Service                    │ asm-ingressgateway                │ asm-ingress       │ Current │            │
│                           │ ServiceAccount             │ asm-ingressgateway                │ asm-ingress       │ Current │            │
│ apps                      │ Deployment                 │ asm-ingressgateway                │ asm-ingress       │ Current │            │
│ networking.istio.io       │ Gateway                    │ asm-ingressgateway                │ asm-ingress       │ Current │            │
│ config.gatekeeper.sh      │ Config                     │ config                            │ gatekeeper-system │ Current │            │
│ mesh.cloud.google.com     │ ControlPlaneRevision       │ asm-managed                       │ istio-system      │ Current │            │
│ security.istio.io         │ PeerAuthentication         │ default                           │ istio-system      │ Current │            │
│ configsync.gke.io         │ RepoSync                   │ repo-sync                         │ onlineboutique    │ Current │            │
│ rbac.authorization.k8s.io │ RoleBinding                │ repo-sync                         │ onlineboutique    │ Current │            │
└───────────────────────────┴────────────────────────────┴───────────────────────────────────┴───────────────────┴─────────┴────────────┘
...
NAME                                                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8srequiredlabels.constraints.gatekeeper.sh/namespace-sidecar-injection-label   deny                 0
NAME                                                                          ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
asmpeerauthnmeshstrictmtls.constraints.gatekeeper.sh/mesh-level-strict-mtls   deny                 0
NAME                                                                               ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
destinationruletlsenabled.constraints.gatekeeper.sh/destination-rule-tls-enabled   deny                 0
NAME                                                                              ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
asmpeerauthnstrictmtls.constraints.gatekeeper.sh/peerauthentication-strict-mtls   deny                 0
NAME                                                                             ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
asmsidecarinjection.constraints.gatekeeper.sh/pod-sidecar-injection-annotation   deny                 0
```

## Enforce AuthorizationPolicies

As an admin, I can enforce that there is a default deny authorization policy for all mesh workloads.

Potential things we should show/explain/illustrate/callout in this section:
- AuthorizationPolicies: FIXME

```
sed -i "s,root-sync/fix-strict-mtls,root-sync/enforce-authorization-policies,g" $WORK_DIR/acm-config.yaml
gcloud beta container hub config-management apply \
    --membership ${CLUSTER} \
    --config $WORK_DIR/acm-config.yaml
```

"Show in GitHub" snippet:
- root-sync/enforce-authorization-policies/policies/default-deny-authorization-policies.yaml

Because we haven't set up yet the default `deny` `AuthorizationPolicy` in our Mesh, running the command below will tell us about this violation:
```
kubectl get asmauthzpolicydefaultdeny.constraints.gatekeeper.sh/default-deny-authorization-policies -ojsonpath='{.status.violations}'  | jq
```
Output:
```
[
  {
    "enforcementAction": "deny",
    "kind": "AsmAuthzPolicyDefaultDeny",
    "message": "Root namespace <istio-system> does not have a default deny AuthorizationPolicy",
    "name": "default-deny-authorization-policies"
  }
]
```

Let's fix this issue by actually deploying this `AuthorizationPolicy` in the `istio-system` in order to deny any ingress to the entire Mesh:
```
sed -i "s,root-sync/enforce-authorization-policies,root-sync/fix-default-deny-authorization-policy,g" $WORK_DIR/acm-config.yaml
gcloud beta container hub config-management apply \
    --membership ${CLUSTER} \
    --config $WORK_DIR/acm-config.yaml
```

"Show in GitHub" snippet:
- istio-system/AuthorizationPolicy

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
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                            managed_resources                                                            │
├───────────────────────────┬────────────────────────────┬─────────────────────────────────────┬───────────────────┬─────────┬────────────┤
│           GROUP           │            KIND            │                 NAME                │     NAMESPACE     │  STATUS │ CONDITIONS │
├───────────────────────────┼────────────────────────────┼─────────────────────────────────────┼───────────────────┼─────────┼────────────┤
│                           │ Namespace                  │ asm-ingress                         │                   │ Current │            │
│                           │ Namespace                  │ gatekeeper-system                   │                   │ Current │            │
│                           │ Namespace                  │ istio-system                        │                   │ Current │            │
│                           │ Namespace                  │ onlineboutique                      │                   │ Current │            │
│ constraints.gatekeeper.sh │ AsmAuthzPolicyDefaultDeny  │ default-deny-authorization-policies │                   │ Current │            │
│ constraints.gatekeeper.sh │ AsmPeerAuthnMeshStrictMtls │ mesh-level-strict-mtls              │                   │ Current │            │
│ constraints.gatekeeper.sh │ AsmPeerAuthnStrictMtls     │ peerauthentication-strict-mtls      │                   │ Current │            │
│ constraints.gatekeeper.sh │ AsmSidecarInjection        │ pod-sidecar-injection-annotation    │                   │ Current │            │
│ constraints.gatekeeper.sh │ DestinationRuleTLSEnabled  │ destination-rule-tls-enabled        │                   │ Current │            │
│ constraints.gatekeeper.sh │ K8sRequiredLabels          │ namespace-sidecar-injection-label   │                   │ Current │            │
│ rbac.authorization.k8s.io │ ClusterRole                │ custom:aggregate-to-edit:istio      │                   │ Current │            │
│                           │ Service                    │ asm-ingressgateway                  │ asm-ingress       │ Current │            │
│                           │ ServiceAccount             │ asm-ingressgateway                  │ asm-ingress       │ Current │            │
│ apps                      │ Deployment                 │ asm-ingressgateway                  │ asm-ingress       │ Current │            │
│ networking.istio.io       │ Gateway                    │ asm-ingressgateway                  │ asm-ingress       │ Current │            │
│ config.gatekeeper.sh      │ Config                     │ config                              │ gatekeeper-system │ Current │            │
│ mesh.cloud.google.com     │ ControlPlaneRevision       │ asm-managed                         │ istio-system      │ Current │            │
│ security.istio.io         │ AuthorizationPolicy        │ deny-all                            │ istio-system      │ Current │            │
│ security.istio.io         │ PeerAuthentication         │ default                             │ istio-system      │ Current │            │
│ configsync.gke.io         │ RepoSync                   │ repo-sync                           │ onlineboutique    │ Current │            │
│ rbac.authorization.k8s.io │ RoleBinding                │ repo-sync                           │ onlineboutique    │ Current │            │
└───────────────────────────┴────────────────────────────┴─────────────────────────────────────┴───────────────────┴─────────┴────────────┘
...
NAME                                                                             ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
asmsidecarinjection.constraints.gatekeeper.sh/pod-sidecar-injection-annotation   deny                 0
NAME                                                                            ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
k8srequiredlabels.constraints.gatekeeper.sh/namespace-sidecar-injection-label   deny                 0
NAME                                                                                      ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
asmauthzpolicydefaultdeny.constraints.gatekeeper.sh/default-deny-authorization-policies   deny                 0
NAME                                                                          ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
asmpeerauthnmeshstrictmtls.constraints.gatekeeper.sh/mesh-level-strict-mtls   deny                 0
NAME                                                                               ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
destinationruletlsenabled.constraints.gatekeeper.sh/destination-rule-tls-enabled   deny                 0
NAME                                                                              ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
asmpeerauthnstrictmtls.constraints.gatekeeper.sh/peerauthentication-strict-mtls   deny                 0
```

But now if we access again the Ingress Gateway's public IP address, we will get an error: `RBAC: access denied`.
```
kubectl get svc asm-ingressgateway -n asm-ingress -o jsonpath="{.status.loadBalancer.ingress[*].ip}"
```

That's because we deployed a default `deny` `AuthorizationPolicy` to the entire Mesh. What we need to do to fix this is actually deploying fine granular `AuthorizationPolicy` resources in order to get our solution working again:
```
sed -i "s,root-sync/fix-default-deny-authorization-policy,root-sync/deploy-authorization-policies,g" $WORK_DIR/acm-config.yaml
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
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                            managed_resources                                                            │
├───────────────────────────┬────────────────────────────┬─────────────────────────────────────┬───────────────────┬─────────┬────────────┤
│           GROUP           │            KIND            │                 NAME                │     NAMESPACE     │  STATUS │ CONDITIONS │
├───────────────────────────┼────────────────────────────┼─────────────────────────────────────┼───────────────────┼─────────┼────────────┤
│                           │ Namespace                  │ asm-ingress                         │                   │ Unknown │            │
│                           │ Namespace                  │ gatekeeper-system                   │                   │ Unknown │            │
│                           │ Namespace                  │ istio-system                        │                   │ Unknown │            │
│                           │ Namespace                  │ onlineboutique                      │                   │ Unknown │            │
│ constraints.gatekeeper.sh │ AsmAuthzPolicyDefaultDeny  │ default-deny-authorization-policies │                   │ Unknown │            │
│ constraints.gatekeeper.sh │ AsmPeerAuthnMeshStrictMtls │ mesh-level-strict-mtls              │                   │ Unknown │            │
│ constraints.gatekeeper.sh │ AsmPeerAuthnStrictMtls     │ peerauthentication-strict-mtls      │                   │ Unknown │            │
│ constraints.gatekeeper.sh │ AsmSidecarInjection        │ pod-sidecar-injection-annotation    │                   │ Unknown │            │
│ constraints.gatekeeper.sh │ DestinationRuleTLSEnabled  │ destination-rule-tls-enabled        │                   │ Unknown │            │
│ constraints.gatekeeper.sh │ K8sRequiredLabels          │ namespace-sidecar-injection-label   │                   │ Unknown │            │
│ rbac.authorization.k8s.io │ ClusterRole                │ custom:aggregate-to-edit:istio      │                   │ Unknown │            │
│                           │ Service                    │ asm-ingressgateway                  │ asm-ingress       │ Unknown │            │
│                           │ ServiceAccount             │ asm-ingressgateway                  │ asm-ingress       │ Unknown │            │
│ apps                      │ Deployment                 │ asm-ingressgateway                  │ asm-ingress       │ Unknown │            │
│ networking.istio.io       │ Gateway                    │ asm-ingressgateway                  │ asm-ingress       │ Unknown │            │
│ security.istio.io         │ AuthorizationPolicy        │ asm-ingressgateway                  │ asm-ingress       │ Unknown │            │
│ config.gatekeeper.sh      │ Config                     │ config                              │ gatekeeper-system │ Unknown │            │
│ mesh.cloud.google.com     │ ControlPlaneRevision       │ asm-managed                         │ istio-system      │ Unknown │            │
│ security.istio.io         │ AuthorizationPolicy        │ deny-all                            │ istio-system      │ Unknown │            │
│ security.istio.io         │ PeerAuthentication         │ default                             │ istio-system      │ Unknown │            │
│ configsync.gke.io         │ RepoSync                   │ repo-sync                           │ onlineboutique    │ Unknown │            │
│ rbac.authorization.k8s.io │ RoleBinding                │ repo-sync                           │ onlineboutique    │ Unknown │            │
└───────────────────────────┴────────────────────────────┴─────────────────────────────────────┴───────────────────┴─────────┴────────────┘
...
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                             managed_resources                                             │
├─────────────────────┬─────────────────────┬───────────────────────┬────────────────┬─────────┬────────────┤
│        GROUP        │         KIND        │          NAME         │   NAMESPACE    │  STATUS │ CONDITIONS │
├─────────────────────┼─────────────────────┼───────────────────────┼────────────────┼─────────┼────────────┤
│                     │ Service             │ adservice             │ onlineboutique │ Unknown │            │
│                     │ Service             │ cartservice           │ onlineboutique │ Unknown │            │
│                     │ Service             │ checkoutservice       │ onlineboutique │ Unknown │            │
│                     │ Service             │ currencyservice       │ onlineboutique │ Unknown │            │
│                     │ Service             │ emailservice          │ onlineboutique │ Unknown │            │
│                     │ Service             │ frontend              │ onlineboutique │ Unknown │            │
│                     │ Service             │ paymentservice        │ onlineboutique │ Unknown │            │
│                     │ Service             │ productcatalogservice │ onlineboutique │ Unknown │            │
│                     │ Service             │ recommendationservice │ onlineboutique │ Unknown │            │
│                     │ Service             │ redis-cart            │ onlineboutique │ Unknown │            │
│                     │ Service             │ shippingservice       │ onlineboutique │ Unknown │            │
│                     │ ServiceAccount      │ adservice             │ onlineboutique │ Unknown │            │
│                     │ ServiceAccount      │ cartservice           │ onlineboutique │ Unknown │            │
│                     │ ServiceAccount      │ checkoutservice       │ onlineboutique │ Unknown │            │
│                     │ ServiceAccount      │ currencyservice       │ onlineboutique │ Unknown │            │
│                     │ ServiceAccount      │ emailservice          │ onlineboutique │ Unknown │            │
│                     │ ServiceAccount      │ frontend              │ onlineboutique │ Unknown │            │
│                     │ ServiceAccount      │ loadgenerator         │ onlineboutique │ Unknown │            │
│                     │ ServiceAccount      │ paymentservice        │ onlineboutique │ Unknown │            │
│                     │ ServiceAccount      │ productcatalogservice │ onlineboutique │ Unknown │            │
│                     │ ServiceAccount      │ recommendationservice │ onlineboutique │ Unknown │            │
│                     │ ServiceAccount      │ redis-cart            │ onlineboutique │ Unknown │            │
│                     │ ServiceAccount      │ shippingservice       │ onlineboutique │ Unknown │            │
│ apps                │ Deployment          │ adservice             │ onlineboutique │ Unknown │            │
│ apps                │ Deployment          │ cartservice           │ onlineboutique │ Unknown │            │
│ apps                │ Deployment          │ checkoutservice       │ onlineboutique │ Unknown │            │
│ apps                │ Deployment          │ currencyservice       │ onlineboutique │ Unknown │            │
│ apps                │ Deployment          │ emailservice          │ onlineboutique │ Unknown │            │
│ apps                │ Deployment          │ frontend              │ onlineboutique │ Unknown │            │
│ apps                │ Deployment          │ loadgenerator         │ onlineboutique │ Unknown │            │
│ apps                │ Deployment          │ paymentservice        │ onlineboutique │ Unknown │            │
│ apps                │ Deployment          │ productcatalogservice │ onlineboutique │ Unknown │            │
│ apps                │ Deployment          │ recommendationservice │ onlineboutique │ Unknown │            │
│ apps                │ Deployment          │ redis-cart            │ onlineboutique │ Unknown │            │
│ apps                │ Deployment          │ shippingservice       │ onlineboutique │ Unknown │            │
│ networking.istio.io │ VirtualService      │ frontend              │ onlineboutique │ Unknown │            │
│ security.istio.io   │ AuthorizationPolicy │ adservice             │ onlineboutique │ Unknown │            │
│ security.istio.io   │ AuthorizationPolicy │ cartservice           │ onlineboutique │ Unknown │            │
│ security.istio.io   │ AuthorizationPolicy │ checkoutservice       │ onlineboutique │ Unknown │            │
│ security.istio.io   │ AuthorizationPolicy │ currencyservice       │ onlineboutique │ Unknown │            │
│ security.istio.io   │ AuthorizationPolicy │ emailservice          │ onlineboutique │ Unknown │            │
│ security.istio.io   │ AuthorizationPolicy │ frontend              │ onlineboutique │ Unknown │            │
│ security.istio.io   │ AuthorizationPolicy │ paymentservice        │ onlineboutique │ Unknown │            │
│ security.istio.io   │ AuthorizationPolicy │ productcatalogservice │ onlineboutique │ Unknown │            │
│ security.istio.io   │ AuthorizationPolicy │ recommendationservice │ onlineboutique │ Unknown │            │
│ security.istio.io   │ AuthorizationPolicy │ redis-cart            │ onlineboutique │ Unknown │            │
│ security.istio.io   │ AuthorizationPolicy │ shippingservice       │ onlineboutique │ Unknown │            │
└─────────────────────┴─────────────────────┴───────────────────────┴────────────────┴─────────┴────────────┘
```

To conclude, you have secured your cluster and your mesh thanks to Policy Controller and Config Sync, if you navigate to Anthos > Security > Policy Audit in the Google Cloud console, you could see that both namespaces: `asm-ingress` and `onlineboutique` are now secured with mTLS `STRICT` and fine granular `AuthorizationPolicies`:
![ASM Security Summary for OnlineBoutique apps](imgs/onlineboutique-asm-policy-summary.png)

## More resources

- Constraint template library: https://cloud.google.com/anthos-config-management/docs/reference/constraint-template-library
- Use Anthos Service Mesh security policy constraints: https://cloud.google.com/anthos-config-management/docs/how-to/using-asm-security-policy
- From edge to mesh: Exposing service mesh applications through GKE Ingress: https://cloud.google.com/architecture/exposing-service-mesh-apps-through-gke-ingress
