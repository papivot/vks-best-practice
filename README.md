# VKS 3.7: Cluster and Addon Best Practices

A practical guide to deploying a VMware vSphere Kubernetes Service 3.7 workload cluster with a
useful addon stack. It is organised around the decisions you have to make, in roughly the order you
have to make them, and it uses one complete manifest as a worked example throughout.

The example manifest is [`reference-profile.yaml`](./reference-profile.yaml). It deploys five addons
(helm-controller, cert-manager, Prometheus, Istio, Headlamp) onto a cluster built from the
`builtin-generic-v3.7.0` ClusterClass, with OIDC authentication and dedicated node volumes. Open it
alongside this guide.

That profile is deliberately permissive in places so the platform can be explored without high
availability or admission control restricting what can run. It is a demonstration, not a baseline.
[Section 1.3](#13-decisions-you-cannot-revisit) lists what to settle before you apply anything, and
[Appendix A](#appendix-a-a-production-starting-point) gives a hardened version you can deploy.

**Audience.** Platform engineers and Kubernetes administrators. Kubernetes fluency is assumed;
VKS-specific concepts are explained.

**Tooling.** Every command needs only `kubectl` and a POSIX shell. Output is shaped with
`-o jsonpath`, `-o custom-columns`, `--show-labels`, and standard utilities. No `jq`, `yq`, or
Python.

**Verification.** Schema constraints, defaults, and platform behaviour were checked against a
running VKS 3.7 environment. Where something depends on your release or licensing, it is listed in
[Appendix E](#appendix-e-check-these-against-your-own-environment) rather than asserted.

**Placeholders.**

| Token | Meaning |
| --- | --- |
| `<VSPHERE_NAMESPACE>` | Your vSphere Namespace on the Supervisor |
| `<CLUSTER_NAME>` | Your workload cluster name |
| `<STORAGE_CLASS>` | A StorageClass associated with that namespace |
| `<OIDC_CLIENT_ID>`, `<OIDC_CLIENT_SECRET>` | Credentials from your identity provider |
| `headlamp.k8s.example.com`, `api.k8s.example.com`, `idp.example.com` | Names you will publish in DNS |

---

## Contents

- [1. Before you start](#1-before-you-start)
    - [1.1 CLI access and contexts](#11-cli-access-and-contexts)
    - [1.2 Platform prerequisites](#12-platform-prerequisites)
    - [1.3 Decisions you cannot revisit](#13-decisions-you-cannot-revisit)
    - [1.4 What to change before production](#14-what-to-change-before-production)
- [2. Choosing a Kubernetes release](#2-choosing-a-kubernetes-release)
    - [Which one to pick](#which-one-to-pick)
- [3. What the platform already gives you](#3-what-the-platform-already-gives-you)
- [4. How addons are delivered](#4-how-addons-are-delivered)
    - [The naming rule](#the-naming-rule)
- [5. Reading an addon's value schema](#5-reading-an-addons-value-schema)
- [6. The addon stack](#6-the-addon-stack)
    - [6.1 helm-controller](#61-helm-controller)
    - [6.2 cert-manager](#62-cert-manager)
    - [6.3 Prometheus](#63-prometheus)
    - [6.4 Istio](#64-istio)
    - [6.5 Headlamp](#65-headlamp)
    - [6.6 The rest of the catalogue](#66-the-rest-of-the-catalogue)
- [7. The cluster](#7-the-cluster)
    - [7.1 Pod and service networking](#71-pod-and-service-networking)
    - [7.2 ClusterClass and version](#72-clusterclass-and-version)
    - [7.3 Node sizing](#73-node-sizing)
    - [7.4 Control plane](#74-control-plane)
    - [7.5 Worker pools](#75-worker-pools)
    - [7.6 Pod Security](#76-pod-security)
    - [7.7 Identity: OIDC and RBAC](#77-identity-oidc-and-rbac)
    - [7.8 Platform settings](#78-platform-settings)
- [8. Couplings that must agree](#8-couplings-that-must-agree)
- [9. Operating the cluster](#9-operating-the-cluster)
    - [9.1 Health checks after deployment](#91-health-checks-after-deployment)
    - [9.2 Upgrades](#92-upgrades)
    - [9.3 Ongoing maintenance](#93-ongoing-maintenance)
    - [9.4 Symptom lookup](#94-symptom-lookup)
- [Appendix A: A production starting point](#appendix-a-a-production-starting-point)
- [Appendix B: In-cluster companion objects](#appendix-b-in-cluster-companion-objects)
    - [B.1 RBAC for OIDC identities](#b1-rbac-for-oidc-identities)
    - [B.2 A trusted certificate for the UI](#b2-a-trusted-certificate-for-the-ui)
    - [B.3 A Prometheus instance](#b3-a-prometheus-instance)
    - [B.4 Worth adding beyond this](#b4-worth-adding-beyond-this)
- [Appendix C: Integrations that need an external system](#appendix-c-integrations-that-need-an-external-system)
    - [C.1 Backup and restore (`velero`)](#c1-backup-and-restore-velero)
    - [C.2 Log forwarding (`fluent-bit`)](#c2-log-forwarding-fluent-bit)
    - [C.3 DNS automation (`external-dns`)](#c3-dns-automation-external-dns)
- [Appendix D: Reading a Kubernetes release object](#appendix-d-reading-a-kubernetes-release-object)
    - [D.1 Core components](#d1-core-components)
    - [D.2 Bootstrap packages](#d2-bootstrap-packages)
    - [D.3 OS images](#d3-os-images)
    - [D.4 A pre-upgrade change report](#d4-a-pre-upgrade-change-report)
    - [D.5 Command reference](#d5-command-reference)
- [Appendix E: Check these against your own environment](#appendix-e-check-these-against-your-own-environment)

---

## 1. Before you start

Four things to get out of the way: access, prerequisites, the handful of choices that are permanent,
and the settings that separate a lab from production.

### 1.1 CLI access and contexts

Everything here runs through `kubectl` against the Supervisor. The `vcf` CLI creates the context and
writes the kubeconfig:

```bash
vcf context create <CONTEXT_NAME> --endpoint https://<SUPERVISOR_FQDN> --type k8s

# if the Supervisor sits behind a private or enterprise CA
vcf context create <CONTEXT_NAME> --endpoint https://<SUPERVISOR_FQDN> \
  --ca-certificate /path/to/ca-cert --type k8s
```

You end up with one context per vSphere Namespace you can reach, and one per workload cluster. Two
of them matter, and confusing them is the most common early stumble:

| Context | What lives there |
| --- | --- |
| Supervisor, scoped to your vSphere Namespace | `Cluster`, `AddonInstall`, `AddonConfig`, `ClusterAddon`, `kr`, `osimage`, `virtualmachineclass`. Everything you author. |
| The workload cluster | `pkgi`, pods, `Gateway`, RBAC, `Prometheus` CRs. Everything that runs inside. |

```bash
kubectl config use-context supervisor:<VSPHERE_NAMESPACE>    # authoring
kubectl config use-context vks:<CLUSTER_NAME>                # in-cluster checks
```

Commands in this guide run against the Supervisor unless stated otherwise.

One useful consequence: `vcf context create` regenerates a working kubeconfig whenever you need one,
so an outage at your identity provider does not lock you out of a cluster that uses OIDC.

### 1.2 Platform prerequisites

VKS accepts objects that reference resources it cannot find, and only stalls later. A `Cluster` whose
VM class is not available to the namespace is admitted happily and then never provisions a machine.
Checking first is cheap.

| Requirement | Check |
| --- | --- |
| CLI access, correct context | `kubectl config current-context` |
| The vSphere Namespace exists and you have rights on it | `kubectl get ns <VSPHERE_NAMESPACE>` |
| VM classes are associated with the namespace | `kubectl get virtualmachineclass` |
| Storage policies are associated with the namespace | `kubectl describe ns <VSPHERE_NAMESPACE>` |
| A suitable Kubernetes release is available | `kubectl get kr` ([section 2](#2-choosing-a-kubernetes-release)) |
| The ClusterClass is present and its variables are ready | `kubectl get clusterclass -n vmware-system-vks-public` |
| OS images are synced | `kubectl get osimage` ([Appendix D](#appendix-d-reading-a-kubernetes-release-object)) |
| The `vcf` CLI is installed | `vcf version` |

StorageClass is cluster-scoped, so a namespace user usually cannot list it. The namespace's storage
quota names every class available to you, which is the list you actually need:

```bash
kubectl describe ns <VSPHERE_NAMESPACE> | grep storageclass
# vsan-esa-default-policy-raid5.storageclass.storage.k8s.io/requests.storage   127Gi   ...
```

The part before `.storageclass.storage.k8s.io` is the name to put in the manifest.

One piece of terminology, because it causes more confusion than anything else in VKS: the
`metadata.namespace` on these objects is a **vSphere Namespace on the Supervisor**, not a namespace
inside the workload cluster. The reference profile contains both kinds. `metadata.namespace` is a
Supervisor namespace; `spec.values.istio.namespace: istio-system` is a namespace in the guest
cluster. Different API servers entirely.

### 1.3 Decisions you cannot revisit

Most of the manifest can be edited later. These cannot, and correcting one means building a new
cluster and migrating workloads.

| Decision | Field | Where it is discussed |
| --- | --- | --- |
| Pod network size | `clusterNetwork.pods.cidrBlocks` | [7.1](#71-pod-and-service-networking) |
| Service network size | `clusterNetwork.services.cidrBlocks` | [7.1](#71-pod-and-service-networking) |
| CNI | `bootstrapAddons.cniRef` | [7.8](#78-platform-settings) |
| Cluster DNS domain | `clusterNetwork.serviceDomain` | [7.1](#71-pod-and-service-networking) |
| Cluster name | `metadata.name` | Embedded in every addon selector |
| Per-node pod block size | Node CIDR mask | [7.1](#71-pod-and-service-networking) |

Two more are technically changeable but expensive enough to treat the same way: `volumes` capacity,
because resizing rebuilds every node in the pool, and moving `controlPlane.replicas` from 1 to 3,
which is a rolling change you need a window for.

Of these, pod network size is the one people get wrong. Read [7.1](#71-pod-and-service-networking)
before you pick a prefix.

### 1.4 What to change before production

The reference profile makes several choices that suit a lab. None of them is a mistake in that
context; all of them should change before real workloads arrive.

| Setting in the profile | Production choice |
| --- | --- |
| `controlPlane.replicas: 1` | `3`, for etcd quorum |
| `vmClass: best-effort-*` | `guaranteed-*` |
| `podSecurityStandard: privileged` | `enforce: baseline`, with `restricted` per namespace |
| PSS `*Version: latest` | An explicit version, such as `v1.36` |
| `pilot.replicas: 1`, ingress `minReplicas: 1` | `2` for both |
| `oidc.clientSecret` inline | A referenced Secret |
| Self-signed UI certificate | A certificate from a trusted CA |
| Pod CIDR `/20` | `/16` |

---

## 2. Choosing a Kubernetes release

`topology.version` takes a single string. Get it wrong and the cluster is admitted and then never
reconciles, so it is worth two minutes to look up rather than copy.

VKS models each available version as a cluster-scoped `KubernetesRelease`, short name `kr`:

```bash
kubectl get kr
```

```
NAME                              VERSION                           READY   COMPATIBLE   DEACTIVATED BY
v1.35.6---vmware.2-vkr.3          v1.35.6+vmware.2-vkr.3            True    True
v1.36.1---vmware.4-vkr.5          v1.36.1+vmware.4-vkr.5            True    True
v1.36.2---vmware.2-vkr.3          v1.36.2+vmware.2-vkr.3            True    True
v1.25.13---vmware.1-fips.1-tkg.1  v1.25.13+vmware.1-fips.1-tkg.1    False   False
```

The list is long and mostly historical. The status columns are what narrow it:

| Column | Meaning |
| --- | --- |
| `READY` | The release's artifacts (OS images, component packages) are present and usable |
| `COMPATIBLE` | The release works with this Supervisor version. This is what rules out the legacy entries. |
| `DEACTIVATED BY` | Names a policy that has explicitly disabled the release. Non-empty means someone blocked it on purpose. |

Only a release that is `READY=True`, `COMPATIBLE=True`, with nothing in `DEACTIVATED BY`, is a valid
choice. In the environment used for this guide that reduced a long list to thirteen entries.

```bash
kubectl get kr --no-headers | awk '$3=="True" && $4=="True" {print $1}' | sort -V
```

Copy the `NAME` into the manifest, not the `VERSION`:

```yaml
    version: v1.36.1---vmware.4-vkr.5
```

The two forms differ. `NAME` uses a triple dash; `VERSION` uses a plus sign and appears in
`Cluster.status` and `kubectl version`. The triple dash is a DNS-safe encoding of the `+` build
separator, which is not legal in a Kubernetes object name. A single dash, a double dash, or the `+`
form all produce a topology that sits there doing nothing.

### Which one to pick

The newest compatible release is not automatically the right one.

| Consideration | Guidance |
| --- | --- |
| Patch level | Among releases of the same minor, prefer the highest patch you have validated. These carry CVE and bug fixes. |
| Validation | Prefer a release you have run somewhere else first, and stay inside the support window. |
| ClusterClass pairing | The Kubernetes version and the ClusterClass move together. Do not bump one on its own. |
| Upgrade path | One minor version at a time. Skipping minors is unsupported. |
| Fleet consistency | Standardise on one release per environment tier. Five patch levels across a fleet adds troubleshooting effort with no benefit. |

A release pins more than the Kubernetes version. It also fixes your etcd version, your CNI version,
and which node OS images you may select. [Appendix D](#appendix-d-reading-a-kubernetes-release-object)
shows how to read all of that out of the object, and includes a pre-upgrade diff worth running before
any version change.

The version used throughout this guide is illustrative. Substitute whatever your own policy selects.

---

## 3. What the platform already gives you

A VKS cluster arrives with a set of addons you do not declare. Knowing what they are saves you
installing something twice.

```bash
kubectl get clusteraddon -n <VSPHERE_NAMESPACE>
```

`ClusterAddon` is a platform-created object joining a cluster to an addon and its resolved release;
[section 4](#4-how-addons-are-delivered) covers it properly. For now it is the quickest way to see
what a fresh cluster has.

| Installed for you | Provides |
| --- | --- |
| antrea, or your chosen CNI | Pod networking and NetworkPolicy |
| vsphere-cpi | Cloud provider integration: node lifecycle, LoadBalancer services |
| vsphere-pv-csi | Persistent volume provisioning from vSphere storage |
| cluster-autoscaler | Node pool scaling |
| gateway-api | The Gateway API CRDs |
| metrics-server | Resource metrics for `kubectl top` and HPA |
| pinniped, guest-cluster-auth-service | Cluster authentication plumbing |
| secretgen-controller | Carvel secret generation and export |
| carvel-repo | The package repository addons are pulled from |
| vks-static-resources | Version-matched platform resources |

Two of those answer questions that come up constantly. The Gateway API CRDs are already present, so
`gatewayApi.enabled: true` has something to bind to without you installing anything. And the Cluster
Autoscaler is already running, which is what makes the machine-deployment scaling annotations in
[7.5](#75-worker-pools) effective rather than decorative.

These are platform-owned. Editing them by hand gets reverted and can block upgrades.

---

## 4. How addons are delivered

You author two objects. The platform creates the rest.

```
   YOU AUTHOR                       PLATFORM CREATES
   AddonInstall  ─┐
   (which addon,  │                 ClusterAddon          PackageInstall (Carvel)
    which         ├──►  addon  ──►   (cluster × addon  ──► in vmware-system-tkg,
    clusters)     │    management     × release)            inside the workload cluster
   AddonConfig   ─┘                                                 │
   (values for                                                      ▼
    one cluster)                                          kapp-controller reconciles
                                                          the addon's resources
```

| Object | Scope | Purpose |
| --- | --- | --- |
| `AddonInstall` | vSphere Namespace | Which addon, and which clusters get it |
| `AddonConfig` | vSphere Namespace | How it is configured on one cluster. Optional. |
| `ClusterAddon` | vSphere Namespace | Platform-created join object, with a `READY` condition |
| `AddonConfigDefinition` (`acd`) | `vmware-system-vks-public` | The value schema for one addon release |
| `PackageInstall` (`pkgi`) | `vmware-system-tkg`, in the workload cluster | The Carvel package that deploys the resources |

Two properties of this model are worth knowing up front, because both differ from what people expect
coming from other Kubernetes distributions.

**Addons are Carvel packages, not Helm releases.** kapp-controller reconciles them inside the
workload cluster, and `pkgi` is where their real status lives:

```bash
# Workload cluster context.
kubectl get pkgi -A
```

```
NAMESPACE           NAME                       PACKAGE NAME                          DESCRIPTION
vmware-system-tkg   <cluster>-cert-manager     cert-manager.kubernetes.vmware.com     Reconcile succeeded
vmware-system-tkg   <cluster>-istio            istio.kubernetes.vmware.com            Reconcile succeeded
```

`kubectl get helmrelease -A` returns nothing. That matters mainly for troubleshooting: the
`DESCRIPTION` column above is the authoritative signal for whether an addon converged.

**Apply order does not matter.** Every addon reconciles independently. Apply the whole manifest in
one go and let the controllers settle. In particular, helm-controller is not a prerequisite for
anything else here; it is an independent addon with its own purpose ([6.1](#61-helm-controller)).

### The naming rule

`AddonConfig.spec` has three fields, and two of them have a behaviour worth reading carefully:

```bash
kubectl explain addonconfig.spec
```

| Field | If unset |
| --- | --- |
| `clusterName` | "the AddonConfig will skip the reconciliation" |
| `addonConfigDefinitionRef` | "the AddonConfig will skip the reconciliation" |
| `values` | Your configuration overlay |

The reference profile sets neither. It relies on naming the object `<clusterName>-<addonName>`, from
which the platform derives both fields and records them. When that name does not resolve, nothing
errors; the config is simply skipped and the addon runs on schema defaults.

So the check is:

```bash
kubectl get addonconfig -n <VSPHERE_NAMESPACE> \
  -o custom-columns='CONFIG:.metadata.name,CLUSTER:.spec.clusterName,DEFINITION:.spec.addonConfigDefinitionRef.name'
```

An empty `CLUSTER` or `DEFINITION` means that config is doing nothing.

In generated or templated manifests, set both explicitly instead. It removes the failure mode, and
naming the `acd` also pins the addon release:

```yaml
spec:
  clusterName: workload-vsphere-vks2
  addonConfigDefinitionRef:
    name: istio.kubernetes.vmware.com.1.30.2---vmware.1-vks.1
    namespace: vmware-system-vks-public
  values: {...}
```

The one command that shows the whole stack at once:

```bash
kubectl get clusteraddon -n <VSPHERE_NAMESPACE>
```

```
NAME                     CLUSTER                 ADDON          ADDONINSTALL           RELEASE                 READY
<cluster>-cert-manager   workload-vsphere-vks2   cert-manager   cluster-cert-manager   cert-manager...1.20.3   True
<cluster>-prometheus     workload-vsphere-vks2   prometheus     cluster-prom           prometheus...3.5.4      True
```

Note the last row. The `AddonInstall` is called `cluster-prom` while the addon is `prometheus`. It is
the `ADDON` column that the `AddonConfig` name has to match.

---

## 5. Reading an addon's value schema

An `AddonConfig` is a sparse overlay. You set a few fields and inherit the rest, so you cannot reason
about your configuration without knowing what you are inheriting. Two equivalent ways to look:

```bash
# with the vcf CLI
vcf addon available list
vcf addon available-releases list --addon-name istio
vcf addon available-releases get <release> -o output.yaml

# with kubectl, against the Supervisor
kubectl get acd -n vmware-system-vks-public
kubectl get acd -n vmware-system-vks-public | grep '^istio'
kubectl get acd <release-name> -n vmware-system-vks-public -o yaml
```

The third command in either column is the one that gets skipped, and it is the one that matters. It
emits the full commented schema, with that release's defaults.

Three reasons to run it rather than trust a document:

- Field names are not guessable. These schemas resemble the upstream Helm chart values they derive
  from without matching them, and a field at the wrong nesting level is rejected or ignored.
- Defaults move between addon releases. Any default written into a guide is a snapshot.
- You need the defaults to know what your overlay is actually changing.

For that reason this guide does not state specific addon default values. Where a value in the
reference profile is a deliberate departure from the default, it says so and points you here.

Record the exact release you validated, and promote new ones deliberately. Naming the `acd` in
`spec.addonConfigDefinitionRef` is how you pin it.

---

## 6. The addon stack

Five addons: package lifecycle, certificates, monitoring, ingress, and a UI. It is a reasonable
baseline, and each one has at least one setting whose effect you would not guess from its name.

Every `AddonInstall` in the profile has the same shape, so here it is once:

```yaml
apiVersion: addons.kubernetes.vmware.com/v1alpha1   # alpha API; expect field changes across releases
kind: AddonInstall
metadata:
  name: cluster-istio                               # arbitrary; names the intent, not the addon
  namespace: <VSPHERE_NAMESPACE>
spec:
  addonRef:
    name: istio                                     # must match `vcf addon available list`
  clusters:
  - selector:                                       # a list, so multiple selectors are allowed
      matchLabels:
        cluster.x-k8s.io/cluster-name: <CLUSTER_NAME>
```

Cluster API applies the `cluster.x-k8s.io/cluster-name` label automatically, so selecting on it always
works with no extra labelling. Selector breadth is a policy choice: matching on cluster name targets
one cluster, while a broader selector such as `environment: dev` becomes a fleet rollout mechanism.
Useful, as long as you meant it.

If you manage several clusters from one vSphere Namespace, note that `metadata.name` must be unique
there, so `cluster-istio` will collide. Prefix with the cluster name.

Every `AddonConfig` carries this annotation, and should:

```yaml
  annotations:
    clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"
```

It ties the config's lifetime to the addon and cluster so it is cleaned up on teardown. Without it
you accumulate orphans, and since binding is name-derived, an orphan gets adopted by the next cluster
that reuses the name.

### 6.1 helm-controller

Declarative Helm chart lifecycle management inside the workload cluster. You create `HelmRelease`
objects; helm-controller and its companion source-controller reconcile them, handling upgrades,
rollbacks, and drift.

This is a day-2 capability for your own charts. It does not deliver the other addons and they do not
wait for it, so install it because you want GitOps-style Helm management, not because the stack needs
it. If you are not managing charts declaratively in this cluster, it is optional.

```yaml
spec:
  values:
    helmController:
      priorityClassName: ""
    sourceController:
      priorityClassName: ""
```

Empty string means no PriorityClass, so both controllers run at default priority and are as evictable
as a batch job under node pressure. If production workloads arrive through Helm releases, the
controller that keeps them in sync being evicted means drift goes uncorrected. `system-cluster-critical`
is the better choice.

This config also demonstrates the overlay model: it sets two fields, and every other
helm-controller setting comes from the schema.

### 6.2 cert-manager

Certificate issuance and rotation, driven by `Issuer` and `ClusterIssuer` resources.

It has no `AddonConfig` in the profile. That is legal and sensible; an addon with no config takes
schema defaults, which for cert-manager is what you want. Do not read the other four as implying a
config is mandatory.

cert-manager is a required component here rather than an optional extra. The Headlamp addon creates its own
`Issuer` and `Certificate` and terminates TLS on its Gateway with the result, so the UI depends on it:

```bash
# Workload cluster context.
kubectl get certificate,issuer -n headlamp
```

The issuer Headlamp creates is self-signed, which is fine in a lab and produces browser warnings
everywhere else. [Appendix B.2](#b2-a-trusted-certificate-for-the-ui) covers replacing it.

### 6.3 Prometheus

The profile enables the exporters and the operator, and deliberately leaves out the packaged
Prometheus server:

```yaml
spec:
  values:
    deploycomponents:
      kube-state-metrics: true       # Kubernetes object state
      node-exporter: true            # per-node OS and hardware metrics
      pushgateway: false
      prometheus: false              # operator-managed instead; see below
      alertmanager: true
      prometheus-operator: true
```

This is the operator-managed pattern. Instead of accepting a packaged server with its baked-in
settings, you enable the operator and declare your own `Prometheus` custom resource, which gives you
control over replicas, retention, persistent storage, scrape configuration, and external labels as
version-controlled objects. For anyone who cares about their monitoring configuration it is the right
choice.

It has one requirement that is easy to miss. Until you create that CR, the operator and CRDs are
installed, the exporters are producing metrics, and nothing is scraping or evaluating them:

```bash
# Workload cluster context.
kubectl get prometheus,alertmanager,servicemonitor -A
# "No resources found" means the operator is idle
```

[Appendix B.3](#b3-a-prometheus-instance) has the CR. This is the most important follow-up action in
the whole profile.

On the individual components: `kube-state-metrics` and `node-exporter` are the minimum for cluster and
node visibility, and node-exporter runs as a DaemonSet needing host access, which makes it a good
candidate for a Pod Security namespace exemption ([7.6](#76-pod-security)) rather than a cluster-wide
relaxation. `pushgateway: false` is correct; Pushgateway retains metrics indefinitely and becomes a
source of stale data outside genuine batch use cases. Alertmanager needs both a rule evaluator and
configured receivers, or it delivers nowhere.

Two spelling traps. The key is lowercase `deploycomponents`, and the component names are hyphenated.
Keys are matched literally, so a tidied-up camelCase key is not applied. And the `AddonInstall` here
is named `cluster-prom` while the addon is `prometheus`, so the config must be
`<cluster>-prometheus`. Calling it `-prom` matches nothing.

### 6.4 Istio

Istio is in this stack for one job: L4 and L7 traffic management. It provides the ingress gateway and
the Gateway API controller that exposes services outside the cluster, including the Headlamp UI.

It is not being used as a service mesh. No sidecars are injected, no workload mTLS is established, no
east-west policy applies. That is a legitimate and increasingly common way to run Istio, and it keeps
the footprint small, but it changes which settings matter. You can confirm the scope on any cluster:

```bash
# Workload cluster context.
kubectl get ns -L istio-injection,istio.io/dataplane-mode
# every namespace blank in both columns means gateway-only
```

```yaml
spec:
  values:
    istio:
      namespace: "istio-system"        # guest namespace; the addon creates and owns it
      ambientMode:
        enabled: false
      istioCNI:
        enabled: false                 # see the note below
      gateways:
        ingress:
          enabled: true                # the reason this addon is here
          namespace: istio-ingress
          autoscaling:
            enabled: true
            minReplicas: 1
            maxReplicas: 5
      pilot:
        replicas: 1
        autoscaling:
          enabled: true
          minReplicas: 1
          maxReplicas: 2
      meshConfig:
        accessLogFile: ""
        enableTracing: false
        meshID: "workload-vsphere-vks2"
```

| Setting | Comment |
| --- | --- |
| `gateways.ingress.namespace: istio-ingress` | Putting the gateway in its own namespace, separate from the control plane, limits blast radius and lets you grant teams access to gateway resources without control-plane access. Worth copying. Note that a Gateway API `Gateway` created with `create: true` provisions its own LoadBalancer in its own namespace and does not share this one. |
| Ingress `minReplicas: 1` | A single ingress pod means all inbound traffic fails during any reschedule: node drain, upgrade, eviction. Use 2, with a PodDisruptionBudget and anti-affinity across nodes. |
| `pilot.replicas: 1` alongside an HPA | When an HPA owns a Deployment it owns `spec.replicas`, so the static value is not authoritative and causes reconcile churn. Drop it and express intent through `minReplicas`. |
| `pilot.autoscaling.minReplicas: 1` | istiod is the Gateway API controller. While it is down, existing gateways keep forwarding on last-known config but no `Gateway` or `HTTPRoute` change takes effect, including istiod's own upgrade. Use 2. |
| `meshConfig.accessLogFile: ""` | Empty disables Envoy access logs. At this scope those are your ingress request logs: what arrived, which route matched, what status came back. That is the tool you reach for when an `HTTPRoute` 404s or an OIDC redirect fails. Enable `/dev/stdout` during bring-up and while validating auth, then decide based on volume. |
| `meshConfig.enableTracing: false` | Of limited value without a mesh; you would see gateway spans only. Setting it true also needs a tracing provider configured. |
| `meshConfig.meshID` | Harmless and tidy in telemetry, largely inert at this scope. It matters if you later federate clusters into one mesh, which needs a shared `meshID` with distinct `network` values. If that is on your roadmap, name the mesh rather than the cluster. |

#### On Istio CNI and Pod Security

There is a widely repeated claim that Istio forces you into `privileged` Pod Security. It is true, but
only under a condition this deployment does not meet, so it is worth being precise.

With sidecar injection enabled and Istio CNI off, traffic redirection is done by an init container in
every injected pod, and that container needs `NET_ADMIN` and `NET_RAW`. Both the `baseline` and
`restricted` standards forbid those capabilities. Enabling `istioCNI` moves the work to a per-node
DaemonSet, so workload pods need nothing elevated.

This cluster injects no sidecars, so no workload pod requires those capabilities and
`istioCNI.enabled: false` places no constraint on your Pod Security choice. The `privileged` setting
in [7.6](#76-pod-security) is a lab decision, not a consequence of this Istio configuration.

| Your Istio scope | `istioCNI` | Pod Security impact |
| --- | --- | --- |
| Gateway and ingress only | `false` is fine | None |
| Sidecar mesh | `false` | Injected pods need `NET_ADMIN`, so `privileged` |
| Sidecar mesh | `true` | Nothing elevated in workload pods |
| Ambient mesh | n/a | Per-node ztunnel, no per-pod privilege |

If you adopt the mesh later, enable `istioCNI` in the same change. Retrofitting it after labelling
namespaces for injection means every injected workload needs `privileged` in the meantime.

If all you need is HTTP ingress and a mesh is not on the roadmap, the `contour` addon is lighter and
has less operational surface. Istio makes sense when you want Gateway API with a path to a mesh, or
richer L4/L7 policy later.

### 6.5 Headlamp

A web UI for the cluster, served through a Gateway API `Gateway` and authenticated with OIDC. It is
the most cross-coupled object in the profile and the source of most "it deployed but does not work"
reports.

```yaml
spec:
  values:
    hostname: headlamp.k8s.example.com
    gatewayApi:
      enabled: true
      gateway:
        className: istio                 # requires a gateway controller
        create: true                     # the addon owns this Gateway and its LoadBalancer
        name: headlamp-gateway
    oidc:
      enabled: true
      issuerURL: https://idp.example.com
      clientID: <OIDC_CLIENT_ID>
      clientSecret: <OIDC_CLIENT_SECRET>
      callbackURL: https://headlamp.k8s.example.com/oidc-callback
      scopes:
      - openid
      - email
      - profile
```

It has one hard prerequisite. `gatewayApi.gateway.className` must name a controller that exists, which
means installing either the `istio` addon with `gateways.ingress.enabled: true`, or `contour`. Without
one the `Gateway` object is created and stays unprogrammed forever: Headlamp runs, nothing routes to
it. This is a completeness requirement rather than an ordering one, since it resolves itself once the
controller appears.

```bash
# Workload cluster context.
kubectl get gateway -A
# PROGRAMMED must be True with an ADDRESS
```

The Gateway API CRDs themselves come with the platform ([section 3](#3-what-the-platform-already-gives-you)),
so their presence tells you nothing about whether a controller exists.

#### Hostname, DNS, and the redirect URI

This is the most common source of lost time, because the three values live in three different systems
and a mismatch surfaces as an error page from your identity provider rather than anything in
Kubernetes.

With `create: true` the addon provisions its own `Gateway` and a dedicated LoadBalancer in the Headlamp
namespace, with its own external IP. That address, not the Istio ingress gateway's, is what DNS must
point at:

```bash
# Workload cluster context.
kubectl get gateway -A          # ADDRESS column
kubectl get svc -n headlamp     # headlamp-gateway-<class>, EXTERNAL-IP
```

Three steps, in any order:

1. Choose the FQDN and use it for both `hostname` and the host in `callbackURL`.
2. Publish a DNS A record for it, pointing at the Gateway's address. Reserve that IP with your
   load-balancer provider so the value is stable.
3. Register `https://<FQDN>/oidc-callback` as an authorized redirect URI at your identity provider.
   Most providers reject anything unregistered outright.

Do all three up front and nothing depends on discovering an IP after the fact, so the whole manifest
can go in at once.

If your starting manifest uses an IP-derived hostname, replace it. Services like `<ip>.sslip.io`
resolve `anything.<ip>.sslip.io` to that address and are handy when you control no DNS, but they wire
the LoadBalancer IP into the hostname, the callback URL, and your provider's redirect registration all
at once, and they make name resolution for your management UI depend on a third party.

| Field | Comment |
| --- | --- |
| `oidc.issuerURL` | Must match the token's `iss` claim byte for byte, trailing slash included, and match the API server's issuer URL. |
| `oidc.clientID` | Must also appear in the API server's `audiences` list. If it does not, users authenticate successfully and then fail every API call, which looks like a broken UI and is not. |
| `oidc.clientSecret` | Plaintext in etcd and in any repository holding the manifest. Check your release's schema for a secret-reference field, and manage the Secret with sealed-secrets, external-secrets, or the `vault-injector` addon. |
| `oidc.callbackURL` | Must be registered at the provider, and its host must equal `hostname`. |
| `oidc.scopes` | `openid` is required by the spec. `email` is load-bearing, because the API server maps the email claim to the Kubernetes username; drop the scope and login appears to work while authorization fails everywhere. If you change the API server's `username.claim`, request the matching scope here. |
| TLS | The addon creates a cert-manager `Issuer` and `Certificate` and terminates TLS automatically, so you get HTTPS with no configuration. It is self-signed. See [Appendix B.2](#b2-a-trusted-certificate-for-the-ui). |
| `gateway.create: true` | The addon owns the object, so hand edits are reverted on the next reconcile. Customise through `AddonConfig` values, or set `create: false` and manage the `Gateway` yourself. |

Logging in successfully grants no permissions. [Section 7.7](#77-identity-oidc-and-rbac) covers the
RBAC side, which is not optional.

### 6.6 The rest of the catalogue

`vcf addon available list` shows what your environment offers. None of the following is a blanket
recommendation; what belongs in your cluster depends on what you already run elsewhere.

Three of them come up in almost every production conversation and are covered separately in
[Appendix C](#appendix-c-integrations-that-need-an-external-system), because each needs a decision
about an external system before it does anything: `velero` for backup, `fluent-bit` for log
forwarding, `external-dns` for DNS automation.

| Addon | When it fits |
| --- | --- |
| `contour` | A focused HTTP ingress controller, lighter than Istio if you want ingress and nothing more. Also a valid `className` provider for the Headlamp Gateway. |
| `ako` | NSX Advanced Load Balancer integration for L4-L7. The right answer if NSX ALB is already your standard. Requires NSX ALB. |
| `multus-cni` | Multiple network interfaces per pod. Standard for NFV and telco workloads, or anything needing a data plane separate from cluster traffic. |
| `whereabouts` | Cluster-wide IPAM for secondary interfaces. Effectively required alongside `multus-cni`, since without it secondary addressing does not coordinate across nodes. |
| `sriov-network-device-plugin` | Exposes SR-IOV virtual functions as schedulable resources for high-throughput, low-latency workloads. |
| `nfs-client` | An NFS CSI driver, for `ReadWriteMany` volumes that vSphere block storage does not provide. |
| `vault-injector` | Injects secrets from HashiCorp Vault into pods. A better answer to credential handling than Kubernetes Secrets alone. Requires an existing Vault. |
| `harbor` | An in-cluster OCI registry with scanning, signing, and replication. Worth considering as a supply-chain control point; needs storage and its own lifecycle. |
| `telegraf` | Metric collection and forwarding, for pushing to an external TSDB such as InfluxDB. Requires an output target. |
| `gatekeeper`, `policy-bundle` | OPA Gatekeeper and VMware's curated compliance policies. Relevant with a specific compliance target; Pod Security already covers pod privilege. |
| `windows-gmsa-webhook` | Group Managed Service Account support for Windows containers. |
| `vsphere-pv-csi-webhook` | Additional validation for the vSphere CSI provider. |

Every addon is a workload. Each consumes CPU, memory, and a pod slot on every node it DaemonSets
onto, and each is another component to upgrade and troubleshoot. Add what you will operate.

---

## 7. The cluster

```yaml
apiVersion: cluster.x-k8s.io/v1beta2      # upstream Cluster API
kind: Cluster
metadata:
  name: <CLUSTER_NAME>                    # matches every addon selector
  namespace: <VSPHERE_NAMESPACE>
```

VKS builds on standard Cluster API, so upstream CAPI concepts and troubleshooting apply directly. Do
not confuse this API group with the `addons.kubernetes.vmware.com` group from section 6.

Renaming a cluster is a rebuild, not an operation: the name is embedded in every addon selector and
every `AddonConfig` name.

### 7.1 Pod and service networking

```yaml
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
      - 240.0.0.0/20
    serviceDomain: cluster.local
    services:
      cidrBlocks:
      - 240.1.0.0/20
```

Both CIDRs are immutable after creation. The service range caps total `ClusterIP` services, and a
`/20` gives about 4,094 of them, which is rarely the constraint. The pod range needs more thought.

`serviceDomain` is the cluster's internal DNS suffix. `cluster.local` is the universal default and
changing it breaks anything that assumes it, including Istio's trust domain if you later adopt the
mesh. Leave it alone.

#### Why 240.0.0.0/4

That range is IANA-reserved and not routable on the public internet or, normally, inside a
datacenter. Using it for pod and service networks guarantees no collision with routable RFC1918
space, which matters when pods reach on-premises systems and when you may later peer or federate
clusters. Overlapping CIDRs are among the hardest problems to unpick after the fact, so this is a
good habit.

Validate it before adopting it, though. Because the range is reserved, some CNIs, OS network stacks,
hardware load balancers, and firewalls reject or mishandle it. It works with Antrea; confirm it
against every appliance in the path.

#### How pod addresses are actually allocated

Allocation happens at two levels, and the second one is what creates the limit.

The cluster CIDR defines one pool. As each node joins, the node IPAM controller carves a fixed-size
block out of that pool and assigns it to that node, visible as `Node.spec.podCIDR`. The block belongs
to that node; pods elsewhere cannot draw from it even when it sits mostly empty. The CNI then
allocates pod IPs only from the block its own node owns.

The per-node block defaults to a `/24`. So the cluster CIDR is not a flat pool of pod IPs, it is a
pool of per-node `/24` blocks, and running out of blocks stops you adding nodes:

```
240.0.0.0/20  =  4096 addresses  =  2^(24-20) = 16 blocks
```

Sixteen blocks is the whole cluster's node budget, control plane included. You can see the allocation
on any running cluster:

```bash
# Workload cluster context.
kubectl get nodes -o custom-columns='NODE:.metadata.name,POD_CIDR:.spec.podCIDR,MAXPODS:.status.capacity.pods'
```

```
NODE                              POD_CIDR       MAXPODS
<cluster>-8dbnf-tf6pb             240.0.0.0/24   110      control plane, block 1
<cluster>-node-pool-1-...-m9b9p   240.0.1.0/24   110      worker, block 2
```

#### Keep a spare block for upgrades

Every rolling operation in Cluster API creates the replacement node before deleting the one it
replaces. A `MachineDeployment` rolling update defaults to `maxSurge: 1`, and a control-plane rollout
scales up before scaling down. For the duration of that overlap the cluster runs N+1 nodes, and the
extra node needs a block.

If every block is allocated, the surge node has nowhere to get a pod CIDR. It joins, may report
`Ready`, and hosts nothing. The rollout neither completes nor fails cleanly; it stalls with the
cluster in a mixed-version state. Since the pod CIDR is immutable, a cluster that has consumed every
block cannot be upgraded out of the situation either.

This applies to every rolling node replacement, not just version changes. A `vmClass` change, a
`volumes` resize, a `maxPods` change, and an OS image change all need the spare block.

Reserve one at minimum. Reserve one per pool that may roll concurrently, one per surge slot if your
rollout strategy allows more than one, and in practice keep two or three plus room for pools you have
not created yet. Block reclamation is not instant either: a node stuck `Terminating` holds its block,
so at zero margin one stuck node blocks the whole rollout.

#### maxPods, and the density lever

The per-node block also bounds pods per node, but a kubelet setting usually binds first. VKS 3.7
exposes it as a ClusterClass variable, which earlier generations did not:

```yaml
        kubeletConfiguration:
          maxPods: 250        # default 110, documented maximum 250, minimum 20
```

```bash
MP='.status.variables[?(@.name=="kubernetes")].definitions[0].schema.openAPIV3Schema.properties.kubeletConfiguration.properties.maxPods'
kubectl get clusterclass builtin-generic-v3.7.0 -n vmware-system-vks-public \
  -o jsonpath="{$MP.description}{\"\nminimum: \"}{$MP.minimum}{\"\n\"}"
```

```
MaxPods is the number of pods that can run on this Kubelet.
Default: 110
NOTE: By default, the maximum allowed value is 250.
minimum: 20
```

At the default of 110, roughly 43% of each `/24` is used and the kubelet refuses the 111th pod long
before addresses run out. At 250 the block becomes the binding limit, with about 254 usable addresses
against 250 pods.

That gives you a lever. Raising `maxPods` increases pod capacity without consuming more blocks, so a
cluster already fixed at a `/20` can go from roughly 1,650 pods to 3,750 without touching the
immutable CIDR. It is not free, and [7.3](#73-node-sizing) covers what the node has to look like to
support it.

#### Sizing table

Assumes default `/24` blocks and one reserved surge block.

| Pod CIDR | Addresses | Blocks | Steady-state nodes | Pods at 110 | Pods at 250 |
| --- | --- | --- | --- | --- | --- |
| `/22` | 1,024 | 4 | 3 | 330 | 750 |
| `/21` | 2,048 | 8 | 7 | 770 | 1,750 |
| `/20` | 4,096 | 16 | 15 | 1,650 | 3,750 |
| `/19` | 8,192 | 32 | 31 | 3,410 | 7,750 |
| `/18` | 16,384 | 64 | 63 | 6,930 | 15,750 |
| `/17` | 32,768 | 128 | 127 | 13,970 | 31,750 |
| `/16` | 65,536 | 256 | 255 | 28,050 | 63,750 |

Applied to real cluster shapes, a `/20` gives:

| Shape | Nodes | Blocks used | Spare on a `/20` |
| --- | --- | --- | --- |
| The reference profile: 1 control plane, 5 workers | 6 | 7 with surge | 9 |
| A production shape: 3 control plane, 10 workers | 13 | 14 with surge | 2 |
| The same, plus a 5-node GPU pool | 18 | 19 with surge | does not fit |

The profile is fine as written. The difficulty is that it is easy to treat as a template, and the
shape it should grow into does not fit a `/20` with any margin, while the field you would need to
change is the one you no longer can.

#### Recommendation

Use a `/16` unless you have a specific reason not to. Drawn from reserved space, address scarcity is
not a real constraint; these addresses are not routable outside the cluster and compete with nothing.
A `/16` gives 255 nodes and removes the whole class of problem.

If you must use something smaller, size for your maximum plausible node count including future pools,
add the surge reserve on top, and cross-check the total against every autoscaler maximum plus
`controlPlane.replicas`. Then alert on block allocation rather than on scheduling failures: "12 of 16
blocks used" is actionable, and discovering it during a stalled upgrade is not.

```bash
# Workload cluster context.
kubectl get nodes -o jsonpath='{range .items[*]}{.spec.podCIDR}{"\n"}{end}' | sort -u | wc -l
```

Other things that consume the per-node budget: DaemonSets take a slot on every node permanently, so
the CNI agent and node-exporter are gone before an application pod lands. Short-lived pods hold IPs
briefly after termination, so high-churn CI runners and CronJobs can pressure a block well above
steady state. Istio sidecars, if you adopt the mesh, cost containers rather than pods or IPs.

### 7.2 ClusterClass and version

```yaml
  topology:
    classRef:
      name: builtin-generic-v3.7.0
      namespace: vmware-system-vks-public
    version: v1.36.1---vmware.4-vkr.5
```

The ClusterClass is the template that turns a short `Cluster` spec into full infrastructure. It is
VMware-supplied, read-only, and version-paired with the VKS release. Objects in `vmware-system-*`
namespaces are platform-managed; edits get reverted and can block upgrades.

The important consequence is that the ClusterClass defines which `topology.variables` exist. The
variable set below is valid because this class declares it, and is not portable across ClusterClass
versions:

```bash
kubectl get clusterclass builtin-generic-v3.7.0 -n vmware-system-vks-public \
  -o jsonpath='{range .status.variables[*]}{.name}{"\n"}{end}'
```

```
bootstrapAddons   kubeAPIServerFQDNs   kubernetes   node   osConfiguration
resourceConfiguration   storageClass   vmClass   volumes   vsphereOptions
```

Ten variables; the profile uses seven. `node` and `resourceConfiguration` are both worth adding and
appear below. `kubeAPIServerFQDNs` is marked deprecated in the schema in favour of
`kubernetes.endpointFQDNs`, so migrate if you have manifests using it.

To read one variable's full schema, including descriptions and constraints:

```bash
kubectl get clusterclass builtin-generic-v3.7.0 -n vmware-system-vks-public \
  -o jsonpath='{.status.variables[?(@.name=="kubernetes")].definitions[0].schema.openAPIV3Schema}' | less
```

Because the ClusterClass is version-pinned in the manifest, moving to a newer VKS generation is a
deliberate act, and the new class may add, rename, or remove variables. Diff the two schemas before
the upgrade rather than after a failed apply.

### 7.3 Node sizing

VM class, node volumes, system reservations, and `maxPods` are one decision, not four. Treating them
separately is how you end up with a node that admits 250 pods and has 20 MiB of memory for each.

```yaml
    variables:
    - name: storageClass
      value: <STORAGE_CLASS>              # the nodes' own disks
    - name: vmClass
      value: guaranteed-medium
    - name: volumes
      value:
      - capacity: 30Gi
        mountPath: /var/lib/containerd
        name: containerd
        storageClass: <STORAGE_CLASS>
      - capacity: 30Gi
        mountPath: /var/lib/kubelet
        name: kubelet
        storageClass: <STORAGE_CLASS>
```

#### VM class: reserve resources in production

The VM class sets each node's CPU and memory shape and, critically, whether those resources are
reserved.

`best-effort-*` classes request without reserving. Under vSphere contention the VM is not guaranteed
the CPU and memory it advertises to Kubernetes. `guaranteed-*` classes reserve.

The reason to be firm about this in production is how the failure presents, not how severe it is.
Kubernetes still believes the node has its full allocatable capacity and schedules accordingly, while
the hypervisor quietly delivers less. What you see is `NotReady` kubelets, etcd leader-election churn,
failing liveness probes, and random timeouts. Those symptoms point at Kubernetes, so that is where
people investigate, and meanwhile capacity planning based on Kubernetes' view of the node is wrong.

Use `best-effort` for dev and test, where density matters more than predictability. Use `guaranteed`
for production, and for the control plane at any tier.

A default catalogue, which you should confirm with `kubectl get virtualmachineclass`:

| Class | vCPU | Memory | Typical use |
| --- | --- | --- | --- |
| `guaranteed-small` | 2 | 4 GiB | Minimal workers |
| `guaranteed-medium` | 2 | 8 GiB | General-purpose workers |
| `guaranteed-large` | 4 | 16 GiB | Control plane, heavier workers |
| `guaranteed-xlarge` | 4 | 32 GiB | Memory-intensive workloads |
| `guaranteed-2xlarge` | 8 | 64 GiB | Large control planes, dense nodes |
| `guaranteed-4xlarge` | 16 | 128 GiB | High-density nodes |

Environments often define their own classes too. Where they do, the `best-effort` / `guaranteed`
naming convention may not apply, so confirm whether a custom class actually reserves.

Changing `vmClass` rebuilds every node in the pool.

#### Dedicated containerd and kubelet volumes

Keep this part of the profile. It is one of its better decisions.

By default both directories sit on the node's root disk. `/var/lib/containerd` holds every image layer
pulled onto the node; `/var/lib/kubelet` holds ephemeral pod storage, `emptyDir` volumes, and pod
logs. Both grow in ways you do not directly control, so on a shared root disk one image-pull storm or
a single pod writing large log volumes can fill the filesystem. When that happens the kubelet degrades,
evicting pods under disk pressure and eventually ceasing to function. One misbehaving workload takes
the node with it.

Separate volumes contain that. A full containerd volume means image pulls fail, which is bad but
diagnosable. A full root disk means the node is gone.

30Gi is modest. Image-heavy workloads, AI/ML frameworks, large JVM or data-science images, and CUDA
layers will exhaust it, and multiple image versions accumulate. Density matters too: a node running
250 pods needs considerably more ephemeral space and image cache than one running 110.

Size generously up front, because adding or resizing a volume is a rolling node replacement, not an
in-place change. Every node in the pool is destroyed and rebuilt. That is survivable with a healthy
multi-node pool and PodDisruptionBudgets, but it is slow, it needs a spare pod CIDR block, and with a
single control-plane replica it is an outage.

#### System reservations

`resourceConfiguration.systemReserved` holds back CPU and memory for the kubelet, container runtime,
and OS, so pods cannot starve them. Starved system daemons produce `NotReady` nodes and cascading
evictions.

```yaml
    - name: resourceConfiguration
      value:
        systemReserved:
          automatic: true
```

At the default `maxPods` of 110, leave `automatic: true` and move on. The upstream Kubernetes
documentation on
[reserving compute resources](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/)
publishes no recommended values, and advises enforcing `systemReserved` only after profiling nodes
exhaustively, since an over-tight reservation can leave critical system services CPU starved, OOM
killed, or unable to fork. Preferring the automatic calculation is the upstream position, not a
hedge.

Where public formulas do exist is at the managed providers, and they are useful because they quantify
the two different things that drive overhead.
[GKE](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/plan-node-sizes) and
[EKS](https://docs.aws.amazon.com/batch/latest/userguide/memory-cpu-batch-eks.html) both reserve CPU
as a tiered percentage of core count: 6% of the first core, 1% of the second, 0.5% of the third and
fourth, 0.25% of anything above four. GKE reserves memory as a tiered percentage of node memory
(25% of the first 4 GiB, 20% of the next 4, 10% of the next 8, 6% of the next 112, 2% above 128) plus
100 MiB for eviction. EKS instead scales memory with pod count, at `(11 MiB x max_pods) + 255 MiB`.

You can see what VKS actually does. The gap between `capacity` and `allocatable` is the total
reservation plus the eviction threshold:

```bash
# Workload cluster context.
kubectl get nodes -o custom-columns='NODE:.metadata.name,\
CPU_CAP:.status.capacity.cpu,CPU_ALLOC:.status.allocatable.cpu,\
MEM_CAP:.status.capacity.memory,MEM_ALLOC:.status.allocatable.memory,\
MAXPODS:.status.capacity.pods'
```

Measured on a VKS 3.7 cluster with `automatic: true` and `maxPods: 110`:

| Node | Capacity | Reserved | GKE formula predicts |
| --- | --- | --- | --- |
| 4 vCPU, 15.6 GiB | 4 / 15.6 GiB | 80m / 2722 MiB | 80m / 2723.2 MiB |
| 2 vCPU, 7.75 GiB | 2 / 7.75 GiB | 70m / 1892 MiB | 70m / 1892.8 MiB |

Exact on CPU and within about 1 MiB on memory, across two node sizes, once the 100 MiB eviction
reserve is included. So the tiered model is a reliable predictor of what `automatic: true` will give
you on a given VM class, which is useful for planning before a node exists. Two data points is a good
fit rather than a guarantee, and VMware documents only the flag and not its calculation, so confirm
against your own nodes.

The gap appears when you raise `maxPods`. That reservation is derived from node size, not pod count,
so going from 110 to 250 more than doubles the kubelet and runtime bookkeeping while the automatic
figure does not move. Both providers acknowledge this: EKS scales memory at 11 MiB per pod, and GKE
documents an extra 400 mCPU once pods per node exceed 110. Combining them gives a defensible starting
point:

```
memory = automatic reservation for the VM class  +  11 MiB x (maxPods - 110)
cpu    = tiered CPU reservation for the core count  +  400m   when maxPods > 110
```

```yaml
    # a pool running maxPods: 250 on guaranteed-2xlarge (8 vCPU, 64 GiB)
    - name: resourceConfiguration
      value:
        systemReserved:
          automatic: false
          cpu: 500m            # 90m tiered plus the 400m density adder
          memory: 7Gi          # about 5.5Gi of node-size tiers plus 1.5Gi for 140 extra pods
```

#### What density costs

Reserved resources come off allocatable, so reservation and pod count together decide what is left per
pod. This is where an ambitious `maxPods` usually falls apart:

| VM class | vCPU / memory | Reserved at 250 | Allocatable | Average memory per pod |
| --- | --- | --- | --- | --- |
| `guaranteed-medium` | 2 / 8 GiB | 470m / 3.35 GiB | 4.4 GiB | 18 MiB, unusable |
| `guaranteed-large` | 4 / 16 GiB | 480m / 4.15 GiB | 11.4 GiB | 47 MiB, unrealistic |
| `guaranteed-xlarge` | 4 / 32 GiB | 480m / 5.10 GiB | 25.9 GiB | 106 MiB, tight |
| `guaranteed-2xlarge` | 8 / 64 GiB | 490m / 6.97 GiB | 55.1 GiB | 226 MiB, workable |
| `guaranteed-4xlarge` | 16 / 128 GiB | 510m / 10.69 GiB | 113.5 GiB | 465 MiB, comfortable |

So: raising `maxPods` to 250 means planning on `guaranteed-2xlarge` or larger. And CPU usually binds
before memory, since 8 vCPU across 250 pods is about 29 millicores each, which suits many small
services and nothing compute-bound. Size from your workload's actual requests rather than from the pod
count.

Other consequences of density worth weighing: losing a node evicts 250 pods rather than 110, so risk
concentrates; draining a 250-pod node takes considerably longer, so upgrades slow down; and DaemonSet
overhead is per node, so fewer denser nodes is a real efficiency gain if you run several. On a `/24`
block there is almost no headroom at 250, so keep sustained density below roughly 220 to absorb pod IP
churn.

Set these per pool rather than cluster-wide, using the `variables.overrides` mechanism shown in
[7.4](#74-control-plane), so a dense pool gets a large reservation without imposing it on pools that
do not need one. Then measure: compare `capacity` against `allocatable` after deployment and watch
actual kubelet and runtime usage under load. Every reserved core is capacity you paid for and cannot
schedule, so do not over-reserve, but err high on memory and low on CPU. Memory is incompressible and
exhausting it OOM-kills system processes, whereas CPU starvation degrades. Upstream makes the same
point: reserving only compressible resources is less likely to cause disruption.

One terminology note. Upstream splits reservations into `kube-reserved` for the kubelet and runtime
and `system-reserved` for OS daemons. The provider formulas above are `kube-reserved`. VKS exposes a
single `systemReserved` knob, so if you override it, size it for total node overhead and validate with
the capacity-versus-allocatable check.

#### Node labels

```yaml
    - name: node
      value:
        labels:
          workload-type: general
          environment: production
```

Use this rather than `kubectl label node`. Labels applied by hand are lost when a node is replaced,
which happens on every upgrade, every `vmClass` change, and every `volumes` resize. Labels set here
are part of the node's declared configuration and survive.

Useful for scheduling with `nodeSelector` or affinity, for environment identification in dashboards
and alerts, for cost attribution, and for scoping compliance controls. The variable also carries
`taints`, `cri.runtimeClasses`, and `firewall.inboundRules`; reach for those when you have a concrete
requirement rather than as part of a baseline.

### 7.4 Control plane

```yaml
    controlPlane:
      metadata:
        annotations:
          run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
      replicas: 1
      variables:
        overrides:
        - name: vmClass
          value: guaranteed-large
```

`replicas: 1` gives you no etcd quorum and no high availability. Any control-plane failure, or any
rolling operation touching it, is a full API-server outage. Workloads already scheduled keep running
and the dataplane survives, but nothing can be scheduled, scaled, or changed, and every controller
stops reconciling. Use 3 in production. Going from 1 to 3 later is supported but is a rolling change
to plan into a window.

The `variables.overrides` block is a pattern worth learning. It overrides a cluster-wide variable for
just the control plane, so you can size it independently without duplicating the whole variable set.
The same mechanism works on machine deployments. One caveat: an override silently shadows the
cluster-wide value, so when a node has unexpected resources, check here before assuming the
cluster-wide variable applies.

Giving the control plane both a larger and a reserved class is the right instinct, since etcd is
latency-sensitive and intolerant of starvation. `guaranteed-large` at 4 vCPU and 16 GiB is a sound
production size for small-to-medium clusters; scale up for high object counts or many nodes.

The `resolve-os-image` annotation selects the node OS image by attribute rather than by a brittle
image name. It is a label selector against `OSImage` objects, which means you can test it before
applying:

```bash
kubectl get osimage -l os-name=ubuntu,os-version=24.04,run.tanzu.vmware.com/tkr=<kr-name>
```

Expect exactly one match. No match means the annotation never resolves and machine creation stalls
with a thin error surface. [Appendix D](#appendix-d-reading-a-kubernetes-release-object) covers the
label set and which OS images a release offers.

Set the same annotation on the control plane and every machine deployment, or the pools drift onto
different images.

### 7.5 Worker pools

```yaml
    workers:
      machineDeployments:
      - class: node-pool
        metadata:
          annotations:
            cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "1"
            cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "5"
            run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
        name: node-pool-1
```

`class` references a machine-deployment class defined by the ClusterClass, and this entry instantiates
it. The autoscaler annotations work, because the Cluster Autoscaler is installed by the platform
([section 3](#3-what-the-platform-already-gives-you)); in some distributions they would be inert.

Note the absence of `replicas`. That is correct with an autoscaler-managed pool, since a static value
and an autoscaler would fight over the same field and oscillate. If you deliberately disable
autoscaling for a pool you must set `replicas` instead. Choose one model, never both.

A minimum of 1 means no capacity redundancy: one node failure and everything is unschedulable. Use at
least 2, ideally 3. Sanity-check the maximum against the pod CIDR arithmetic in
[7.1](#71-pod-and-service-networking), against vSphere capacity, and against VM-class reservations.

Multiple pools are how you separate workloads: different VM classes, different `maxPods`, taints for
GPU or memory-optimised work. A single undifferentiated pool means every workload shares one node
shape. Combine extra pools with the per-pool overrides from 7.4:

```yaml
      - class: node-pool
        name: node-pool-memory
        variables:
          overrides:
          - name: vmClass
            value: guaranteed-2xlarge
          - name: node
            value:
              labels:
                workload-type: memory-optimised
```

### 7.6 Pod Security

```yaml
        security:
          podSecurityStandard:
            enforce: privileged
            enforceVersion: latest
            audit: privileged
            auditVersion: latest
            warn: privileged
            warnVersion: latest
            deactivated: false
```

These are lab settings. `privileged` at all three levels means Pod Security enforces nothing, which is
right for exploring a platform without admission control rejecting workloads. It is not required by anything
else in the profile; see [the Istio note](#on-istio-cni-and-pod-security).

Three levels, weakest to strongest: `privileged` permits everything; `baseline` blocks known privilege
escalations while staying broadly compatible; `restricted` enforces hardening, requiring non-root, no
privilege escalation, dropped capabilities, and a seccomp profile. Three modes: `enforce` rejects at
admission and is the only one that blocks anything, while `audit` records violations and `warn`
returns a warning to the client.

`*Version: latest` is a genuine upgrade hazard. It means whatever the running Kubernetes version
defines, and PSS definitions tighten over time as escape vectors close. So a cluster upgrade can
silently make your admission policy stricter and reject workloads that deployed cleanly the day
before, during an upgrade, when you are already busy. Pin an explicit version and raise it as a
separate, tested change.

There is also an option worth taking that costs nothing. Setting `enforce: privileged` while raising `audit`
and `warn` to `restricted` gives you a complete inventory of exactly which workloads would break under
stricter enforcement, without rejecting anything. It is the standard way to plan a PSS migration and
it costs nothing. Do it today.

#### Two levers, and when to use each

Once you enforce anything above `privileged`, you need an approach for namespaces that legitimately
need more. There are two mechanisms, they behave differently, and picking the wrong one is the most common
Pod Security mistake after leaving it at `privileged`.

| | Per-namespace labels | `exemptions.namespaces` |
| --- | --- | --- |
| Lives on | The `Namespace` object | The `Cluster` manifest, in the API server admission config |
| Effect | Applies a different level to that namespace | Bypasses Pod Security entirely |
| Granularity | Per level and per mode | All or nothing: no enforce, audit, or warn |
| Controlled by | Whoever can label namespaces | Platform team only |
| Cost to change | A label edit | A `Cluster` edit and a control-plane reconfiguration |
| Keeps audit trail | Yes, violations still logged | No |

Prefer labels. An exemption does not grant a lower level, it turns the policy off, and you lose
`audit` and `warn` visibility with it. A label keeps the policy engaged at a level the workload can
actually meet:

```yaml
# Apply in the workload cluster context, not the vSphere Namespace.
apiVersion: v1
kind: Namespace
metadata:
  name: legacy-app
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.36
    # keep audit and warn stricter than enforce, so you still learn what
    # would fail under restricted without blocking the workload today
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.36
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.36
```

Reserve exemptions for platform namespaces that cannot run under any enforced level: a short, stable
list the platform team owns.

```yaml
          podSecurityStandard:
            enforce: baseline
            enforceVersion: v1.36
            audit: restricted
            auditVersion: v1.36
            warn: restricted
            warnVersion: v1.36
            exemptions:
              namespaces:
              - tanzu-system-monitoring     # node-exporter needs host access
              - vmware-system-antrea        # CNI agent
```

#### Making Helm charts install without friction

Here is the practical problem. `helm install --create-namespace` creates a bare namespace with no PSA
labels, so it inherits the cluster default. If the chart's pods cannot meet that level they are
rejected, and the operator has to go back, create the namespace by hand with labels, and reinstall.
Same for any CI pipeline that assumes it can create its own namespace.

The most useful thing to know is that most of this friction lives at `restricted`, not `baseline`.
`baseline` blocks host namespaces, privileged containers, and dangerous capabilities, which most
application charts never request; charts that break there are usually infrastructure components you
were going to treat specially anyway. `restricted` additionally demands `runAsNonRoot`,
`allowPrivilegeEscalation: false`, a `seccompProfile`, and all capabilities dropped, and plenty of
ordinary charts simply do not set those in their pod specs.

So `enforce: baseline` provides most of the security benefit with little disruption, and it is the pragmatic
cluster default for a platform serving charts you do not control. Treat `restricted` as a per-namespace
goal you raise deliberately.

For namespace creation itself, four options:

| Approach | Seamless install | Assessment |
| --- | --- | --- |
| Platform pre-creates namespaces with PSA labels; teams install into an existing namespace | No, one extra step | The recommended default. Fine-grained, self-documenting, no control-plane changes, and namespace creation becomes a reviewable action, which is usually what you want. |
| The `Namespace` object is committed alongside the release in Git, labels included | Yes | Best if you already do GitOps. You have helm-controller, so a labelled `Namespace` can sit next to the `HelmRelease` and apply together. Seamless, fine-grained, and auditable. |
| Pre-declare namespace names in `exemptions.namespaces` before they exist | Yes | A narrow escape hatch. It works because the exemption is a name match evaluated at admission, so the name can be listed before the namespace exists. But it disables Pod Security for those names, does not scale past a handful, and needs a control-plane change each time. |
| Keep the cluster default at `privileged` and label individual namespaces stricter | Yes | Not recommended. It inverts the posture, so every namespace anyone forgets is unprotected, and the failure is silent. Acceptable only as a transitional state. |

One caveat to plan for: if application teams can label their own namespaces, they can grant themselves
`privileged`. Per-namespace labels are only as strong as the RBAC around namespace creation. The simple
control is not granting namespace `create` or `patch` to application teams, which the first approach
above does naturally. If teams must self-serve namespaces, restricting which PSA label values they can
set needs a policy engine such as `gatekeeper`.

To find out what would break before it does, `audit` and `warn` set stricter than `enforce` is the
whole point:

```bash
# Workload cluster context.
# warn violations come back to the client on apply
helm template my-release ./chart --namespace my-ns | kubectl apply --dry-run=server -f -

# what level is each namespace actually at
kubectl get ns -L pod-security.kubernetes.io/enforce,pod-security.kubernetes.io/enforce-version
```

`audit` violations land in the API server audit log, which is the durable record and another reason to
ship those logs somewhere.

#### A migration order

1. Today, at no risk: set `audit` and `warn` to `restricted`, leave `enforce: privileged`, change
   nothing else. Collect violations across a full workload cycle including scheduled jobs.
2. Pin all three versions explicitly.
3. Move `enforce` to `baseline`. Best security-to-friction ratio of any step here.
4. Decide your namespace creation policy and label existing workload namespaces explicitly rather than
   relying on the cluster default.
5. Add `exemptions.namespaces` for platform components that genuinely cannot run enforced.
6. Raise individual workload namespaces to `restricted` as their charts are fixed.
7. Once nothing depends on the looser default, raise the cluster default to `restricted`.

If you adopt the Istio sidecar mesh at any point, enable `istioCNI` in the same change or injected pods
will need `privileged` and undo the work.

#### Resource quotas

```yaml
          resourceQuotaConfiguration:
            enabled: true       # schema default is false
```

Without quotas one namespace can consume all cluster capacity and starve everything else, so this is a
basic multi-tenancy guardrail and enabling it is a deliberate improvement over the default. The catch
is that enabling quota enforcement can cause previously schedulable workloads to be rejected, because
pods without resource requests may no longer be admitted. Turn it on in non-production first and make
sure workloads declare requests and limits.

### 7.7 Identity: OIDC and RBAC

This configures structured authentication on the API server, so JWTs from an external identity
provider are accepted as Kubernetes credentials and humans authenticate with corporate identity rather
than long-lived kubeconfig certificates. Central revocation, MFA, and short-lived credentials come
with it.

The mechanism is generic OIDC. Any compliant provider works: Entra ID, Okta, Keycloak, Dex, Ping,
Auth0, Google, GitLab, or something self-hosted. The fields are the same in every case. Where providers
differ is in which claims they emit, and that is the one thing you have to check against your own
tokens.

In this profile it exists to support Headlamp login, since the UI obtains ID tokens the API server has
to accept. It is equally useful for `kubectl` with an OIDC-aware credential plugin.

```yaml
        apiServerConfiguration:
          extraAuthentication:
            jwt:
            - issuer:
                url: https://idp.example.com
                audiences:
                - <OIDC_CLIENT_ID>
              claimMappings:
                username:
                  claim: email
                  prefix: "oidc:"
                groups:
                  claim: groups
                  prefix: "oidc-groups:"
              claimValidationRules:
              - expression: 'claims.?email_verified.orValue(false) == true'
                message: "email must be verified"
```

| Field | Comment |
| --- | --- |
| `issuer.url` | Must match the token's `iss` claim exactly, trailing slash included, and match what your client uses. The API server must also be able to reach the URL and its JWKS endpoint; in a restricted-egress environment external OIDC simply does not work, and the failure looks like universally invalid tokens. |
| `audiences` | A token is accepted only if its `aud` claim is listed, which is what stops a token minted for another application being replayed here. Must contain the client ID your UI or CLI uses. |
| `claimMappings.username.claim` | Which claim becomes the Kubernetes username. `email` is stable and readable and matches how people think about identity. Requires the corresponding scope. Note that email is not immutable at every provider, so if an address changes the Kubernetes identity changes and the RBAC bindings stop applying. An immutable subject identifier is more robust at the cost of unreadable RBAC. |
| `claimMappings.username.prefix` | A security control, not decoration. The prefix namespaces external identities so they cannot collide with, and therefore cannot impersonate, built-in subjects like `system:masters` members or ServiceAccounts. Without it, a provider that lets a user set an arbitrary email could mint an identity RBAC already trusts. Never leave it empty, and note that changing it orphans every existing binding at once. |
| `claimMappings.groups.claim` | Enables group-based RBAC, which is the only approach that scales. Verify your provider actually emits this claim: many do not by default, some need a specific scope, some need it added to the token explicitly, some need a directory-API integration or a broker. If the claim is absent, group bindings silently do nothing, and you find out after designing your RBAC around them. Inspect a real token first. |
| `claimMappings.groups.prefix` | Same reasoning, more urgently. Without it, a provider group literally named `system:masters` would grant cluster-admin. |
| `claimValidationRules` | Additional trust conditions, in two forms: `claim` with `requiredValue` for a simple equality check, or `expression` for CEL. |

Fields the profile does not use but enterprise deployments often need:

| Field | When |
| --- | --- |
| `issuer.certificateAuthority` | Essential for an internal provider. Supplies the CA bundle the API server uses to trust the issuer's TLS certificate. Without it, a provider behind an enterprise or self-signed CA is unreachable and every token fails validation. |
| `issuer.egressSelectorType` | `controlplane` or `cluster`, selecting the network path the API server uses to reach the provider. Matters when it is reachable from only one. |
| `issuer.discoveryURL` | When discovery metadata is not at the standard path relative to the issuer URL. |
| `issuer.audienceMatchPolicy` | `MatchAny`, to accept a token matching any listed audience. |
| `claimMappings.uid`, `claimMappings.extra` | Map claims to the user UID or to extra attributes, for audit correlation or authorization webhooks. |
| `userValidationRules` | CEL rules validating the mapped user rather than the raw claims, for example asserting the username carries the expected prefix. Useful defence in depth. |

#### Writing claim validation rules

```
claims.?email_verified.orValue(false) == true
```

Read it piece by piece. `claims.?email_verified` is CEL optional chaining: it yields the claim's value
if present, or an empty optional if absent, and the `?` is what stops a missing claim erroring.
`.orValue(false)` unwraps the optional and supplies `false` when the claim is absent. Then `== true`
compares.

The substituted value is the part to think about. It decides what happens to a token that omits the
claim, and `false` is what you want, because a token you cannot evaluate is rejected rather than
admitted. Since the Kubernetes username is derived from the email claim here, a provider or client
that omits `email_verified` should not be able to present an unverified address as an identity.

Apply the same reasoning to every CEL rule you write. Choose the substituted value so that a missing
claim denies access, and make the `message` describe the condition the rule actually enforces so the
error is useful to whoever hits it.

#### Authentication is not authorization

A successfully authenticated user has no permissions until an RBAC binding names them, and the
affected users report a broken cluster rather than a missing binding. This happens often enough to be
worth stating plainly.

The subject name is the prefix concatenated with the mapped claim value. With `prefix: "oidc:"` and
`claim: email`, a user whose email is `user@example.com` is the Kubernetes user `oidc:user@example.com`.
Bindings are applied inside the workload cluster, not the vSphere Namespace.

Always verify the exact string the API server derives rather than assuming:

```bash
# Workload cluster context.
kubectl auth whoami
kubectl auth can-i --list
```

[Appendix B.1](#b1-rbac-for-oidc-identities) has the bindings, including group-based ones and the
least-privilege pattern.

### 7.8 Platform settings

The remaining `kubernetes` sub-variables and cluster-level settings, grouped by what they affect.

#### Certificates

```yaml
        certificateRotation:
          enabled: true
          renewalDaysBeforeExpiry: 90
```

Keep this on. Control-plane certificates expire, and expired certificates mean a dead control plane you
cannot fix through the API, because the API is what stopped working. Automating renewal removes a class
of self-inflicted outage that has taken down clusters run by competent teams. The only real failure mode
of manual rotation is forgetting.

The window must be comfortably shorter than certificate validity, and long enough that renewal lands
inside a maintenance window rather than at the expiry cliff. 90 days on a one-year certificate is
sensible.

Rotation is silent when it works, which makes it an untested assumption. Confirm once that it has
actually happened, then alert on certificate expiry as a backstop so a silently broken rotation surfaces
months before it becomes an outage.

#### etcd

```yaml
        etcdConfiguration:
          maximumDBSizeGiB: 4
```

etcd stores every object plus revision history until compaction, and an addon stack of this size is
exactly the workload that grows it: five addons bring many CRDs, and operators write frequently, with the
Prometheus operator, cert-manager, and kapp-controller all writing status and events continuously.

Exceeding the quota raises a `NOSPACE` alarm and puts etcd read-only. Every write fails. It presents as
a total cluster outage and does not clear on its own; you must compact, defragment, and explicitly
disarm the alarm, all needing etcd-level access at the moment your control plane appears dead.

Two things follow. The etcd volume must actually have the space, since a quota larger than the disk
converts a clean stop into a disk-full condition. And you should alert on database size at 60 to 70% of
the quota, which turns an outage into a ticket. Watch the trend rather than just the threshold: steady
growth usually means excessive object churn, a controller writing status in a loop, or auto-compaction
not running.

4 GiB suits a small-to-medium cluster with this addon set. High object counts, heavy CRD use, or high
event churn need more. `maxRequestSizeKiB` is also available; raise it only for genuinely large objects,
remembering that it lets clients push more per request and so grows etcd faster.

Related: `kubeControllerManagerConfiguration.terminatedPodGCThreshold` sets how many terminated pods are
retained before garbage collection. Lowering it helps on high-churn clusters, since retained pod objects
consume etcd space.

#### Logging

```yaml
        apiServerConfiguration:
          logs:
            format: json
        kubeletConfiguration:
          logging:
            format: json
```

Structured JSON is parsed reliably by log aggregators with no fragile regex extraction, and fields
become queryable. Setting it on both the API server and the kubelet is right; mixed formats in one
pipeline defeat the purpose.

This only pays off if the logs leave the node, because node logs are lost when a node is replaced, which
happens on every upgrade. If you do not already forward logs, see
[Appendix C.2](#c2-log-forwarding-fluent-bit).

Human readability drops noticeably, so keep a JSON formatter available and remember that if your
pipeline is down you are reading JSON directly. Structured lines are also more verbose, so account for the volume.

`logs.verbosity` is available; leave it at the default unless debugging, since high verbosity on a busy
API server generates enormous volume.

#### Other kubelet settings worth knowing

| Field | Comment |
| --- | --- |
| `podPidsLimit` | Caps PIDs per pod. Worthwhile defence against fork-bomb resource exhaustion. |
| `containerLogMaxSizeMiB`, `containerLogMaxFiles` | Bound per-container log growth, which protects the `/var/lib/kubelet` volume from a pod writing large log volumes. |
| `imageGCHighThresholdPercent`, `imageGCLowThresholdPercent` | Disk-usage thresholds for image garbage collection. Tune together with the containerd volume size. |
| `serializeImagePulls`, `maxParallelImagePulls` | Image-pull concurrency. Parallel pulls speed up scale-out and can saturate the registry or network. |
| `imagePullCredentialsVerificationPolicy` | `AlwaysVerify` prevents a pod using an image another tenant cached on the node without presenting its own credentials. A real multi-tenancy control in shared clusters. |
| `allowedUnsafeSysctls` | Specialised tuning only. Taint such pools and schedule only workloads that need them. |

#### API server tuning and TLS

`maxRequestsInFlight` and `maxMutatingRequestsInFlight` are concurrency limits; raise them for very
large or controller-heavy clusters, since the defaults suit most. `requestTimeout` bounds
non-long-running requests. `profiling` exposes `/debug/pprof`; it can be disabled, but support tooling
uses those endpoints, so leave it at the default unless a specific policy says otherwise.

`security.minimumTLSProtocol` and `security.tlsCipherSuites` are available and deserve caution rather
than enthusiasm. Raising the TLS floor or narrowing ciphers breaks any client, controller, webhook, or
integration that has not been validated against it, and the failures are handshake errors scattered
across unrelated components that do not name cipher negotiation as the cause. Change them only against
a specific compliance requirement, with a tested inventory of everything that talks to the API server.

`kubeProxyConfiguration.enabled` allows disabling kube-proxy for CNIs that replace it, such as some
Cilium modes. Do not disable it unless your CNI explicitly requires it; service networking stops
working otherwise.

#### API server FQDNs

```yaml
        endpointFQDNs:
        - api.k8s.example.com
```

Adds FQDN aliases for the control plane endpoint, and the name goes into the API server certificate's
SANs so TLS validation succeeds on it. Without that, connecting via the name fails validation even
though the endpoint is reachable, and the error indicates a certificate problem rather than the
configuration gap that caused it.

Set this in the same domain you publish the UI on. If Headlamp is at `headlamp.k8s.example.com`, put
the API server at `api.k8s.example.com`, and then kubeconfigs, CI runners, and the UI all reference
stable names in one zone instead of a mix of names and IPs. The variable adds the name to the
certificate; publishing the DNS record is still your job.

This replaces the deprecated `kubeAPIServerFQDNs` variable.

#### CNI

```yaml
    - name: bootstrapAddons
      value:
        cniRef:
          name: antrea
          namespace: vmware-system-vks-public
```

The CNI must be installed as the cluster bootstraps, since nodes cannot become `Ready` and pods cannot
get addresses without it, which is why it is configured here rather than as an `AddonInstall`.

This is a create-time decision with no supported in-place migration: changing it means building a new
cluster. A default catalogue offers `antrea`, `calico`, and `cilium`.

Antrea is the VMware-supported default. It integrates with the vSphere networking stack, supports
Kubernetes NetworkPolicy plus richer Antrea policy CRDs, and handles the reserved pod CIDR used here.
Choose it unless you have a specific reason not to. Calico is mature and widely deployed with a strong
policy model, reasonable if it is your organisational standard. Cilium is eBPF-based with advanced
observability and L7 policy, and the option to replace kube-proxy; choose it if you want those
capabilities and will operate them.

The CNI and Istio both manipulate pod networking, which is normal and supported. At gateway-only Istio
scope there is no interaction at all.

#### Workload storage

```yaml
    - name: vsphereOptions
      value:
        persistentVolumes:
          availableStorageClasses:
          - <STORAGE_CLASS>
          defaultStorageClass: <STORAGE_CLASS>
```

Distinct from the `storageClass` variable in [7.3](#73-node-sizing): that one is for the nodes' own
disks, this one is for workload PersistentVolumeClaims.

Explicitly listing the available classes is good practice, since workloads cannot then request policies
you have not sanctioned. Setting a default matters too, because without one any chart that omits
`storageClassName` produces a permanently pending PVC. Consider whether your default should be your most
expensive protection level; many teams default to something cheaper and require explicit opt-in to
premium storage.

The class must be associated with your vSphere Namespace, and the error surfaces late. A class named
here but not associated is accepted at cluster-create time and fails at PVC-create time, so the first
symptom is a workload whose PVC pends indefinitely with an unhelpful event, long after the cluster
looked healthy.

A protection policy also needs enough hosts to satisfy placement. A RAID-5 or RAID-6 erasure-coding
policy on too small a cluster is non-compliant, and provisioning either fails or silently produces
objects that do not meet the stated protection level. Check compliance in vCenter after deployment.

Consider offering a tier set rather than one class, so workloads can match storage to need.

#### Time and access notices

```yaml
    - name: osConfiguration
      value:
        ntp:
          servers:
          - ntp.example.com
        sshd:
          banner: '---AUTHORIZED ACCESS ONLY--- Use of this system is restricted to
            authorized users for authorized activities only. '
```

NTP matters more here than it looks. Time synchronisation is load-bearing for three separate mechanisms
in this manifest: TLS, where certificate validity is a time window and a skewed clock rejects valid
certificates; etcd, whose leases and election timeouts are time-based and where skew between members
causes election churn; and OIDC, where every ID token carries `exp`, `iat`, and often `nbf` claims
validated against the API server's clock.

That last one is the specific risk. This cluster authenticates humans with short-lived tokens, so a skew
of a few minutes rejects valid tokens or accepts expired ones, and the symptom is intermittent
user-specific authentication failures that come and go as tokens refresh. That is close to
undiagnosable if you are not looking at clocks.

Configure multiple servers, use the same sources as the rest of your infrastructure including vCenter
and ESXi, and alert on skew. An unreachable NTP server is worse than none configured, because you
believe time is managed.

The SSH banner is a compliance control; many frameworks require an explicit authorized-use notice
before interactive access. Have legal or compliance supply the wording, since a generic banner may not
satisfy the regime you are audited against. Note the trailing space and the single-quoted scalar folded
across two lines: YAML flow folding turns the newline plus indentation into one space, so the value is a
single line, and switching to a block scalar changes the string. A banner is a notice, not an access
control; restrict node SSH by network policy, jump hosts, and key management.

---

## 8. Couplings that must agree

Values that have to match something else, in another object, in DNS, or at your identity provider.
These are the edits that break something apparently unrelated, so it is worth keeping the list next to
your manifest. Each is explained where it belongs; this is a checklist, not a second explanation.

| Must agree | Where |
| --- | --- |
| Cluster name | `Cluster.metadata.name` and every `AddonInstall` selector |
| AddonConfig name | `<clusterName>-<addonRef.name>`, using the addon name not the install name |
| vSphere Namespace | Identical on every `AddonInstall`, `AddonConfig`, and the `Cluster` |
| OIDC client ID | Headlamp `oidc.clientID` and the API server's `audiences` |
| OIDC issuer URL | Headlamp `oidc.issuerURL` and the API server's `issuer.url`, byte for byte |
| Username string | `claimMappings.username.prefix` plus the mapped claim, and every RBAC subject in the cluster |
| OIDC scope and claim | `oidc.scopes` must include the scope producing `claimMappings.username.claim` |
| UI hostname | Headlamp `hostname`, the `callbackURL` host, a published DNS record, and a redirect URI registered at the provider |
| DNS target | The Headlamp `Gateway`'s own LoadBalancer address, not the Istio ingress gateway's |
| Gateway class | `gatewayApi.gateway.className` needs a matching controller installed |
| StorageClass name | `storageClass`, each `volumes[].storageClass`, `availableStorageClasses`, `defaultStorageClass`, and all associated with the vSphere Namespace |
| Node budget | Pod CIDR blocks must cover `controlPlane.replicas` plus every autoscaler maximum, plus a surge spare |
| Density | `maxPods`, the VM class, and `systemReserved` sized together |
| Block headroom | `maxPods` below the usable addresses in a per-node block, about 254 for a `/24` |
| Version pairing | `topology.version` a `READY` and `COMPATIBLE` `kr`, supported by the ClusterClass |
| OS image | The same `resolve-os-image` annotation on the control plane and every machine deployment |
| API server names | Any name clients use listed in `endpointFQDNs` and published in DNS |
| Namespace policy | A namespace whose workloads cannot meet the cluster PSS default needs explicit labels or an exemption |
| Mesh and privilege | Adopting sidecar injection requires `istioCNI: true` for any PSS above `privileged` |

---

## 9. Operating the cluster

### 9.1 Health checks after deployment

Work outward from the Supervisor. Each layer depends on the one before, so checking in order localises
a failure rather than leaving you guessing among five components.

**The cluster exists.**

```bash
kubectl get cluster,machinedeployment,machine -n <VSPHERE_NAMESPACE>
kubectl describe cluster <CLUSTER_NAME> -n <VSPHERE_NAMESPACE>   # conditions live here
```

The `Cluster` should be `Provisioned` and `AVAILABLE=True`, every `Machine` `Running`, and
`CP AVAILABLE` matching `CP DESIRED`. Machines stuck here usually mean the `resolve-os-image` selector
does not resolve, the VM class or storage policy is not associated with the namespace, or
`topology.version` names a release that is not `READY` and `COMPATIBLE`.

**Addons are installed, bound, and ready.**

```bash
kubectl get clusteraddon -n <VSPHERE_NAMESPACE>

kubectl get addonconfig -n <VSPHERE_NAMESPACE> \
  -o custom-columns='CONFIG:.metadata.name,CLUSTER:.spec.clusterName,DEFINITION:.spec.addonConfigDefinitionRef.name'
```

`READY=True` on every `ClusterAddon`, and no empty `CLUSTER` or `DEFINITION` on any config. The second
command is the one that catches a silently skipped config.

**Packages reconciled inside the cluster.** Switch context.

```bash
# Workload cluster context.
kubectl get pkgi -A
kubectl describe pkgi <name> -n vmware-system-tkg      # if one failed
```

Every `PackageInstall` should say `Reconcile succeeded`.

**Istio and the ingress gateway.**

```bash
# Workload cluster context.
kubectl get pods -n istio-system      # istiod
kubectl get pods -n istio-ingress
kubectl get svc -n istio-ingress      # EXTERNAL-IP must be populated
```

An `EXTERNAL-IP` stuck at `<pending>` is a load-balancer problem, VIP pool exhaustion or LB
misconfiguration, not an Istio problem.

**Gateway API, the UI, and its certificate.**

```bash
# Workload cluster context.
kubectl get gateway,httproute -A       # PROGRAMMED True, ADDRESS populated
kubectl get svc -n headlamp            # the LoadBalancer DNS should point at

kubectl get certificate,issuer -n headlamp

JP='{range .spec.listeners[*]}{.name}{"  "}{.protocol}{":"}{.port}'
JP+='{"  hostname="}{.hostname}{"  tls="}{.tls.mode}'
JP+='{"  secret="}{.tls.certificateRefs[0].name}{"\n"}{end}'
kubectl get gateway headlamp-gateway -n headlamp -o jsonpath="$JP"
```

Then confirm the chain from outside:

```bash
# Run from a workstation with network access to the FQDN.
openssl s_client -connect headlamp.k8s.example.com:443 -servername headlamp.k8s.example.com </dev/null 2>&1 | head -20
```

**Storage.**

```bash
# Workload cluster context.
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: storage-smoke-test
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc storage-smoke-test          # Bound within seconds
kubectl describe pvc storage-smoke-test     # events explain any delay
kubectl delete pvc storage-smoke-test
```

Omitting `storageClassName` is deliberate: it verifies the default class is in effect.

**Identity.** The question is not whether you can log in, it is what username the API server thinks you
are, because everything in RBAC depends on that exact string.

```bash
# Workload cluster context.
kubectl auth whoami
kubectl auth can-i --list
```

Expect the prefix plus the claim value, so `oidc:user@example.com`. When claims are in doubt, read the
token rather than guessing at what your provider emits:

```bash
echo "<ID_TOKEN>" | cut -d. -f2 | tr '_-' '/+' | sed 's/$/===/' | base64 -d 2>/dev/null; echo
```

**Observability.**

```bash
# Workload cluster context.
kubectl get pods -n tanzu-system-monitoring
kubectl get prometheus,alertmanager,servicemonitor,prometheusrule -A
```

Know where rule evaluation happens: either a `Prometheus` CR here, or an external Prometheus scraping
this cluster.

**Pod CIDR headroom.** Run this before any upgrade or rolling change.

```bash
# Workload cluster context.
kubectl get nodes -o custom-columns='NODE:.metadata.name,POD_CIDR:.spec.podCIDR,MAXPODS:.status.capacity.pods'
kubectl get nodes -o jsonpath='{range .items[*]}{.spec.podCIDR}{"\n"}{end}' | sort -u | wc -l
```

Compare against your total, 16 for a `/20` and 256 for a `/16`. If the count equals the total, do not
start.

### 9.2 Upgrades

The dependency runs bottom-up, and going the other way produces objects referencing things that do not
exist yet.

1. Upgrade the Supervisor and VKS platform first; it provides everything below.
2. Confirm the new ClusterClass and `kr` releases are available and compatible.
3. Diff the new ClusterClass variable schema against the old one. Variables can be added, renamed, or
   removed between generations.
4. Diff the two `kr` objects. A Kubernetes patch bump can carry a CNI upgrade, a kapp-controller minor
   bump, and an etcd patch. [Appendix D](#appendix-d-reading-a-kubernetes-release-object) has a
   ready-made change report, and while you are there confirm your `resolve-os-image` annotation still
   resolves against the target release.
5. Bump `topology.version`, one minor version at a time.
6. Bump `topology.classRef.name` when moving to a new ClusterClass generation.
7. Addon releases last.

Before starting: a free pod CIDR block exists, PodDisruptionBudgets cover critical workloads, the pool
has capacity to lose a node, backups are current if you run them and a restore has been tested, and you
have a window if the control plane is single-replica. Upgrade non-production first.

Which changes replace nodes, so you can predict the disruption:

| Change | Effect |
| --- | --- |
| `topology.version`, `topology.classRef.name` | Rolling replacement of all nodes, control plane first |
| `vmClass` | Rolling replacement of the affected pool |
| `volumes`, add or resize | Rolling replacement |
| `kubeletConfiguration`, including `maxPods` | Rolling replacement; the schema states this explicitly |
| `resourceConfiguration.systemReserved` | Rolling replacement |
| `node.labels`, `node.taints` | Rolling replacement |
| `resolve-os-image` annotation | Rolling replacement |
| `replicas` or autoscaler bounds | Scale operation only, existing nodes untouched |
| `certificateRotation`, `etcdConfiguration`, PSS, `apiServerConfiguration` | Control-plane reconfiguration, no worker replacement |
| `clusterNetwork` CIDRs | Immutable, requires a new cluster |
| `bootstrapAddons.cniRef` | Effectively immutable, requires a new cluster |

Every rolling replacement needs a free pod CIDR block, which is the fact that ties most of this
together.

### 9.3 Ongoing maintenance

**Addon versions.** Naming the `acd` in `spec.addonConfigDefinitionRef` pins a release; otherwise the
platform resolves one and it can move on upgrade. After any addon upgrade, re-verify your `AddonConfig`
against the new schema, because fields can be added, renamed, or relocated, and a field at the wrong
nesting level is not applied. An upgrade can quietly revert a setting to its default.

**Orphaned configs.** The `owned-for-deletion` annotation handles cleanup. If a manifest lacked it,
sweep with `kubectl get addonconfig -n <VSPHERE_NAMESPACE>` and delete configs whose cluster or addon no
longer exists.

**Adding an addon** is a normal day-2 operation. Apply the `AddonInstall` and optional `AddonConfig`; no
ordering to observe, no restart.

**Monitoring worth having**, given what this profile is sensitive to:

| Alert | Reason |
| --- | --- |
| Allocated pod CIDR blocks against total | An exhausted CIDR blocks upgrades and cannot be fixed in place |
| etcd database size above 60% of quota | Turns a read-only outage into a ticket |
| Node clock skew | Otherwise near-undiagnosable OIDC failures |
| Control-plane certificate expiry | Backstop for a silently broken rotation |
| containerd and kubelet volume fill | Those volumes are the node's protection |
| Pods per node approaching `maxPods` | Early warning before scheduling failures |
| PVC pending duration | Catches the storage-class association problem |
| istiod and gateway availability | Both default to a single replica in the profile |
| Failed authentication rate | Detects misconfiguration and credential attacks |
| `PackageInstall` reconcile failures | The authoritative signal that an addon stopped converging |

**Identity hygiene.** Plan client secret rotation, and prefer a Secret reference so rotating does not
mean editing a manifest. Audit RBAC bindings against current staff, which is the argument for
group-based bindings once your provider emits groups.

**If the UI address changes**, four things move together or login breaks: the Headlamp `hostname`, the
`callbackURL`, the DNS record, and the redirect URI registered at your provider. Reserving a static
LoadBalancer IP removes the trigger; [Appendix C.3](#c3-dns-automation-external-dns) automates the DNS
step.

**Backups** are not part of this profile. Whether they belong in-cluster depends on how you already
protect workloads, since many environments handle it at the vSphere or storage layer.
[Appendix C.1](#c1-backup-and-restore-velero) covers the in-cluster option. Either way, test a restore.
Keep these manifests in version control regardless: they are the declarative source of truth, and for
the immutable fields in [1.3](#13-decisions-you-cannot-revisit) rebuilding from them is the only remedy.

### 9.4 Symptom lookup

Most of these present at a different layer than their cause, so recognising the symptom saves the
search.

| Symptom | Likely cause | Read |
| --- | --- | --- |
| Addon deployed but your values had no effect | `AddonConfig` name did not resolve, so it was skipped | [4](#the-naming-rule) |
| Addon never appears at all | `AddonInstall` selector matches nothing | [6](#6-the-addon-stack) |
| Nodes `Ready` but pods stay `Pending` | Pod CIDR blocks exhausted | [7.1](#71-pod-and-service-networking) |
| Rolling upgrade stalls in a mixed-version state | No free pod CIDR block for the surge node | [7.1](#71-pod-and-service-networking) |
| Machines never provision | OS image selector, VM class, or storage policy not available to the namespace | [1.2](#12-platform-prerequisites), [7.4](#74-control-plane) |
| PVCs pend indefinitely | StorageClass not associated with the vSphere Namespace | [7.8](#78-platform-settings) |
| Intermittent, user-specific auth failures that resolve on retry | Clock skew against token `exp` and `iat` | [7.8](#78-platform-settings) |
| Login succeeds, every API call is unauthorised | Client ID missing from the API server `audiences` | [7.7](#77-identity-oidc-and-rbac) |
| Login succeeds, user has no permissions | No RBAC binding for the mapped username | [7.7](#77-identity-oidc-and-rbac) |
| All tokens rejected as untrusted issuer | `issuer.url` mismatch, or an untrusted provider CA | [7.7](#77-identity-oidc-and-rbac) |
| Group-based bindings never apply | Provider emits no `groups` claim | [7.7](#77-identity-oidc-and-rbac) |
| Provider shows a redirect-mismatch error | `callbackURL`, DNS, and registered redirect URI disagree | [6.5](#65-headlamp) |
| UI pods healthy, URL unreachable | `Gateway` unprogrammed, no matching controller | [6.5](#65-headlamp) |
| Monitoring deployed, no metrics and no alerts | No `Prometheus` CR, so the operator is idle | [6.3](#63-prometheus) |
| Cluster-wide write failures, reads fine | etcd hit its size quota and went read-only | [7.8](#78-platform-settings) |
| Workloads rejected at admission after an upgrade | PSS versions pinned to `latest` tightened | [7.6](#76-pod-security) |
| `helm install --create-namespace` rejected | Bare namespace inherits a stricter cluster default | [7.6](#76-pod-security) |
| `NotReady` kubelets, etcd election churn, random timeouts | `best-effort` VM class under hypervisor contention | [7.3](#73-node-sizing) |
| Nodes accept many pods then destabilise | `maxPods` raised without node resources or reservations | [7.3](#73-node-sizing) |
| istiod replica count oscillates | Static `pilot.replicas` fighting its own HPA | [6.4](#64-istio) |
| Node pool never scales | Static `replicas` set alongside autoscaler annotations | [7.5](#75-worker-pools) |
| Topology never reconciles | `topology.version` is not a valid `kr` name | [2](#2-choosing-a-kubernetes-release) |

---

## Appendix A: A production starting point

The same stack, hardened. Substitute the placeholders, verify version strings and class names against
your environment, and read the inline notes. A few settings depend on schema field names you should
confirm with the discovery commands from [section 5](#5-reading-an-addons-value-schema) rather than
trust here.

These objects go to the vSphere Namespace on the Supervisor. The in-cluster companions are in
[Appendix B](#appendix-b-in-cluster-companion-objects).

[Section 1.4](#14-what-to-change-before-production) lists what changed and why. Two further changes
not on that list: `priorityClassName` on the Helm controllers, and `meshID` naming the mesh rather than
the cluster so multi-cluster federation stays available.

```yaml
# Addons. Order is irrelevant; all reconcile independently.
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonInstall
metadata:
  name: cluster-helm-controller
  namespace: <VSPHERE_NAMESPACE>
spec:
  addonRef:
    name: helm-controller
  clusters:
  - selector:
      matchLabels:
        cluster.x-k8s.io/cluster-name: workload-vsphere-vks2
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonConfig
metadata:
  annotations:
    clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"
  # In generated manifests, set spec.clusterName and spec.addonConfigDefinitionRef
  # explicitly instead of relying on this name. It removes the silent-skip failure
  # mode and pins the addon release.
  name: workload-vsphere-vks2-helm-controller
  namespace: <VSPHERE_NAMESPACE>
spec:
  values:
    helmController:
      priorityClassName: system-cluster-critical
    sourceController:
      priorityClassName: system-cluster-critical
---
# cert-manager: no AddonConfig needed. Headlamp uses it for Gateway TLS.
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonInstall
metadata:
  name: cluster-cert-manager
  namespace: <VSPHERE_NAMESPACE>
spec:
  addonRef:
    name: cert-manager
  clusters:
  - selector:
      matchLabels:
        cluster.x-k8s.io/cluster-name: workload-vsphere-vks2
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonInstall
metadata:
  name: cluster-prometheus
  namespace: <VSPHERE_NAMESPACE>
spec:
  addonRef:
    name: prometheus
  clusters:
  - selector:
      matchLabels:
        cluster.x-k8s.io/cluster-name: workload-vsphere-vks2
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonConfig
metadata:
  annotations:
    clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"
  name: workload-vsphere-vks2-prometheus
  namespace: <VSPHERE_NAMESPACE>
spec:
  values:
    deploycomponents:
      kube-state-metrics: true
      node-exporter: true
      pushgateway: false
      # Operator-managed. NOTHING SCRAPES until you create the Prometheus CR
      # in Appendix B.3.
      prometheus: false
      alertmanager: true          # configure receivers, or alerts go nowhere
      prometheus-operator: true
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonInstall
metadata:
  name: cluster-istio
  namespace: <VSPHERE_NAMESPACE>
spec:
  addonRef:
    name: istio
  clusters:
  - selector:
      matchLabels:
        cluster.x-k8s.io/cluster-name: workload-vsphere-vks2
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonConfig
metadata:
  annotations:
    clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"
  name: workload-vsphere-vks2-istio
  namespace: <VSPHERE_NAMESPACE>
spec:
  values:
    istio:
      namespace: "istio-system"
      # L4/L7 ingress only. If you later label namespaces for sidecar
      # injection, set istioCNI.enabled: true in the SAME change.
      ambientMode:
        enabled: false
      istioCNI:
        enabled: false
      gateways:
        ingress:
          enabled: true
          namespace: istio-ingress
          autoscaling:
            enabled: true
            minReplicas: 2
            maxReplicas: 5
      pilot:
        # No static replicas: the HPA owns the field.
        autoscaling:
          enabled: true
          minReplicas: 2
          maxReplicas: 4
      meshConfig:
        accessLogFile: "/dev/stdout"
        enableTracing: false
        meshID: "prod-mesh"
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonInstall
metadata:
  name: cluster-headlamp
  namespace: <VSPHERE_NAMESPACE>
spec:
  addonRef:
    name: headlamp
  clusters:
  - selector:
      matchLabels:
        cluster.x-k8s.io/cluster-name: workload-vsphere-vks2
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonConfig
metadata:
  annotations:
    clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"
  name: workload-vsphere-vks2-headlamp
  namespace: <VSPHERE_NAMESPACE>
spec:
  values:
    # Published in DNS, pointing at THIS Gateway's LoadBalancer. Reserve the IP.
    hostname: headlamp.k8s.example.com
    gatewayApi:
      enabled: true
      gateway:
        className: istio          # requires the istio addon above, or contour
        create: true
        name: headlamp-gateway
        # The addon issues a SELF-SIGNED certificate by default. Point this at
        # a trusted-CA certificate (Appendix B.2); check the schema for the
        # TLS field names.
    oidc:
      enabled: true
      issuerURL: https://idp.example.com
      clientID: <OIDC_CLIENT_ID>          # must equal the apiserver audience
      # Do not inline this. Check the schema for a secret-reference field and
      # manage the Secret with sealed-secrets, external-secrets, or
      # vault-injector so this file stays committable.
      clientSecret: <OIDC_CLIENT_SECRET>
      callbackURL: https://headlamp.k8s.example.com/oidc-callback
      scopes:
      - openid
      - email                             # feeds the username claim mapping
      - profile
---
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
metadata:
  name: workload-vsphere-vks2
  namespace: <VSPHERE_NAMESPACE>
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
      # 256 per-node /24 blocks, so 255 usable nodes after a surge reserve.
      # IMMUTABLE. A /20 gives only 16 blocks, which the 3 + 10 shape below
      # would nearly exhaust. Validate 240/4 against your firewalls and LBs.
      - 240.0.0.0/16
    serviceDomain: cluster.local
    services:
      cidrBlocks:
      - 240.1.0.0/20
  topology:
    classRef:
      name: builtin-generic-v3.7.0
      namespace: vmware-system-vks-public
    controlPlane:
      metadata:
        annotations:
          run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
      replicas: 3
      variables:
        overrides:
        - name: vmClass
          value: guaranteed-large
    variables:
    - name: storageClass
      value: <STORAGE_CLASS>
    - name: vmClass
      # guaranteed-* reserves CPU and memory. best-effort-* is dev/test only:
      # the symptoms point at Kubernetes, not at resource pressure.
      value: guaranteed-medium
    - name: volumes
      # Resizing these is a rolling node replacement. Size generously, and scale
      # with pod density.
      value:
      - capacity: 100Gi
        mountPath: /var/lib/containerd
        name: containerd
        storageClass: <STORAGE_CLASS>
      - capacity: 100Gi
        mountPath: /var/lib/kubelet
        name: kubelet
        storageClass: <STORAGE_CLASS>
    - name: node
      value:
        # Declarative labels survive node replacement; hand-applied ones do not.
        labels:
          workload-type: general
          environment: production
    - name: resourceConfiguration
      value:
        systemReserved:
          # Correct at maxPods 110, where the reservation is derived from node
          # size. If you raise maxPods, override explicitly and move to a
          # larger VM class: see 7.3.
          automatic: true
    - name: kubernetes
      value:
        # API server FQDN aliases, in the same domain as the UI. Adds the name
        # to the certificate SANs; you still publish the DNS record.
        endpointFQDNs:
        - api.k8s.example.com
        certificateRotation:
          enabled: true
          renewalDaysBeforeExpiry: 90
        etcdConfiguration:
          # The etcd volume must have this space. Alert at 60-70%.
          maximumDBSizeGiB: 8
        kubeControllerManagerConfiguration:
          terminatedPodGCThreshold: 500
        security:
          # minimumTLSProtocol and tlsCipherSuites are deliberately NOT set.
          # Raising the TLS floor or narrowing ciphers breaks unvalidated
          # clients and webhooks, and the failures are hard to attribute.
          podSecurityStandard:
            # baseline blocks real privilege escalation while most application
            # charts still install unmodified. restricted is a per-NAMESPACE
            # goal you raise deliberately, via namespace labels.
            enforce: baseline
            enforceVersion: v1.36
            # Stricter audit/warn inventories the path to restricted without
            # rejecting anything.
            audit: restricted
            auditVersion: v1.36
            warn: restricted
            warnVersion: v1.36
            deactivated: false
            # Exemptions DISABLE Pod Security entirely for these namespaces.
            # Use them only for platform components that cannot run enforced at
            # any level; for workload namespaces use labels instead.
            exemptions:
              namespaces:
              - tanzu-system-monitoring
              - vmware-system-antrea
          resourceQuotaConfiguration:
            enabled: true
        apiServerConfiguration:
          logs:
            format: json
          extraAuthentication:
            jwt:
            - issuer:
                url: https://idp.example.com
                audiences:
                - <OIDC_CLIENT_ID>
                # For an internal provider behind a private CA, also set:
                #   certificateAuthority: | ...
                #   egressSelectorType: cluster
              claimMappings:
                username:
                  claim: email
                  # Never empty. Changing it orphans every RBAC binding.
                  prefix: "oidc:"
                groups:
                  # Verify your provider actually emits this claim.
                  claim: groups
                  prefix: "oidc-groups:"
              claimValidationRules:
              # orValue(false) fails closed: a token that omits the claim is
              # rejected rather than admitted.
              - expression: 'claims.?email_verified.orValue(false) == true'
                message: "email must be verified"
        kubeletConfiguration:
          logging:
            format: json
          # Default 110, documented maximum 250. Raising it buys density without
          # consuming pod CIDR blocks, but size the VM class and systemReserved
          # to match. Changing it requires a machine rollout.
          maxPods: 110
          podPidsLimit: 4096
          containerLogMaxSizeMiB: 50
          containerLogMaxFiles: 5
    - name: bootstrapAddons
      value:
        cniRef:
          # Effectively immutable. Alternatives: calico, cilium.
          name: antrea
          namespace: vmware-system-vks-public
    - name: vsphereOptions
      value:
        persistentVolumes:
          availableStorageClasses:
          - <STORAGE_CLASS>
          defaultStorageClass: <STORAGE_CLASS>
    - name: osConfiguration
      value:
        ntp:
          # A functional prerequisite: TLS validity, etcd leases, and OIDC
          # exp/iat all depend on synchronised clocks. Same sources as vCenter
          # and ESXi. Alert on skew.
          servers:
          - ntp1.example.com
          - ntp2.example.com
        sshd:
          banner: '---AUTHORIZED ACCESS ONLY--- Use of this system is restricted to
            authorized users for authorized activities only. '
    # Must be the NAME of a kr that is READY and COMPATIBLE. Note the triple
    # dash. Check with: kubectl get kr
    version: v1.36.1---vmware.4-vkr.5
    workers:
      machineDeployments:
      - class: node-pool
        metadata:
          annotations:
            # The platform installs the Cluster Autoscaler, so these are
            # effective. Do not also set replicas.
            cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "3"
            cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "10"
            run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
        name: node-pool-1
```

Node budget check: 3 control plane plus 10 workers is 13 nodes, 14 blocks with the surge spare. On the
`/16` above that is 14 of 256. On a `/20` it would be 14 of 16, which is why the CIDR changed.

---

## Appendix B: In-cluster companion objects

Three things the reference profile is missing. All are applied inside the workload cluster, not to the
vSphere Namespace, so switch context first.

### B.1 RBAC for OIDC identities

The most important addition. Without it, OIDC login succeeds and grants nothing.

```yaml
# Apply in the workload cluster context, not the vSphere Namespace.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-platform-admins
subjects:
- kind: User
  name: "oidc:platform-admin@example.com"     # prefix plus the mapped claim
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-readonly
subjects:
- kind: User
  name: "oidc:developer@example.com"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
---
# Once your provider emits a groups claim, bind groups instead of users.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-group-platform-team
subjects:
- kind: Group
  name: "oidc-groups:platform-team"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

Get the prefix right: `oidc:user@example.com`, not `user@example.com`. A binding without it matches
nobody. Verify with `kubectl auth whoami` rather than assuming.

Prefer namespace-scoped `RoleBinding`s wherever the role does not genuinely need cluster scope, and
grant `cluster-admin` to a small named set rather than a team. Per-user bindings do not scale and go
stale as people join and leave, so move to groups once you can.

Bindings accumulate. Review them periodically against current staff and roles.

### B.2 A trusted certificate for the UI

The Headlamp addon creates a self-signed cert-manager `Issuer` and `Certificate` automatically, so
HTTPS works without configuration, with a browser warning. Check what you have:

```bash
# Workload cluster context.
kubectl get certificate,issuer -n headlamp

JP='{.metadata.annotations.cert-manager\.io/issuer-kind}{"/"}'
JP+='{.metadata.annotations.cert-manager\.io/issuer-name}'
JP+='{"  CN="}{.metadata.annotations.cert-manager\.io/common-name}{"\n"}'
kubectl get secret <tls-secret> -n headlamp -o jsonpath="$JP"
```

```
Issuer/headlamp-issuer  CN=Headlamp CA
```

If the FQDN is publicly resolvable, an ACME issuer is usually less work overall, because it avoids
distributing a CA certificate to client trust stores:

```yaml
# Apply in the workload cluster context, not the vSphere Namespace.
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform-team@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    # DNS-01 avoids needing inbound HTTP reachability. Use the solver for your
    # DNS provider; this is illustrative.
    - dns01:
        cnameStrategy: Follow
```

For an internal cluster, issue from your enterprise PKI instead:

```yaml
# Apply in the workload cluster context, not the vSphere Namespace.
# Create the CA keypair out-of-band from your PKI:
#   kubectl create secret tls enterprise-ca --cert=ca.crt --key=ca.key -n cert-manager
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: enterprise-ca-issuer
spec:
  ca:
    secretName: enterprise-ca
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: headlamp-tls
  namespace: istio-ingress          # must be readable by the Gateway
spec:
  secretName: headlamp-tls
  dnsNames:
  - headlamp.k8s.example.com        # must equal Headlamp's hostname
  duration: 2160h                   # 90 days
  renewBefore: 360h                 # 15 days
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: enterprise-ca-issuer
    kind: ClusterIssuer
    group: cert-manager.io
```

Then reference the resulting secret from the Gateway's TLS listener, through `AddonConfig` values or by
setting `gateway.create: false` and managing the `Gateway` yourself.

A `Gateway` can normally only read TLS secrets in its own namespace, which is why the `Certificate`
above targets `istio-ingress`; cross-namespace references need an explicit `ReferenceGrant`. Test the
chain with `openssl s_client` before assuming the OIDC redirect flow works, because a broken chain
breaks login rather than just the padlock.

### B.3 A Prometheus instance

This is what makes the monitoring stack functional. Adjust to your operator version; it needs a
ServiceAccount with cluster-wide read access, which the operator's own RBAC usually provides.

```yaml
# Apply in the workload cluster context, not the vSphere Namespace.
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: main
  namespace: tanzu-system-monitoring
spec:
  replicas: 2
  retention: 30d
  serviceAccountName: prometheus
  # Empty selectors discover every ServiceMonitor, PodMonitor, and rule in the
  # cluster. Narrow them if you want explicit opt-in.
  serviceMonitorSelector: {}
  podMonitorSelector: {}
  ruleSelector: {}
  alerting:
    alertmanagers:
    - namespace: tanzu-system-monitoring
      name: alertmanager
      port: web
  # Without persistent storage you lose all metrics on every pod restart.
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: <STORAGE_CLASS>
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 100Gi
```

[Section 9.3](#93-ongoing-maintenance) lists the alerts worth defining for this particular stack.

### B.4 Worth adding beyond this

| Area | Recommendation |
| --- | --- |
| NetworkPolicy | The highest-value control missing from the profile. Default-deny per namespace, using Kubernetes NetworkPolicy or your CNI's richer CRDs. As it stands any pod can reach any other. |
| PodDisruptionBudgets | Required for safe rolling operations. Without them an upgrade can take all replicas of a service down at once. |
| Secrets management | Externalise every credential with `vault-injector`, sealed-secrets, or external-secrets. Consider etcd encryption at rest. |
| GitOps | These manifests are the source of truth. Reconcile from Git with review rather than applying by hand. |
| Audit logging | Ship API server audit logs off-cluster. With OIDC in place they become a real record of who did what. |
| Istio mTLS | If you adopt the mesh, enable strict mTLS and a minimum TLS protocol version, and `istioCNI` in the same change. |
| Cluster labels | Ownership, environment, and cost-attribution labels. Trivial now, invaluable at fleet scale. |

---

## Appendix C: Integrations that need an external system

Backup, log forwarding, and DNS automation come up in almost every production conversation, and all
three are useful. None of them does anything on its own. Each needs a decision about an external system
that differs in every environment, which is why they are here rather than in the recommended baseline:
installing one without the integration behind it leaves you with a running pod and false confidence.

### C.1 Backup and restore (`velero`)

Backs up Kubernetes resources and persistent volumes, restores them, and supports migration between
clusters.

Cluster-level backup is not a universal practice. Many environments protect workloads at a different
layer, with vSphere-level VM protection, storage array snapshots, or application-native backup, and a
database's own dump and WAL shipping is usually better than a volume snapshot of the same database.
Whether an in-cluster tool adds anything depends on what you already have.

| Decision | Notes |
| --- | --- |
| Object storage target | Velero needs an S3-compatible bucket and credentials. No bucket, no backup. |
| Scope | Cluster resources only, or volumes too? Volume backup needs a snapshot mechanism, and behaviour differs by CSI driver. |
| Overlap with existing protection | If your storage layer already snapshots these volumes on a schedule you trust, this may be duplicate cost and complexity. |
| Retention and schedule | Backups never expired become an unbounded storage bill. |
| Restore target | Same cluster, or a rebuild? Cross-cluster restore has extra constraints around CIDRs and storage classes. |

Whatever you use, including nothing, test a restore. That applies equally to vSphere-level protection:
confirm you can recover a workload, not just that a job reported success.

### C.2 Log forwarding (`fluent-bit`)

Collects container and node logs and forwards them.

Fluent Bit is only as useful as its output plugin, and the output is entirely environment-specific:
Splunk, Elasticsearch or OpenSearch, Loki, an OTLP collector, a cloud logging service, plain syslog.
Each needs its own configuration, endpoint, credentials, and index conventions. There is no sensible
default to recommend, and a Fluent Bit with no configured output is a DaemonSet taking a pod slot on
every node and shipping nothing.

| Decision | Notes |
| --- | --- |
| Output plugin and endpoint | The core decision; it determines the whole configuration. |
| Credentials and TLS | Most outputs need auth. Manage it as a Secret, not inline. |
| Scope | All container logs, or filtered? Node and systemd logs too? Unfiltered collection from a busy cluster generates surprising volume. |
| Parsing | The API server and kubelet emit JSON in this profile, so those parse cleanly. Application logs are whatever your applications emit. |
| Volume and cost | Log platforms usually bill on ingest. Estimate before turning it on cluster-wide. |
| Retention | Where it is enforced and for how long, often a compliance question. |

If you have no log pipeline at all, this is the most urgent of the three. Node logs are destroyed when
a node is replaced, which happens on every upgrade and every `vmClass` or `volumes` change, so without
forwarding, your diagnostic history lasts only as long as a node and the JSON log format provides no
benefit.

### C.3 DNS automation (`external-dns`)

Watches `Service`, `Ingress`, and `Gateway` resources and creates or updates DNS records to match.

It requires write access to a DNS provider: a provider integration, an API credential, and a delegated
zone. That is an organisational decision as much as a technical one, since many teams will not grant a
cluster write access to corporate DNS.

| Decision | Notes |
| --- | --- |
| DNS provider | Each has its own provider configuration and credential type. |
| Credential scope | The cluster gets write access to a zone. Scope it to a delegated subdomain, never the apex. |
| Zone delegation | Cleanest model is delegating a subdomain to be cluster-managed and leaving the rest alone. |
| Ownership | External-DNS tracks ownership with TXT records. Decide how it coexists with manually managed records so it does not fight your DNS team. |
| Policy | `sync` deletes records when resources go away; `upsert-only` never deletes. Start with `upsert-only`. |

What it solves here is the manual A record for the UI, and the drift when the LoadBalancer IP changes.
A simpler alternative solves most of it: reserve a static IP with your load-balancer provider, and the
record never needs updating. Do that first, and add automation only if you are publishing enough
services for manual records to be a real burden.

---

## Appendix D: Reading a Kubernetes release object

[Section 2](#2-choosing-a-kubernetes-release) covers which release to pick. This covers what is inside
one, because the `kr` object is the authoritative manifest of everything a release ships. Reading it
answers three questions you otherwise guess at: which OS images you may select, what component
versions you are getting, and what actually changes if you upgrade.

```bash
kubectl explain kr.spec
```

```
FIELDS:
  bootstrapPackages	<[]Object>
    BootstrapPackages lists references to all bootstrap packages shipped with
    this KubernetesRelease.

  kubernetes	<Object> -required-
    Kubernetes is Kubernetes

  osImages	<[]Object>
    OSImages lists references to all OSImage objects shipped with this
    KubernetesRelease.

  version	<string> -required-
    Version is the fully qualified Semantic Versioning conformant version of the
    KubernetesRelease.
```

### D.1 Core components

```bash
JP='{"kubernetes: "}{.spec.kubernetes.version}'
JP+='{"\netcd:       "}{.spec.kubernetes.etcd.imageTag}'
JP+='{"\ncoredns:    "}{.spec.kubernetes.coredns.imageTag}'
JP+='{"\npause:      "}{.spec.kubernetes.pause.imageTag}'
JP+='{"\nrepository: "}{.spec.kubernetes.imageRepository}{"\n"}'

kubectl get kr v1.36.2---vmware.2-vkr.3 -o jsonpath="$JP"
```

```
kubernetes: v1.36.2+vmware.2
etcd:       v3.6.12_vmware.7-fips
coredns:    v1.14.3_vmware.7-fips
pause:      3.10.2
repository: localhost:5000/vmware.io
```

The etcd version is the number to look at, since etcd carries its own upgrade constraints and is your
cluster's state store. CoreDNS matters if you have custom Corefile configuration. `imageRepository`
pointing at `localhost:5000` indicates a node-local mirror rather than an external registry, which is
useful when diagnosing image pulls and reassuring in air-gapped environments.

One caution on reading these tags: `_fips` suffixes appear on CoreDNS and etcd above even though this
is not a FIPS release, meaning those components are FIPS-validated builds regardless. Do not infer a
release's overall posture from a component tag.

Comparing two patch releases of the same minor shows why this is worth doing before an upgrade:

| Component | `v1.36.1---vmware.4-vkr.5` | `v1.36.2---vmware.2-vkr.3` |
| --- | --- | --- |
| Kubernetes | `v1.36.1+vmware.4` | `v1.36.2+vmware.2` |
| etcd | `v3.6.11_vmware.3-fips` | `v3.6.12_vmware.7-fips` |
| CoreDNS | `v1.14.3_vmware.3-fips` | `v1.14.3_vmware.7-fips` |

A Kubernetes patch bump brought an etcd patch upgrade with it.

### D.2 Bootstrap packages

```bash
kubectl get kr v1.36.2---vmware.2-vkr.3 \
  -o jsonpath='{range .spec.bootstrapPackages[*]}{.name}{"\n"}{end}'
```

```
antrea.tanzu.vmware.com.2.6.2+vmware.1-tkg.1
calico.tanzu.vmware.com.3.31.5+vmware.3-fips-tkg.1
gateway-api.tanzu.vmware.com.1.5.1+vmware.4-tkg.1
guest-cluster-auth-service.tanzu.vmware.com.1.4.8+vmware.1-tkg.1
kapp-controller.tanzu.vmware.com.0.60.4+vmware.3-fips-tkg.1
metrics-server.tanzu.vmware.com.0.8.1+vmware.3-fips-tkg.1
pinniped.tanzu.vmware.com.0.46.0+vmware.8-tkg.2
secretgen-controller.tanzu.vmware.com.0.21.1+vmware.4-fips-tkg.1
vsphere-cpi.tanzu.vmware.com.1.36.0+vmware.2-tkg.1
vsphere-pv-csi.tanzu.vmware.com.3.8.0+vmware.3-tkg.1
```

This is the version-pinned list of the platform components from
[section 3](#3-what-the-platform-already-gives-you). Two things stand out. Both CNI options appear,
pinned to the release, so `bootstrapAddons.cniRef` selects among them while the release determines the
version; you do not choose a CNI version independently. And the Gateway API version is here, which is
what makes `gatewayApi.enabled: true` work with no CRD installation on your part.

The reason to diff this before upgrading is that a Kubernetes patch release is not only a patch. Across
the two v1.36 releases above, all ten packages changed:

| Component | Change | Why it matters |
| --- | --- | --- |
| antrea | `2.6.1+vmware.1` to `2.6.2+vmware.1` | A CNI upgrade, so your dataplane |
| kapp-controller | `0.59.8+vmware.1` to `0.60.4+vmware.3` | A minor bump in the component reconciling every addon |
| secretgen-controller | `0.20.1+vmware.3` to `0.21.1+vmware.4` | Minor bump |
| guest-cluster-auth-service | `1.4.7` to `1.4.8` | Authentication path |
| pinniped | `0.46.0+vmware.1` to `0.46.0+vmware.8` | Rebuild, authentication path |
| vsphere-pv-csi | `3.8.0+vmware.2` to `3.8.0+vmware.3` | Rebuild, storage |
| vsphere-cpi | `1.36.0+vmware.1` to `1.36.0+vmware.2` | Rebuild, LoadBalancer services |
| gateway-api, metrics-server, calico | Rebuilds | |

A CNI upgrade and a kapp-controller minor bump inside a Kubernetes patch release is the kind of scope
you want to know about while planning.

### D.3 OS images

```bash
kubectl get kr v1.36.2---vmware.2-vkr.3 \
  -o jsonpath='{range .spec.osImages[*]}{.name}{"\n"}{end}'
```

```
vmi-3219b260b6bb7ccf7
vmi-5c9649f9fba751a8b
vmi-a32066fa312b9ab5f
```

Opaque hashes. Resolve them through the cluster-scoped `OSImage` object, short name `osimg`:

```bash
for i in $(kubectl get kr v1.36.2---vmware.2-vkr.3 \
             -o jsonpath='{range .spec.osImages[*]}{.name}{" "}{end}'); do
  kubectl get osimage "$i" \
    -o jsonpath='{.metadata.name}{"  "}{.spec.os.name}/{.spec.os.version}{"  "}{.spec.os.arch}{"  "}{.spec.image.type}{"\n"}'
done
```

```
vmi-3219b260b6bb7ccf7  ubuntu/22.04   amd64  cvmi
vmi-5c9649f9fba751a8b  ubuntu/24.04   amd64  cvmi
vmi-a32066fa312b9ab5f  photon/5       amd64  cvmi
```

That is the authoritative list of OS choices for that release. In the environment used here, every
`READY` and `COMPATIBLE` release offered the same three, but that is an observation about one
environment rather than a guarantee.

The connection to the manifest is that `resolve-os-image` is a label selector against these objects:

```bash
kubectl get osimage vmi-5c9649f9fba751a8b --show-labels --no-headers \
  | awk '{print $NF}' | tr ',' '\n'
```

```
content-library=cl-3bd805269e1c8c7c1
image-type=cvmi
name=vmi-5c9649f9fba751a8b
os-arch=amd64
os-name=ubuntu
os-type=linux
os-version=24.04
run.tanzu.vmware.com/kubernetesVersion=v1.36.2---vmware.2
run.tanzu.vmware.com/tkr=v1.36.2---vmware.2-vkr.3
v1.36.2---vmware.2=
v1.36.2---vmware=
v1.36.2=
v1.36=
v1=
```

So you can test the annotation before applying it, which replaces a stalled cluster with a quick
check:

```bash
kubectl get osimage \
  -l os-name=ubuntu,os-version=24.04,run.tanzu.vmware.com/tkr=v1.36.2---vmware.2-vkr.3
```

Exactly one match is what you want. Selectable labels are `os-name`, `os-version`, `os-arch`, and
`os-type`; add `os-arch` if your environment ever carries mixed architectures. The empty-valued version
labels allow selection by version prefix as well as exact release. `content-library` tells you which
vSphere content library holds the image, which is where to look if an image is listed but unusable.

Two notes on the objects themselves. `cvmi` is a cluster-scoped `ClusterVirtualMachineImage` and `vmi`
is the older namespaced `VirtualMachineImage`; current releases use `cvmi`, and in the environment here
96 of 138 `OSImage` objects were legacy `vmi` belonging to non-compatible releases. `OSImage` has no
`status`: it is a projection owned by a `ClusterVirtualMachineImage`, so check that owner if an image
looks wrong.

On choosing: Ubuntu 24.04 is the current LTS with the longest support horizon and the sensible default.
Ubuntu 22.04 is the previous LTS, worth choosing only for a specific compatibility reason since you
start closer to end of support. Photon 5 is VMware's minimal purpose-built host OS, with a smaller
footprint and attack surface but fewer familiar userspace tools if you need to debug on a node.

### D.4 A pre-upgrade change report

Run this against your current and target releases before scheduling anything.

```bash
#!/usr/bin/env bash
# usage: kr-diff.sh <current-kr> <target-kr>
# Requires only kubectl and coreutils.
set -u
CUR="$1"; TGT="$2"
TMP=$(mktemp -d); trap 'rm -rf "$TMP"' EXIT

echo "### Core components"
for k in "$CUR" "$TGT"; do
  echo "  $k"
  kubectl get kr "$k" -o jsonpath='{"    kubernetes: "}{.spec.kubernetes.version}{"\n    etcd:       "}{.spec.kubernetes.etcd.imageTag}{"\n    coredns:    "}{.spec.kubernetes.coredns.imageTag}{"\n"}'
done

echo; echo "### Bootstrap package changes"
kubectl get kr "$CUR" -o jsonpath='{range .spec.bootstrapPackages[*]}{.name}{"\n"}{end}' \
  | sort > "$TMP/cur"
kubectl get kr "$TGT" -o jsonpath='{range .spec.bootstrapPackages[*]}{.name}{"\n"}{end}' \
  | sort > "$TMP/tgt"
diff "$TMP/cur" "$TMP/tgt" | sed 's/^</  - /; s/^>/  + /' || true

echo; echo "### OS images available in target"
for i in $(kubectl get kr "$TGT" -o jsonpath='{range .spec.osImages[*]}{.name}{" "}{end}'); do
  kubectl get osimage "$i" \
    -o jsonpath='  {.spec.os.name}/{.spec.os.version} {.spec.os.arch} ({.metadata.name}){"\n"}'
done

echo; echo "### Does the current resolve-os-image annotation still resolve?"
n=$(kubectl get osimage --no-headers \
      -l "os-name=ubuntu,os-version=24.04,run.tanzu.vmware.com/tkr=$TGT" 2>/dev/null | wc -l | tr -d ' ')
echo "  matches: $n (expect exactly 1)"
```

What to do with the output:

| Finding | Action |
| --- | --- |
| etcd version changed | Review etcd release notes, confirm quota headroom, and confirm you can recover it if the upgrade goes badly |
| CNI version changed | The highest-attention item, since it is your dataplane. Validate in non-production and check for NetworkPolicy or CRD behaviour changes. |
| kapp-controller version changed | It reconciles every addon. An addon that stops converging afterwards points here first. |
| CPI or CSI version changed | Test a LoadBalancer service and a PVC after the upgrade |
| Authentication components changed | Re-verify OIDC login end to end |
| Your OS choice absent from the target | Stop and resolve it, or machine creation will stall |
| Zero label matches | Fix the annotation before applying anything |

### D.5 Command reference

```bash
# Usable releases only
kubectl get kr --no-headers | awk '$3=="True" && $4=="True" {print $1}' | sort -V

# Everything a release ships
kubectl get kr <name> -o yaml

# What fields exist, with descriptions
kubectl explain kr.spec

# Core component versions
kubectl get kr <name> -o jsonpath='{.spec.kubernetes.version}{"  etcd "}{.spec.kubernetes.etcd.imageTag}{"  coredns "}{.spec.kubernetes.coredns.imageTag}{"\n"}'

# Version-pinned platform addons
kubectl get kr <name> -o jsonpath='{range .spec.bootstrapPackages[*]}{.name}{"\n"}{end}'

# OS image references, opaque
kubectl get kr <name> -o jsonpath='{range .spec.osImages[*]}{.name}{"\n"}{end}'

# ...resolved to OS name and version
kubectl get osimage -L os-name,os-version,os-arch -l run.tanzu.vmware.com/tkr=<name>

# Test a resolve-os-image annotation before applying it
kubectl get osimage -l os-name=ubuntu,os-version=24.04,run.tanzu.vmware.com/tkr=<name>

# What the running cluster landed on
kubectl get cluster <CLUSTER_NAME> -n <VSPHERE_NAMESPACE> \
  -o custom-columns='NAME:.metadata.name,CLUSTERCLASS:.spec.topology.classRef.name,VERSION:.status.version'
```

---

## Appendix E: Check these against your own environment

Everything in this guide is either derivable from the reference profile, verified against a running
VKS 3.7 environment, or stable Kubernetes behaviour. The items below are environment- or
release-specific and deliberately not asserted.

| Item | How to check |
| --- | --- |
| Addon default values. No specific default appears anywhere in this guide, because they move between releases. | `vcf addon available-releases get <release> -o output.yaml`, or `kubectl get acd <release> -n vmware-system-vks-public -o yaml`. Repeat after every addon upgrade. |
| Your ClusterClass variable set. The ten described here are `builtin-generic-v3.7.0`'s. | `kubectl get clusterclass <name> -n vmware-system-vks-public -o jsonpath='{range .status.variables[*]}{.name}{"\n"}{end}'` |
| Exact semantics of `resourceQuotaConfiguration.enabled`: which quota objects are created, and how they relate to vSphere Namespace limits. | Test in non-production; check your VKS documentation. |
| Headlamp secret-reference and Gateway TLS field names, needed to externalise the client secret and use a trusted CA. | Headlamp addon schema output. |
| Istio strict-mTLS and resource field names, referenced without asserting spellings. | Istio addon schema output. |
| The `maxSurge` your ClusterClass rollout strategy applies. This guide assumes the Cluster API default of 1, which sets your minimum spare-block reserve. | ClusterClass and `MachineDeployment` rollout strategy. |
| Whether the per-node pod CIDR mask is tunable. This guide assumes the IPv4 default of `/24`, confirmed on live nodes. A `/25` would double your block count but is a create-time decision. | `kubectl get nodes -o custom-columns='NODE:.metadata.name,POD_CIDR:.spec.podCIDR'` and the ClusterClass schema. |
| API server behaviour when a mapped `groups` claim is absent: tolerated, or does it reject? | Test with a real token and read the result from `kubectl auth whoami`. |
| Whether `240.0.0.0/4` is handled correctly by every appliance in your path. Antrea supports it; your firewalls, hardware load balancers, and monitoring may not. | Test pod-to-external and external-to-service traffic end to end before standardising. |
| Addon licensing and entitlement. Some addons imply other products, such as `ako` requiring NSX Advanced Load Balancer. Catalogue availability is not entitlement. | Your licensing, and `vcf addon available list`. |
| Custom VM classes. The `best-effort` and `guaranteed` naming convention may not apply, so confirm whether a custom class reserves. | `kubectl get virtualmachineclass` and vCenter. |
| PSS version strings valid for your Kubernetes minor. This guide uses `v1.36` to match `v1.36.1`. | Kubernetes documentation for your minor version. |
| The practical `maxPods` ceiling for your node sizes. The schema documents 250 as the maximum; what your VM classes sustain is a capacity question. | Load-test at your intended density. |
| The `systemReserved` formula. VKS documents the `automatic` flag but not its calculation, and the tiered model matched on two node sizes rather than being published. | Compare `capacity` against `allocatable` on your own nodes. |

Answered by direct inspection, and stated as fact above:

| Question | Answer | Confirm with |
| --- | --- | --- |
| Are the Gateway API CRDs installed automatically? | Yes, the `gateway-api` addon is platform-installed | `kubectl get clusteraddon -n <ns>` |
| Is the Cluster Autoscaler deployed by default? | Yes, so the machine-deployment annotations are effective | `kubectl get clusteraddon -n <ns>` |
| Are addons delivered as Helm releases? | No, as Carvel `PackageInstall` objects | `kubectl get pkgi -A`; `kubectl get helmrelease -A` returns nothing |
| Is there an addon ordering requirement? | No, all reconcile independently | Addon framework design |
| What happens if an `AddonConfig` name does not resolve? | It skips reconciliation and `spec.clusterName` stays empty | `kubectl explain addonconfig.spec` |
| `maxPods` default and maximum? | Default 110, documented maximum 250, minimum 20 | ClusterClass schema |
| Does the Headlamp addon use cert-manager? | Yes, it creates a self-signed `Issuer` and `Certificate` | `kubectl get certificate,issuer -n headlamp` |
| Does the Headlamp Gateway share the Istio ingress IP? | No, it provisions its own LoadBalancer | `kubectl get svc -n headlamp` |
