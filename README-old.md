# VKS 3.7 Workload Cluster + Addons — Annotated Reference & Best Practices

An annotated walkthrough of a VMware vSphere Kubernetes Service (VKS) 3.7 workload cluster and
its recommended addon stack — **helm-controller, cert-manager, Prometheus, Istio, and
Headlamp** — on the `builtin-generic-v3.7.0` ClusterClass, with OIDC authentication, Pod
Security Standards, and dedicated node volumes.

Every field is explained: what it does, why you would set it, and where it will bite you.
Sections follow the manifest order so you can read top to bottom alongside your own file.

**Audience.** Platform engineers and Kubernetes administrators deploying VKS. Kubernetes
fluency is assumed; VKS-specific concepts are explained.

**On accuracy.** Schema constraints, defaults, and platform behaviour in this document were
verified against a live VKS 3.7 environment — `builtin-generic-v3.7.0` and its addon catalogue.
Statements that are environment-specific are marked, and every command shown is one you can run
to confirm the behaviour yourself rather than take on trust. Where something depends on your
release or licensing, it is listed in [Appendix C](#appendix-c--verify-for-your-environment)
rather than asserted.

**Tooling.** Every command in this document runs with **`kubectl` and a POSIX shell only** —
output is shaped with `-o jsonpath`, `-o custom-columns`, `--show-labels`, and standard utilities
(`awk`, `sort`, `diff`). No `jq`, `yq`, or Python is required. Where a JSON formatter would merely
improve readability, it is noted as optional.

**Conventions.**

| Placeholder | Meaning |
| --- | --- |
| `<VSPHERE_NAMESPACE>` | Your vSphere Namespace on the Supervisor — the `metadata.namespace` of every object except the in-cluster ones in Appendix B |
| `<CLUSTER_NAME>` | Your workload cluster name. The examples use `workload-vsphere-vks2`. |
| `<STORAGE_CLASS>` | A StorageClass associated with your vSphere Namespace |
| `<OIDC_CLIENT_ID>` | The OAuth 2.0 / OIDC client ID from your identity provider |
| `<OIDC_CLIENT_SECRET>` | The corresponding client secret — never commit the real value |
| `headlamp.k8s.example.com` | The FQDN you will publish for the Headlamp UI |

---

## Read this first: this is a reference profile, not a production baseline

The manifest annotated here is a **working reference** — ideal for learning the object model and
for a proof of concept. Several values are deliberately permissive for a lab and should change
before production. Nothing here is broken; it is simply scoped for dev/test.

| Value | Fine for dev/test because | Production choice |
| --- | --- | --- |
| `controlPlane.replicas: 1` | Halves the footprint; a lab can tolerate API downtime. | **`3`** — etcd quorum and HA |
| `vmClass: best-effort-*` | Higher density on a shared lab cluster. | **`guaranteed-*`** — reserved CPU/memory |
| `podSecurityStandard: privileged` | Nothing is blocked, so nothing distracts from learning the platform. | **`enforce: baseline`** cluster-wide, `restricted` per namespace ([7.7.3](#773-securitypodsecuritystandard)) |
| `*Version: latest` (PSS) | Convenient; upgrade drift is irrelevant in a lab. | Pin explicitly (e.g. `v1.36`) |
| `pilot.replicas: 1`, ingress `minReplicas: 1` | One of each is enough to demonstrate routing. | **`2`** minimum for both |
| `oidc.clientSecret` inline | Fast to iterate on. | A referenced Secret |
| Self-signed Headlamp TLS (addon default) | Browser warnings are tolerable in a lab. | A certificate from a **trusted CA** |
| Pod CIDR `/20` | 16 node blocks is plenty for a small lab. | **`/16`** — immutable, so decide now ([7.1.1](#711-pod-cidr-sizing-node-blocks-maxpods-and-the-upgrade-spare)) |
| `claims.?email_verified.orValue(true)` | — | **`orValue(false)`** — the rule as written fails open ([Pitfall 1](#1-a-claim-validation-rule-that-fails-open)) |

A hardened, copy-pasteable version of the whole manifest is in
[Appendix A](#appendix-a--production-baseline-manifest).

> **The one item on that list that is not a dev/prod trade-off** is the
> `orValue(true)` claim validation rule. It is a logic error in either environment — see
> [Pitfall 1](#1-a-claim-validation-rule-that-fails-open).

---

## Contents

Numbered sections build on each other and are best read in order. The appendices are reference
material you can jump to directly. Each section opens with what it covers and why it matters.

- [Read this first: this is a reference profile, not a production baseline](#read-this-first-this-is-a-reference-profile-not-a-production-baseline)
- [The reference profile](#the-reference-profile)
    - [What is in it](#what-is-in-it)
    - [The shape of it](#the-shape-of-it)
    - [How to read this document alongside it](#how-to-read-this-document-alongside-it)
- [1. Prerequisites](#1-prerequisites)
    - [Getting CLI access to the Supervisor](#getting-cli-access-to-the-supervisor)
    - [Platform prerequisites](#platform-prerequisites)
- [2. Choosing a Kubernetes release: the `kr` object](#2-choosing-a-kubernetes-release-the-kr-object)
    - [The object](#the-object)
    - [Selecting one](#selecting-one)
    - [Which one to pick](#which-one-to-pick)
    - [Related objects](#related-objects)
- [3. What VKS installs for you](#3-what-vks-installs-for-you)
- [4. How addons work — the object model](#4-how-addons-work--the-object-model)
    - [The chain](#the-chain)
    - [How addons are delivered: Carvel packages](#how-addons-are-delivered-carvel-packages)
    - [Apply order does not matter](#apply-order-does-not-matter)
    - [The name convention is a defaulting mechanism](#the-name-convention-is-a-defaulting-mechanism)
    - [Verifying the whole stack in one command](#verifying-the-whole-stack-in-one-command)
- [5. Discovering addons and their value schemas](#5-discovering-addons-and-their-value-schemas)
    - [With the `vcf` CLI](#with-the-vcf-cli)
    - [With `kubectl`, against the Supervisor](#with-kubectl-against-the-supervisor)
- [6. The addons, annotated](#6-the-addons-annotated)
    - [6.1 helm-controller](#61-helm-controller)
    - [6.2 cert-manager](#62-cert-manager)
    - [6.3 Prometheus](#63-prometheus)
    - [6.4 Istio — deployed for L4/L7, not as a service mesh](#64-istio--deployed-for-l4l7-not-as-a-service-mesh)
    - [6.5 Headlamp](#65-headlamp)
    - [6.6 Additional addons worth considering](#66-additional-addons-worth-considering)
- [7. The `Cluster` object, annotated](#7-the-cluster-object-annotated)
    - [7.1 Metadata and cluster networking](#71-metadata-and-cluster-networking)
    - [7.1.1 Pod CIDR sizing: node blocks, `maxPods`, and the upgrade spare](#711-pod-cidr-sizing-node-blocks-maxpods-and-the-upgrade-spare)
    - [7.2 Topology and ClusterClass](#72-topology-and-clusterclass)
    - [7.3 Control plane](#73-control-plane)
    - [7.4 `storageClass`](#74-storageclass)
    - [7.5 `vmClass` — use `guaranteed-*` for production](#75-vmclass--use-guaranteed--for-production)
    - [7.6 `volumes` — dedicated containerd and kubelet disks](#76-volumes--dedicated-containerd-and-kubelet-disks)
    - [7.7 `kubernetes` — certificates, etcd, security, and API server](#77-kubernetes--certificates-etcd-security-and-api-server)
    - [7.8 `bootstrapAddons` — CNI selection](#78-bootstrapaddons--cni-selection)
    - [7.9 `vsphereOptions` — persistent volume storage classes](#79-vsphereoptions--persistent-volume-storage-classes)
    - [7.10 `osConfiguration` — NTP and SSH banner](#710-osconfiguration--ntp-and-ssh-banner)
    - [7.11 `version`](#711-version)
    - [7.12 `workers.machineDeployments` — the node pool](#712-workersmachinedeployments--the-node-pool)
    - [7.13 3.7 variables not used in the sample](#713-37-variables-not-used-in-the-sample)
- [8. Cluster decisions you cannot change later](#8-cluster-decisions-you-cannot-change-later)
- [9. Cross-object dependency map](#9-cross-object-dependency-map)
- [10. Consolidated pitfalls](#10-consolidated-pitfalls)
    - [1. A claim validation rule that fails open](#1-a-claim-validation-rule-that-fails-open)
    - [2. An `AddonConfig` that is silently skipped](#2-an-addonconfig-that-is-silently-skipped)
    - [3. OIDC client secret in plaintext](#3-oidc-client-secret-in-plaintext)
    - [4. Pod CIDR exhaustion blocks upgrades](#4-pod-cidr-exhaustion-blocks-upgrades)
    - [5. Missing RBAC after OIDC login](#5-missing-rbac-after-oidc-login)
    - [6. A Gateway with no controller](#6-a-gateway-with-no-controller)
    - [7. Hostname, DNS, and redirect URI drift](#7-hostname-dns-and-redirect-uri-drift)
    - [8. Prometheus operator with no `Prometheus` CR](#8-prometheus-operator-with-no-prometheus-cr)
    - [9. The IdP emits no `groups` claim](#9-the-idp-emits-no-groups-claim)
    - [10. Pod Security versions pinned to `latest`](#10-pod-security-versions-pinned-to-latest)
    - [11. StorageClass or VM class not associated with the vSphere Namespace](#11-storageclass-or-vm-class-not-associated-with-the-vsphere-namespace)
    - [12. `best-effort` VM classes in production](#12-best-effort-vm-classes-in-production)
    - [13. Clock skew breaks OIDC intermittently](#13-clock-skew-breaks-oidc-intermittently)
    - [14. etcd read-only after exceeding its quota](#14-etcd-read-only-after-exceeding-its-quota)
    - [15. Istio `pilot.replicas` fighting its own HPA](#15-istio-pilotreplicas-fighting-its-own-hpa)
    - [16. `maxPods` raised without matching node resources](#16-maxpods-raised-without-matching-node-resources)
- [11. Verification and troubleshooting](#11-verification-and-troubleshooting)
    - [Layer 1 — Supervisor: does the cluster exist?](#layer-1--supervisor-does-the-cluster-exist)
    - [Layer 2 — Addons: installed, bound, and ready?](#layer-2--addons-installed-bound-and-ready)
    - [Layer 3 — Guest cluster: did the packages reconcile?](#layer-3--guest-cluster-did-the-packages-reconcile)
    - [Layer 4 — Istio and the ingress gateway](#layer-4--istio-and-the-ingress-gateway)
    - [Layer 5 — Gateway API, Headlamp, and TLS](#layer-5--gateway-api-headlamp-and-tls)
    - [Layer 6 — Storage](#layer-6--storage)
    - [Layer 7 — Identity: what username does the API server see?](#layer-7--identity-what-username-does-the-api-server-see)
    - [Layer 8 — Observability: is anything actually evaluating rules?](#layer-8--observability-is-anything-actually-evaluating-rules)
    - [Layer 9 — Pod CIDR headroom](#layer-9--pod-cidr-headroom)
- [12. Day-2 lifecycle](#12-day-2-lifecycle)
    - [Upgrade sequencing](#upgrade-sequencing)
    - [Which changes replace nodes](#which-changes-replace-nodes)
    - [Addon lifecycle](#addon-lifecycle)
    - [Certificate rotation](#certificate-rotation)
    - [etcd growth](#etcd-growth)
    - [Tightening Pod Security](#tightening-pod-security)
    - [OIDC and hostname changes](#oidc-and-hostname-changes)
    - [Backup and restore](#backup-and-restore)
- [Appendix A — Production baseline manifest](#appendix-a--production-baseline-manifest)
- [Appendix B — Additional objects and hardening](#appendix-b--additional-objects-and-hardening)
    - [B.1 RBAC for OIDC identities](#b1-rbac-for-oidc-identities)
    - [B.2 Trusted TLS for the Headlamp Gateway](#b2-trusted-tls-for-the-headlamp-gateway)
    - [B.3 A `Prometheus` instance via the operator](#b3-a-prometheus-instance-via-the-operator)
    - [B.4 Further hardening worth considering](#b4-further-hardening-worth-considering)
- [Appendix C — Verify for your environment](#appendix-c--verify-for-your-environment)
- [Appendix D — Optional integrations requiring environment-specific setup](#appendix-d--optional-integrations-requiring-environment-specific-setup)
    - [D.1 Backup and restore (`velero`)](#d1-backup-and-restore-velero)
    - [D.2 Log forwarding (`fluent-bit`)](#d2-log-forwarding-fluent-bit)
    - [D.3 DNS automation (`external-dns`)](#d3-dns-automation-external-dns)
- [Appendix E — Inspecting a Kubernetes release (`kr`)](#appendix-e--inspecting-a-kubernetes-release-kr)
    - [E.1 `.spec.kubernetes` — core control-plane components](#e1-speckubernetes--core-control-plane-components)
    - [E.2 `.spec.bootstrapPackages` — the platform addons pinned to the release](#e2-specbootstrappackages--the-platform-addons-pinned-to-the-release)
    - [E.3 `.spec.osImages` and the `osimage` object](#e3-specosimages-and-the-osimage-object)
    - [E.4 A pre-upgrade change report](#e4-a-pre-upgrade-change-report)
    - [E.5 Command reference](#e5-command-reference)
- [Summary — the short list](#summary--the-short-list)
    - [Before you apply](#before-you-apply)
    - [Fix before production](#fix-before-production)
    - [Addons to consider](#addons-to-consider)
    - [Keep — these parts of the sample are good practice](#keep--these-parts-of-the-sample-are-good-practice)
    - [Four points worth being explicit about](#four-points-worth-being-explicit-about)

---

## The reference profile

> **What this covers.** The single manifest that the rest of this document annotates — what is in it,
> why each piece is there, and where to find the full file.
>
> **Why it matters.** Every section from 6 onward explains fields *of this manifest*. Seeing the whole
> artifact first turns the document into a walkthrough rather than a list of disconnected settings.

This document is not a survey of options — it annotates **one complete, working manifest**, field by
field. Knowing what that manifest contains before you start makes the rest of the document read as a
walkthrough rather than a list of disconnected settings.

**The full manifest lives alongside this document as [`reference-profile.yaml`](./reference-profile.yaml).**
It carries the same values discussed here, with a comment block above each object explaining what it
is and why it is in the stack. Open it in a second window and read the two together.

> **It is a learning and demonstration profile, not a deployable baseline.** Several values are
> deliberately permissive so the platform can be explored without HA or admission control in the way
> — see [Read this first](#read-this-first-this-is-a-reference-profile-not-a-production-baseline) for
> the full list, and [Appendix A](#appendix-a--production-baseline-manifest) for the hardened
> counterpart.

### What is in it

Ten objects: five addons (each an `AddonInstall`, four with a matching `AddonConfig`) and the
`Cluster` itself. All ten live in your **vSphere Namespace on the Supervisor**.

| # | Object | What it does | Why it is in the profile | Annotated in |
| --- | --- | --- | --- | --- |
| 1 | `AddonInstall` + `AddonConfig` **helm-controller** | Declarative Helm chart lifecycle management inside the cluster | A day-2 capability for *your own* charts. Notably **not** a prerequisite for the others. | [6.1](#61-helm-controller) |
| 2 | `AddonInstall` **cert-manager** | Certificate issuance and rotation | A real dependency — Headlamp uses it to terminate TLS on its Gateway. Ships with **no** `AddonConfig`, which is legal and deliberate. | [6.2](#62-cert-manager) |
| 3 | `AddonInstall` + `AddonConfig` **prometheus** | Exporters plus the prometheus-operator | The operator-managed pattern: you declare your own `Prometheus` CR instead of accepting a packaged server. | [6.3](#63-prometheus) |
| 4 | `AddonInstall` + `AddonConfig` **istio** | L4/L7 ingress gateway and Gateway API controller | The cluster's north-south entry point. **Not** deployed as a service mesh — no sidecars. | [6.4](#64-istio--deployed-for-l4l7-not-as-a-service-mesh) |
| 5 | `AddonInstall` + `AddonConfig` **headlamp** | A web UI, exposed via Gateway API and authenticated with OIDC | The human-facing entry point, and the reason the API server carries an OIDC configuration. | [6.5](#65-headlamp) |
| 6 | `Cluster` | The workload cluster, on the `builtin-generic-v3.7.0` ClusterClass | Networking, node shape, storage, Pod Security, and the OIDC trust that makes the UI login work. | [7](#7-the-cluster-object-annotated) |

### The shape of it

An outline of all ten objects and every `Cluster` variable, so nothing below comes as a surprise:

```text
── Addons (vSphere Namespace on the Supervisor) ─────────────────────────────
AddonInstall/cluster-helm-controller          → addonRef: helm-controller
AddonConfig/<cluster>-helm-controller           priorityClassName × 2
AddonInstall/cluster-cert-manager             → addonRef: cert-manager
AddonInstall/cluster-prom                     → addonRef: prometheus
AddonConfig/<cluster>-prometheus                deploycomponents × 6
AddonInstall/cluster-istio                    → addonRef: istio
AddonConfig/<cluster>-istio                     namespace, ambientMode, istioCNI,
                                                gateways.ingress, pilot, meshConfig
AddonInstall/cluster-headlamp                 → addonRef: headlamp
AddonConfig/<cluster>-headlamp                  hostname, gatewayApi, oidc

── Cluster ──────────────────────────────────────────────────────────────────
Cluster/<cluster>
  clusterNetwork          pods /20 · services /20 · serviceDomain
  topology.classRef       builtin-generic-v3.7.0
  topology.controlPlane   replicas 1 · vmClass override · OS image annotation
  topology.variables
    storageClass          node disks
    vmClass               worker node shape
    volumes               /var/lib/containerd · /var/lib/kubelet
    kubernetes            certificateRotation · etcdConfiguration · security
                          apiServerConfiguration (incl. OIDC) · kubeletConfiguration
    bootstrapAddons       cniRef: antrea
    vsphereOptions        PV storage classes
    osConfiguration       ntp · sshd banner
  topology.version        v1.36.1---vmware.4-vkr.5
  topology.workers        one machineDeployment, autoscaler-annotated
```

Three ClusterClass variables are available but **not** used by this profile — `node`,
`resourceConfiguration`, and the deprecated `kubeAPIServerFQDNs`. Two of them are worth adding;
see [7.13](#713-37-variables-not-used-in-the-sample).

### How to read this document alongside it

| If you want to… | Go to |
| --- | --- |
| Understand one field | [6](#6-the-addons-annotated) for addons, [7](#7-the-cluster-object-annotated) for the `Cluster` |
| Know what you cannot change later | [8](#8-cluster-decisions-you-cannot-change-later) |
| See which values must agree with each other | [9](#9-cross-object-dependency-map) |
| Know what will bite you | [10](#10-consolidated-pitfalls) |
| Deploy something production-ready instead | [Appendix A](#appendix-a--production-baseline-manifest) |


---

## 1. Prerequisites

> **What this covers.** Getting `kubectl` access to the Supervisor, and confirming that the VM
> classes, storage policies, and Kubernetes release the manifest references actually exist.
>
> **Why it matters.** These objects are accepted by the API server and *then* stall. An unchecked
> prerequisite does not fail at apply time — it surfaces later as a machine that never provisions or
> a PVC that pends forever, with a thin error surface either way.

Objects that reference resources not associated with your vSphere Namespace do not fail fast —
they are accepted by the API server and then stall, often with a thin error surface. These
checks are cheap and save real debugging time.

### Getting CLI access to the Supervisor

Everything in this document is driven through `kubectl` against the **Supervisor**, so start here.
The `vcf` CLI creates the context and writes the kubeconfig for you:

```bash
# Authenticate to the Supervisor and create a context
vcf context create <CONTEXT_NAME> --endpoint https://<SUPERVISOR_FQDN_OR_IP> --type k8s

# If the Supervisor uses a private or enterprise CA, supply the bundle
vcf context create <CONTEXT_NAME> --endpoint https://<SUPERVISOR_FQDN_OR_IP> \
  --ca-certificate /path/to/ca-cert --type k8s
```

That produces one context per vSphere Namespace you can reach, plus one per workload cluster:

```bash
kubectl config get-contexts
```

```
CURRENT   NAME                          CLUSTER
          supervisor                    supervisor:10.0.0.6        ← Supervisor, no namespace
*         supervisor:<VSPHERE_NAMESPACE> supervisor:10.0.0.6       ← Supervisor, namespace-scoped
          vks                           vks:10.0.0.6
          vks:<CLUSTER_NAME>            vks:10.0.0.12              ← inside the workload cluster
```

**Two contexts matter, and mixing them up is the most common early confusion:**

| Context | Use it for |
| --- | --- |
| **Supervisor**, scoped to your vSphere Namespace | `Cluster`, `AddonInstall`, `AddonConfig`, `ClusterAddon`, `kr`, `osimage`, `virtualmachineclass` — everything you *author* |
| **Workload cluster** | `pkgi`, pods, `Gateway`, RBAC, `Prometheus` CRs — everything *inside* the cluster |

```bash
kubectl config use-context supervisor:<VSPHERE_NAMESPACE>    # authoring
kubectl config use-context vks:<CLUSTER_NAME>                # in-cluster verification
```

Throughout this document, commands are run against the **Supervisor** unless a section says
otherwise — [section 5](#5-discovering-addons-and-their-value-schemas) and
[Layer 3](#layer-3--guest-cluster-did-the-packages-reconcile) onward call out the switch explicitly.

> **This is also your recovery path.** `vcf context create` regenerates a working kubeconfig at any
> time, so a failure of the external identity provider used for
> [OIDC](#776-apiserverconfigurationextraauthentication--oidc) does not lock you out of the cluster.

### Platform prerequisites

| # | Requirement | Verify with |
| --- | --- | --- |
| 1 | **CLI access to the Supervisor**, with a context selected. | `kubectl config current-context` |
| 2 | A **vSphere Namespace** exists and you have permissions on it. This is `metadata.namespace` throughout. | `kubectl get ns <VSPHERE_NAMESPACE>` |
| 3 | **VM classes** are associated with the namespace. | `kubectl get virtualmachineclass` |
| 4 | **Storage policies** are associated with the namespace. | `kubectl describe ns <VSPHERE_NAMESPACE>` — the storage quota lists every associated class (see below) |
| 5 | A **Kubernetes release** matching your target version is available and compatible. | `kubectl get kr` — see [section 2](#2-choosing-a-kubernetes-release-the-kr-object) |
| 6 | The **ClusterClass** you intend to use is present and its variables are ready. | `kubectl get clusterclass -n vmware-system-vks-public` |
| 7 | **OS images** are synced so the requested image resolves. | `kubectl get osimage` — see [Appendix E.3](#e3-specosimages-and-the-osimage-object) |
| 8 | The **`vcf` CLI** is available, for addon discovery. | `vcf version` |

**The storage-class discovery trick.** `StorageClass` is cluster-scoped, so a namespace user
usually cannot list it. But the namespace's storage quota names every associated class, which is
exactly the list you need:

```bash
kubectl describe ns <VSPHERE_NAMESPACE> | grep storageclass
# → vsan-esa-default-policy-raid5.storageclass.storage.k8s.io/requests.storage   127Gi   ...
```

The prefix before `.storageclass.storage.k8s.io` is the StorageClass name to use in the
manifest. If a class you intend to use is not in this output, **it is not available to this
namespace** and every reference to it will fail later, not now.

> **The most common conceptual error in VKS.** `metadata.namespace` on these objects is a
> **vSphere Namespace on the Supervisor**, not a namespace inside the workload cluster. The
> manifest contains both kinds: `metadata.namespace: <VSPHERE_NAMESPACE>` is a Supervisor
> namespace, while `spec.values.istio.namespace: istio-system` is a namespace *inside the
> workload cluster*. Different API servers entirely.

---

## 2. Choosing a Kubernetes release: the `kr` object

> **What this covers.** How to list the Kubernetes releases your Supervisor offers, read their status
> columns, and pick a valid one.
>
> **Why it matters.** `topology.version` is a single string that either matches a release or silently
> never reconciles. The release you choose also pins your etcd version, your CNI version, and which
> node OS images you may select — so this is a bigger decision than the version number suggests.

`topology.version` must exactly match a Kubernetes release the Supervisor offers. This is a
frequent source of clusters that are accepted and then never reconcile, and it is entirely
avoidable — the available releases are queryable.

### The object

VKS models each available Kubernetes version as a cluster-scoped **`KubernetesRelease`**
(`kubernetes.vmware.com/v1alpha1`, short name **`kr`**):

```bash
kubectl get kr
```

```
NAME                              VERSION                           READY   COMPATIBLE   DEACTIVATED BY   CREATED
v1.35.6---vmware.2-vkr.3          v1.35.6+vmware.2-vkr.3            True    True                          61d
v1.36.1---vmware.4-vkr.5          v1.36.1+vmware.4-vkr.5            True    True                          5d
v1.36.2---vmware.2-vkr.3          v1.36.2+vmware.2-vkr.3            True    True                          2d
v1.25.13---vmware.1-fips.1-tkg.1  v1.25.13+vmware.1-fips.1-tkg.1    False   False                         124d
```

An environment typically lists **many** releases, most of them historical. The list is not a
menu — the status columns are.

| Column | Meaning | Consequence |
| --- | --- | --- |
| `READY` | The release's artifacts (OS images, component packages) are present and usable. | `False` → the release cannot be deployed even though the object exists. |
| `COMPATIBLE` | The release is compatible with **this Supervisor / VKS version**. | `False` → not a valid choice. This is what filters out legacy releases. |
| `DEACTIVATED BY` | Names a policy or object that has explicitly disabled this release. | Non-empty → deliberately blocked, likely on purpose. Ask before overriding. |

**Only a release that is `READY=True` and `COMPATIBLE=True` with an empty `DEACTIVATED BY` is a
valid `topology.version`.** In the environment this document was verified against, that reduced
a long list to **13** usable releases.

### Selecting one

```bash
# the only releases you should be choosing from — READY and COMPATIBLE are
# columns 3 and 4 of the default output
kubectl get kr --no-headers | awk '$3=="True" && $4=="True" {print $1}' | sort -V
```

Then copy the **`NAME`**, not the `VERSION`, into `topology.version`:

```yaml
    version: v1.36.1---vmware.4-vkr.5     # ← the kr NAME
```

> **The version used throughout this document is illustrative.** Any release that is `READY=True`
> and `COMPATIBLE=True` in *your* environment is a valid choice — substitute whatever your version
> policy selects. The point of this section is the selection *method*, not the specific string.

| Field | Format | Use |
| --- | --- | --- |
| `kr` `NAME` | `v1.36.1---vmware.4-vkr.5` — **triple** dash | **This is what `topology.version` takes.** |
| `kr` `VERSION` | `v1.36.1+vmware.4-vkr.5` — plus sign | The semver form. Appears in `Cluster.status` and `kubectl version`. |

> **The `---` is not a typo and not interchangeable with the `+` form.** The triple dash is a
> DNS-safe encoding of the `+` build separator, because the `+` is not legal in a Kubernetes
> object name. Using the `+` form, or a single or double dash, produces a topology that never
> reconciles.

### Which one to pick

| Consideration | Guidance |
| --- | --- |
| **Newest vs. proven** | The newest compatible release is not automatically the right one. Prefer a release you have validated, and stay within the support window. |
| **Patch level** | Among releases of the same minor, prefer the highest patch you have validated — these carry CVE and bug fixes. |
| **ClusterClass compatibility** | The Kubernetes version and the ClusterClass move together. Do not bump one independently. |
| **Upgrade path** | **One minor version at a time.** Skipping minors is unsupported. Plan `1.35 → 1.36 → 1.37`, not `1.35 → 1.37`. |
| **Fleet consistency** | Standardise on one release per environment tier. A fleet on five different patch levels is a support and troubleshooting burden with no upside. |

> **To see what is actually inside a release** — component versions, the pinned platform addons, and
> the OS images it offers — see [Appendix E](#appendix-e--inspecting-a-kubernetes-release-kr). That is
> also where you test a `resolve-os-image` annotation before applying it.

### Related objects

| Object | Purpose |
| --- | --- |
| `kr` / `kubernetesreleases` | The VKS-native release object. **Use this.** |
| `tkr` / `tanzukubernetesreleases` | The older TKG-era equivalent, still present for compatibility. |
| `clusterclass` | Defines the variable schema; version-paired with the release. |

```bash
# after deployment, confirm what the cluster actually landed on
kubectl get cluster <CLUSTER_NAME> -n <VSPHERE_NAMESPACE> \
  -o custom-columns='NAME:.metadata.name,CLUSTERCLASS:.spec.topology.classRef.name,VERSION:.status.version'
```

---

## 3. What VKS installs for you

> **What this covers.** The platform-managed addons a VKS cluster arrives with, before you declare
> anything.
>
> **Why it matters.** It stops you installing something you already have, and it explains two things
> that otherwise look like magic: why the autoscaler annotations in the manifest are effective, and
> why Gateway API works for Headlamp without you installing any CRDs.

Before adding anything, know what is already there. A VKS 3.7 cluster arrives with a set of
platform-managed addons you do not declare — so some of what you might reach for is already
running.

```bash
# lists platform-managed and your own addons together
kubectl get clusteraddon -n <VSPHERE_NAMESPACE>
```

`ClusterAddon` is the platform-created object that joins a cluster to an addon and its resolved
release; [section 4](#4-how-addons-work--the-object-model) covers it and the rest of the addon object
model. For now it is simply the fastest way to see what a fresh cluster already has.

| Auto-installed | What it provides |
| --- | --- |
| **antrea** (or your chosen CNI) | Pod networking and NetworkPolicy |
| **vsphere-cpi** | Cloud provider integration — node lifecycle, LoadBalancer services |
| **vsphere-pv-csi** | Persistent volume provisioning from vSphere storage |
| **cluster-autoscaler** | **Already present** — this is why the machine-deployment autoscaler annotations work |
| **gateway-api** | **The Gateway API CRDs** — so `gatewayApi.enabled: true` has something to bind to |
| **metrics-server** | Resource metrics for `kubectl top` and HPA |
| **pinniped** / **guest-cluster-auth-service** | Cluster authentication plumbing |
| **secretgen-controller** | Carvel secret generation and export |
| **carvel-repo** | The package repository the addons are pulled from |
| **vks-static-resources** | Version-matched static platform resources |

Two of these resolve questions people routinely ask:

- **You do not need to install the Gateway API CRDs.** They arrive with the platform. Confirm:
  `kubectl get crd | grep gateway.networking.k8s.io`
- **You do not need to deploy the Cluster Autoscaler.** It is already running, which is what
  makes the `cluster-api-autoscaler-node-group-*-size` annotations effective.

> **Do not manage these by hand.** They are platform-owned; changes will be reverted and can
> block upgrades.

---

## 4. How addons work — the object model

> **What this covers.** The objects between the two you author and the pods that eventually run —
> `ClusterAddon`, `AddonConfigDefinition`, and the Carvel `PackageInstall`.
>
> **Why it matters.** It tells you where to look when an addon does not appear, and it settles the two
> questions that waste the most time: how addons are actually delivered, and whether apply order
> matters. It also explains the naming rule that makes configuration silently vanish.

Understanding this model removes most addon troubleshooting guesswork, and it corrects two
assumptions people commonly bring from other Kubernetes distributions.

### The chain

You author the two objects on the left. The platform creates the rest.

```
   YOU AUTHOR                        PLATFORM CREATES
   ─────────────                     ────────────────
   AddonInstall  ──┐
   (which addon,   │                 ClusterAddon           PackageInstall (Carvel)
    which clusters)├──►  addon  ──►  (the join: cluster  ──► in vmware-system-tkg
   AddonConfig   ──┘   management     × addon × release)      inside the workload cluster
   (values for one                                                    │
    cluster)                                                          ▼
                                                            kapp-controller reconciles
                                                            the addon's resources
```

| Object | Scope | Purpose |
| --- | --- | --- |
| `AddonInstall` | vSphere Namespace | *Which* addon, and *which clusters* get it (by label selector). |
| `AddonConfig` | vSphere Namespace | *How* it is configured on one specific cluster. Optional. |
| `ClusterAddon` | vSphere Namespace | Platform-created join object: cluster × addon × resolved release, with a `READY` condition. **The best single object for verifying addon state.** |
| `AddonConfigDefinition` (`acd`) | `vmware-system-vks-public` | The value schema for one addon release. |
| `PackageInstall` (`pkgi`) | `vmware-system-tkg`, **inside the workload cluster** | The Carvel package that actually deploys the addon's resources. |

### How addons are delivered: Carvel packages

Every addon in this stack is delivered as a **Carvel `PackageInstall`**, reconciled by
kapp-controller inside the workload cluster. You can see this directly:

```bash
# in the workload cluster
kubectl get pkgi -A
```

```
NAMESPACE           NAME                          PACKAGE NAME                          DESCRIPTION
vmware-system-tkg   <cluster>-cert-manager        cert-manager.kubernetes.vmware.com     Reconcile succeeded
vmware-system-tkg   <cluster>-headlamp            headlamp.kubernetes.vmware.com         Reconcile succeeded
vmware-system-tkg   <cluster>-istio               istio.kubernetes.vmware.com            Reconcile succeeded
vmware-system-tkg   <cluster>-prometheus          prometheus.kubernetes.vmware.com       Reconcile succeeded
...
```

**This matters because it changes where you look when something is wrong.** `pkgi` and its
`DESCRIPTION` column (`Reconcile succeeded` / `Reconcile failed`) is the authoritative status for
addon delivery. Looking for `HelmRelease` objects returns nothing:

```bash
kubectl get helmrelease -A
# → No resources found
```

### Apply order does not matter

**All addons are reconciled independently by the addon management layer. There is no ordering
requirement and no dependency chain you need to sequence.** Apply the whole manifest in one
`kubectl apply` and let the controllers converge.

In particular, **`helm-controller` is not a prerequisite for the other addons.** It does not deliver
them and they do not wait for it. It is an independent addon providing Helm lifecycle management for
*your own* charts, as a day-2 capability inside the cluster. See [6.1](#61-helm-controller).

The only *practical* sequencing concern is not an ordering requirement at all: the Headlamp
`hostname` and `callbackURL` must match a DNS name and a registered OIDC redirect URI. If you
publish the FQDN in advance — which you should — you can apply everything at once. See
[6.5](#65-headlamp).

### The name convention is a defaulting mechanism

`AddonConfig.spec` has three fields:

```bash
kubectl explain addonconfig.spec
```

| Field | Behaviour if unset |
| --- | --- |
| `clusterName` | *"If not specified, the AddonConfig will skip the reconciliation."* |
| `addonConfigDefinitionRef` | *"If not specified, the AddonConfig will skip the reconciliation."* |
| `values` | Your configuration overlay |

The sample manifest sets **neither** of the first two. Instead it relies on naming the object
`<clusterName>-<addonName>`, from which the platform derives and then records both fields. You
can verify that resolution happened:

```bash
kubectl get addonconfig <CLUSTER_NAME>-istio -n <VSPHERE_NAMESPACE> \
  -o jsonpath='{.spec.clusterName}{"  "}{.spec.addonConfigDefinitionRef.name}{"\n"}'
# → workload-vsphere-vks2  istio.kubernetes.vmware.com.1.30.2---vmware.1-vks.1
```

**If those fields come back empty, the config is being skipped** and the addon is running on pure
schema defaults. This is the precise, checkable form of
[Pitfall 2](#2-an-addonconfig-that-is-silently-skipped).

> **You can also set them explicitly**, which is worth doing in generated or templated manifests
> where a naming typo would otherwise be silent:
> ```yaml
> spec:
>   clusterName: workload-vsphere-vks2
>   addonConfigDefinitionRef:
>     name: istio.kubernetes.vmware.com.1.30.2---vmware.1-vks.1
>     namespace: vmware-system-vks-public
>   values: {...}
> ```
> Naming the `acd` explicitly also **pins the addon release**, rather than accepting whatever the
> platform resolves.

### Verifying the whole stack in one command

```bash
kubectl get clusteraddon -n <VSPHERE_NAMESPACE>
```

```
NAME                     CLUSTER                 ADDON                ADDONINSTALL        RELEASE                    READY
<cluster>-cert-manager   workload-vsphere-vks2   cert-manager         cluster-cert-manager cert-manager...1.20.3      True
<cluster>-istio          workload-vsphere-vks2   istio                cluster-istio        istio...1.30.2             True
<cluster>-prometheus     workload-vsphere-vks2   prometheus           cluster-prom         prometheus...3.5.4         True
```

This is the single most useful addon command: it shows the resolved release, the owning
`AddonInstall`, and readiness in one view. Note the `prometheus` row — the `AddonInstall` is named
`cluster-prom` while the addon is `prometheus`. **The `ADDON` column, not the `AddonInstall`
name, is what the `AddonConfig` name must match.**

---

## 5. Discovering addons and their value schemas

> **What this covers.** Two equivalent ways to read an addon release's value schema — the `vcf` CLI
> and `kubectl`.
>
> **Why it matters.** `AddonConfig` is a sparse overlay: you set a few fields and inherit everything
> else. You cannot reason about your configuration without knowing what those inherited defaults are,
> and they move between addon releases — which is why this document quotes commands rather than
> default values.

You now know that an `AddonConfig` is validated against an `AddonConfigDefinition`
([section 4](#4-how-addons-work--the-object-model)). This section is how you read that definition —
because before writing any `AddonConfig`, you need to know what fields exist and what they default
to. There are two ways, and they return the same schemas.

### With the `vcf` CLI

```bash
vcf addon available list                                    # every addon in your environment
vcf addon available-releases list --addon-name istio         # releases of one addon
vcf addon available-releases get <release> -o output.yaml    # the full commented value schema
```

### With `kubectl`, against the Supervisor

Each addon release is represented by an `AddonConfigDefinition` (short name `acd`), which
carries the OpenAPI schema the addon management layer validates your `AddonConfig` against:

```bash
# every addon release available, as objects
kubectl get acd -n vmware-system-vks-public

# just the releases of one addon
kubectl get acd -n vmware-system-vks-public | grep '^istio'

# the schema itself
kubectl get acd <release-name> -n vmware-system-vks-public -o yaml
```

**Step 3 / the schema dump is the step that gets skipped, and it is the important one.**

- **`AddonConfig` is a sparse overlay, not a complete specification.** You set only what you
  want to change; everything else comes from the schema defaults. You cannot reason about your
  configuration without knowing those defaults.
- **Field names and nesting are not guessable.** These schemas resemble the upstream Helm chart
  values they derive from but are not identical, and a field at the wrong nesting level is
  rejected or ignored rather than silently working.
- **Defaults move between addon releases.** Any default written into a document — including this
  one — is a snapshot. The command is not. **This document therefore does not assert specific
  addon default values**; where a value looks like a deliberate deviation, it says so and points
  you here.

> **Pin what you deploy.** These commands return specific, versioned releases. Record the exact
> release you validated against and promote new ones deliberately — see
> [Day-2](#12-day-2-lifecycle).

---

## 6. The addons, annotated

> **What this covers.** Every field of the five addons in the reference profile: what it does, why it
> is set that way, and where it will bite you.
>
> **Why it matters.** Each of these addons has at least one value whose effect is not deducible from
> its name — a monitoring stack that collects nothing, an ingress controller that is not a mesh, a UI
> that needs three systems to agree before anyone can log in.

The five addons in this stack are a sensible baseline: package lifecycle, certificates,
monitoring, ingress, and a UI. [6.6](#66-additional-addons-worth-considering) covers the rest of
the catalogue.

Every `AddonInstall` in the manifest follows the same shape, so it is explained once here and not
repeated per addon:

```yaml
apiVersion: addons.kubernetes.vmware.com/v1alpha1   # ← VKS addon framework API group (alpha)
kind: AddonInstall                                  # ← "install this addon on these clusters"
metadata:
  name: cluster-istio                               # ← arbitrary; names the intent, not the addon
  namespace: <VSPHERE_NAMESPACE>                    # ← vSphere Namespace, NOT a guest namespace
spec:
  addonRef:
    name: istio                                     # ← must match `vcf addon available list`
  clusters:
  - selector:                                       # ← a LIST — multiple selectors allowed
      matchLabels:
        cluster.x-k8s.io/cluster-name: <CLUSTER_NAME>
```

| Field | Why | Pitfall |
| --- | --- | --- |
| `apiVersion` | `v1alpha1` — an alpha API. Expect field changes across VKS releases. | No compatibility guarantee. Keep manifests in version control so you can diff after an upgrade. |
| `metadata.name` | Free-form. `cluster-<addon>` reads as "this addon, at cluster scope". | Must be unique in the namespace. If you manage several clusters from one vSphere Namespace, `cluster-istio` collides — prefix with the cluster name. **This name is *not* what the `AddonConfig` matches** — `spec.addonRef.name` is. |
| `spec.addonRef.name` | Selects the addon. | No version field here; the platform resolves a release. To pin, name the `acd` explicitly in the `AddonConfig` ([section 4](#the-name-convention-is-a-defaulting-mechanism)). |
| `spec.clusters[].selector` | `cluster.x-k8s.io/cluster-name` is applied to every `Cluster` automatically by Cluster API, so it always works with no extra labelling. | **Selector breadth is a policy decision.** Matching on cluster name targets one cluster; a broader selector (`environment: dev`) is a fleet-rollout tool — and a foot-gun if you thought you were configuring one cluster. |

Likewise, every `AddonConfig` carries this annotation:

```yaml
  annotations:
    clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"
```

**Set this on every `AddonConfig`.** It ties the config's lifecycle to the addon and cluster so it
is garbage-collected on teardown. Without it you leave orphans behind — and because binding is
name-derived, an orphan can be silently adopted by a future cluster that reuses the name,
inheriting configuration nobody remembers writing.

---

### 6.1 helm-controller

```yaml
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
        cluster.x-k8s.io/cluster-name: <CLUSTER_NAME>
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonConfig
metadata:
  annotations:
    clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"
  name: <CLUSTER_NAME>-helm-controller
  namespace: <VSPHERE_NAMESPACE>
spec:
  values:
    helmController:
      priorityClassName: ""
    sourceController:
      priorityClassName: ""
```

> **What this addon is for.** helm-controller (with its companion source-controller) provides
> **declarative Helm chart lifecycle management inside the workload cluster** — a day-2 capability
> for *your own* application charts. You create `HelmRelease` objects; it reconciles them,
> handling upgrades, rollbacks, and drift.
>
> **What it is not.** It does not deliver the other addons in this stack and they do not depend on
> it. Those are Carvel packages reconciled by kapp-controller ([section 4](#how-addons-are-delivered-carvel-packages)).
> Install it because you want GitOps-style Helm management, not because something else needs it.

| Field | Why | Pitfall |
| --- | --- | --- |
| Two controllers | `helmController` reconciles releases; `sourceController` fetches chart artifacts. Both are needed. | If your `HelmRelease` objects stall, check both pods — they run in `vmware-system-helm` in the workload cluster. |
| `priorityClassName: ""` | Empty means **no PriorityClass**, so these pods run at default (zero) priority. | Under node pressure they are as evictable as a batch job. If you are running production workloads through Helm releases, the controller that reconciles them being evicted means drift goes uncorrected. Consider `system-cluster-critical`. |
| Only two fields set | A good illustration that `AddonConfig` is a **sparse overlay** — every other helm-controller setting is at its schema default. | See [section 5](#5-discovering-addons-and-their-value-schemas). |

**Do you need it?** If you are not managing Helm charts declaratively in this cluster, this addon
is optional. It is in the recommended set because most teams eventually want it, not because the
stack requires it.

---

### 6.2 cert-manager

```yaml
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
        cluster.x-k8s.io/cluster-name: <CLUSTER_NAME>
```

**No `AddonConfig`, deliberately.** An addon with no config takes every schema default, which for
cert-manager is the right choice. Do not infer from the other addons that a config is required.

| Consideration | Detail |
| --- | --- |
| **Why it is a real dependency here** | The Headlamp addon uses it. On deployment, Headlamp creates its own cert-manager `Issuer` and `Certificate` and terminates TLS on its Gateway with the resulting secret — so cert-manager is load-bearing for the UI, not decoration. See [6.5](#65-headlamp). |
| What it does more generally | Issues and rotates certificates for admission webhooks, ingress/gateway TLS, and internal service TLS, driven by `Issuer`/`ClusterIssuer` resources. |
| Ordering | None required. If Headlamp reconciles first, its `Certificate` simply stays pending until cert-manager is ready, then resolves. |
| **Production consideration** | The issuer Headlamp creates is **self-signed**, so browsers will warn. For production, add a `ClusterIssuer` backed by a **trusted CA** — your enterprise PKI or ACME — and point the Gateway at a certificate from it. See [Appendix B](#appendix-b--additional-objects-and-hardening). |

---

### 6.3 Prometheus

```yaml
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonInstall
metadata:
  name: cluster-prom                    # ← install name is arbitrary; the ADDON is "prometheus"
  namespace: <VSPHERE_NAMESPACE>
spec:
  addonRef:
    name: prometheus
  clusters:
  - selector:
      matchLabels:
        cluster.x-k8s.io/cluster-name: <CLUSTER_NAME>
---
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonConfig
metadata:
  annotations:
    clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"
  name: <CLUSTER_NAME>-prometheus        # ← "prometheus", NOT "prom" — matches the addon name
  namespace: <VSPHERE_NAMESPACE>
spec:
  values:
    deploycomponents:                    # ← lowercase, hyphenated keys — match exactly
      kube-state-metrics: true           # ← Kubernetes object-state metrics
      node-exporter: true                # ← per-node OS/hardware metrics
      pushgateway: false                 # ← correctly disabled
      prometheus: false                  # ← intentional: you deploy the server via the operator
      alertmanager: true                 # ← alert routing and delivery
      prometheus-operator: true          # ← the CRDs + operator you will drive
```

> **The intended pattern: operator-managed Prometheus.** `prometheus: false` with
> `prometheus-operator: true` is deliberate. Rather than accepting the packaged Prometheus server
> and its baked-in settings, you enable the **operator** and then declare your own `Prometheus`
> custom resource — giving you direct control over replica count, retention, persistent storage,
> scrape configuration, and external labels, all as version-controlled Kubernetes objects.
>
> This is the right choice for anyone who cares about their monitoring configuration. It has one
> requirement: **you must actually create the `Prometheus` CR.** Until you do, the operator and
> CRDs are installed, the exporters are producing metrics, and nothing is scraping or evaluating
> them. Confirm with:
>
> ```bash
> kubectl get prometheus,alertmanager,servicemonitor -A
> # "No resources found" means the operator is idle — see Appendix B.3
> ```

**This is the manifest's most important follow-up action.** The `Prometheus` CR is in
[Appendix B.3](#b3-a-prometheus-instance-via-the-operator).

| Component | Value | Role | Guidance |
| --- | --- | --- | --- |
| `kube-state-metrics` | `true` | Exports the state of Kubernetes objects — deployment replica counts, pod phases, PVC status. | Keep enabled. There is no substitute for cluster-state signals. |
| `node-exporter` | `true` | Per-node OS and hardware metrics: CPU, memory, disk, filesystem fill. | Keep enabled. Runs as a DaemonSet, so it needs elevated host access — relevant to your Pod Security choice, and a good candidate for a **namespace exemption** rather than a cluster-wide relaxation ([7.7.3](#773-securitypodsecuritystandard)). |
| `pushgateway` | `false` | Accepts pushed metrics from short-lived batch jobs. | Correctly disabled. Pushgateway retains metrics indefinitely, becoming a source of stale data. Enable only for genuine batch use cases. |
| `prometheus` | `false` | The packaged Prometheus server. | Disabled on purpose — see the callout above. |
| `alertmanager` | `true` | Routes, groups, deduplicates, and delivers alerts. | Needs two things to be useful: a rule evaluator (your `Prometheus` CR) and **configured receivers**. An Alertmanager with no receivers delivers nowhere. |
| `prometheus-operator` | `true` | Installs the CRDs (`Prometheus`, `ServiceMonitor`, `PodMonitor`, `PrometheusRule`, `Alertmanager`) and the reconciling operator. | The foundation of this pattern. Also lets application teams declare their own scrape targets declaratively. |

**Watch the key spelling.** `deploycomponents` is lowercase and the component keys are hyphenated
(`kube-state-metrics`, not `kubeStateMetrics`). Keys are matched literally; a "corrected" camelCase
key is not applied. Verify against the schema output.

**Note the naming.** The `AddonInstall` is `cluster-prom`, but the addon is `prometheus`, so the
`AddonConfig` must be `<CLUSTER_NAME>-prometheus`. Naming it `-prom` would match nothing.

---

### 6.4 Istio — deployed for L4/L7, not as a service mesh

> **Scope matters here, so start with it.** In this stack Istio is deployed for one purpose:
> **L4 and L7 traffic management** — the ingress gateway and the Gateway API controller that
> exposes services outside the cluster, including the Headlamp UI. It is **not** being used as a
> service mesh. No sidecars are injected, no mTLS is established between workloads, and no
> east-west traffic policy is applied.
>
> That is a legitimate and increasingly common way to run Istio, and it keeps the footprint small.
> But it changes which settings matter and which are inert, so it is worth stating explicitly
> rather than leaving readers to infer a mesh they are not getting.
>
> Verify the scope on any cluster with:
> ```bash
> kubectl get ns -L istio-injection,istio.io/dataplane-mode
> # every namespace blank in both columns → gateway-only, no mesh dataplane
> ```

```yaml
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonConfig
metadata:
  annotations:
    clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"
  name: <CLUSTER_NAME>-istio
  namespace: <VSPHERE_NAMESPACE>
spec:
  values:
    istio:
      namespace: "istio-system"        # ← GUEST cluster namespace; addon creates and owns it
      ambientMode:
        enabled: false                 # ← no ambient dataplane (not needed for gateway-only)
      istioCNI:
        enabled: false                 # ← not needed without sidecar injection; see below
      gateways:
        ingress:
          enabled: true                # ← THE point of this addon: the L4/L7 entry point
          namespace: istio-ingress     # ← separate from istio-system: good practice
          autoscaling:
            enabled: true
            minReplicas: 1             # ← single ingress pod = outage on reschedule
            maxReplicas: 5
      pilot:
        replicas: 1                    # ← conflicts with the HPA below
        autoscaling:
          enabled: true
          minReplicas: 1               # ← istiod is the Gateway controller: SPOF
          maxReplicas: 2
      meshConfig:
        accessLogFile: ""              # ← gateway access logging DISABLED
        enableTracing: false
        meshID: "<CLUSTER_NAME>"
```

#### A note on `istioCNI` and Pod Security

There is a widely cited coupling between Istio and Pod Security Standards, and it is worth being
precise about it because **it does not apply to this deployment.**

The coupling is real **only when sidecar injection is enabled**. In that case, if Istio CNI is
off, traffic redirection is performed by an init container in every injected pod, and that init
container needs `NET_ADMIN`/`NET_RAW` — capabilities that the `baseline` and `restricted` Pod
Security Standards both forbid. Enabling `istioCNI` moves that work to a per-node DaemonSet so
workload pods need no elevated capabilities.

**This cluster injects no sidecars**, so no workload pod requires those capabilities and
`istioCNI.enabled: false` imposes no constraint on your Pod Security Standard. The
`podSecurityStandard: privileged` setting elsewhere in the manifest is a **dev/test reference
choice**, not a consequence of this Istio configuration — see
[7.7.3](#773-securitypodsecuritystandard).

| Your Istio scope | `istioCNI` | Pod Security impact |
| --- | --- | --- |
| **Gateway/ingress only** (this stack) | `false` is fine | **None.** Choose your PSS level freely. |
| Sidecar mesh | `false` | Injected pods need `NET_ADMIN`/`NET_RAW` → `privileged` required |
| Sidecar mesh | `true` | No elevated capabilities in workload pods → `baseline`/`restricted` viable |
| Ambient mesh | n/a | Per-node ztunnel; no per-pod privilege |

**If you later adopt the mesh**, enable `istioCNI` at the same time. Retrofitting it after
labelling namespaces for injection means every injected workload needs `privileged` in the interim.

#### Istio field reference

| Field | Value | Why | Pitfall |
| --- | --- | --- | --- |
| `istio.namespace` | `istio-system` | Control-plane namespace **inside the workload cluster**. Conventional and expected by tooling. | The addon creates and owns it. Pre-creating it by hand, or managing it with another tool, causes ownership conflicts. |
| `ambientMode.enabled` | `false` | Ambient's per-node ztunnel dataplane is a mesh feature. With no mesh, there is nothing for it to do. | Correct for this scope. If you adopt ambient later, treat it as a migration — it changes how every packet is handled — not a toggle. |
| `istioCNI.enabled` | `false` | Only needed for sidecar injection. | See the note above. Inert at this scope. |
| `gateways.ingress.enabled` | `true` | **The reason this addon is here.** Deploys the ingress gateway and its LoadBalancer Service — the cluster's north-south entry point, and the Istio Gateway API controller that programs `Gateway` resources. | The LoadBalancer needs an available VIP. If `EXTERNAL-IP` stays `<pending>`, the problem is your load-balancer provider or IP pool, not Istio. |
| `gateways.ingress.namespace` | `istio-ingress` | **Good practice.** Separating the data-plane gateway from the control plane limits blast radius and lets you grant teams access to gateway resources without control-plane access. | Note that a Gateway API `Gateway` with `create: true` provisions its **own** LoadBalancer in its **own** namespace — it does not share this one. See [6.5](#65-headlamp). |
| `gateways.ingress.autoscaling` | `1`–`5` | Scales ingress capacity with traffic. | **`minReplicas: 1` is a production risk.** During any reschedule — node drain, upgrade, eviction — you have zero ingress pods and all inbound traffic fails. Use `2` minimum, with a `PodDisruptionBudget` and anti-affinity across nodes. |
| `pilot.replicas` | `1` | Static replica count for istiod. | **Conflicts with the HPA immediately below it.** When an HPA owns a Deployment it owns `spec.replicas`; a static value is not authoritative and invites reconcile churn. Express intent through `minReplicas` and drop this field. |
| `pilot.autoscaling` | `1`–`2` | istiod is the Gateway API controller and configures the gateway proxies. | **A single istiod is a single point of failure for all ingress configuration.** Existing gateways keep forwarding on last-known config, so it is not an instant outage — but no `Gateway` or `HTTPRoute` change takes effect while it is down, and that includes its own upgrade. Use `minReplicas: 2`. |
| `meshConfig.accessLogFile` | `""` | Empty **disables** Envoy access logs. Saves CPU and log volume. | **At gateway-only scope these are your ingress access logs** — the per-request record of what arrived, which route matched, and what status was returned. That is the primary tool for debugging a 404 from a misconfigured `HTTPRoute` or a failing OIDC redirect. Enable it (`/dev/stdout`) during bring-up and while validating any auth path. |
| `meshConfig.enableTracing` | `false` | Distributed tracing off. | Of limited value without a mesh — you would only see gateway spans. Setting `true` also requires configuring a tracing provider, not just this flag. |
| `meshConfig.meshID` | `<CLUSTER_NAME>` | Identifies the mesh; harmless and tidy in telemetry. | Largely inert at gateway-only scope. It matters if you later federate multiple clusters into one mesh — that requires a **shared** `meshID` with distinct `network` values, so a cluster-derived value would need changing. If mesh federation is on your roadmap, name the mesh (`prod-mesh`) rather than the cluster. |

> **Alternative: Contour.** If all you need is HTTP ingress and you are not planning to adopt a
> mesh, the **`contour`** addon is a lighter-weight ingress controller with a smaller footprint
> and less operational surface. Istio makes sense if you want Gateway API with a path to a mesh,
> richer L4/L7 policy, or mTLS later. Contour makes sense if you want ingress and nothing more.

---

### 6.5 Headlamp

A web UI for the cluster, exposed through a Gateway API `Gateway` and authenticated with OIDC.

```yaml
apiVersion: addons.kubernetes.vmware.com/v1alpha1
kind: AddonConfig
metadata:
  annotations:
    clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"
  name: <CLUSTER_NAME>-headlamp
  namespace: <VSPHERE_NAMESPACE>
spec:
  values:
    hostname: headlamp.k8s.example.com     # ← a real FQDN; must resolve to the Gateway's IP
    gatewayApi:
      enabled: true                        # ← route via Gateway API
      gateway:
        className: istio                   # ← REQUIRES a gateway controller: Istio or Contour
        create: true                       # ← the addon owns this Gateway (and its LoadBalancer)
        name: headlamp-gateway
    oidc:
      enabled: true
      issuerURL: https://idp.example.com   # ← must match the token's `iss` claim exactly
      # This client ID must also appear in the API server's extraAuthentication
      # audiences list, or tokens issued here are rejected by the cluster.
      clientID: <OIDC_CLIENT_ID>
      clientSecret: <OIDC_CLIENT_SECRET>   # ← plaintext in etcd and in Git
      callbackURL: https://headlamp.k8s.example.com/oidc-callback
      scopes:
      - openid                             # ← mandatory for OIDC
      - email                              # ← load-bearing: feeds the username claim
      - profile                            # ← convenience only
```

> **The one hard prerequisite: a gateway controller.**
>
> `gatewayApi.gateway.className` must name a controller that exists in the cluster. That means
> **you must install either the `istio` addon (with `gateways.ingress.enabled: true`) or the
> `contour` addon** before Headlamp can be reached. Without a matching controller the `Gateway`
> object is created and stays unprogrammed forever — Headlamp runs, but nothing routes to it.
>
> This is not an ordering requirement (the addons reconcile independently and it will resolve
> itself once the controller appears) — it is a **completeness** requirement. Confirm with:
> ```bash
> kubectl get gateway -A
> # PROGRAMMED must be True and ADDRESS must be populated
> ```
>
> The Gateway API CRDs themselves are installed by the platform — you do not add them
> ([section 3](#3-what-vks-installs-for-you)).

#### The hostname, DNS, and the OIDC redirect must agree

This is the part that most often costs an afternoon, because the three values live in three
different systems and a mismatch surfaces as an error from your identity provider rather than from
Kubernetes.

With `gateway.create: true`, the addon provisions its **own** `Gateway` and a dedicated
LoadBalancer Service in the Headlamp namespace — separate from the Istio ingress gateway, with its
**own external IP**:

```bash
kubectl get gateway -A
# NAME               CLASS   ADDRESS          PROGRAMMED
# headlamp-gateway   istio   10.x.x.x         True        ← this is the IP DNS must point to

kubectl get svc -n headlamp
# headlamp-gateway-istio   LoadBalancer   ...   10.x.x.x   443:31596/TCP
```

**Use a real FQDN you publish in DNS.** This document uses `headlamp.k8s.example.com` throughout,
and there are three things to line up.

> **A note if your starting manifest uses a wildcard-DNS shortcut.** Lab manifests sometimes set the
> hostname to an IP-derived name — the `<ip>.sslip.io` and `<ip>.nip.io` services resolve
> `anything.<ip>.sslip.io` to that IP, which is genuinely handy when you control no DNS. Replace it
> before production: it hardwires the LoadBalancer IP into the hostname, the `callbackURL`, and your
> IdP's redirect-URI registration simultaneously, and it makes name resolution for your management UI
> depend on a third-party service.

**The three-step setup:**

| Step | Action |
| --- | --- |
| 1 | Choose the FQDN (`headlamp.k8s.example.com`) and set it as both `hostname` and the host in `callbackURL`. |
| 2 | **Publish a DNS A record** for that FQDN pointing at the Gateway's `ADDRESS`. Reserve the IP with your load-balancer provider so it is stable. |
| 3 | **Register `https://<FQDN>/oidc-callback` as an authorized redirect URI** with your identity provider. Most providers reject unregistered redirects outright. |

Do all three and you can apply the whole manifest at once, because nothing depends on discovering
an IP after the fact.

#### Headlamp field reference

| Field | Why | Pitfall |
| --- | --- | --- |
| `hostname` | The FQDN the UI is served on, and the `Gateway` listener's hostname. | Must match the `callbackURL` host and a published DNS record. An IP-derived hostname breaks whenever the IP changes — taking DNS, the callback, and the IdP registration with it. |
| `gatewayApi.enabled` | Routes via Gateway API (`Gateway`/`HTTPRoute`) rather than the legacy `Ingress` resource. The correct modern choice. | CRDs are platform-provided, so nothing to install — but a controller is still required. |
| `gatewayApi.gateway.className` | Binds the Gateway to a controller: `istio`, or the equivalent class for Contour. | See the prerequisite callout. A class with no controller stays unprogrammed silently. |
| `gatewayApi.gateway.create` | `true` — the addon creates and manages the `Gateway`, including its LoadBalancer. | The addon **owns** the object; hand-edits are reverted on the next reconcile. To customise it (a different TLS certificate, extra listeners), do so through `AddonConfig` values, or set `create: false` and manage the `Gateway` yourself. |
| **TLS** | The addon creates a cert-manager `Issuer` and `Certificate` and terminates TLS on the Gateway listener automatically — you get HTTPS with no configuration. | **The issuer is self-signed**, so browsers show a warning and strict clients may refuse the OIDC redirect. For production, issue from a **trusted CA** — see [Appendix B.2](#b2-trusted-tls-for-the-headlamp-gateway). Verify what you have: `kubectl get certificate,issuer -n headlamp`. |
| `oidc.issuerURL` | The OIDC issuer, used for discovery and to validate the `iss` claim. | **Must match byte-for-byte, including any trailing slash**, and must equal the API server's `extraAuthentication.jwt[].issuer.url`. |
| `oidc.clientID` | The OIDC client that mints ID tokens for UI logins. | **The most important cross-reference in the manifest.** It must also appear in the API server's `audiences` list, or the API server rejects the tokens Headlamp obtains: users log in successfully and then hit permission errors on every call. |
| `oidc.clientSecret` | The client secret for the authorization-code flow. | **Plaintext** in etcd and in any repository holding the manifest. See [Pitfall 3](#3-oidc-client-secret-in-plaintext). |
| `oidc.callbackURL` | The post-authentication redirect target. | Must be registered with the IdP, and its host must equal `hostname`. Failures appear on the provider's error page, not in Kubernetes. |
| `oidc.scopes` | `openid` is required by the spec. `email` requests the email claim. `profile` requests name and picture. | **`email` is load-bearing.** The API server maps the `email` claim to the Kubernetes username. Drop the scope and the claim is absent, so username mapping fails — login appears to work while authorization fails everywhere. If you change the API server's `username.claim`, request the matching scope here. |

> **Authentication is not authorization — you must also create RBAC.**
>
> Configuring OIDC proves *who* a user is. It grants them **nothing**. A user who logs into
> Headlamp successfully with no RBAC binding sees permission errors everywhere and will report the
> UI as broken.
>
> You need at least one RBAC binding **inside the workload cluster** naming the fully composed
> username — the `prefix` from the API server's `claimMappings.username.prefix` concatenated with
> the claim value:
>
> ```yaml
> # applied INSIDE the workload cluster, not the vSphere Namespace
> apiVersion: rbac.authorization.k8s.io/v1
> kind: ClusterRoleBinding
> metadata:
>   name: oidc-admin
> subjects:
> - kind: User
>   name: "oidc:user@example.com"      # ← prefix + email claim, exactly
>   apiGroup: rbac.authorization.k8s.io
> roleRef:
>   kind: ClusterRole
>   name: cluster-admin                # ← prefer a least-privilege role
>   apiGroup: rbac.authorization.k8s.io
> ```
>
> Confirm the exact string the API server sees with `kubectl auth whoami`. Full guidance, including
> group-based bindings and least privilege, is in [Appendix B.1](#b1-rbac-for-oidc-identities).

---

### 6.6 Additional addons worth considering

The five above are a good baseline. The catalogue is larger — list yours with
`vcf addon available list`. The additions below are the ones most often missing from a stack that
is otherwise production-ready.

The catalogue is larger than the five above — list yours with `vcf addon available list`. Nothing
here is a blanket recommendation: which of these belongs in your cluster depends entirely on what
you already run elsewhere.

Three of them — backup, log forwarding, and DNS automation — are frequently valuable but need
environment-specific decisions before they do anything useful, so they are written up separately in
[Appendix D](#appendix-d--optional-integrations-requiring-environment-specific-setup).

#### Ingress and load balancing

| Addon | When to choose it |
| --- | --- |
| **`contour`** | A focused HTTP ingress controller. Lighter than Istio if you want ingress and nothing more. Also a valid `className` provider for the Headlamp Gateway. |
| **`ako`** | Integrates NSX Advanced Load Balancer (Avi) for L4–L7. The right choice if NSX ALB is already your standard — enterprise LB features, WAF, and GSLB rather than a basic VIP. Requires NSX ALB. |

#### Networking extensions

| Addon | Use case |
| --- | --- |
| **`multus-cni`** | Attaches multiple network interfaces to a pod. A standard requirement for NFV, telco, and workloads needing a data plane separate from cluster traffic. |
| **`whereabouts`** | Cluster-wide IPAM for secondary interfaces. Effectively required alongside `multus-cni` — without it, secondary-interface addressing does not coordinate across nodes. |
| **`sriov-network-device-plugin`** | Exposes SR-IOV virtual functions as schedulable resources for high-throughput, low-latency workloads. |

#### Storage, secrets, registry, and specialised

| Addon | Use case |
| --- | --- |
| **`nfs-client`** | An NFS CSI driver. Useful when you need `ReadWriteMany` volumes, which vSphere block storage does not provide. |
| **`vault-injector`** | Injects secrets from HashiCorp Vault into pods. A stronger answer to secret handling than Kubernetes Secrets alone, and directly relevant to the `clientSecret` issue in [Pitfall 3](#3-oidc-client-secret-in-plaintext). Requires an existing Vault deployment. |
| **`harbor`** | An in-cluster OCI registry with image scanning, signing, and replication. Worth considering if you want a supply-chain control point inside the cluster; needs storage capacity and its own lifecycle. |
| **`telegraf`** | Metric collection and forwarding. Complements Prometheus when you need to push to an external TSDB such as InfluxDB. Requires an output target. |
| **`gatekeeper`** / **`policy-bundle`** | OPA Gatekeeper for admission policy, and VMware's curated compliance policies. Relevant if you have a specific compliance target; otherwise Pod Security Standards already cover pod privilege. |
| **`windows-gmsa-webhook`** | Group Managed Service Account support for Windows containers. Only if you run Windows nodes. |
| **`vsphere-pv-csi-webhook`** | Additional validation for the vSphere CSI provider. |

> **Every addon is a workload.** Each consumes CPU, memory, and pod slots on every node it
> DaemonSets onto, and each is another component to upgrade and troubleshoot. Add what you will
> operate, not everything available.

---

## 7. The `Cluster` object, annotated

> **What this covers.** Every field of the `Cluster` object — networking, node shape, storage,
> certificates, etcd, Pod Security, OIDC, CNI, and the node pool.
>
> **Why it matters.** This is the densest and most consequential part of the manifest. It holds the
> decisions you cannot reverse, the settings with the widest blast radius, and the one outright logic
> error in the profile. If you read one section closely, read this one.

### 7.1 Metadata and cluster networking

```yaml
apiVersion: cluster.x-k8s.io/v1beta2      # ← upstream Cluster API, not the VKS addon group
kind: Cluster
metadata:
  name: <CLUSTER_NAME>                    # ← must match every addon selector above
  namespace: <VSPHERE_NAMESPACE>          # ← same vSphere Namespace as the addons
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
      - 240.0.0.0/20                      # ← reserved "Class E" space; see 7.1.1
    serviceDomain: cluster.local
    services:
      cidrBlocks:
      - 240.1.0.0/20
```

| Field | Why | Pitfall |
| --- | --- | --- |
| `apiVersion` | Upstream Cluster API. VKS builds on standard CAPI, so upstream concepts and troubleshooting apply directly. | Do not confuse this group with `addons.kubernetes.vmware.com` used above. |
| `metadata.name` | The cluster's identity, and the value every `AddonInstall` selector matches. | Renaming is a rebuild, not an operation. |
| `pods.cidrBlocks` | The pool pod addresses are allocated from. | The most consequential number in the manifest — see 7.1.1. |
| `services.cidrBlocks` | The pool for `ClusterIP` Services. `/20` gives ~4,094 usable service IPs. | Must not overlap the pod CIDR — these do not (pods occupy `240.0.0.0`–`240.0.15.255`, services `240.1.0.0`–`240.1.15.255`) — nor anything routable in your datacenter. **Immutable.** |
| `serviceDomain` | The internal DNS suffix. `cluster.local` is the universal default. | Changing it breaks anything assuming `.cluster.local`, and would need to be matched by Istio's trust domain if you ever adopt the mesh. Leave it alone. |

> **Why `240.0.0.0/4`?** That range is IANA-reserved ("Class E") and not routable on the public
> internet or, normally, inside a datacenter. Using it for pod and service networks **guarantees no
> collision with routable RFC1918 space** (`10/8`, `172.16/12`, `192.168/16`) elsewhere in your
> environment — which matters when pods reach on-premises systems, and when you may later peer or
> federate clusters. Overlapping CIDRs are among the hardest problems to unpick after the fact, so
> this is genuinely good practice.
>
> **Validate it before adopting it.** Because the range is reserved, some CNIs, OS network stacks,
> hardware load balancers, and firewalls reject or mishandle Class E addresses. It works with
> Antrea; confirm it against every appliance in the path.

### 7.1.1 Pod CIDR sizing: node blocks, `maxPods`, and the upgrade spare

> **`clusterNetwork.pods.cidrBlocks` is immutable after cluster creation.** Get it wrong and the
> remedy is a new cluster. Read this before accepting `240.0.0.0/20`.

#### Allocation happens at two levels

The second level is what creates the limit, and it is the part that surprises people.

1. **Cluster level.** `pods.cidrBlocks` defines one pool for the whole cluster.
2. **Node level.** As each node joins, the node IPAM controller carves a **fixed-size block out of
   that pool and assigns it to that node** (`Node.spec.podCIDR`). The block is **dedicated** to
   that node — pods elsewhere cannot draw from it, even when it sits mostly empty. The CNI then
   allocates pod IPs only from the block its own node owns.

The per-node block size defaults to **`/24`** (256 addresses) for IPv4. So the cluster pool is not
a flat pool of pod IPs — it is a **pool of per-node `/24` blocks**. Run out of blocks and adding
nodes stops working.

```
240.0.0.0/20  →  4096 addresses  →  2^(24−20) = 16 blocks of /24
```

**16 blocks is your total node budget — control plane and workers combined.** Not 16 workers.

You can see the allocation on any running cluster:

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,POD_CIDR:.spec.podCIDR,MAXPODS:.status.capacity.pods'
```

```
NODE                                POD_CIDR       MAXPODS
<cluster>-8dbnf-tf6pb               240.0.0.0/24   110      ← control plane: block 1
<cluster>-node-pool-1-...-m9b9p     240.0.1.0/24   110      ← worker:        block 2
```

#### Reserve at least one spare block for upgrades

Every rolling operation in Cluster API is **surge-then-remove**: a replacement node is created and
brought to `Ready` *before* the node it replaces is deleted. A `MachineDeployment` rolling update
defaults to `maxSurge: 1`, and a control-plane rollout likewise scales up before scaling down. For
the duration of that overlap the cluster runs **N + 1 nodes**, and that extra node needs a block of
its own.

**If every block is allocated, the surge node has no pod CIDR to receive.** It joins, may report
`Ready`, and hosts nothing — pods scheduled to it stay `Pending`. The rollout neither completes nor
cleanly fails; it stalls with the cluster in a mixed-version state.

> **A cluster that has consumed its entire pod CIDR cannot be upgraded — and because the pod CIDR
> is immutable, it cannot be fixed in place either.** Running to the block limit paints you into a
> corner you can only leave by rebuilding.

This applies to **every** rolling node replacement, not just version upgrades: a `vmClass` change,
a `volumes` resize, and an OS image change all need the spare block
([which changes replace nodes](#which-changes-replace-nodes)).

| Situation | Spare blocks to reserve |
| --- | --- |
| Minimum, ever | **1** |
| Multiple machine deployments that may roll concurrently | 1 per pool rolling at once |
| A rollout strategy with `maxSurge` above 1 | 1 per surge slot |
| **Recommended working margin** | **2–3**, plus blocks for pools you have not created yet |

A second reason not to run to the edge: **block reclamation is not instantaneous.** A deleted
node's block is released when its `Node` object is removed, so a node stuck `Terminating` holds its
block meanwhile. At zero margin, one stuck node blocks the whole rollout.

#### `maxPods` — new in ClusterClass 3.7, and the lever for pod density

The per-node block also bounds pods per node, but a kubelet setting usually binds first. **VKS 3.7
exposes it as a ClusterClass variable**, which earlier generations did not:

```yaml
        kubeletConfiguration:
          maxPods: 250        # default 110; documented maximum 250; minimum 20
```

Verify the constraints for your ClusterClass directly:

```bash
# jsonpath renders string fields with real newlines, so this reads cleanly.
# Keep only the path in the variable — never flags, which would rely on word
# splitting and breaks under zsh.
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

**Why this matters against the CIDR arithmetic:**

| `maxPods` | Pods per node | Usable IPs in a `/24` | Which limit binds | Block utilisation |
| --- | --- | --- | --- | --- |
| `110` (default) | 110 | ~254 | **kubelet** | ~43% — over half the block is never used |
| `250` (maximum) | 250 | ~254 | **the `/24` block** — essentially exactly matched | ~98% |

This gives you a genuine lever: **raising `maxPods` increases pod capacity without consuming
additional blocks.** On a cluster already constrained to a `/20`, going from `110` to `250` takes
the ceiling from ~1,650 pods to ~3,750 — with no change to the immutable CIDR.

It is not free:

| Consideration | Detail |
| --- | --- |
| **Node resources must support it** | 250 pods on a 2-vCPU / 8 GiB node is not realistic. Size the VM class to the pod count — and remember each pod carries kubelet, CRI, and networking overhead beyond its own requests. |
| **Reserve resources for the system** | At high pod density, kubelet and runtime overhead grows materially — and VKS's automatic reservation is derived from **node size, not pod count**, so it does not grow with `maxPods`. Override `resourceConfiguration.systemReserved` explicitly; [7.13](#713-37-variables-not-used-in-the-sample) has the published formulas and a per-VM-class table. |
| **Blast radius** | 250 pods per node means losing a node evicts 250 pods at once. Fewer, denser nodes concentrate risk; more, smaller nodes spread it — at one block each. |
| **DaemonSet cost is per node, not per pod** | Denser nodes mean fewer DaemonSet copies, which is a real efficiency gain when you run several. |
| **Rollout time** | Draining a 250-pod node takes considerably longer than a 110-pod node. Upgrades get slower. |
| **`/24` becomes the binding limit at 250** | With ~254 usable addresses there is almost no headroom. Pod IP churn from short-lived pods can transiently exhaust a node's block. Do not plan to exceed ~220–230 sustained per node on a `/24`. |
| **Changing it later** | `kubeletConfiguration` changes **require a machine rollout** — which needs a spare block. |

#### Sizing table

Assumes default `/24` per-node blocks and one reserved surge block.

| Pod CIDR | Addresses | `/24` blocks | Steady-state nodes | Pod ceiling @ `maxPods 110` | @ `maxPods 250` |
| --- | --- | --- | --- | --- | --- |
| `/22` | 1,024 | 4 | 3 | 330 | 750 |
| `/21` | 2,048 | 8 | 7 | 770 | 1,750 |
| **`/20`** | **4,096** | **16** | **15** | **1,650** | **3,750** |
| `/19` | 8,192 | 32 | 31 | 3,410 | 7,750 |
| `/18` | 16,384 | 64 | 63 | 6,930 | 15,750 |
| `/17` | 32,768 | 128 | 127 | 13,970 | 31,750 |
| **`/16`** | **65,536** | **256** | **255** | **28,050** | **63,750** |

#### What a `/20` means for real cluster shapes

| Cluster shape | Nodes | Blocks used | Left over on a `/20` | Verdict |
| --- | --- | --- | --- | --- |
| **The sample:** 1 CP + 5 workers (autoscaler max) | 6 | 6 + 1 surge = 7 | 9 | Comfortable |
| **Production shape:** 3 CP + 10 workers | 13 | 13 + 1 surge = 14 | 2 | **Too tight** — one extra pool or one stuck node and you are stuck |
| Same, plus a 5-node GPU pool | 18 | 18 + 1 = 19 | — | **Does not fit** |

The sample is fine as written. The trap is that it looks like a template, and the production shape
it should grow into **does not fit a `/20`** with real margin — while the field you would need to
change is the one you no longer can.

#### What else eats the per-node budget

- **DaemonSets consume a slot on every node, permanently.** This stack runs at least the CNI agent
  and node-exporter everywhere, so a few of each node's slots are gone before an application pod
  lands. Every DaemonSet you add costs one slot per node forever.
- **Short-lived pods hold IPs briefly after termination.** High-churn CI runners, Jobs, and
  CronJobs can pressure a node's block well above its steady-state pod count.
- **Sidecars cost containers, not pods or IPs.** If you later adopt the Istio mesh, sidecars share
  their pod's IP and consume no extra slot. Service mesh will not exhaust your CIDR.

#### Recommendation

**Use a `/16` unless you have a specific reason not to.** Drawn from reserved `240.0.0.0/4` space,
address scarcity is not a real constraint — these addresses are not routable outside the cluster
and compete with nothing. A `/16` gives 255 nodes and removes the entire class of problem at no
cost.

If you must use a smaller block:

1. Size for **maximum plausible node count**, including future pools for GPU or memory-optimised
   workloads.
2. **Add the surge reserve on top** — 2–3 blocks, not the bare minimum of 1.
3. **Cross-check against every `autoscaler-node-group-max-size` plus `controlPlane.replicas`.**
4. **Consider raising `maxPods`** to buy pod density without buying blocks — sizing the VM class and
   `systemReserved` accordingly ([7.13](#713-37-variables-not-used-in-the-sample)).
5. **Alert on block allocation**, not just on scheduling failures. "12 of 16 blocks used" is
   actionable; discovering it during a stalled upgrade is not.

#### If you are already on a `/20`

- Compute the real ceiling: `16 − controlPlane.replicas − surge reserve` = maximum workers, summed
  across all pools. Cap every autoscaler `max-size` at or below it.
- Audit before any rolling operation:
  ```bash
  # how many blocks are in use
  kubectl get nodes -o jsonpath='{range .items[*]}{.spec.podCIDR}{"\n"}{end}' | sort -u | wc -l
  ```
  **Confirm at least one free block before starting.**
- Prefer fewer, larger nodes with a higher `maxPods`. Vertical scaling costs no blocks; horizontal
  scaling costs one each. Note that changing `vmClass` or `maxPods` is itself a rolling
  replacement, so it needs the spare block too.

### 7.2 Topology and ClusterClass

```yaml
  topology:
    classRef:
      name: builtin-generic-v3.7.0            # ← VMware-supplied ClusterClass
      namespace: vmware-system-vks-public     # ← VMware-managed namespace
```

| Field | Why | Pitfall |
| --- | --- | --- |
| `classRef.name` | The ClusterClass is the template that turns a short `Cluster` spec into full infrastructure. Version-paired with the VKS release. | **The ClusterClass defines which `topology.variables` exist.** Every variable below is valid only because this class declares it — the set is not portable across ClusterClass versions. |
| `classRef.namespace` | `vmware-system-vks-public` holds VMware-provided, read-only ClusterClasses. | Do not modify objects in `vmware-system-*` namespaces. They are platform-managed; changes are reverted and can block upgrades. |

**Inspect your own ClusterClass rather than copying variables hopefully.** The variable set is
published in `status.variables`:

```bash
# which variables exist on this ClusterClass
kubectl get clusterclass builtin-generic-v3.7.0 -n vmware-system-vks-public \
  -o jsonpath='{range .status.variables[*]}{.name}{"\n"}{end}'
```

For `builtin-generic-v3.7.0` that returns ten:

```
bootstrapAddons   kubeAPIServerFQDNs   kubernetes   node   osConfiguration
resourceConfiguration   storageClass   vmClass   volumes   vsphereOptions
```

**The sample manifest uses seven of them.** `node`, `resourceConfiguration`, and
`kubeAPIServerFQDNs` are unused — see [7.13](#713-37-variables-not-used-in-the-sample). Note that
`kubeAPIServerFQDNs` is marked deprecated in the schema in favour of `kubernetes.endpointFQDNs`.

To read the full schema for one variable, including descriptions, defaults, and constraints:

```bash
# the whole schema for one variable (add `| jq` or `| yq` if you want it indented)
kubectl get clusterclass builtin-generic-v3.7.0 -n vmware-system-vks-public \
  -o jsonpath='{.status.variables[?(@.name=="kubernetes")].definitions[0].schema.openAPIV3Schema}' | less

# or read the descriptions field by field, which is usually what you want
kubectl explain cluster.spec.topology.variables
```

> **The upgrade implication.** Because the ClusterClass is version-pinned in the manifest, moving to
> a newer VKS generation is a deliberate act, and the new class may add, rename, or remove
> variables. **Diff the two variable schemas before the upgrade**, not after a failed apply.

### 7.3 Control plane

```yaml
    controlPlane:
      metadata:
        annotations:
          run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
      replicas: 1                          # ← dev/test only: no HA, no etcd quorum
      variables:
        overrides:                         # ← per-pool override of a cluster-wide variable
        - name: vmClass
          value: guaranteed-large          # ← larger AND reserved: correct for a control plane
```

| Field | Why | Pitfall |
| --- | --- | --- |
| `annotations[resolve-os-image]` | Selects the node OS image by attribute rather than a brittle image name. Ubuntu 24.04 is a current LTS with a long support horizon. **This is a label selector against `OSImage` objects, so you can test it in advance** — see [Appendix E.3](#e3-specosimages-and-the-osimage-object). | **Two traps.** (1) The selector must resolve to an image available *for your chosen release*, or machine creation stalls with a thin error surface. One command confirms it: `kubectl get osimage -l os-name=ubuntu,os-version=24.04,run.tanzu.vmware.com/tkr=<kr-name>` — expect exactly one match. (2) It must be set on the control plane **and every machine deployment**, or pools drift onto different OS images. |
| `replicas: 1` | One control-plane node halves a lab's footprint. | **No high availability and no etcd quorum.** A single etcd member means any control-plane failure — or any rolling operation touching it — is a full API-server outage. Workloads already scheduled keep running (the dataplane survives), but nothing can be scheduled, scaled, or changed, and every controller stops reconciling. **Use 3 in production.** Scaling 1→3 later is supported but is a rolling change to plan into a window. |
| `variables.overrides` | **A pattern worth learning.** Overrides a cluster-wide variable for just the control plane, so you can size it independently without duplicating the whole variable set. | An override silently shadows the cluster-wide value. When a node has unexpected resources, check for overrides here before assuming the cluster-wide variable applies. |
| `vmClass: guaranteed-large` | **The right instinct, correctly executed.** The control plane runs etcd, which is latency-sensitive and intolerant of resource starvation — so it gets both a larger class *and* a reserved one. | `guaranteed-large` is 4 vCPU / 16 GiB in a default catalogue. That is a sound production control-plane size for small-to-medium clusters; scale up for high object counts or many nodes. |

### 7.4 `storageClass`

```yaml
    variables:
    - name: storageClass
      value: <STORAGE_CLASS>                 # ← for node/system disks
```

The StorageClass backing the cluster's own machine disks.

| Consideration | Detail |
| --- | --- |
| Why it matters | Provisions the disks the nodes run on. Wrong or unavailable, and machines never come up. |
| **Pitfall — namespace association** | A StorageClass that exists in vCenter but is **not associated with your vSphere Namespace** cannot be used. The `Cluster` object is accepted regardless; the failure appears later as machines that never provision. Verify with `kubectl describe ns <VSPHERE_NAMESPACE> \| grep storageclass` ([section 1](#1-prerequisites)). |
| **Pitfall — protection-policy host count** | A RAID-5 or RAID-6 erasure-coding policy requires enough hosts to satisfy placement. On a smaller cluster the policy is non-compliant and provisioning either fails or silently produces objects that do not meet the stated protection level — you believe you have RAID-5 protection when you do not. Check compliance in vCenter after deployment. |

### 7.5 `vmClass` — use `guaranteed-*` for production

```yaml
    - name: vmClass
      value: guaranteed-medium               # ← cluster-wide default (workers)
```

The VM class sets each node's CPU and memory shape and — critically — whether those resources are
**reserved**.

| Class family | Behaviour | Use for |
| --- | --- | --- |
| **`best-effort-*`** | Requests resources with **no reservation**. Under vSphere contention the VM is not guaranteed the CPU and memory it advertises to Kubernetes. | **Dev and test.** Higher consolidation ratio, lower cost, and unpredictable performance is acceptable. |
| **`guaranteed-*`** | **Reserves** the resources. The VM always has what it claims. | **Production. Recommended.** Also always for the control plane, at any tier. |

> **Why `best-effort` in production fails in a misleading way.** Kubernetes still believes the node
> has its full allocatable capacity and schedules accordingly, while the hypervisor quietly delivers
> less. The symptoms — `NotReady` kubelets, etcd leader-election churn, failing liveness probes,
> random request timeouts — read as "Kubernetes is flaky," sending you to debug the wrong layer
> entirely. Meanwhile capacity planning based on Kubernetes' view of the cluster is simply wrong.
>
> This is why the recommendation is unqualified for production: the cost is not slower performance,
> it is *unpredictable* performance that presents as instability.

**A default catalogue** (confirm yours with `kubectl get virtualmachineclass`):

| Class | vCPU | Memory | Typical use |
| --- | --- | --- | --- |
| `guaranteed-small` | 2 | 4 GiB | Minimal workers |
| `guaranteed-medium` | 2 | 8 GiB | General-purpose workers |
| `guaranteed-large` | 4 | 16 GiB | **Control plane**; heavier workers |
| `guaranteed-xlarge` | 4 | 32 GiB | Memory-intensive workloads |
| `guaranteed-2xlarge` | 8 | 64 GiB | Large control planes; dense nodes |
| `guaranteed-4xlarge` | 16 | 128 GiB | High-density nodes |

**Match the class to your `maxPods`.** Raising `maxPods` to 250 ([7.1.1](#711-pod-cidr-sizing-node-blocks-maxpods-and-the-upgrade-spare))
on a `guaranteed-medium` (2 vCPU / 8 GiB) is not realistic — that is roughly 32 MiB per pod before
any overhead. If you want dense nodes, use `guaranteed-xlarge` or larger and set
`resourceConfiguration.systemReserved` so system daemons are not starved.

| Pitfall | Detail |
| --- | --- |
| Availability | The class must be associated with your vSphere Namespace: `kubectl get virtualmachineclass`. |
| Changing it | A `vmClass` change is a **rolling node replacement** — and needs a spare pod CIDR block. |
| Custom classes | Environments often define their own (e.g. a 20-vCPU/98-GiB class). Custom classes are fine; confirm whether they reserve resources, since the `best-effort`/`guaranteed` naming convention may not apply. |

### 7.6 `volumes` — dedicated containerd and kubelet disks

```yaml
    - name: volumes
      value:
      - capacity: 30Gi
        mountPath: /var/lib/containerd        # ← container image layers
        name: containerd
        storageClass: <STORAGE_CLASS>
      - capacity: 30Gi
        mountPath: /var/lib/kubelet           # ← ephemeral pod storage, emptyDir, logs
        name: kubelet
        storageClass: <STORAGE_CLASS>
```

**This is a genuine best practice and one of the strongest parts of the sample. Keep it.**

By default both directories live on the node's root disk. `/var/lib/containerd` holds every
container image layer pulled onto the node; `/var/lib/kubelet` holds ephemeral pod storage,
`emptyDir` volumes, and pod logs. Both grow in ways you do not directly control.

On a shared root disk, one image-pull storm or a single log-happy pod can fill the root filesystem.
When that happens the **kubelet itself degrades** — evicting pods under disk pressure and, if the
disk fills, ceasing to function. One misbehaving workload takes out the whole node.

Dedicated volumes contain that failure. A full containerd volume means image pulls fail: bad, but
survivable and diagnosable. A full root disk means the node is gone.

| Field | Why | Pitfall |
| --- | --- | --- |
| `capacity: 30Gi` | Sized for a modest workload set. | **30Gi is modest.** Image-heavy workloads — AI/ML frameworks, large JVM or data-science images, CUDA layers — exhaust a 30Gi containerd volume quickly, and multiple image versions accumulate. `/var/lib/kubelet` fills fast with `emptyDir`-heavy workloads. **Denser nodes need proportionally more:** a node running 250 pods needs far more ephemeral space and image cache than one running 110. |
| `mountPath` | The two highest-growth, least-predictable directories on a node. | Both paths must be exactly as the kubelet and containerd expect. Do not improvise alternatives. |
| `storageClass` | Must be associated with your vSphere Namespace. | The same class name appears here twice and in two other variables — keep all occurrences consistent. |
| **Changing later** | — | **Adding or resizing a volume is a rolling node-replacement operation**, not an in-place change. Every node in the pool is destroyed and rebuilt — survivable with a healthy multi-node pool and PodDisruptionBudgets, but disruptive, slow, needing a spare pod CIDR block, and with `controlPlane.replicas: 1` an outage. **This is the strongest argument for sizing generously on day one.** |

### 7.7 `kubernetes` — certificates, etcd, security, and API server

The densest and most consequential variable in the manifest.

```yaml
    - name: kubernetes
      value:
        certificateRotation:
          enabled: true                    # ← keep this on
          renewalDaysBeforeExpiry: 90
        etcdConfiguration:
          maximumDBSizeGiB: 4
        security:
          podSecurityStandard:
            audit: privileged              # ← dev/test reference values
            auditVersion: latest
            deactivated: false
            enforce: privileged
            enforceVersion: latest
            warn: privileged
            warnVersion: latest
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
                - <OIDC_CLIENT_ID>         # ← ≡ Headlamp's clientID
              claimMappings:
                username:
                  claim: email
                  prefix: "oidc:"          # ← mandatory anti-impersonation guard
                groups:
                  claim: groups
                  prefix: "oidc-groups:"
              claimValidationRules:
              - expression: 'claims.?email_verified.orValue(true) == true'   # ← FAILS OPEN
                message: "email must be verified"
        kubeletConfiguration:
          logging:
            format: json
```

#### 7.7.1 `certificateRotation`

| Field | Why | Pitfall |
| --- | --- | --- |
| `enabled: true` | **Correct — keep it.** Control-plane certificates expire, and expired certificates mean a dead control plane you cannot fix through the API, because the API is what stopped working. Automating renewal removes an entire class of self-inflicted outage that has taken down clusters run by competent teams. | The only real failure mode of manual rotation is forgetting. Automation is strictly better. |
| `renewalDaysBeforeExpiry: 90` | Renews 90 days ahead, a wide window for rotation to happen and be noticed. | Must be comfortably shorter than certificate validity, and long enough that renewal lands inside a maintenance window rather than at the expiry cliff. 90 days on a one-year certificate is sensible. |
| Verification | — | Rotation is silent when it works, which makes it an untested assumption. Confirm at least once — see [Day-2](#12-day-2-lifecycle). |

#### 7.7.2 `etcdConfiguration`

```yaml
        etcdConfiguration:
          maximumDBSizeGiB: 4
          # also available: maxRequestSizeKiB
```

| Consideration | Detail |
| --- | --- |
| Why size it explicitly | etcd stores every object plus revision history until compaction. **An addon stack of this size is exactly the workload that grows etcd** — five addons bring many CRDs, and operators are chatty: the Prometheus operator, cert-manager, and kapp-controller all write status and events continuously. |
| **The read-only alarm** | Exceeding the quota raises a `NOSPACE` alarm and puts etcd **read-only**. Every write fails. It presents as a total cluster outage and **does not clear on its own** — you must compact, defragment, and explicitly disarm the alarm, all requiring etcd-level access at the moment your control plane appears dead. |
| Prerequisite | The **etcd volume must actually have the space.** Setting a quota larger than the disk converts a clean quota stop into a disk-full condition, which is worse. |
| **Recommendation** | **Monitor etcd database size and alert at 60–70% of the quota.** You have Prometheus in this stack; use it. That converts an outage into a ticket. Also confirm auto-compaction is running. |
| `maxRequestSizeKiB` | Also available in 3.7. Raise only if you have genuinely large objects — large CRDs or ConfigMaps. Raising it lets clients push more per request, which grows etcd faster. |
| Sizing | 4 GiB is reasonable for a small-to-medium cluster with this addon set. High object counts, heavy CRD use, or high event churn need more. Check the schema default rather than assuming 4 is a change. |

#### 7.7.3 `security.podSecurityStandard`

> **These values are dev/test reference settings.** `privileged` at all three levels means Pod
> Security enforces nothing — which is exactly right for a lab where you want to explore the
> platform without admission control getting in the way. **It is not a production recommendation**,
> and it is **not** required by anything else in this manifest (see the
> [Istio CNI note](#a-note-on-istiocni-and-pod-security) — with no sidecar injection, Istio imposes
> no Pod Security constraint here).

Three levels, weakest to strongest:

| Level | Meaning | Appropriate for |
| --- | --- | --- |
| `privileged` | **No restrictions.** Any pod may request host namespaces, privileged containers, arbitrary capabilities, host paths. | Dev/test, or a lab |
| `baseline` | Blocks known privilege escalations while staying broadly compatible with common workloads. | **A realistic production floor** |
| `restricted` | Enforces hardening: non-root, no privilege escalation, dropped capabilities, seccomp. | **The production target** |

Three modes:

| Mode | Effect |
| --- | --- |
| `enforce` | **Rejects** non-compliant pods at admission. The only mode with teeth. |
| `audit` | Records violations in the audit log. Pod is admitted. |
| `warn` | Returns a warning to the user's client. Pod is admitted. |

| Field | Sample value | Assessment |
| --- | --- | --- |
| `enforce` | `privileged` | Nothing is blocked. For production, `baseline` minimum, `restricted` as the target. |
| `audit` / `warn` | `privileged` | **A free win being left on the table.** Setting `enforce: privileged` while raising `audit` and `warn` to `restricted` gives you **a complete, zero-risk inventory of exactly which workloads would break under stricter enforcement** — nothing is rejected, but every violation is logged and surfaced to users. This is the standard way to plan a PSS migration and it costs nothing. **Do this today.** |
| `deactivated: false` | Correct | The machinery is active; only the level is permissive. |
| `*Version: latest` | Unpinned | **A real upgrade hazard.** `latest` means "whatever this Kubernetes version defines," and PSS definitions tighten over time as new escape vectors close. A cluster upgrade can silently make your admission policy stricter, rejecting workloads that deployed cleanly yesterday — during an upgrade, when you are already busy. **Pin explicitly** (e.g. `v1.36`), then raise deliberately as a separate, tested change. |

##### Namespace strategy: two levers, and when to use each

Once PSS enforces anything above `privileged`, you need a story for namespaces that legitimately
need more privilege than the cluster default. There are **two** mechanisms, they behave very
differently, and choosing the wrong one is the most common Pod Security mistake after leaving it at
`privileged`.

| | **Per-namespace labels** | **`exemptions.namespaces`** |
| --- | --- | --- |
| Where it lives | Labels on the `Namespace` object | The `Cluster` manifest (API server admission config) |
| What it does | Applies a **different level** to that namespace | **Bypasses PSS entirely** for that namespace |
| Granularity | Per level and per mode — grant `baseline` where the default is `restricted` | All-or-nothing; no enforce, audit, *or* warn |
| Visibility | Visible to anyone who can read the namespace | Visible only to whoever reads the `Cluster` manifest |
| Who controls it | Whoever can label namespaces | Platform team only |
| Cost to change | A label edit | A `Cluster` edit + control-plane reconfiguration |
| Audit trail retained? | **Yes** — violations still logged and warned | **No** — you lose all signal from that namespace |

**Prefer labels.** An exemption is a blunt instrument: it does not grant a lower level, it turns the
policy off, and you lose `audit`/`warn` visibility along with it. A namespace label keeps the policy
engaged at a level that workload can actually meet.

```yaml
# The standard pattern: grant `baseline` in a namespace whose cluster default is `restricted`.
apiVersion: v1
kind: Namespace
metadata:
  name: legacy-app
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.36
    # Keep audit/warn STRICTER than enforce — you still get told what would
    # fail under restricted, without blocking the workload today.
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.36
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.36
```

Reserve `exemptions.namespaces` for **platform and infrastructure namespaces** that cannot run under
any enforced level — a short, stable list the platform team owns:

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

##### The seamless-install problem

Here is the friction in practice. `helm install --create-namespace` creates a **bare** namespace with
no PSA labels, so it inherits the cluster default. If the chart's pods cannot meet that level, they
are rejected — and the operator has to go back, create the namespace by hand with labels, and
reinstall. The same applies to any CI pipeline or `kubectl apply -f` that assumes it can create its
own namespace.

**The most useful thing to know: most of this friction lives at `restricted`, not `baseline`.**

| Cluster default | What typically breaks |
| --- | --- |
| `privileged` | Nothing. No policy. |
| **`baseline`** | Little. `baseline` blocks host namespaces, privileged containers, and dangerous capabilities — things most application charts never request. Charts that break here are usually infrastructure components, which you were going to treat specially anyway. |
| `restricted` | **A great deal.** It additionally requires `runAsNonRoot`, `allowPrivilegeEscalation: false`, a `seccompProfile`, and all capabilities dropped. Many perfectly ordinary charts do not set these in their pod specs, so they fail even though they need no real privilege. |

**So `enforce: baseline` is the setting that buys most of the security with little of the pain**, and
it is the pragmatic cluster default for a platform serving charts you do not control. Treat
`restricted` as a per-namespace goal you raise deliberately, not a cluster-wide starting point.

##### Four ways to handle namespace creation

| # | Approach | Seamless install? | Assessment |
| --- | --- | --- | --- |
| **A** | **Platform pre-creates namespaces with PSA labels.** App teams install into an existing namespace, without `--create-namespace`. | No — one extra step | **The recommended default.** Fine-grained, self-documenting, standard upstream practice, no control-plane changes. Namespace creation becomes a deliberate, reviewable platform action, which is usually what you want anyway. |
| **B** | **Namespace committed alongside the release in Git**, labels included. | Yes | **Best if you already do GitOps.** You have the `helm-controller` addon ([6.1](#61-helm-controller)), so a `Namespace` object with PSA labels can sit next to the `HelmRelease` in the same directory and be applied together. Seamless *and* fine-grained *and* auditable. |
| **C** | **Pre-declare namespace names in `exemptions.namespaces`** before they exist. | Yes | A narrow escape hatch. Works because the exemption is a **name match evaluated at admission**, so the name can be listed before the namespace is created — but it fully disables PSS for those names, does not scale past a handful, and needs a control-plane change each time. Use only where you must support `--create-namespace` unchanged for a known, fixed set of namespaces. |
| **D** | **Keep the cluster default at `privileged`, label individual namespaces stricter.** | Yes | **Not recommended.** It inverts the posture: every namespace anyone forgets is unprotected, and the failure is silent. Acceptable only as a transitional state while you inventory workloads. |

**A recommended combination:** cluster default `enforce: baseline` with `audit`/`warn: restricted`;
approach **A** or **B** for workload namespaces; `exemptions.namespaces` reserved for platform
components. That gives seamless installs for the common case, real enforcement everywhere, full
visibility into what stands between you and `restricted`, and a short exemption list you can defend
in a review.

##### One caveat worth planning for

**If app teams can label their own namespaces, they can grant themselves `privileged`.** Per-namespace
labels are only as strong as the RBAC around namespace creation and modification. Two mitigations:

- **Do not grant namespace `create`/`patch` to app teams.** The platform creates namespaces
  (approach A), which is the simplest and most robust control.
- If teams must self-serve namespaces, restrict which PSA label values they can set. That needs a
  policy engine — `gatekeeper` is in the catalogue ([6.6](#66-additional-addons-worth-considering))
  if you go that route.

##### Finding out what would break, before it does

`audit` and `warn` set stricter than `enforce` is the whole point — it tells you the answer without
blocking anything:

```bash
# `warn` violations come back to the client on apply — the fastest signal
kubectl apply -f my-chart-output.yaml
# Warning: would violate PodSecurity "restricted:v1.36": allowPrivilegeEscalation != false ...

# Dry-run a rendered chart against the live admission chain without creating anything
helm template my-release ./chart --namespace my-ns | kubectl apply --dry-run=server -f -

# What level is each namespace actually at right now?
kubectl get ns -L pod-security.kubernetes.io/enforce,pod-security.kubernetes.io/enforce-version
```

`audit` violations land in the API server audit log, which is the durable record — another reason to
ship those logs somewhere ([Appendix D.2](#d2-log-forwarding-fluent-bit)).

##### The migration path, in order

1. **Today, risk-free:** set `audit` and `warn` to `restricted`, leave `enforce: privileged`. Change
   nothing else. Collect violations across a full workload cycle, including scheduled jobs.
2. Pin all three versions explicitly.
3. Move `enforce` to **`baseline`** — the step with the best security-to-friction ratio.
4. Decide namespace creation policy (approach **A** or **B** above) and label existing workload
   namespaces explicitly rather than relying on the cluster default.
5. Add `exemptions.namespaces` for the platform components that genuinely cannot run enforced.
6. Raise individual workload namespaces to `restricted` as their charts are fixed — per namespace,
   not cluster-wide.
7. Once nothing depends on the looser default, raise the cluster default to `restricted`.
8. **If you later adopt the Istio sidecar mesh**, enable `istioCNI` at the same time — otherwise
   injected pods need `privileged` and undo this work.

#### 7.7.4 `security.resourceQuotaConfiguration`

```yaml
          resourceQuotaConfiguration:
            enabled: true       # schema default is false
```

| Consideration | Detail |
| --- | --- |
| Why enable it | Without quotas, one namespace can consume all cluster capacity and starve everything else. A basic multi-tenancy and cost-control guardrail. |
| **Pitfall** | Enabling quota enforcement can cause **previously schedulable workloads to be rejected**, because pods without resource requests may not be admitted once a quota applies. Turn it on in non-production first and ensure workloads declare requests and limits. |
| Note | The schema default is `false`, so `true` here is a deliberate choice — a good one. Exact behaviour is release-specific; see [Appendix C](#appendix-c--verify-for-your-environment). |

#### 7.7.5 Structured logging

```yaml
        apiServerConfiguration:
          logs:
            format: json
        kubeletConfiguration:
          logging:
            format: json
```

| Consideration | Detail |
| --- | --- |
| Why | Structured JSON is parsed reliably by log aggregators with no fragile regex extraction, and fields become queryable. For any cluster shipping logs off-node — which should be every production cluster — this is right. |
| Consistency | Setting it on **both** API server and kubelet is good practice; mixed formats in one pipeline defeat the purpose. |
| **Pair it with log shipping** | This decision only pays off if the logs leave the node — node logs are lost when a node is replaced, which happens on every upgrade. If you do not already forward logs, see [Appendix D.2](#d2-log-forwarding-fluent-bit). |
| Pitfalls | Human readability drops noticeably — keep `jq` handy, and remember that if your pipeline is down you are reading JSON by hand. Structured lines are also more verbose; account for the volume. |
| Also available | `logs.verbosity` (0–10) and `logs.flushFrequency`. Leave verbosity at its default unless debugging; high verbosity on a busy API server generates enormous volume. |

#### 7.7.6 `apiServerConfiguration.extraAuthentication` — OIDC

This configures **structured authentication**: JWTs from an external identity provider are accepted
as Kubernetes credentials, so humans authenticate with corporate identity instead of long-lived
kubeconfig certificates. Central revocation, MFA, and short-lived credentials come with it — a
significant security improvement.

> **The provider in the sample is an example, not a requirement.** This mechanism is **generic
> OIDC**. Any compliant provider works — Entra ID, Okta, Keycloak, Dex, Ping, Auth0, Google, GitLab,
> or a self-hosted issuer. Nothing below is provider-specific, and the fields you need are the same
> in every case. Where providers differ is in *which claims they emit*, which is the one thing you
> must check against your own IdP's token.
>
> **This configuration exists to support OIDC login for Headlamp** ([6.5](#65-headlamp)) — the UI
> obtains ID tokens and the API server must accept them. It is equally useful for `kubectl` access
> with an OIDC-aware credential plugin.

##### OIDC field reference

| Field | Why | Pitfall |
| --- | --- | --- |
| `issuer.url` | The trusted issuer. The API server fetches OIDC discovery metadata and signing keys from here. | **Must match the token's `iss` claim exactly**, trailing slash included, and match the value your client uses. The API server must also be able to **reach** this URL and its JWKS endpoint — in a restricted-egress environment external OIDC simply does not work, and the failure looks like universally invalid tokens. See `egressSelectorType` below. |
| `audiences` | A token is accepted only if its `aud` claim is in this list. This is what stops a token minted for another application being replayed against your cluster. | **Must contain the client ID your UI or CLI uses.** If it does not, users authenticate successfully at the IdP and then every API call fails — a symptom that points nowhere near the cause. |
| `claimMappings.username.claim` | Which claim becomes the Kubernetes username. `email` is stable, human-readable, and matches how people think about identity. | Requires the corresponding scope to be requested by the client. Note `email` is not immutable at every IdP — if a user's address changes, their Kubernetes identity changes and their RBAC bindings stop applying. Where available, an immutable subject identifier (`sub`) is more robust at the cost of unreadable RBAC. |
| `claimMappings.username.prefix` | **A mandatory security control.** The prefix namespaces external identities so they cannot collide with — and therefore cannot impersonate — built-in Kubernetes subjects such as `system:masters` members or ServiceAccounts (`system:serviceaccount:...`). Without it, an IdP that lets a user set an arbitrary email could mint an identity RBAC already trusts. | **Never leave it empty.** And note that **changing it orphans every existing RBAC binding at once** — the username string changes, bindings naming the old form stop matching, and every user loses access simultaneously. Choose it once. |
| `claimMappings.groups.claim` | Maps an IdP group claim to Kubernetes groups, enabling group-based RBAC — the only approach that scales. | **Verify your IdP actually emits this claim.** Many do not by default: some require a specific scope, some need the claim explicitly added to the token, and some require a directory-API integration or a broker (Dex, Keycloak) to enrich tokens. If the claim is absent, group bindings silently do nothing — and you discover it only after designing your RBAC around them. **Inspect a real token before relying on groups.** |
| `claimMappings.groups.prefix` | Same anti-impersonation reasoning, and more urgent here: without it, an IdP group literally named `system:masters` would grant cluster-admin outright. | Not optional. |
| `claimValidationRules` | Additional trust conditions. Two forms: `claim` + `requiredValue` for a simple equality check, or `expression` for CEL. | The sample's rule **fails open** — see below. |
| `message` | The error returned on rejection. | Make it describe what actually happened. A message claiming email verification is required, on a rule that does not require it, is worse than no message — it stops you looking. |

##### Fields not in the sample that matter for enterprise IdPs

Verify availability against your ClusterClass schema:

| Field | When you need it |
| --- | --- |
| `issuer.certificateAuthority` | **Essential for an internal or private IdP.** Supplies the CA bundle the API server uses to trust the issuer's TLS certificate. Without it, an IdP behind an enterprise CA or a self-signed certificate is unreachable and every token fails validation. |
| `issuer.egressSelectorType` | `controlplane` or `cluster` — selects the network path the API server uses to reach the IdP. Important when the IdP is only reachable from one of them. |
| `issuer.discoveryURL` | When discovery metadata is not at the standard `.well-known` path relative to the issuer URL. |
| `issuer.audienceMatchPolicy` | `MatchAny` — accept a token whose `aud` matches any listed audience. |
| `claimMappings.uid` | Maps a claim to the Kubernetes user UID, useful for audit correlation. |
| `claimMappings.extra` | Maps arbitrary claims into extra user attributes, consumable by authorization webhooks. |
| `userValidationRules` | CEL rules validating the **mapped user** rather than the raw claims — for example asserting the username carries the expected prefix. A useful defence-in-depth layer on top of `claimValidationRules`. |

##### `claims.?email_verified.orValue(true)` admits unverified identities

```
claims.?email_verified.orValue(true) == true
```

Piece by piece:

- `claims.?email_verified` — CEL **optional chaining**. Yields the claim's value if present, or an
  *empty optional* if absent. The `?` is what prevents a missing claim from erroring.
- `.orValue(true)` — unwraps the optional, substituting **`true`** when the claim is absent.
- `== true` — compares.

So a token that **omits `email_verified` entirely** is treated as verified and **accepted**. The
rule rejects only tokens explicitly asserting `email_verified: false` — inverting the intent stated
in the very next line. Since the username derives from the email claim, a misconfigured client or an
IdP that omits the claim can present an unverified — potentially someone else's — address as a
Kubernetes identity.

**The fix is one word:**

```yaml
              claimValidationRules:
              - expression: 'claims.?email_verified.orValue(false) == true'
                message: "email must be verified"
```

`orValue(false)` **fails closed**: absent claim means rejected. This is the correct default for any
security predicate — when you cannot prove the condition holds, deny. Apply the same reasoning to
every CEL rule you write.

> **You must also create RBAC inside the workload cluster.**
>
> This configures authentication only. **A successfully authenticated user has zero permissions**
> until an RBAC binding names them, and the resulting experience looks like a broken cluster rather
> than a missing binding.
>
> The subject name is the **fully composed username**: `claimMappings.username.prefix` concatenated
> with the mapped claim value. With `prefix: "oidc:"` and `claim: email`, a user with the email
> `user@example.com` is the Kubernetes user `oidc:user@example.com`.
>
> Bindings are applied **inside the workload cluster**, not the vSphere Namespace:
>
> ```yaml
> apiVersion: rbac.authorization.k8s.io/v1
> kind: ClusterRoleBinding
> metadata:
>   name: oidc-admin
> subjects:
> - kind: User
>   name: "oidc:user@example.com"
>   apiGroup: rbac.authorization.k8s.io
> roleRef:
>   kind: ClusterRole
>   name: cluster-admin
>   apiGroup: rbac.authorization.k8s.io
> ```
>
> Always verify the exact string the API server derives, rather than assuming:
> ```bash
> kubectl auth whoami
> ```
> Full guidance — group bindings and least privilege — in
> [Appendix B.1](#b1-rbac-for-oidc-identities).

##### Other `apiServerConfiguration` fields

| Field | Guidance |
| --- | --- |
| `profiling` | Exposes `/debug/pprof` on the API server. Available if you want to disable it, but **be cautious** — profiling endpoints are used by support tooling and diagnostics, so turning them off can complicate troubleshooting. Leave at the default unless a specific policy requires otherwise. |
| `maxRequestsInFlight` / `maxMutatingRequestsInFlight` | API server concurrency limits. Raise for very large or controller-heavy clusters; the defaults suit most. |
| `requestTimeout` | Timeout for non-long-running requests. |
| `oidc.serviceAccountIssuerURL` | **A different feature from `extraAuthentication`.** This sets the issuer for *bound ServiceAccount tokens*, enabling workload identity federation with external systems. Unrelated to human login. |

#### 7.7.7 Other `kubernetes` sub-variables

| Field | Guidance |
| --- | --- |
| `security.minimumTLSProtocol` | `TLS_1.2` or `TLS_1.3`. **Handle with care — do not set this reflexively.** Raising the floor to `TLS_1.3` breaks any client, controller, webhook, or integration that has not been validated against it, and the failures are TLS handshake errors scattered across unrelated components. Only change it against a specific compliance requirement, with a tested inventory of every client that talks to the API server. |
| `security.tlsCipherSuites` | Restricts accepted cipher suites. **Same caution, more sharply** — an over-narrow list is one of the easier ways to make a cluster partially unreachable, and the resulting errors do not name cipher negotiation as the cause. Leave at the platform default unless you have a defined cryptographic policy *and* a way to validate every client against it. |
| `endpointFQDNs` | Additional FQDNs to include in the API server certificate SANs. Needed when reaching the API server through a load balancer or vanity DNS name — otherwise TLS validation fails on that name. Replaces the deprecated `kubeAPIServerFQDNs` variable. |
| `kubeControllerManagerConfiguration.terminatedPodGCThreshold` | How many terminated pods are retained before garbage collection. Lower it on high-churn clusters — retained pod objects consume etcd space, which ties back to [7.7.2](#772-etcdconfiguration). |
| `kubeProxyConfiguration.enabled` | Allows disabling kube-proxy, for CNIs that replace it (some Cilium modes). **Do not disable it unless your CNI explicitly requires it** — service networking stops working otherwise. |
| `kubeletConfiguration.podPidsLimit` | Caps PIDs per pod. A worthwhile defence against fork-bomb-style resource exhaustion. |
| `kubeletConfiguration.imageGCHighThresholdPercent` / `imageGCLowThresholdPercent` | Disk-usage thresholds for image garbage collection. Relevant to the containerd volume in [7.6](#76-volumes--dedicated-containerd-and-kubelet-disks) — tune together. |
| `kubeletConfiguration.containerLogMaxSizeMiB` / `containerLogMaxFiles` | Bound per-container log growth. **Directly protects the `/var/lib/kubelet` volume** from a log-happy pod. |
| `kubeletConfiguration.serializeImagePulls` / `maxParallelImagePulls` | Control image-pull concurrency. Parallel pulls speed up scale-out but can saturate the registry or the network. |
| `kubeletConfiguration.imagePullCredentialsVerificationPolicy` | `NeverVerify`, `NeverVerifyPreloadedImages`, `NeverVerifyAllowlistedImages`, `AlwaysVerify`. `AlwaysVerify` prevents a pod from using an image cached on the node by another tenant without presenting its own credentials — **a genuine multi-tenancy control** worth enabling in shared clusters. |
| `kubeletConfiguration.allowedUnsafeSysctls` | Only for specialised tuning. Taint such node pools and schedule only workloads that need them. |

### 7.8 `bootstrapAddons` — CNI selection

```yaml
    - name: bootstrapAddons
      value:
        cniRef:
          name: antrea                        # ← the cluster's CNI
          namespace: vmware-system-vks-public
```

| Consideration | Detail |
| --- | --- |
| Why it is separate | The CNI must be installed as the cluster bootstraps — nodes cannot become `Ready` and pods cannot get addresses without it. Hence "bootstrap" addons, configured here rather than as an `AddonInstall`. |
| **Effectively irreversible** | **CNI is a create-time decision.** There is no supported in-place migration: changing it means building a new cluster and migrating workloads. |
| Your options | List them with `kubectl get acd -n vmware-system-vks-public \| grep -E '^(antrea\|calico\|cilium)'`. A default catalogue offers **`antrea`**, **`calico`**, and **`cilium`**. |
| **`antrea`** | The VMware-supported default. Integrates with the vSphere networking stack, supports Kubernetes NetworkPolicy plus richer Antrea policy CRDs, and handles the reserved `240.0.0.0/4` pod CIDR used here. **Choose this unless you have a specific reason not to.** |
| **`calico`** | Mature, widely deployed, strong policy model. Reasonable if it is your organisational standard. |
| **`cilium`** | eBPF-based, with advanced observability (Hubble), L7 policy, and the option to replace kube-proxy. Choose it if you want those capabilities and will operate them — note the `kubeProxyConfiguration.enabled` interaction in [7.7.7](#777-other-kubernetes-sub-variables). |
| Interaction with Istio | Both manipulate pod networking. Normal and supported. At gateway-only Istio scope there is no interaction at all; if you later enable `istioCNI`, validate the combination in non-production first. |

### 7.9 `vsphereOptions` — persistent volume storage classes

```yaml
    - name: vsphereOptions
      value:
        persistentVolumes:
          availableStorageClasses:
          - <STORAGE_CLASS>                   # ← classes offered to workloads
          defaultStorageClass: <STORAGE_CLASS>
```

Distinct from the `storageClass` variable in [7.4](#74-storageclass): **that one is for the nodes'
own disks, this one is for workload PersistentVolumeClaims.**

| Field | Why | Pitfall |
| --- | --- | --- |
| `availableStorageClasses` | The allow-list exposed inside the workload cluster. Explicitly listing them is good practice — workloads cannot request policies you have not sanctioned. | **The class must be associated with your vSphere Namespace, and the error surfaces late.** A class named here but not associated is accepted at cluster-create time and fails at *PVC*-create time — so the first symptom is a workload whose PVC pends indefinitely with an unhelpful event, long after the cluster looked healthy. |
| `defaultStorageClass` | The class used by PVCs that do not name one. **Setting a default matters** — without it, any chart that omits `storageClassName` produces a permanently pending PVC. | Only one class should be default. Consider whether your default should be your most expensive protection level; many teams default to a cheaper class and require explicit opt-in to premium storage. |
| Single class | Simple and predictable. | Consider a tier set — a mirrored class for latency-sensitive databases, erasure-coded for capacity — so workloads can match storage to need. Every class must be namespace-associated. |

> **One name, four places.** The StorageClass appears in `storageClass`, in both `volumes` entries,
> and twice here. When you substitute your own, change all of them — a missed occurrence produces a
> cluster that half-works.

### 7.10 `osConfiguration` — NTP and SSH banner

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

#### NTP — a functional prerequisite, not hygiene

| Consideration | Detail |
| --- | --- |
| Why it matters far more than it looks | Time synchronisation is load-bearing for **three** mechanisms in this manifest. **(1) TLS:** certificate validity is a time window; a skewed clock rejects valid certificates as not-yet-valid or expired. **(2) etcd:** leases and election timeouts are time-based, and skew between members causes election churn. **(3) OIDC:** every ID token carries `exp`, `iat`, and often `nbf` claims validated against the API server's clock. |
| **The specific risk here** | This cluster authenticates humans with short-lived OIDC tokens. **A skew of even a few minutes rejects valid tokens** — or accepts expired ones. The symptom is intermittent, user-specific authentication failures that come and go as tokens refresh, which is close to undiagnosable if you are not looking at clocks. |
| Pitfall | **The server must be reachable from the node network.** An unreachable NTP server is worse than none configured, because you believe time is managed. Verify sync after deployment. |
| Recommendation | Configure **multiple** servers, and use the same sources as the rest of your infrastructure — including vCenter and ESXi — so the whole stack agrees. **Alert on clock skew.** |

#### SSH banner

| Consideration | Detail |
| --- | --- |
| Why | A legal and compliance control. Many frameworks require an explicit authorized-use notice before interactive access. |
| Pitfall — YAML | Note the **trailing space** before the closing quote, and that this is a single-quoted scalar folded across two lines: YAML flow folding turns the newline plus indentation into one space, so the value is a single line. Preserve the quoting style — switching to a block scalar (`\|` or `>`) changes the resulting string, usually adding a newline you did not intend. |
| Pitfall — content | Have legal or compliance supply the wording. A generic banner may not satisfy the regime you are audited against. |
| Note | A banner is a deterrent and a notice, **not** an access control. Restrict node SSH by network policy, jump hosts, and key management. |

### 7.11 `version`

```yaml
    version: v1.36.1---vmware.4-vkr.5
```

Covered in full in [section 2](#2-choosing-a-kubernetes-release-the-kr-object). In short: this must
be the **`NAME`** of a `kr` object that is `READY=True` and `COMPATIBLE=True`, using the **triple
dash**, and it must be compatible with the ClusterClass.

[Appendix E](#appendix-e--inspecting-a-kubernetes-release-kr) covers what the release actually
contains — etcd and CoreDNS versions, the version-pinned platform addons, and the OS images it offers
— plus a pre-upgrade diff you can run between two releases.

### 7.12 `workers.machineDeployments` — the node pool

```yaml
    workers:
      machineDeployments:
      - class: node-pool                       # ← a machine-deployment class from the ClusterClass
        metadata:
          annotations:
            cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "5"
            cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "1"
            run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
        name: node-pool-1
```

| Field | Why | Pitfall |
| --- | --- | --- |
| `class: node-pool` | References a machine-deployment class defined by the ClusterClass; this entry instantiates it. | The class must exist in your ClusterClass. |
| `name: node-pool-1` | Names the pool. | **Multiple pools are the mechanism for workload separation** — different VM classes, different `maxPods`, taints for GPU or memory-optimised workloads. One undifferentiated pool means every workload shares a node shape. Combine with the `node.labels` and `node.taints` variables ([7.13](#713-37-variables-not-used-in-the-sample)). |
| autoscaler `min-size` / `max-size` | Declares the bounds the Cluster Autoscaler respects for this pool. | **These work** — the Cluster Autoscaler is installed by the platform ([section 3](#3-what-vks-installs-for-you)), so unlike some distributions you do not need to deploy it. Confirm with `kubectl get clusteraddon -n <VSPHERE_NAMESPACE> \| grep autoscaler`. |
| `min-size: "1"` | — | A single worker means no capacity redundancy: one node failure and everything is unschedulable. **Use at least 2, ideally 3**, so a node can be lost or drained without an outage. |
| `max-size: "5"` | — | **Sanity-check against the pod CIDR.** With a `/20` you have 16 blocks total, so 5 is safe — but raising this without widening the CIDR walks into the wall in [7.1.1](#711-pod-cidr-sizing-node-blocks-maxpods-and-the-upgrade-spare). Also check against vSphere capacity and VM-class reservations. |
| The **absence** of `replicas` | **Correct and deliberate.** A static `replicas` alongside an autoscaler-managed pool means two controllers fighting over the same field, producing oscillation. | If you deliberately disable autoscaling for a pool, you **must** set `replicas` — otherwise it has no size directive. Choose one model, never both. |
| `resolve-os-image` | Matches the control plane's, keeping both on the same OS image. | Set it on **every** machine deployment you add. |

### 7.13 3.7 variables not used in the sample

Three ClusterClass variables the manifest does not touch.

#### `resourceConfiguration.systemReserved` — how to size it

Reserves CPU and memory for system daemons — kubelet, container runtime, OS — so pods cannot starve
them. Starved system daemons produce `NotReady` nodes and cascading evictions.

```yaml
    - name: resourceConfiguration
      value:
        systemReserved:
          automatic: true       # schema default
```

##### What the upstream documentation says

The mechanism is upstream Kubernetes' **Node Allocatable** feature, documented in
[Reserve Compute Resources for System Daemons](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/).
Two points from that page matter here:

1. **Upstream deliberately publishes no recommended values.** There is no default and no formula —
   the numbers are considered site-specific.
2. **It warns explicitly against enforcing `systemReserved` casually.** The guidance is to *"enforce
   `systemReserved` only if a user has profiled their nodes exhaustively to come up with precise
   estimates and is confident in their ability to recover if any process in that group is
   oom-killed,"* because an over-tight reservation can leave *"critical system services CPU
   starved, OOM killed, or unable to fork on the node."*

So: **prefer `automatic: true`, and override only with evidence.** That is not a cop-out — it is the
upstream recommendation.

##### Where public formulas do exist

The two managed-Kubernetes providers that publish concrete reservation formulas are the usual
reference points. They are worth knowing because they quantify the two different things that drive
overhead:

**CPU — a tiered percentage of core count** (identical in both
[GKE](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/plan-node-sizes) and
[EKS](https://docs.aws.amazon.com/batch/latest/userguide/memory-cpu-batch-eks.html)):

| Cores | Reserved |
| --- | --- |
| 1st core | 6% |
| 2nd core | 1% |
| 3rd–4th core | 0.5% |
| Every core above 4 | 0.25% |

**Memory — GKE uses a tiered percentage of node memory:**

| Memory tier | Reserved |
| --- | --- |
| Nodes under 1 GiB | 255 MiB flat |
| First 4 GiB | 25% |
| Next 4 GiB (4–8 GiB) | 20% |
| Next 8 GiB (8–16 GiB) | 10% |
| Next 112 GiB (16–128 GiB) | 6% |
| Above 128 GiB | 2% |

plus **100 MiB on every node** held back for pod eviction.

**Memory — EKS instead scales with pod count**, which is the number relevant to `maxPods`:

```
kubeReserved memory = (11 MiB × max_pods) + 255 MiB
```

##### What VKS actually reserves

You do not have to guess, and this is worth doing before you change anything:

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,\
CPU_CAP:.status.capacity.cpu,CPU_ALLOC:.status.allocatable.cpu,\
MEM_CAP:.status.capacity.memory,MEM_ALLOC:.status.allocatable.memory,\
MAXPODS:.status.capacity.pods'
```

The gap between `capacity` and `allocatable` **is** the total reservation plus the eviction threshold.

Measured on a VKS 3.7 cluster with `automatic: true` and the default `maxPods: 110`:

| Node | Class | Capacity | Allocatable | Reserved |
| --- | --- | --- | --- | --- |
| control plane | `guaranteed-large`-equivalent (4 vCPU / 16 GiB) | 4 / 15.6 GiB | 3920m / 12.96 GiB | **80m / 2722 MiB** |
| worker | `guaranteed-medium`-equivalent (2 vCPU / 8 GiB) | 2 / 7.75 GiB | 1930m / 5.91 GiB | **70m / 1892 MiB** |

**Those values match the GKE-published tiered formulas exactly** — CPU to the millicore, memory to
within 1.2 MiB on both node sizes, once the 100 MiB eviction reserve is included:

| Node | Predicted (GKE tiers + 100 MiB) | Measured | Delta |
| --- | --- | --- | --- |
| 4 vCPU / 15.6 GiB | 80m / 2723.2 MiB | 80m / 2722 MiB | 0m / +1.2 MiB |
| 2 vCPU / 7.75 GiB | 70m / 1892.8 MiB | 70m / 1892 MiB | 0m / +0.8 MiB |

So the tiered model above is a reliable predictor of what `automatic: true` will give you on a given
VM class — useful for capacity planning before a node exists.

> **Caveat on the fit.** Two node sizes is a good fit, not a guarantee, and this is not a
> VMware-documented formula — VKS documents only the `automatic` flag, not its calculation. Treat the
> model as a planning aid and confirm against your own nodes. It could change between releases.

##### The gap when you raise `maxPods`

Here is the reason this variable matters for pod density: **the memory reservation above is derived
from node size, not pod count.** Raising `maxPods` from 110 to 250 more than doubles the kubelet and
runtime bookkeeping per node while the automatic reservation does not move at all.

The two providers each acknowledge this, and their numbers give you a defensible adder:

| Source | Adjustment for higher pod density |
| --- | --- |
| EKS | Memory scales at **11 MiB per pod** |
| GKE | *"If you increase the maximum Pods per node beyond 110, GKE reserves an extra **400 mCPU**"* |

Combining them gives a starting point for an explicit override — automatic's node-size reservation,
**plus** a pod adder:

```
memory = <automatic reservation for the VM class>  +  11 MiB × (maxPods − 110)
cpu    = <tiered CPU reservation for the core count>  +  400m   (when maxPods > 110)
```

```yaml
    # Example: a pool running maxPods: 250 on guaranteed-2xlarge (8 vCPU / 64 GiB)
    - name: resourceConfiguration
      value:
        systemReserved:
          automatic: false
          cpu: 500m            #  90m tiered + 400m density adder
          memory: 7Gi          # ~5.5Gi node-size tiers + 1.5Gi (11 MiB × 140 pods)
```

##### The consequence: high `maxPods` demands large nodes

Reserved resources are subtracted from allocatable, so reservation and pod count together determine
what is actually left per pod. Run this arithmetic before choosing a VM class — it is where an
ambitious `maxPods` usually falls apart. Figures below use the tiered model plus the pod adder:

| VM class | vCPU / Mem | Reserved @ 110 | Alloc @ 110 | Reserved @ 250 | Alloc @ 250 | Avg mem/pod @ 250 | Verdict |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `guaranteed-medium` | 2 / 8 GiB | 70m / 1.85 GiB | 5.9 GiB | 470m / 3.35 GiB | 4.4 GiB | **~18 MiB** | **Unusable** |
| `guaranteed-large` | 4 / 16 GiB | 80m / 2.65 GiB | 12.9 GiB | 480m / 4.15 GiB | 11.4 GiB | ~47 MiB | Unrealistic |
| `guaranteed-xlarge` | 4 / 32 GiB | 80m / 3.60 GiB | 27.4 GiB | 480m / 5.10 GiB | 25.9 GiB | ~106 MiB | Tight; CPU-bound |
| `guaranteed-2xlarge` | 8 / 64 GiB | 90m / 5.46 GiB | 56.6 GiB | 490m / 6.97 GiB | 55.1 GiB | ~226 MiB | **Workable** |
| `guaranteed-4xlarge` | 16 / 128 GiB | 110m / 9.19 GiB | 115.0 GiB | 510m / 10.69 GiB | 113.5 GiB | ~465 MiB | Comfortable |

> **The practical rule: if you raise `maxPods` to 250, plan on `guaranteed-2xlarge` or larger.**
> And note **CPU usually binds before memory** — 8 vCPU across 250 pods is ~29 millicores each, which
> suits many small services and nothing compute-bound. Size from your workload's actual requests, not
> from the pod count.

##### Practical guidance

| Guidance | Detail |
| --- | --- |
| **At `maxPods: 110`, leave `automatic: true`** | The tiered reservation is well-matched to node size, and upstream advises against overriding without profiling. |
| **Override only when you change pod density** | That is the case automatic does not cover, because its memory tier is node-size derived. |
| **Set it per pool** | Use `variables.overrides` on the machine deployment so a dense pool gets a large reservation without imposing it on pools that do not need one. |
| **Measure, then adjust** | Compare `capacity` against `allocatable` after deployment, and watch actual kubelet and runtime usage under load. The formulas are a starting point; your workload decides. |
| **Do not over-reserve** | Every reserved core and gigabyte is capacity you paid for and cannot schedule. |
| **Prefer erring high on memory, low on CPU** | Memory is incompressible — exhausting it OOM-kills system processes. CPU starvation degrades rather than kills, and upstream notes that reserving only compressible resources *"is less likely to cause disruption."* |
| **Changing it** | Requires a machine rollout — which needs a spare pod CIDR block. |

**References**

- [Reserve Compute Resources for System Daemons](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/) — upstream mechanism and cautions (no values published)
- [GKE: Plan node sizes](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/plan-node-sizes) — tiered CPU and memory formulas, 100 MiB eviction reserve, +400 mCPU above 110 pods
- [EKS / AWS Batch: Memory and vCPU considerations](https://docs.aws.amazon.com/batch/latest/userguide/memory-cpu-batch-eks.html) — `(11 MiB × max_pods) + 255 MiB`, and the tiered CPU table with worked examples

> **On terminology.** Upstream splits reservations into `kube-reserved` (kubelet, container runtime)
> and `system-reserved` (OS daemons, kernel). The provider formulas above are `kube-reserved`. VKS
> exposes a single `systemReserved` knob, so if you override it, size it for **total** node overhead
> rather than only the OS portion — and validate with the capacity-versus-allocatable check.

#### `node.labels` — declarative node labels

```yaml
    - name: node
      value:
        labels:
          workload-type: general
          environment: production
          cost-center: platform-eng
```

**Use this rather than labelling nodes by hand.** Labels applied with `kubectl label node` are lost
when the node is replaced — which happens on every upgrade, every `vmClass` change, and every
`volumes` resize. Labels set here are part of the node's declared configuration and survive
replacement.

| Use | Example |
| --- | --- |
| Scheduling | `workload-type: general` / `workload-type: memory-optimised`, targeted with `nodeSelector` or affinity |
| Environment identification | `environment: production` — useful in dashboards, alerts, and audit queries |
| Cost attribution | `cost-center: platform-eng` — chargeback and showback reporting |
| Compliance scoping | A label marking nodes subject to a particular control set |

Because this is a ClusterClass variable, it supports **per-pool overrides** — the same
`variables.overrides` mechanism used for `vmClass` in [7.3](#73-control-plane). That is how you label
one pool differently without duplicating the whole variable set:

```yaml
      machineDeployments:
      - class: node-pool
        name: node-pool-memory
        variables:
          overrides:
          - name: node
            value:
              labels:
                workload-type: memory-optimised
          - name: vmClass
            value: guaranteed-2xlarge
```

The `node` variable also carries `taints`, `cri.runtimeClasses`, and `firewall.inboundRules`. Those
are specialised — reach for them when you have a concrete requirement, not as part of a baseline.

#### `endpointFQDNs` — API server FQDN aliases

```yaml
    - name: kubernetes
      value:
        # Keep this in the same domain you publish the UI on, so the whole
        # cluster is reachable by name rather than by IP.
        endpointFQDNs:
        - api.k8s.example.com
```

Adds FQDN aliases for the control plane endpoint — per the schema, *"for example to allow users to
connect to the cluster using `https://k8s.prod.example.com/`"*. The name is added to the API server
certificate's SANs, so TLS validation succeeds on it.

**Set this alongside the Headlamp FQDN, in the same domain.** If you are publishing
`headlamp.k8s.example.com` for the UI ([6.5](#65-headlamp)), publish `api.k8s.example.com` for the API
server too — then a kubeconfig, a CI runner, and the UI all reference stable names in one zone
instead of a mix of names and IPs.

| Consideration | Detail |
| --- | --- |
| **Why it is needed** | Without the name in the certificate SANs, connecting via that name fails TLS validation — even though the endpoint is reachable. The error looks like a certificate problem, not a configuration gap. |
| What to include | Every name clients will use: a vanity DNS name, a load balancer name, and any per-environment alias. |
| DNS is still your job | This variable adds the name to the certificate; it does not create the DNS record. Publish an A record pointing at the control plane endpoint. |
| Pair with a stable endpoint | An FQDN in front of the API server endpoint is what lets you change the underlying address without reissuing every kubeconfig. |
| Supported scopes | `cluster` and `controlPlane`. |
| Migration note | This **replaces the deprecated `kubeAPIServerFQDNs` variable**, which the schema marks for removal. If you have manifests using the old name, migrate them. |

---

## 8. Cluster decisions you cannot change later

> **What this covers.** The short list of `Cluster` fields that are immutable, or expensive enough to
> treat as immutable.
>
> **Why it matters.** Getting one of these wrong is not a configuration change, it is a cluster
> rebuild and a workload migration. They are worth a deliberate decision before the first apply.

A recap of the previous section, isolating what matters most. Most of the `Cluster` object is
editable after deployment — but a small number of fields are not, and correcting one of them means
building a new cluster and migrating every workload.

**Settle these before you apply.** If you review nothing else in section 7, review this table.

| Decision | Field | Why it is fixed | Get it right by |
| --- | --- | --- | --- |
| **Pod network size** | `clusterNetwork.pods.cidrBlocks` | Immutable. Determines your maximum node count, and an exhausted pod CIDR also blocks upgrades. | Sizing for maximum plausible scale plus a surge reserve — [7.1.1](#711-pod-cidr-sizing-node-blocks-maxpods-and-the-upgrade-spare) |
| **Service network size** | `clusterNetwork.services.cidrBlocks` | Immutable. Caps total `ClusterIP` Services. | Same exercise; `/20` is ~4,094 services |
| **CNI** | `bootstrapAddons.cniRef` | No supported in-place migration path. | Choosing between `antrea`, `calico`, and `cilium` on day one — [7.8](#78-bootstrapaddons--cni-selection) |
| **Cluster DNS domain** | `clusterNetwork.serviceDomain` | Baked into every service address and workload identity. | Leaving it as `cluster.local` unless you have a specific reason |
| **Cluster name** | `metadata.name` | Embedded in every addon selector and `AddonConfig` name. | Following a naming convention from the start |
| **Per-node pod block size** | Node CIDR mask | Create-time. Governs how the pod CIDR is subdivided. | Understanding the arithmetic in 7.1.1 first |

Two more that are *technically* changeable but expensive enough to treat as day-one decisions:

| Decision | Cost of changing later |
| --- | --- |
| `volumes` capacity | **Rolling node replacement** of the whole pool — size generously up front |
| `controlPlane.replicas` 1 → 3 | Supported, but a rolling change to plan into a maintenance window |

---

## 9. Cross-object dependency map

> **What this covers.** Every value in the profile that must agree with a value somewhere else — in
> another object, in DNS, or at your identity provider.
>
> **Why it matters.** These are the failures where the symptom points away from the cause. Editing one
> line in isolation breaks something that looks unrelated, and the error message rarely names the
> field that is actually wrong.

These values must agree across objects. Each is a place where editing one line in isolation breaks
something that looks unrelated — and where the symptom points away from the cause. Worth keeping
next to your manifest.

| # | Must match | Where | Symptom if mismatched |
| --- | --- | --- | --- |
| 1 | **Cluster name** | `Cluster.metadata.name` ≡ every `AddonInstall.spec.clusters[].selector.matchLabels[cluster.x-k8s.io/cluster-name]` | The addon is never installed. No error — the selector matches nothing. |
| 2 | **AddonConfig name** | `AddonConfig.metadata.name` ≡ `<clusterName>-<AddonInstall.spec.addonRef.name>` — note this is the **addon** name, not the `AddonInstall` object name | Config skipped; addon runs on schema defaults. Check `spec.clusterName` is populated. |
| 3 | **vSphere Namespace** | `metadata.namespace` identical on every `AddonInstall`, `AddonConfig`, and the `Cluster` | Objects never associate. |
| 4 | **OIDC client ID** | Headlamp `oidc.clientID` ≡ apiserver `extraAuthentication.jwt[].issuer.audiences[]` | UI login succeeds, then every API call is unauthorised. |
| 5 | **OIDC issuer URL** | Headlamp `oidc.issuerURL` ≡ apiserver `extraAuthentication.jwt[].issuer.url`, byte-for-byte | All tokens rejected as from an untrusted issuer. |
| 6 | **Username composition** | apiserver `claimMappings.username.prefix` + mapped claim ≡ `subjects[].name` in every RBAC binding **inside the workload cluster** | Authentication works, authorization does not. Valid identity, zero permissions. |
| 7 | **OIDC scope ↔ claim** | Headlamp `oidc.scopes` must include the scope producing `claimMappings.username.claim` | Token lacks the mapped claim; username mapping fails. |
| 8 | **FQDN, DNS, and redirect URI** | Headlamp `hostname` ≡ `callbackURL` host ≡ a published DNS A record ≡ a redirect URI registered with the IdP | IdP refuses the redirect. The error appears on the provider's page, not in Kubernetes. |
| 9 | **Gateway class ↔ controller** | Headlamp `gatewayApi.gateway.className` ⇒ the `istio` addon (with ingress enabled) **or** `contour` installed | `Gateway` never programmed; Headlamp unreachable with no clear cause. |
| 10 | **DNS ↔ Gateway address** | The DNS record must point at the **Headlamp `Gateway`'s** own LoadBalancer IP, not the Istio ingress gateway's | Name resolves to the wrong endpoint. |
| 11 | **StorageClass name** | `storageClass` ≡ each `volumes[].storageClass` ≡ `vsphereOptions.availableStorageClasses[]` ≡ `defaultStorageClass` — **and all associated with the vSphere Namespace** | Nodes fail to provision, or workload PVCs pend indefinitely. |
| 12 | **Pod CIDR ↔ total nodes** | `pods.cidrBlocks` must supply one `/24` per node — all `autoscaler-node-group-max-size` values **plus** `controlPlane.replicas` — **plus a spare for upgrade surge** | Pods stay `Pending`; or a rolling upgrade stalls. Immutable, so the fix is a rebuild. |
| 13 | **`maxPods` ↔ VM class ↔ `systemReserved`** | Pod density must be supported by node CPU/memory and system reservations | Node instability, evictions, `NotReady` kubelets under load. |
| 14 | **`maxPods` ↔ per-node block** | `maxPods` must stay below the usable addresses in a per-node block (~254 for a `/24`) | Pod IP exhaustion on a single node while the cluster looks healthy. |
| 15 | **K8s version ↔ ClusterClass** | `topology.version` must be a `READY`+`COMPATIBLE` `kr`, supported by `topology.classRef.name` | Topology never reconciles. |
| 16 | **OS image annotation** | Identical `resolve-os-image` on the control plane and every machine deployment | Pools drift onto different OS images. |
| 17 | **API server FQDN ↔ certificate SANs** | Any name clients use must be listed in `kubernetes.endpointFQDNs` **and** published in DNS | TLS validation fails on that name; looks like a certificate problem, not a config gap. |
| 18 | **PSS level ↔ namespace labels** | A namespace whose workloads cannot meet the cluster default needs explicit PSA labels, or membership of `exemptions.namespaces` | Pods rejected at admission; `helm install --create-namespace` fails ([7.7.3](#773-securitypodsecuritystandard)). |
| 19 | **`maxPods` ↔ `systemReserved` ↔ VM class** | All three must be sized together | Node instability at density; or a node whose allocatable capacity cannot host the pods it admits. |
| 20 | **PSS level ↔ Istio scope** | If you adopt sidecar injection, `istioCNI.enabled: true` is required for PSS above `privileged` | Injected pods rejected at admission. |

---

## 10. Consolidated pitfalls

> **What this covers.** The failure modes this profile can produce, each as symptom → cause → fix,
> ordered by severity combined with how hard it is to diagnose.
>
> **Why it matters.** Most of these present at a different layer than their cause — a broken login
> that is really a missing RBAC binding, a stalled upgrade that is really an exhausted IP range,
> cluster instability that is really hypervisor contention. Recognising the symptom saves the search.

Ordered by severity combined with how hard the problem is to diagnose from its symptom.

### 1. A claim validation rule that fails open

| | |
| --- | --- |
| **Symptom** | None. The cluster works and the rule appears to enforce email verification. That is what makes it dangerous. |
| **Cause** | `claims.?email_verified.orValue(true) == true` substitutes `true` when the claim is **absent**, so a token omitting it is accepted. Only an explicit `false` is rejected. Since the username derives from the email claim, an unverified address becomes a cluster identity. |
| **Fix** | `orValue(false)` — fail closed. |
| **Principle** | Every CEL security predicate should use `orValue(false)`. If you cannot prove the condition holds, deny. |

### 2. An `AddonConfig` that is silently skipped

| | |
| --- | --- |
| **Symptom** | The addon deploys and runs, but none of your configured values took effect. |
| **Cause** | `spec.clusterName` and `spec.addonConfigDefinitionRef` are normally derived from the `<clusterName>-<addonName>` object name. If the name does not resolve — a typo, a renamed cluster, or using the `AddonInstall` name instead of the addon name — both stay empty and, per the API docs, *"the AddonConfig will skip the reconciliation."* |
| **Detect** | `kubectl get addonconfig <name> -n <ns> -o jsonpath='{.spec.clusterName}'` — empty means skipped. Also check `READY` on `kubectl get addonconfig` and `kubectl get clusteraddon`. |
| **Fix / prevent** | Set `spec.clusterName` and `spec.addonConfigDefinitionRef` **explicitly** in generated manifests. It also pins the addon release. |
| **Classic instance** | The `AddonInstall` named `cluster-prom` installs the addon `prometheus`, so the config must be `<cluster>-prometheus`. Naming it `-prom` matches nothing. |

### 3. OIDC client secret in plaintext

| | |
| --- | --- |
| **Symptom** | None until the credential is abused. |
| **Cause** | `oidc.clientSecret` inline in an `AddonConfig` is stored unencrypted in etcd, visible to anyone with read access, present in `kubectl get -o yaml`, and committed to any repository holding the manifest — including its history. |
| **Fix** | Reference a Kubernetes Secret instead. Check your addon's schema for the supported field (commonly an `existingSecret` or `clientSecretRef` pattern), and manage the Secret with sealed-secrets, external-secrets, or the **`vault-injector`** addon so the manifest stays committable. |
| **If already committed** | **Rotate the secret at the identity provider.** Removing it from the working tree does not remove it from Git history. |

### 4. Pod CIDR exhaustion blocks upgrades

| | |
| --- | --- |
| **Symptom** | Two forms. **(a)** Nodes join and report `Ready` but pods stay `Pending`, with no error mentioning addressing. **(b)** Worse: a rolling upgrade stalls partway, leaving a mixed-version cluster, because the surge node has no pod CIDR block. |
| **Cause** | The pod CIDR is a pool of fixed-size per-node blocks (`/24` by default), not a flat pool of IPs. A `/20` yields 16 blocks for the whole cluster. Every rolling replacement is surge-then-remove, so **one block must always be free**. |
| **Fix** | None in place — **the pod CIDR is immutable.** A cluster that has consumed all blocks cannot even be upgraded out of the situation. |
| **Prevention** | Size for maximum plausible nodes **plus 2–3 spare blocks**. Use a `/16`. Consider raising `maxPods` for density instead of adding nodes. See [7.1.1](#711-pod-cidr-sizing-node-blocks-maxpods-and-the-upgrade-spare). |
| **If already deployed** | Cap total autoscaler `max-size` below `blocks − controlPlane.replicas − reserve`, and verify a free block before every rolling operation. |

### 5. Missing RBAC after OIDC login

| | |
| --- | --- |
| **Symptom** | Users authenticate successfully and then see permission errors everywhere. The UI looks broken. |
| **Cause** | Authentication is not authorization. A valid identity with no RBAC binding has zero permissions. |
| **Fix** | Create bindings **inside the workload cluster** naming the fully composed username — prefix plus claim. Verify the exact string with `kubectl auth whoami`. See [Appendix B.1](#b1-rbac-for-oidc-identities). |

### 6. A Gateway with no controller

| | |
| --- | --- |
| **Symptom** | Headlamp pods run and are healthy, but the URL is unreachable. |
| **Cause** | `gatewayApi.gateway.className` names a controller that does not exist — neither the `istio` addon (with `gateways.ingress.enabled: true`) nor `contour` is installed. |
| **Detect** | `kubectl get gateway -A` — `PROGRAMMED` is not `True` and `ADDRESS` is empty. |
| **Fix** | Install a gateway controller addon. The Gateway API **CRDs** are platform-provided, so their presence does not imply a controller exists. |

### 7. Hostname, DNS, and redirect URI drift

| | |
| --- | --- |
| **Symptom** | Login fails at the identity provider with a redirect-mismatch error, or the URL does not resolve. |
| **Cause** | `hostname`, `callbackURL`, the DNS record, and the IdP's registered redirect URI must all agree. Any hostname derived from the LoadBalancer IP — including wildcard-DNS shortcuts — breaks all four at once whenever that IP changes. |
| **Fix** | Use a stable FQDN with a reserved LoadBalancer IP. If you want the record maintained automatically, see [Appendix D.3](#d3-dns-automation-external-dns). |

### 8. Prometheus operator with no `Prometheus` CR

| | |
| --- | --- |
| **Symptom** | Monitoring appears deployed — operator, exporters, and Alertmanager all running — but there are no metrics to query and no alerts ever fire. |
| **Cause** | `prometheus: false` deploys the operator without a server. The operator does nothing until you create a `Prometheus` custom resource. |
| **Detect** | `kubectl get prometheus,alertmanager,servicemonitor -A` → `No resources found`. |
| **Fix** | Create the `Prometheus` CR ([Appendix B.3](#b3-a-prometheus-instance-via-the-operator)), or point a central Prometheus at this cluster's exporters. |

### 9. The IdP emits no `groups` claim

| | |
| --- | --- |
| **Symptom** | Group-based RBAC bindings never apply. Individual user bindings work. |
| **Cause** | Many providers do not include a `groups` claim by default — some need a specific scope, some need the claim added to the token explicitly, some need a directory-API integration. |
| **Fix** | **Inspect a real ID token before designing RBAC around groups.** Then either configure the IdP to emit the claim, use an identity broker (Dex, Keycloak) to enrich tokens, or fall back to user bindings. |

### 10. Pod Security versions pinned to `latest`

| | |
| --- | --- |
| **Symptom** | Workloads that deployed cleanly before a cluster upgrade are rejected at admission afterwards. |
| **Cause** | `enforceVersion: latest` tracks whatever the running Kubernetes version defines, and PSS definitions tighten over time. |
| **Fix** | Pin an explicit version, then raise it deliberately as a separate, tested change. |

### 11. StorageClass or VM class not associated with the vSphere Namespace

| | |
| --- | --- |
| **Symptom** | Machines never provision, or PVCs pend indefinitely with an unhelpful event — long after the cluster looked healthy. |
| **Cause** | The resource exists in vCenter but is not added to the vSphere Namespace. The `Cluster` object is accepted regardless; validation happens at consumption time. |
| **Fix** | Verify with `kubectl describe ns <ns> \| grep storageclass` and `kubectl get virtualmachineclass` **before** applying. |

### 12. `best-effort` VM classes in production

| | |
| --- | --- |
| **Symptom** | Intermittent `NotReady` kubelets, etcd leader-election churn, failed liveness probes, random timeouts. Reads as "Kubernetes is flaky." |
| **Cause** | `best-effort` classes carry no resource reservation. Under vSphere contention the hypervisor delivers less than the node advertises, while Kubernetes schedules against the advertised capacity. |
| **Fix** | `guaranteed-*` classes, always for the control plane. |
| **Why it is worth calling out** | The failure presents at the wrong layer, so teams spend days debugging Kubernetes for a hypervisor resource problem. |

### 13. Clock skew breaks OIDC intermittently

| | |
| --- | --- |
| **Symptom** | Authentication fails for some users some of the time, resolving on retry. |
| **Cause** | ID tokens carry `exp`/`iat`/`nbf` claims validated against the API server's clock. |
| **Fix** | Verify NTP is actually synchronising on every node — a configured but unreachable server is worse than none. Alert on skew. |

### 14. etcd read-only after exceeding its quota

| | |
| --- | --- |
| **Symptom** | Every write fails; reads work. Presents as a total outage. |
| **Cause** | A `NOSPACE` alarm on exceeding `maximumDBSizeGiB` put etcd read-only. It does not clear on its own. |
| **Fix** | Compact, defragment, disarm the alarm — all needing etcd-level access when the control plane appears dead. |
| **Prevention** | Alert on etcd DB size at 60–70% of the quota. Consider lowering `terminatedPodGCThreshold` on high-churn clusters. |

### 15. Istio `pilot.replicas` fighting its own HPA

| | |
| --- | --- |
| **Symptom** | istiod replica count oscillates; reconcile loops churn. |
| **Cause** | A static `replicas` alongside `autoscaling.enabled: true`. The HPA owns `spec.replicas`. |
| **Fix** | Remove the static value; express intent through `minReplicas`/`maxReplicas`. |

### 16. `maxPods` raised without matching node resources

| | |
| --- | --- |
| **Symptom** | Nodes accept many pods, then become unstable — evictions, `NotReady`, slow kubelet responses. |
| **Cause** | Pod density raised without CPU/memory to match, or without `systemReserved` protecting system daemons. |
| **Fix** | Size the VM class to the pod count and override `resourceConfiguration.systemReserved` — the automatic reservation is node-size derived and does not grow with `maxPods` ([7.13](#713-37-variables-not-used-in-the-sample)). Keep sustained density below ~220–230 on a `/24` block. |

---

## 11. Verification and troubleshooting

> **What this covers.** Nine checks, working outward from the Supervisor to identity and IP headroom,
> each with the command to run and what a healthy result looks like.
>
> **Why it matters.** Checking in order localises a failure instead of leaving you guessing among five
> components. Several of these checks also answer questions the object status does not surface.

Work outward from the Supervisor. Each layer depends on the one before, so checking in order
localises a failure instead of leaving you guessing among five components.

### Layer 1 — Supervisor: does the cluster exist?

```bash
kubectl get cluster,machinedeployment,machine -n <VSPHERE_NAMESPACE>
kubectl describe cluster <CLUSTER_NAME> -n <VSPHERE_NAMESPACE>   # conditions live here
```

**Healthy:** `Cluster` is `Provisioned` with `AVAILABLE=True`; every `Machine` is `Running`;
`CP AVAILABLE` matches `CP DESIRED`.

**If machines are stuck:** check that the `resolve-os-image` selector resolves, that the VM class and
storage policy are associated with the namespace, and that `topology.version` names a `READY` +
`COMPATIBLE` `kr`.

### Layer 2 — Addons: installed, bound, and ready?

**The one command that shows everything:**

```bash
kubectl get clusteraddon -n <VSPHERE_NAMESPACE>
```

`READY=True` on every row, with the resolved `RELEASE` visible. This also shows the
platform-installed addons, so it confirms the autoscaler and Gateway API are present.

**Then check config binding** — the silent-skip failure in
[Pitfall 2](#2-an-addonconfig-that-is-silently-skipped):

```bash
# every AddonConfig, with the cluster and definition it resolved to
kubectl get addonconfig -n <VSPHERE_NAMESPACE> \
  -o custom-columns='CONFIG:.metadata.name,CLUSTER:.spec.clusterName,DEFINITION:.spec.addonConfigDefinitionRef.name'
```

**Any empty `CLUSTER` or `DEFINITION` means that config is being skipped** and the addon is running
on schema defaults.

```bash
kubectl get addoninstall -n <VSPHERE_NAMESPACE> \
  -o custom-columns='INSTALL:.metadata.name,ADDON:.spec.addonRef.name'
```

Each `AddonConfig` name must be `<CLUSTER_NAME>-<ADDON>` using the **`ADDON`** column.

### Layer 3 — Guest cluster: did the packages reconcile?

Switch to the workload cluster context. Addons are **Carvel packages**, so this is where their real
status lives:

```bash
kubectl get pkgi -A
```

**Healthy:** every `PackageInstall` shows `Reconcile succeeded` in `DESCRIPTION`.

**If one failed**, that is where the error is:

```bash
kubectl describe pkgi <name> -n vmware-system-tkg
kubectl logs -n <kapp-controller-namespace> -l app=kapp-controller --tail=100
```

> **Do not look for `HelmRelease` objects** — the addons do not use them. `kubectl get helmrelease -A`
> returns nothing unless you have created your own via the helm-controller addon.

### Layer 4 — Istio and the ingress gateway

```bash
kubectl get pods -n istio-system      # istiod
kubectl get pods -n istio-ingress     # the gateway
kubectl get svc -n istio-ingress      # EXTERNAL-IP must be populated

# confirm the deployment scope: blank columns = gateway-only, no mesh
kubectl get ns -L istio-injection,istio.io/dataplane-mode
```

**If `EXTERNAL-IP` is `<pending>`:** a load-balancer problem — VIP pool exhaustion or LB
misconfiguration — not an Istio problem.

### Layer 5 — Gateway API, Headlamp, and TLS

```bash
# CRDs are platform-provided; confirm anyway
kubectl get crd | grep gateway.networking.k8s.io

# is the Gateway programmed, and what is its address?
kubectl get gateway,httproute -A
```

**Healthy:** `PROGRAMMED=True` with a populated `ADDRESS`. **That address is what DNS must point
to** — it is the Gateway's own LoadBalancer, not the Istio ingress gateway's:

```bash
kubectl get svc -n headlamp        # headlamp-gateway-<class> LoadBalancer + EXTERNAL-IP
```

**Check what is terminating TLS and who issued it:**

```bash
kubectl get certificate,issuer -n headlamp

# which listener terminates TLS, with what, and for which hostname
JP='{range .spec.listeners[*]}{.name}{"  "}{.protocol}{":"}{.port}'
JP+='{"  hostname="}{.hostname}{"  tls="}{.tls.mode}'
JP+='{"  secret="}{.tls.certificateRefs[0].name}{"\n"}{end}'
kubectl get gateway headlamp-gateway -n headlamp -o jsonpath="$JP"
```

```
https-headlamp  HTTPS:443  hostname=headlamp.k8s.example.com  tls=Terminate  secret=headlamp-tls-cert
```

A `READY=True` `Certificate` from a self-signed `Issuer` is the addon default. For production you
want an issuer backed by a trusted CA ([Appendix B.2](#b2-trusted-tls-for-the-headlamp-gateway)).

**Then confirm the chain from outside the cluster:**

```bash
openssl s_client -connect headlamp.k8s.example.com:443 -servername headlamp.k8s.example.com </dev/null 2>&1 | head -20
```

### Layer 6 — Storage

```bash
kubectl get storageclass                                     # in the workload cluster

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

kubectl get pvc storage-smoke-test          # expect Bound within seconds
kubectl describe pvc storage-smoke-test     # events explain any delay
kubectl delete pvc storage-smoke-test
```

Omitting `storageClassName` is deliberate — it verifies `defaultStorageClass` is in effect.
**If it pends:** the class is almost certainly not associated with the vSphere Namespace, or the
storage policy is non-compliant for your host count.

### Layer 7 — Identity: what username does the API server see?

The critical question is not "can I log in" but **"what username does the API server think I am"** —
everything in RBAC depends on that exact string.

```bash
kubectl auth whoami
kubectl auth whoami --token="<ID_TOKEN>"     # or with a token directly
```

**Expect** `<prefix><claim value>` — e.g. `oidc:user@example.com`.

| Observation | Likely cause |
| --- | --- |
| Rejected: invalid issuer | `issuer.url` does not match the token's `iss` byte-for-byte |
| Rejected: invalid audience | The client ID is not in the API server's `audiences` list |
| Username missing its prefix, or unexpected | `claimMappings.username` does not match a claim actually present in the token |
| Groups list empty | Your IdP is not emitting a `groups` claim ([Pitfall 9](#9-the-idp-emits-no-groups-claim)) |
| Intermittent failures that resolve on retry | Clock skew ([Pitfall 13](#13-clock-skew-breaks-oidc-intermittently)) |
| Rejected with a TLS error | The IdP's certificate is not trusted — set `issuer.certificateAuthority` |

**Inspect the actual token** when claims are in doubt — do not guess at what your IdP emits:

```bash
# the JWT payload is base64url-encoded; translate the alphabet and pad it
echo "<ID_TOKEN>" | cut -d. -f2 | tr '_-' '/+' | sed 's/$/===/' | base64 -d 2>/dev/null; echo
```

```
{"email":"user@example.com","email_verified":true,"iss":"https://idp.example.com", ...}
```

Then verify authorization **separately** — authentication succeeding says nothing about permissions:

```bash
kubectl auth can-i --list
kubectl auth can-i get pods --all-namespaces
```

An authenticated user with no binding can do nothing. That is
[Pitfall 5](#5-missing-rbac-after-oidc-login), not a broken login.

### Layer 8 — Observability: is anything actually evaluating rules?

```bash
kubectl get pods -n tanzu-system-monitoring
kubectl get prometheus,alertmanager,servicemonitor,prometheusrule -A
```

**`No resources found` on the CRs means the operator is idle** — exporters are producing metrics and
nothing is scraping them ([Pitfall 8](#8-prometheus-operator-with-no-prometheus-cr)). Know where rule
evaluation happens: a `Prometheus` CR here, or an external Prometheus scraping this cluster.

### Layer 9 — Pod CIDR headroom

Run this **before** any upgrade or rolling change:

```bash
# what each node holds
kubectl get nodes -o custom-columns='NODE:.metadata.name,POD_CIDR:.spec.podCIDR,MAXPODS:.status.capacity.pods'

# how many blocks are consumed
kubectl get nodes -o jsonpath='{range .items[*]}{.spec.podCIDR}{"\n"}{end}' | sort -u | wc -l
```

Compare against your total — 16 for a `/20`, 256 for a `/16`. **If the count equals the total, do not
start a rolling operation** ([Pitfall 4](#4-pod-cidr-exhaustion-blocks-upgrades)).

---

## 12. Day-2 lifecycle

> **What this covers.** Upgrades and their sequencing, which field edits replace nodes, addon version
> management, certificate rotation, etcd growth, and tightening Pod Security after the fact.
>
> **Why it matters.** Several values in the profile only reveal their cost at upgrade time — a
> single-replica control plane, an undersized pod CIDR, a 30Gi volume. This section is where those
> bills come due, and where the pre-flight checks live.

### Upgrade sequencing

The dependency runs bottom-up. Going the other way produces objects referencing things that do not
exist yet.

1. **Supervisor / VKS platform** — first; it provides everything below.
2. **Confirm the new ClusterClass and `kr` releases are available and compatible**
   (`kubectl get kr`, `kubectl get clusterclass -n vmware-system-vks-public`).
3. **Diff the new ClusterClass variable schema against the old one.** Variables can be added,
   renamed, or removed between generations. Do this *before* the upgrade.
4. **Diff the two `kr` objects.** A Kubernetes *patch* bump can carry a CNI upgrade, a kapp-controller
   minor bump, and an etcd patch — verified between two v1.36 releases in
   [Appendix E.2](#e2-specbootstrappackages--the-platform-addons-pinned-to-the-release). There is a
   ready-made change report in [Appendix E.4](#e4-a-pre-upgrade-change-report). Confirm your
   `resolve-os-image` annotation still resolves against the target release while you are there.
5. **`topology.version`** — one minor version at a time. Skipping minors is unsupported.
6. **`topology.classRef.name`** — when moving to a new ClusterClass generation.
7. **Addon releases** — last.

**Pre-flight checklist:**

| Check | Why |
| --- | --- |
| A **free pod CIDR block** exists | Surge nodes need one; without it the rollout stalls ([Layer 9](#layer-9--pod-cidr-headroom)) |
| The **`kr` change report** has been read | A patch bump may move the CNI and kapp-controller ([Appendix E.4](#e4-a-pre-upgrade-change-report)) |
| PodDisruptionBudgets exist for critical workloads | Otherwise a drain can take all replicas down at once |
| The pool can lose a node | Capacity headroom for the surge and the drain |
| A maintenance window, if `controlPlane.replicas: 1` | Every control-plane step is an API outage |
| Backups are current, if you run them, and **a restore has been tested** | See [Appendix D.1](#d1-backup-and-restore-velero) |
| Non-production upgraded first | Always |

### Which changes replace nodes

| Change | Effect |
| --- | --- |
| `topology.version` | Rolling replacement of all nodes, control plane first |
| `topology.classRef.name` | Rolling replacement |
| `vmClass` | Rolling replacement of the affected pool |
| `volumes` (add or resize) | **Rolling replacement** — not in-place |
| `kubeletConfiguration` (including `maxPods`) | **Rolling replacement** — the schema states this explicitly |
| `resourceConfiguration.systemReserved` | Rolling replacement |
| `node.labels` / `node.taints` | Rolling replacement |
| `resolve-os-image` annotation | Rolling replacement |
| `replicas` / autoscaler bounds | Scale operation only; existing nodes untouched |
| `certificateRotation`, `etcdConfiguration`, PSS, `apiServerConfiguration` | Control-plane reconfiguration; no worker replacement |
| `clusterNetwork` CIDRs | **Immutable** — requires a new cluster |
| `bootstrapAddons.cniRef` | **Effectively immutable** — requires a new cluster |

**Every rolling replacement needs a free pod CIDR block.** That single fact ties most of this section
together.

### Addon lifecycle

- **Pin and promote deliberately.** Naming the `acd` explicitly in `spec.addonConfigDefinitionRef`
  pins the release; otherwise the platform resolves one and it can move on upgrade.
- **Re-verify your `AddonConfig` against the new schema after any addon upgrade.** Fields can be
  added, renamed, or relocated, and a field at the wrong nesting level is not applied — so an upgrade
  can quietly revert a setting to its default. Diff against a fresh
  `vcf addon available-releases get` or `kubectl get acd ... -o yaml`.
- **Deletion:** `owned-for-deletion: "true"` ties each config's lifecycle to its addon and cluster.
  If a manifest lacked it, sweep for orphans — because binding is name-derived, an orphan will be
  silently adopted by a future cluster reusing the name:
  ```bash
  kubectl get addonconfig -n <VSPHERE_NAMESPACE>
  ```
- **Adding an addon is a normal day-2 operation.** Apply the `AddonInstall` (and optional
  `AddonConfig`); no ordering to observe, no cluster restart.

### Certificate rotation

Rotation is silent when it works, which makes it an untested assumption. Confirm at least once that
it has run — check control-plane certificate expiry on a node and verify the dates advance after the
renewal window passes. Then **alert on certificate expiry as a backstop**, so a silently broken
rotation surfaces months before it becomes an outage.

### etcd growth

Scrape etcd with the Prometheus stack you already have and alert on database size at 60–70% of
`maximumDBSizeGiB`. Watch the **trend**, not just the threshold: steady growth usually means
excessive object churn, a controller writing status in a loop, or auto-compaction not running.
Lowering `kubeControllerManagerConfiguration.terminatedPodGCThreshold` helps on high-churn clusters.

### Tightening Pod Security

A planned migration, with step 1 completely risk-free. Full sequence in
[7.7.3](#773-securitypodsecuritystandard). Raise `audit` and `warn` to `restricted` today, leave
`enforce` alone, and you get the full inventory of what would break at zero risk. Use
`exemptions.namespaces` only for platform components that cannot run enforced at any level, and
per-namespace PSA labels — not exemptions — for workload namespaces that need a lower level.

### OIDC and hostname changes

If the Gateway IP or FQDN changes, four things must move together or login breaks:

1. Headlamp `hostname`
2. Headlamp `callbackURL`
3. The DNS record
4. The authorized redirect URI registered with the identity provider

Reserving a static LoadBalancer IP removes the trigger entirely. Automating step (3) is possible —
see [Appendix D.3](#d3-dns-automation-external-dns).

Also review periodically:

- **IdP client secret rotation** — plan it, and prefer a Secret reference so rotation does not mean
  editing a manifest.
- **RBAC drift** — audit bindings against current staff. This is the argument for group-based
  bindings once your IdP emits groups.

### Backup and restore

Not part of the base manifest, and whether it belongs in-cluster depends on how you already protect
workloads — many environments handle this at the vSphere or storage layer instead. If you do want it
in-cluster, see [Appendix D.1](#d1-backup-and-restore-velero). Either way, **test a restore**: an
untested backup is a hypothesis.

Independently of any backup tooling, keep these manifests in version control. They are the declarative source of truth for the
cluster, and rebuilding from them is often faster than repairing — which matters for the immutable
fields in [section 8](#8-cluster-decisions-you-cannot-change-later), where rebuild is the only remedy.

---

## Appendix A — Production baseline manifest

> **What this is.** The same stack, hardened and copy-pasteable — the counterpart to
> [`reference-profile.yaml`](./reference-profile.yaml).
>
> **Why it matters.** It turns the recommendations scattered through sections 6–12 into a single
> artifact you can start from, with a change table explaining every deviation from the reference
> profile.

A hardened version of the same stack. Substitute the placeholders, verify version strings and class
names against your environment, and read the inline notes — a few settings depend on schema field
names you should confirm with the discovery commands rather than trust from a document.

**Scope:** these objects are applied to the **vSphere Namespace on the Supervisor**. Objects that
belong *inside* the workload cluster are in
[Appendix B](#appendix-b--additional-objects-and-hardening).

**What changed, and why:**

| Change | Reason |
| --- | --- |
| Pod CIDR `/20` → `/16` | `/20` gives only 16 per-node blocks; a 3 + 10 node shape plus surge nearly exhausts it, and an exhausted CIDR blocks upgrades. Immutable. |
| `controlPlane.replicas` 1 → 3 | etcd quorum; upgrades stop being outages |
| `best-effort-*` → `guaranteed-*` | Reserved resources; removes contention-induced instability that presents as Kubernetes flakiness |
| Control plane → `guaranteed-large` | etcd is latency-sensitive and intolerant of starvation |
| PSS `privileged` → `enforce: baseline`, `audit`/`warn: restricted` | Real admission control, plus a zero-risk inventory of the path to `restricted` |
| PSS versions `latest` → `v1.36` | Upgrades no longer change admission policy silently |
| Added `podSecurityStandard.exemptions` | Narrow, explicit exceptions instead of a weakened cluster default |
| Added `kubeletConfiguration` log bounds and `podPidsLimit` | Protects the kubelet volume; limits fork-bomb exposure |
| Added `resourceConfiguration.systemReserved` | Explicitly protects system daemons |
| Added `node.labels` | Declarative labels survive node replacement; hand-applied labels do not |
| Added `kubernetes.endpointFQDNs` | API server reachable by a stable name in the same domain as the UI |
| `maxPods` set explicitly | Explicit beats implicit, and documents the density decision |
| `volumes` 30Gi → 100Gi | Resizing later is a rolling node replacement |
| `etcdConfiguration` 4 → 8 GiB | Headroom for a CRD- and event-heavy addon stack |
| `orValue(true)` → `orValue(false)` | The claim validation rule now fails closed |
| `clientSecret` inline → secret reference | Keeps the credential out of etcd and Git |
| `pilot`: static `replicas` removed, `minReplicas` → 2 | No HPA conflict; istiod no longer a single point of failure |
| Ingress `minReplicas` 1 → 2 | Survives a reschedule without an ingress outage |
| `accessLogFile` `""` → `/dev/stdout` | Restores L7 debugging at the gateway |
| `prometheus` operator + explicit `Prometheus` CR (Appendix B.3) | Rule evaluation actually exists |
| Worker `min-size` 1 → 3 | Capacity redundancy |
| `priorityClassName` `""` → `system-cluster-critical` | The Helm controllers stop being evictable |
| `meshID` cluster name → mesh name | Leaves multi-cluster federation available |

```yaml
# =============================================================================
#  Addons. Order is irrelevant — all are reconciled independently.
# =============================================================================
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
  # Name contract: <clusterName>-<addonRef.name>. Set spec.clusterName and
  # spec.addonConfigDefinitionRef explicitly if you generate these manifests —
  # it removes the silent-skip failure mode and pins the addon release.
  name: workload-vsphere-vks2-helm-controller
  namespace: <VSPHERE_NAMESPACE>
spec:
  values:
    # These reconcile your own HelmReleases. Do not leave them evictable.
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
      # Operator-managed: you declare the Prometheus CR yourself (Appendix B.3),
      # which gives you control over retention, storage, and scrape config.
      # NOTHING SCRAPES until that CR exists.
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
      # Deployed for L4/L7 ingress only — no service mesh, no sidecar injection.
      ambientMode:
        enabled: false
      # Not required without sidecar injection, so it imposes no Pod Security
      # constraint here. If you later label namespaces for injection, set this
      # to true AT THE SAME TIME or injected pods will need `privileged`.
      istioCNI:
        enabled: false
      gateways:
        ingress:
          enabled: true
          namespace: istio-ingress
          autoscaling:
            enabled: true
            minReplicas: 2        # 1 = ingress outage on every reschedule
            maxReplicas: 5
      pilot:
        # No static `replicas` — the HPA owns it. Setting both causes churn.
        autoscaling:
          enabled: true
          minReplicas: 2          # istiod programs all gateways; do not run one
          maxReplicas: 4
      meshConfig:
        accessLogFile: "/dev/stdout"   # your ingress request log
        enableTracing: false           # true also needs a tracing provider
        # Name the MESH, not the cluster — federation needs a shared meshID.
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
    # A DNS name you control, published as an A record pointing at the Headlamp
    # Gateway's own LoadBalancer IP (NOT the istio-ingress gateway's). Reserve
    # that IP so this value is stable.
    hostname: headlamp.k8s.example.com
    gatewayApi:
      enabled: true
      gateway:
        # REQUIRES a gateway controller: the istio addon above, or contour.
        # The Gateway API CRDs are platform-provided; a controller is not.
        className: istio
        create: true
        name: headlamp-gateway
        # The addon creates a cert-manager Issuer + Certificate automatically,
        # but SELF-SIGNED. Point this at a trusted-CA certificate for production
        # (Appendix B.2); check your release's schema for the TLS field names.
    oidc:
      enabled: true
      # Any OIDC-compliant provider. Must match the token's `iss` byte-for-byte.
      issuerURL: https://idp.example.com
      # MUST equal the apiserver's extraAuthentication audience below, or users
      # log in successfully and then get permission errors on every API call.
      clientID: <OIDC_CLIENT_ID>
      # DO NOT inline the secret. Reference a Kubernetes Secret — check your
      # release's schema for the supported field (commonly `existingSecret` /
      # `clientSecretRef`) and manage it with sealed-secrets, external-secrets,
      # or the vault-injector addon so this file stays committable.
      clientSecret: <OIDC_CLIENT_SECRET>
      # Must be registered as an authorized redirect URI with your IdP.
      callbackURL: https://headlamp.k8s.example.com/oidc-callback
      scopes:
      - openid
      - email      # load-bearing: feeds the username claim mapping
      - profile
---
# =============================================================================
#  Cluster
# =============================================================================
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
metadata:
  name: workload-vsphere-vks2
  namespace: <VSPHERE_NAMESPACE>
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
      # /16 = 256 per-node /24 blocks = 255 usable nodes after reserving one for
      # upgrade surge. IMMUTABLE: the pool is carved into fixed /24 blocks per
      # node, and every rolling replacement needs a FREE block for its surge
      # node or it stalls. A /20 gives only 16 blocks total.
      # Reserved 240/4 space avoids RFC1918 collisions — validate it against
      # your CNI, firewalls, and load balancers first.
      - 240.0.0.0/16
    serviceDomain: cluster.local
    services:
      cidrBlocks:
      - 240.1.0.0/20                    # no overlap with the pod range above
  topology:
    classRef:
      name: builtin-generic-v3.7.0
      namespace: vmware-system-vks-public
    controlPlane:
      metadata:
        annotations:
          run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
      replicas: 3                       # etcd quorum; upgrades become rolling
      variables:
        overrides:
        - name: vmClass
          # Guaranteed, and larger than the workers: etcd under CPU contention
          # causes leader-election churn that looks like cluster-wide instability.
          value: guaranteed-large
    variables:
    - name: storageClass
      value: <STORAGE_CLASS>
    - name: vmClass
      # guaranteed-* reserves CPU and memory. best-effort-* is acceptable for
      # dev/test only: it presents as Kubernetes flakiness, not resource pressure.
      # Size this to your maxPods setting below.
      value: guaranteed-medium
    - name: volumes
      # Resizing these later is a ROLLING NODE REPLACEMENT. Size generously now,
      # and scale with pod density — a 250-pod node needs far more than a 110-pod one.
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
        # Use per-pool overrides to differentiate pools (see 7.13).
        labels:
          workload-type: general
          environment: production
    - name: resourceConfiguration
      value:
        systemReserved:
          # Protects kubelet, runtime, and OS from being starved by pods.
          # `automatic` is correct at the default maxPods of 110 — it derives a
          # tiered reservation from node size. If you raise maxPods, that tier
          # does NOT grow with pod count, so override explicitly (add ~11 MiB
          # per pod above 110, and ~400m CPU) AND move to a larger VM class.
          # See 7.13 for the formulas, sources, and per-class arithmetic.
          automatic: true
    - name: kubernetes
      value:
        # FQDN aliases for the API server, in the same domain as the UI, so the
        # whole cluster is reachable by name. Adds the name to the cert SANs;
        # you still publish the DNS record yourself.
        endpointFQDNs:
        - api.k8s.example.com
        certificateRotation:
          enabled: true
          renewalDaysBeforeExpiry: 90
        etcdConfiguration:
          # The etcd volume must actually have this space. Alert at 60-70%.
          maximumDBSizeGiB: 8
        kubeControllerManagerConfiguration:
          terminatedPodGCThreshold: 500   # bounds etcd growth on churn
        security:
          # NOTE: minimumTLSProtocol and tlsCipherSuites are deliberately NOT set
          # here. Raising the TLS floor or narrowing ciphers breaks clients,
          # webhooks, and integrations that have not been validated against it,
          # and the failures are hard to attribute. Change only against a
          # specific compliance requirement, with a tested client inventory.
          podSecurityStandard:
            # enforce=baseline blocks known privilege escalations.
            # audit/warn=restricted inventories the path to `restricted`
            # WITHOUT rejecting anything — risk-free reconnaissance.
            enforce: baseline
            enforceVersion: v1.36
            audit: restricted
            auditVersion: v1.36
            warn: restricted
            warnVersion: v1.36
            deactivated: false
            # `baseline` is the pragmatic cluster default: it blocks real
            # privilege escalation while most application charts still install
            # unmodified. `restricted` is a per-NAMESPACE goal you raise
            # deliberately, via namespace labels — not a cluster-wide start.
            #
            # EXEMPTIONS DISABLE PSS ENTIRELY for the listed namespaces (no
            # enforce, audit, OR warn). Use them only for platform components
            # that cannot run enforced at any level. For workload namespaces
            # that need a LOWER level, use namespace labels instead — see 7.7.3.
            # Verify these names against your own cluster.
            exemptions:
              namespaces:
              - tanzu-system-monitoring     # node-exporter needs host access
              - vmware-system-antrea        # CNI agent
          resourceQuotaConfiguration:
            enabled: true
        apiServerConfiguration:
          logs:
            format: json
          # Generic OIDC — any compliant provider. Supports human login via
          # Headlamp and OIDC-aware kubectl plugins.
          extraAuthentication:
            jwt:
            - issuer:
                url: https://idp.example.com
                audiences:
                - <OIDC_CLIENT_ID>
                # For an internal IdP behind an enterprise or self-signed CA,
                # supply the CA bundle here or every token fails validation:
                # certificateAuthority: |
                #   -----BEGIN CERTIFICATE-----
                #   ...
                # And select the network path the apiserver uses to reach it:
                # egressSelectorType: cluster
              claimMappings:
                username:
                  claim: email
                  # NEVER empty: prevents an external identity impersonating
                  # system: subjects. Changing it orphans ALL RBAC bindings.
                  prefix: "oidc:"
                groups:
                  # VERIFY your IdP actually emits this claim — many do not by
                  # default. Inspect a real token before designing group RBAC.
                  claim: groups
                  prefix: "oidc-groups:"
              claimValidationRules:
              # orValue(FALSE) — fails closed. A token omitting email_verified
              # is REJECTED. orValue(true) would silently accept it.
              - expression: 'claims.?email_verified.orValue(false) == true'
                message: "email must be verified"
        kubeletConfiguration:
          logging:
            format: json
          # Default 110; documented maximum 250. Raising it buys pod density
          # WITHOUT consuming more pod CIDR blocks — but size the VM class and
          # systemReserved to match, and stay under ~220 sustained on a /24.
          # Changing this requires a machine rollout.
          maxPods: 110
          podPidsLimit: 4096              # limits fork-bomb exposure
          containerLogMaxSizeMiB: 50      # protects the kubelet volume
          containerLogMaxFiles: 5
    - name: bootstrapAddons
      value:
        cniRef:
          # Effectively immutable — changing the CNI means a new cluster.
          # Alternatives in this catalogue: calico, cilium.
          name: antrea
          namespace: vmware-system-vks-public
    - name: vsphereOptions
      value:
        persistentVolumes:
          # Every class must be associated with the vSphere Namespace, or PVCs
          # pend indefinitely long after the cluster looks healthy.
          availableStorageClasses:
          - <STORAGE_CLASS>
          defaultStorageClass: <STORAGE_CLASS>
    - name: osConfiguration
      value:
        ntp:
          # A functional prerequisite: TLS validity windows, etcd leases, and
          # OIDC exp/iat claims all depend on synchronised clocks. Use the same
          # sources as vCenter and ESXi. Alert on skew.
          servers:
          - ntp1.example.com
          - ntp2.example.com
        sshd:
          banner: '---AUTHORIZED ACCESS ONLY--- Use of this system is restricted to
            authorized users for authorized activities only. '
    # Must be the NAME of a kr that is READY=True and COMPATIBLE=True.
    # Note the TRIPLE dash. Verify with: kubectl get kr
    version: v1.36.1---vmware.4-vkr.5
    workers:
      machineDeployments:
      - class: node-pool
        metadata:
          annotations:
            # The Cluster Autoscaler is installed by the platform, so these are
            # effective. Do not also set `replicas` — they would fight.
            cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "3"
            cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "10"
            run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
        name: node-pool-1
```

> **Node budget check for this manifest:** 3 control plane + 10 workers = 13 nodes, plus a surge
> block = 14 blocks. On the `/16` above that is 14 of 256 — ample. On a `/20` it would be 14 of 16,
> which is why the CIDR changed.

---

## Appendix B — Additional objects and hardening

> **What this is.** The objects the reference profile is missing, applied **inside** the workload
> cluster rather than to the vSphere Namespace.
>
> **Why it matters.** Three gaps make the difference between a stack that runs and a stack that
> works: OIDC without RBAC grants nothing, a self-signed certificate warns every user, and a
> monitoring operator with no `Prometheus` CR collects nothing.

The base manifest has three gaps: it authenticates without authorizing, it accepts a self-signed
certificate for the UI, and it installs a monitoring operator without giving it anything to do.
This appendix closes them.

> **Context.** Unlike Appendix A, these objects are applied **inside the workload cluster**, not
> to the vSphere Namespace. Switch your `kubectl` context first.

### B.1 RBAC for OIDC identities

The most important addition. Without it, OIDC login succeeds and grants nothing — the UI looks
broken.

The subject name is the **fully composed username**: the `prefix` from
`claimMappings.username.prefix` concatenated with the mapped claim value.

```yaml
# Applied INSIDE the workload cluster.
# subjects[].name = claimMappings.username.prefix + the mapped claim value.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-platform-admins
subjects:
- kind: User
  name: "oidc:platform-admin@example.com"     # note the "oidc:" prefix
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
---
# Least privilege for everyone else. Prefer namespace-scoped RoleBindings where
# the role does not genuinely need cluster scope.
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
# Once your IdP emits a groups claim, bind groups instead of users.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-group-platform-team
subjects:
- kind: Group
  name: "oidc-groups:platform-team"           # groups prefix + group name
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

| Guidance | Detail |
| --- | --- |
| **Get the prefix right** | `oidc:user@example.com`, not `user@example.com`. A binding without the prefix silently matches nobody. **Always verify with `kubectl auth whoami`** rather than assuming what the API server derived. |
| **Bind groups once you can** | Per-user bindings do not scale and go stale as people join and leave. Groups are the correct unit — but confirm your IdP emits the claim first ([Pitfall 9](#9-the-idp-emits-no-groups-claim)). |
| **Avoid `cluster-admin`** | Grant it to a small, named set of break-glass identities. Everyone else gets `view`, `edit`, or a purpose-built role. |
| **You already have a fallback** | If the identity provider is unreachable, OIDC authentication fails — but this is **not** a lockout. VKS lets you regenerate a working kubeconfig through the Supervisor at any time: `vcf context create <name> --endpoint https://<supervisor> --type k8s`, then select the namespace and cluster. So there is no need to pre-stage and rotate a separate emergency credential. Do confirm your vSphere SSO account retains the namespace permissions to do this. |
| **Audit periodically** | Bindings accumulate. Review them against current staff and roles. |

### B.2 Trusted TLS for the Headlamp Gateway

The Headlamp addon creates a **self-signed** cert-manager `Issuer` and `Certificate` automatically,
so you get HTTPS out of the box — with a browser warning. For production, issue from a trusted CA.

Check what you currently have:

```bash
kubectl get certificate,issuer -n headlamp

# who issued the certificate currently in use
JP='{.metadata.annotations.cert-manager\.io/issuer-kind}{"/"}'
JP+='{.metadata.annotations.cert-manager\.io/issuer-name}'
JP+='{"  CN="}{.metadata.annotations.cert-manager\.io/common-name}{"\n"}'
kubectl get secret <tls-secret> -n headlamp -o jsonpath="$JP"
```

```
Issuer/headlamp-issuer  CN=Headlamp CA      ← self-signed, hence the browser warning
```

#### Option 1 — ACME (best when the FQDN is publicly resolvable)

```yaml
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

#### Option 2 — Your enterprise CA (typical for internal clusters)

```yaml
# The CA's signing keypair, created out-of-band from your PKI.
# kubectl create secret tls enterprise-ca --cert=ca.crt --key=ca.key -n cert-manager
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
  - headlamp.k8s.example.com        # must equal Headlamp's `hostname`
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

Then reference the resulting secret from the Gateway's TLS listener — through `AddonConfig` values
(check the schema for field names), or set `gateway.create: false` and manage the `Gateway` yourself.

| Guidance | Detail |
| --- | --- |
| Secret namespace | A `Gateway` can normally only read TLS secrets in its own namespace. Cross-namespace references need an explicit `ReferenceGrant`. |
| Private CA cost | A private CA means distributing the CA certificate to client trust stores. ACME avoids that entirely if the name is publicly resolvable. |
| **Test the chain** | `openssl s_client -connect headlamp.k8s.example.com:443` before assuming the OIDC redirect flow works. A broken chain breaks login, not just the padlock. |

### B.3 A `Prometheus` instance via the operator

This is the piece that makes the monitoring stack functional. Adjust to your operator version.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: main
  namespace: tanzu-system-monitoring
spec:
  replicas: 2
  retention: 30d
  serviceAccountName: prometheus     # the operator's RBAC normally provides this
  # Empty selectors discover every ServiceMonitor / PodMonitor / rule in the
  # cluster. Narrow them if you want explicit opt-in.
  serviceMonitorSelector: {}
  podMonitorSelector: {}
  ruleSelector: {}
  alerting:
    alertmanagers:
    - namespace: tanzu-system-monitoring
      name: alertmanager
      port: web
  # Without persistent storage you lose all metrics on every pod restart —
  # a common and entirely avoidable surprise.
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: <STORAGE_CLASS>
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 100Gi
```

**Alerts this cluster specifically needs**, derived from the pitfalls above:

| Alert | Why |
| --- | --- |
| **Allocated pod CIDR blocks vs. total** | Alert *before* the last block is taken. An exhausted CIDR blocks upgrades and the CIDR is immutable ([Pitfall 4](#4-pod-cidr-exhaustion-blocks-upgrades)) |
| etcd DB size > 60% of `maximumDBSizeGiB` | Turns [Pitfall 14](#14-etcd-read-only-after-exceeding-its-quota) from an outage into a ticket |
| Node clock skew | [Pitfall 13](#13-clock-skew-breaks-oidc-intermittently) — otherwise close to undiagnosable |
| Control-plane certificate expiry | Backstop for silently broken rotation |
| containerd / kubelet volume fill % | Those volumes are your protection; know before they fill |
| Pods-per-node approaching `maxPods` | Early warning before scheduling failures |
| PVC pending duration | Catches the storage-class association problem ([Pitfall 11](#11-storageclass-or-vm-class-not-associated-with-the-vsphere-namespace)) |
| istiod and gateway availability | Both default to `minReplicas: 1` in the sample |
| Failed authentication rate | Detects both misconfiguration and credential attacks |
| `PackageInstall` reconcile failures | The authoritative signal that an addon has stopped converging |

### B.4 Further hardening worth considering

| Area | Recommendation |
| --- | --- |
| **NetworkPolicy** | **The highest-value control missing from this stack.** A default-deny posture per namespace, using Kubernetes NetworkPolicy or your CNI's richer CRDs. The manifest has none, so any pod can reach any other. |
| **PodDisruptionBudgets** | Required for safe rolling operations. Without them, an upgrade can take all replicas of a service down at once. |
| **Backups, log shipping, DNS automation** | See [Appendix D](#appendix-d--optional-integrations-requiring-environment-specific-setup) — each needs an environment-specific decision before it does anything. |
| **Secrets management** | Externalise every credential (`vault-injector`, sealed-secrets, external-secrets). Consider etcd encryption at rest. |
| **GitOps** | These manifests are the source of truth. Reconcile from Git with review rather than applying by hand. |
| **Audit logging** | Ship API server audit logs off-cluster. With OIDC in place they become a genuine record of who did what. |
| **Istio mTLS** | If you later adopt the mesh, enable strict mTLS and a minimum TLS protocol version — and `istioCNI` at the same time. |
| **Labels** | Ownership, environment, and cost-attribution labels on the `Cluster`. Trivial now, invaluable at fleet scale. |

---

## Appendix C — Verify for your environment

> **What this is.** The claims this document deliberately does **not** assert, and the ones it
> confirmed by direct inspection.
>
> **Why it matters.** Addon defaults, schema field names, and platform behaviour move between
> releases. Listing what to re-check — rather than asserting it and going stale — is what keeps the
> rest of the document trustworthy.

This document asserts only what was verified against a live VKS 3.7 environment, or what follows
from stable Kubernetes behaviour. The items below are **environment- or release-specific and
deliberately not asserted.**

| # | Item | How to check |
| --- | --- | --- |
| 1 | **Addon default values.** No specific addon default is stated anywhere in this document, because defaults move between releases. | `vcf addon available-releases get <release> -o output.yaml`, or `kubectl get acd <release> -n vmware-system-vks-public -o yaml`. Repeat after every addon upgrade. |
| 2 | **Your ClusterClass variable set.** The ten variables described here are `builtin-generic-v3.7.0`'s. | `kubectl get clusterclass <name> -n vmware-system-vks-public -o jsonpath='{range .status.variables[*]}{.name}{"\n"}{end}'` |
| 3 | **Exact semantics of `resourceQuotaConfiguration.enabled`** — which quota objects are created and how they relate to vSphere Namespace limits. | Test in non-production; check your VKS documentation. |
| 4 | **Headlamp secret-reference and Gateway TLS field names** — needed to externalise `clientSecret` and use a trusted CA. | Headlamp addon schema output. |
| 5 | **Istio strict-mTLS and resource field names**, referenced in B.4 without asserting spellings. | Istio addon schema output. |
| 6 | **The `maxSurge` your ClusterClass rollout strategy applies.** This document assumes the Cluster API default of 1, which sets your minimum spare-block reserve. | ClusterClass and `MachineDeployment` rollout strategy. |
| 7 | **Whether the per-node pod CIDR mask is tunable.** This document assumes the IPv4 default of `/24`, confirmed on live nodes. A `/25` would double your block count but is a create-time decision. | `kubectl get nodes -o custom-columns='NODE:.metadata.name,POD_CIDR:.spec.podCIDR'` and the ClusterClass schema. |
| 8 | **API server behaviour when a mapped `groups` claim is absent** from the token — tolerated, or does it reject? | Test with a real token and read the result from `kubectl auth whoami`. |
| 9 | **Whether `240.0.0.0/4` is handled correctly** by every appliance in your path — firewalls, hardware load balancers, monitoring. Antrea supports it; your wider environment may not. | Test pod-to-external and external-to-service traffic end to end before standardising. |
| 10 | **Addon licensing and entitlement.** Some addons imply other products — `ako` requires NSX Advanced Load Balancer, for example. Availability in the catalogue is not entitlement. | Your licensing, and `vcf addon available list` for what your environment actually offers. |
| 11 | **Custom VM classes.** Environments often define their own; the `best-effort`/`guaranteed` naming convention may not apply, so confirm whether a custom class reserves resources. | `kubectl get virtualmachineclass` and vCenter. |
| 12 | **PSS version strings valid for your Kubernetes minor.** This document uses `v1.36` to match `v1.36.1`. | Kubernetes documentation for your minor version. |
| 13 | **Practical `maxPods` ceiling for your node sizes.** The schema documents 250 as the maximum; what your VM classes can actually sustain is a capacity question. | Load-test at your intended density before standardising. |

**Answered by direct inspection.** These are commonly asked, and each can be confirmed in your own
environment with the command shown:

| Question | Answer | Verified by |
| --- | --- | --- |
| Are the Gateway API CRDs installed automatically? | **Yes** — the `gateway-api` addon is platform-installed | `kubectl get clusteraddon -n <ns>` |
| Is the Cluster Autoscaler deployed by default? | **Yes** — so the machine-deployment annotations are effective | `kubectl get clusteraddon -n <ns>` |
| Are addons delivered as Helm releases? | **No** — Carvel `PackageInstall` objects | `kubectl get pkgi -A`; `kubectl get helmrelease -A` returns nothing |
| Is there an addon ordering requirement? | **No** — all reconciled independently | Addon framework design |
| What happens if an `AddonConfig` name does not resolve? | It **skips reconciliation**; `spec.clusterName` stays empty | `kubectl explain addonconfig.spec` |
| `maxPods` default and maximum? | Default **110**, documented maximum **250**, minimum 20 | ClusterClass schema |
| Does the Headlamp addon use cert-manager? | **Yes** — it creates a self-signed `Issuer` and `Certificate` | `kubectl get certificate,issuer -n headlamp` |
| Does the Headlamp Gateway share the Istio ingress IP? | **No** — it provisions its own LoadBalancer | `kubectl get svc -n headlamp` |

---

## Appendix D — Optional integrations requiring environment-specific setup

> **What this is.** Three addons — backup, log forwarding, and DNS automation — and the decisions each
> one requires before it is worth installing.
>
> **Why it matters.** All three are commonly recommended as production essentials. They are not
> optional because they lack value; they are separated out because each depends on an external system
> that differs in every environment, and installing one without that integration leaves you with a
> running pod and false confidence.

These three addons come up in almost every production conversation, and they are all genuinely
useful — but **none of them does anything on its own.** Each needs a decision about an external
system before it produces value, and that decision is different in every environment. That is why
they are here rather than in the recommended baseline: installing them without the integration
behind them leaves you with a running pod and a false sense of coverage.

Take them if the corresponding gap is real for you and you know what you are pointing them at.

### D.1 Backup and restore (`velero`)

**What it does.** Backs up Kubernetes resources and persistent volumes, restores them, and supports
migration between clusters.

**Why it is not in the baseline.** Cluster-level backup is not a universal practice. Many
environments protect workloads at a different layer — vSphere-level VM protection, storage array
snapshots, or an application-native backup (a database's own dump and WAL shipping is usually better
than a volume snapshot of the same database). Whether an in-cluster backup tool adds anything depends
on what you already have.

**What you must decide first:**

| Decision | Notes |
| --- | --- |
| **Object storage target** | Velero needs an S3-compatible bucket, plus credentials. This is the hard prerequisite — no bucket, no backup. |
| **What is in scope** | Cluster resources only, or volumes too? Volume backup needs a snapshot mechanism, and behaviour differs by CSI driver. |
| **Whether it duplicates existing protection** | If your storage layer already snapshots these volumes on a schedule you trust, this may be redundant cost and complexity. |
| **Retention and schedule** | Backups that are never expired become an unbounded storage bill. |
| **Restore target** | Same cluster, or a rebuild? Cross-cluster restore has extra constraints around CIDRs and storage classes. |

> **Whatever you use — including nothing — test a restore.** An untested backup is a hypothesis, not
> a control. This applies equally to vSphere-level protection: confirm you can actually recover a
> workload, not just that a job reported success. This matters most for the immutable fields in
> [section 8](#8-cluster-decisions-you-cannot-change-later), where rebuild-and-restore is the only remedy.

### D.2 Log forwarding (`fluent-bit`)

**What it does.** Collects container and node logs and forwards them to a destination.

**Why it is not in the baseline.** Fluent Bit is only as useful as its **output plugin**, and the
output is entirely environment-specific — Splunk, Elasticsearch/OpenSearch, Loki, an OTLP collector,
a cloud logging service, or plain syslog. Each needs its own configuration, endpoint, credentials,
and index or stream conventions. There is no sensible default to recommend, and a Fluent Bit with no
configured output is a DaemonSet consuming a pod slot on every node and shipping nothing.

**What you must decide first:**

| Decision | Notes |
| --- | --- |
| **Output plugin and endpoint** | The core decision. Determines the whole configuration. |
| **Credentials and TLS** | Most outputs need auth. Manage it as a Secret, not inline. |
| **What to collect** | All container logs, or filtered? Node and systemd logs too? Unfiltered collection from a busy cluster generates surprising volume. |
| **Parsing** | You set `format: json` on the API server and kubelet ([7.7.5](#775-structured-logging)), so those parse cleanly — but application logs are whatever your applications emit. |
| **Volume and cost** | Log platforms usually bill on ingest. Estimate before turning it on cluster-wide. |
| **Retention** | Where retention is enforced, and for how long — often a compliance question. |

> **The reason this pairs with the manifest:** you already set structured JSON logging on both the API
> server and the kubelet, and **node logs are destroyed when a node is replaced** — which happens on
> every upgrade, `vmClass` change, and `volumes` resize. Without forwarding, the JSON setting buys you
> nothing and your diagnostic history has the lifespan of a node. If you have no log pipeline at all,
> this is the more urgent of the three.
>
> It is also what makes PSS `audit` violations durable
> ([7.7.3](#773-securitypodsecuritystandard)) — those land in the API server audit log.

### D.3 DNS automation (`external-dns`)

**What it does.** Watches `Service`, `Ingress`, and `Gateway` resources and creates or updates DNS
records to match.

**Why it is not in the baseline.** It requires **write access to a DNS provider** — a specific
provider integration, an API credential, and a delegated zone. That is an organisational decision as
much as a technical one: many teams will not grant a cluster write access to corporate DNS, and in
those environments a change request is the process, not an API call.

**What you must decide first:**

| Decision | Notes |
| --- | --- |
| **DNS provider** | Each has its own provider configuration and credential type. |
| **Credential and blast radius** | The cluster gets write access to a DNS zone. Scope the credential to a delegated subdomain (e.g. `k8s.example.com`), never the apex zone. |
| **Zone delegation** | Cleanest model: delegate a subdomain to be cluster-managed and leave the rest alone. |
| **Ownership and conflicts** | External-DNS uses TXT records to track ownership. Decide how it coexists with manually managed records so it does not fight your DNS team. |
| **Policy** | `sync` (delete records when resources go away) versus `upsert-only` (never delete). `upsert-only` is safer to start with. |

> **What it solves in this manifest:** the Headlamp FQDN currently needs a manually maintained A
> record pointing at the Gateway's LoadBalancer IP ([6.5](#65-headlamp)), and if that IP changes the
> record, the `callbackURL`, and the IdP registration all drift
> ([Pitfall 7](#7-hostname-dns-and-redirect-uri-drift)). Automating the record removes the most
> commonly forgotten of those steps.
>
> **A simpler alternative that solves most of it:** reserve a static IP with your load-balancer
> provider. The IP then never changes, so the record never needs updating and the drift problem
> disappears without granting anyone DNS write access. Do that first; add automation only if you are
> publishing enough services for manual records to be a real burden.

---

## Appendix E — Inspecting a Kubernetes release (`kr`)

> **What this is.** What is actually inside a `kr` object: component versions, the version-pinned
> platform addons, and the OS images it offers — plus a ready-made pre-upgrade diff.
>
> **Why it matters.** A Kubernetes *patch* bump can carry a CNI upgrade and a kapp-controller minor
> version bump. Reading the release before you upgrade turns that from a surprise into a plan. It is
> also how you test a `resolve-os-image` annotation before it stalls a cluster.


[Section 2](#2-choosing-a-kubernetes-release-the-kr-object) covers *which* release to pick. This
appendix covers what is actually **inside** one, because the `kr` object is the authoritative
manifest of everything a release ships — and reading it answers three questions you otherwise guess
at:

1. **Which OS images may I select?** — the valid values for your `resolve-os-image` annotation.
2. **What component versions am I getting?** — etcd, CoreDNS, CNI, CSI, CPI.
3. **What actually changes if I upgrade?** — a real diff, before you touch anything.

The object has four keys:

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

### E.1 `.spec.kubernetes` — core control-plane components

```bash
# Build the jsonpath across lines in a variable — a trailing backslash inside
# single quotes is LITERAL, not a line continuation, so it must not be split.
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

| Field | What it tells you |
| --- | --- |
| `version` | The semver form of the Kubernetes version. Note this is the `+` form — `topology.version` takes the object **`NAME`** with the triple dash instead ([section 2](#2-choosing-a-kubernetes-release-the-kr-object)). |
| `etcd.imageTag` | **The etcd version, which is the single most important number here.** etcd carries its own upgrade constraints and behavioural changes, and it is your cluster's state store. |
| `coredns.imageTag` | The CoreDNS version. Relevant if you have custom Corefile configuration or have hit DNS behaviour changes before. |
| `pause.imageTag` | The sandbox/infra container image. Rarely interesting, but it confirms the CRI baseline. |
| `imageRepository` | Where control-plane images are pulled from. `localhost:5000/vmware.io` indicates a node-local mirror rather than an external registry — useful to know when diagnosing image pulls, and reassuring for air-gapped environments. |
| `_fips` suffixes | Present on the CoreDNS and etcd tags above even though this is a non-FIPS release, meaning those components are FIPS-validated builds regardless. Do not infer a release's overall FIPS posture from a component tag — use the `-fips` marker in the release **name**. |

> **Why read this before an upgrade:** the Kubernetes version is only part of what moves. Comparing
> the deployed release against the target across two patch versions of the *same minor*:
>
> | Component | `v1.36.1---vmware.4-vkr.5` | `v1.36.2---vmware.2-vkr.3` |
> | --- | --- | --- |
> | Kubernetes | `v1.36.1+vmware.4` | `v1.36.2+vmware.2` |
> | **etcd** | `v3.6.11_vmware.3-fips` | **`v3.6.12_vmware.7-fips`** |
> | CoreDNS | `v1.14.3_vmware.3-fips` | `v1.14.3_vmware.7-fips` (rebuild) |
>
> A Kubernetes patch bump brought an **etcd patch upgrade** with it. That is worth knowing in
> advance rather than discovering afterwards.

### E.2 `.spec.bootstrapPackages` — the platform addons pinned to the release

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

**This is the version-pinned list of the platform components from
[section 3](#3-what-vks-installs-for-you)** — the things installed for you. Two useful observations:

- **Both CNI options appear here** (`antrea` and `calico`), pinned to the release. Your
  `bootstrapAddons.cniRef` selects among them ([7.8](#78-bootstrapaddons--cni-selection)); the
  release determines the *version* you get. So you do not choose a CNI version independently — it
  comes with the Kubernetes release.
- The Gateway API version is here too, which is what makes `gatewayApi.enabled: true` work for
  Headlamp ([6.5](#65-headlamp)) with no CRD installation on your part.

> **A Kubernetes "patch" release is not only a patch.** Diffing the two v1.36 releases above,
> **all ten bootstrap packages changed:**
>
> | Component | `vkr.5` → `vkr.3` | Significance |
> | --- | --- | --- |
> | **antrea** | `2.6.1+vmware.1` → **`2.6.2+vmware.1`** | **A CNI upgrade — your dataplane** |
> | **kapp-controller** | `0.59.8+vmware.1` → **`0.60.4+vmware.3`** | **A minor bump in the component that reconciles every addon** |
> | secretgen-controller | `0.20.1+vmware.3` → `0.21.1+vmware.4` | Minor bump |
> | guest-cluster-auth-service | `1.4.7+vmware.1` → `1.4.8+vmware.1` | Patch — authentication path |
> | pinniped | `0.46.0+vmware.1` → `0.46.0+vmware.8` | Rebuild — authentication path |
> | vsphere-pv-csi | `3.8.0+vmware.2` → `3.8.0+vmware.3` | Rebuild — storage |
> | vsphere-cpi | `1.36.0+vmware.1` → `1.36.0+vmware.2` | Rebuild — LoadBalancer services |
> | gateway-api | `1.5.1+vmware.1` → `1.5.1+vmware.4` | Rebuild |
> | metrics-server | `0.8.1+vmware.1` → `0.8.1+vmware.3` | Rebuild |
> | calico | `3.31.5+vmware.1` → `3.31.5+vmware.3` | Rebuild |
>
> A CNI upgrade and a kapp-controller minor version bump inside a Kubernetes patch release is exactly
> the kind of scope you want to know about while planning, not while troubleshooting. **Run this diff
> as a standard pre-upgrade step** — see [E.4](#e4-a-pre-upgrade-change-report).

### E.3 `.spec.osImages` and the `osimage` object

```bash
kubectl get kr v1.36.2---vmware.2-vkr.3 \
  -o jsonpath='{range .spec.osImages[*]}{.name}{"\n"}{end}'
```

```
vmi-3219b260b6bb7ccf7
vmi-5c9649f9fba751a8b
vmi-a32066fa312b9ab5f
```

Order is not significant — and the names are opaque hashes, which is why the next step matters.

Opaque hashes on their own. Resolve them through the cluster-scoped **`OSImage`** object
(`run.tanzu.vmware.com/v1alpha3`, short name `osimg`):

```bash
for i in $(kubectl get kr v1.36.2---vmware.2-vkr.3 \
             -o jsonpath='{range .spec.osImages[*]}{.name}{" "}{end}'); do
  kubectl get osimage "$i" -o jsonpath='{.metadata.name}{"  "}{.spec.os.name}/{.spec.os.version}{"  "}{.spec.os.arch}{"  "}{.spec.image.type}{"\n"}'
done
```

```
vmi-a32066fa312b9ab5f  photon/5       amd64  cvmi
vmi-3219b260b6bb7ccf7  ubuntu/22.04   amd64  cvmi
vmi-5c9649f9fba751a8b  ubuntu/24.04   amd64  cvmi
```

**This is the authoritative list of OS choices for that release.** In the environment this was
verified against, every `READY` + `COMPATIBLE` release offers the same three — **photon 5,
ubuntu 22.04, ubuntu 24.04, all amd64** — but that is an observation about one environment, not a
guarantee. Check your own release rather than assuming a given OS is available.

#### The key connection: `resolve-os-image` is a label selector

The annotation in the manifest —

```yaml
          run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
```

— is a **label selector matched against `OSImage` labels.** Inspect the labels and it becomes obvious:

```bash
# --show-labels appends a LABELS column; $NF isolates it (labels contain no spaces)
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

**So you can test your annotation before applying it**, which turns a stalled cluster into a
five-second check:

```bash
# Exactly what the annotation will resolve to for a given release
kubectl get osimage \
  -l os-name=ubuntu,os-version=24.04,run.tanzu.vmware.com/tkr=v1.36.2---vmware.2-vkr.3
```

```
NAME                    K8S VERSION        OS NAME   OS VERSION   ARCH    TYPE
vmi-5c9649f9fba751a8b   v1.36.2+vmware.2   ubuntu    24.04        amd64    cvmi
```

**Exactly one match is what you want.** No match means the annotation will never resolve and machine
creation stalls with a thin error surface — the failure mode flagged in
[7.3](#73-control-plane) and [7.12](#712-workersmachinedeployments--the-node-pool).

| Observation | Detail |
| --- | --- |
| **Selectable labels** | `os-name`, `os-version`, `os-arch`, `os-type`. Add `os-arch=amd64` if your environment ever carries mixed architectures. |
| **Version-prefix labels** | The empty-valued `v1`, `v1.36`, `v1.36.2` labels allow selection by version prefix as well as by exact release. |
| **`run.tanzu.vmware.com/tkr`** | Ties the image to its release. Use it to scope a query to one release, as above. |
| **`content-library`** | The vSphere content library holding the image. If an image is listed but unusable, this is where to look — the library must be synced and available to the namespace. |
| **`vmi` vs `cvmi`** | `cvmi` is a cluster-scoped `ClusterVirtualMachineImage`; `vmi` is the older namespaced `VirtualMachineImage`. Current releases use `cvmi`. In the verified environment, 42 of 138 `OSImage` objects were `cvmi` and 96 were legacy `vmi` — the `vmi` ones belong to old, non-compatible releases. |
| **No `status`** | `OSImage` is spec and metadata only. It is a projection, owned by a `ClusterVirtualMachineImage` — check that owner if an image looks wrong. |

#### Choosing the OS

| OS | Consideration |
| --- | --- |
| **ubuntu 24.04** | Current LTS with the longest support horizon. The sensible default for a new cluster, and what this document uses. |
| **ubuntu 22.04** | Previous LTS. Choose it only for a specific compatibility reason — you are starting closer to end of support. |
| **photon 5** | VMware's minimal, purpose-built host OS. Smaller footprint and attack surface; fewer familiar userspace tools if you need to debug on a node. |

Whatever you choose, **set the same annotation on the control plane and every machine deployment**, or
pools drift onto different images ([7.12](#712-workersmachinedeployments--the-node-pool)).

### E.4 A pre-upgrade change report

Combining the above into one pre-upgrade step. Run it against your current and target releases and
read the output before scheduling anything:

```bash
#!/usr/bin/env bash
# usage: kr-diff.sh <current-kr> <target-kr>
# Requires only kubectl and coreutils — no jq, no python.
set -u
CUR="$1"; TGT="$2"
TMP=$(mktemp -d); trap 'rm -rf "$TMP"' EXIT

echo "### Core components"
for k in "$CUR" "$TGT"; do
  echo "  $k"
  kubectl get kr "$k" -o jsonpath='\
{"    kubernetes: "}{.spec.kubernetes.version}\
{"\n    etcd:       "}{.spec.kubernetes.etcd.imageTag}\
{"\n    coredns:    "}{.spec.kubernetes.coredns.imageTag}{"\n"}'
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
      -l "os-name=ubuntu,os-version=24.04,run.tanzu.vmware.com/tkr=$TGT" 2>/dev/null | wc -l)
echo "  matches: $n (expect exactly 1)"
```

**What to do with the output:**

| Finding | Action |
| --- | --- |
| etcd version changed | Review etcd release notes. Confirm `maximumDBSizeGiB` headroom ([7.7.2](#772-etcdconfiguration)) and that you can recover it if the upgrade goes badly. |
| **CNI version changed** | The highest-attention item — it is your dataplane. Validate in non-production, and check for NetworkPolicy or CRD behaviour changes. |
| **kapp-controller version changed** | It reconciles every addon. An addon that stops converging after an upgrade points here first (`kubectl get pkgi -A`). |
| CPI / CSI version changed | Test a LoadBalancer service and a PVC after the upgrade ([Layer 6](#layer-6--storage)). |
| Authentication components changed | Re-verify OIDC login end to end ([Layer 7](#layer-7--identity-what-username-does-the-api-server-see)). |
| Your OS choice is absent from the target | Stop. Resolve this before upgrading, or machine creation will stall. |
| Zero label matches | Fix the annotation before applying anything. |

### E.5 Command reference

```bash
# Usable releases only
kubectl get kr

# Everything a release ships
kubectl get kr <name> -o yaml

# What fields exist, with descriptions
kubectl explain kr.spec

# Core component versions
kubectl get kr <name> -o jsonpath='{.spec.kubernetes.version}{"  etcd "}{.spec.kubernetes.etcd.imageTag}{"  coredns "}{.spec.kubernetes.coredns.imageTag}{"\n"}'

# Version-pinned platform addons
kubectl get kr <name> -o jsonpath='{range .spec.bootstrapPackages[*]}{.name}{"\n"}{end}'

# OS image references (opaque) ...
kubectl get kr <name> -o jsonpath='{range .spec.osImages[*]}{.name}{"\n"}{end}'

# ... resolved to OS name and version
kubectl get osimage --show-labels -l run.tanzu.vmware.com/tkr=<name>

# Test a resolve-os-image annotation before applying it
kubectl get osimage -l os-name=ubuntu,os-version=24.04,run.tanzu.vmware.com/tkr=<name>

# Every OS image in the environment, with the release each belongs to
kubectl get osimage -L os-name,os-version,os-arch,run.tanzu.vmware.com/tkr

# What the running cluster actually landed on
kubectl get cluster <CLUSTER_NAME> -n <VSPHERE_NAMESPACE> \
  -o custom-columns='NAME:.metadata.name,VERSION:.status.version'
```

---

## Summary — the short list

> **What this is.** Everything actionable from the document, condensed: what to settle before you
> apply, what to change before production, and which parts of the reference profile to keep as-is.


### Before you apply

| # | Action | Why |
| --- | --- | --- |
| 1 | **Size the pod CIDR for maximum scale plus 2–3 spare blocks.** Use a `/16`. | Immutable, and an exhausted CIDR blocks upgrades ([7.1.1](#711-pod-cidr-sizing-node-blocks-maxpods-and-the-upgrade-spare)) |
| 2 | **Choose your CNI deliberately.** | Effectively irreversible ([7.8](#78-bootstrapaddons--cni-selection)) |
| 3 | **Pick a `kr` that is `READY` *and* `COMPATIBLE`**, and read what it ships. | A wrong version string never reconciles; and the release pins your etcd, CNI, and OS image choices ([section 2](#2-choosing-a-kubernetes-release-the-kr-object), [Appendix E](#appendix-e--inspecting-a-kubernetes-release-kr)) |
| 4 | **Confirm VM classes and storage policies are associated with the namespace.** | Failures surface late and unhelpfully ([section 1](#1-prerequisites)) |
| 5 | **Publish the Headlamp FQDN in DNS and register the redirect URI with your IdP.** | Removes the only real sequencing constraint ([6.5](#65-headlamp)) |
| 6 | **Size `volumes` generously.** | Resizing later is a rolling node replacement ([7.6](#76-volumes--dedicated-containerd-and-kubelet-disks)) |
| 7 | **Decide your namespace-creation policy** before enforcing Pod Security above `privileged`. | Otherwise `helm install --create-namespace` starts failing ([7.7.3](#773-securitypodsecuritystandard)) |
| 8 | **If raising `maxPods`, size the VM class and `systemReserved` together.** | The automatic reservation does not scale with pod count; 250 pods needs `guaranteed-2xlarge` or larger ([7.13](#713-37-variables-not-used-in-the-sample)) |

### Fix before production

| # | Change | From → to |
| --- | --- | --- |
| 1 | **The claim validation rule** — it fails open regardless of environment | `orValue(true)` → `orValue(false)` |
| 2 | **Control-plane HA** | `replicas: 1` → `3` |
| 3 | **VM classes** | `best-effort-*` → `guaranteed-*`, control plane `guaranteed-large` |
| 4 | **Pod Security** | `privileged` → `enforce: baseline` + `audit`/`warn: restricted`; raise to `restricted` per namespace, and decide your namespace-creation policy ([7.7.3](#773-securitypodsecuritystandard)) |
| 5 | **Pin PSS versions** | `latest` → explicit (e.g. `v1.36`) |
| 6 | **Istio availability** | `pilot` and ingress `minReplicas: 1` → `2`; drop the static `pilot.replicas` |
| 7 | **Headlamp FQDN** | Publish the DNS A record and register the redirect URI with your IdP — the manifest value is only one of three places it must match ([6.5](#65-headlamp)) |
| 8 | **Headlamp TLS** | self-signed → a trusted CA ([B.2](#b2-trusted-tls-for-the-headlamp-gateway)) |
| 9 | **OIDC client secret** | inline → a Secret reference; rotate if ever committed |
| 10 | **RBAC** | none → bindings for the OIDC username ([B.1](#b1-rbac-for-oidc-identities)) |
| 11 | **Prometheus** | operator only → create the `Prometheus` CR ([B.3](#b3-a-prometheus-instance-via-the-operator)) |
| 12 | **Gateway access logs** | `""` → `/dev/stdout` while validating |

### Addons to consider

Nothing beyond the five is a blanket recommendation — it depends on what you already run. Browse the
catalogue in [6.6](#66-additional-addons-worth-considering). The three that most often come up —
backup, log forwarding, and DNS automation — each need an environment-specific decision first, and
are written up in [Appendix D](#appendix-d--optional-integrations-requiring-environment-specific-setup).

### Keep — these parts of the sample are good practice

- Dedicated `/var/lib/containerd` and `/var/lib/kubelet` volumes
- `certificateRotation.enabled: true`
- Explicit NTP configuration — a functional prerequisite for OIDC, TLS, and etcd
- `owned-for-deletion` on every `AddonConfig`
- The gateway in its own namespace, separate from the Istio control plane
- Username **and** group prefixes on OIDC claim mappings
- Structured JSON logging on both API server and kubelet
- Reserved `240.0.0.0/4` space to avoid RFC1918 collisions
- `resourceQuotaConfiguration.enabled: true` (a deliberate change from the schema default)
- Pinned Kubernetes version and ClusterClass
- `pushgateway: false`
- Operator-managed Prometheus rather than the packaged server
- The `controlPlane.variables.overrides` pattern for independent control-plane sizing
- Autoscaler bounds with **no** static `replicas` — the correct pairing

### Four points worth being explicit about

1. **There is no addon ordering requirement.** All addons are Carvel packages reconciled
   independently. **helm-controller is not a prerequisite** for the others — it exists for day-2 Helm
   lifecycle management of *your own* charts.
2. **Istio here provides L4/L7 only.** No sidecars, no mesh. Consequently `istioCNI: false` imposes
   **no** Pod Security constraint on this cluster — the `privileged` setting is a dev/test choice, not
   a consequence of Istio. That coupling only applies once you enable sidecar injection.
3. **The pod CIDR is a pool of per-node `/24` blocks, not a flat pool of IPs** — and one block must
   always stay free, or no rolling upgrade can complete.
4. **PSS namespace labels and `exemptions.namespaces` are not two ways to do the same thing.** A label
   applies a *different level* and keeps audit and warn working; an exemption switches Pod Security
   *off* for that namespace. Prefer labels.
