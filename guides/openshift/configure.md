# RBLN Cloud Native Stack (RCNS) - Configuration Guide for Red Hat OpenShift

This guide explains how to configure the RBLN NPU stack on an existing OpenShift cluster. It does not cover
installing OpenShift.

The configuration installs Node Feature Discovery (NFD), installs the RBLN NPU Operator with Operator Lifecycle
Manager (OLM), and then creates an `RBLNClusterPolicy` and one `RBLNDriver` per NPU family. The component versions
and the image registries, repositories, and tags come from the OpenShift values file.

The stack can be configured in two ways. Both produce the same RBLN NPU stack.

- [Automatic configuration](#automatic-configuration) (recommended) runs one playbook.
- [Manual configuration](#manual-configuration) runs and verifies each step by hand.

## Choosing a Values File

RCNS batch 2026.10.x supports OpenShift 4.19 through 4.21 and provides one values file per OpenShift minor
version. Choose the file that matches the minor version of the cluster. For more information about batches,
see [Version Policy](../../README.md#version-policy).

| OpenShift | Values file |
|---|---|
| 4.19 | `automation/values/rcns_openshift_values_4.19.yaml` |
| 4.20 | `automation/values/rcns_openshift_values_4.20.yaml` |
| 4.21 | `automation/values/rcns_openshift_values_4.21.yaml` |

Check the cluster version with `oc get clusterversion`.

This guide uses the following keys from the values file.

| Values file key | Used for |
|---|---|
| `components.rbln-operator` | RBLN NPU Operator version |
| `stacks.rbln-npu-operator.olm` | OLM Subscription settings (`package`, `source`, `channel`, `version`). Automatic configuration applies `source`, `channel`, and `version` (`version` sets the Subscription's `startingCSV`) |
| `stacks.card.<family>.image` | RBLN Driver image per NPU family (`registry`, `repository`, `tag`, `pullPolicy`) |
| `stacks.card.<family>.smd` | RBLN System Management Daemon (RSMD) image per NPU family (`registry`, `repository`). RSMD uses the RBLN Driver tag |

The NPU families are Rebel100™ (`rebel100`) and ATOM™ (`atom`).

## Prerequisites

- An OpenShift 4.19, 4.20, or 4.21 cluster and a kubeconfig with `cluster-admin` privileges.
- A workstation that can reach the cluster API and has the following tools:
  - For both methods: Git (to clone this repository)
  - For automatic configuration: Python 3, Ansible (ansible-core 2.15 or later), Helm, and `kubectl`

    ```sh
    python3 -m venv .venv && . .venv/bin/activate   # on Ubuntu/Debian, install the python3-venv package first
    python3 -m pip install 'ansible-core>=2.15'
    ansible --version
    ```
  - For manual configuration: `oc`, `jq`, and Python 3 with PyYAML

    Get the `oc` client that matches the OpenShift minor version of the cluster. The following example is for
    Linux x86_64 and OpenShift 4.21. The client is also available from the web console: in the upper-right
    corner, click **?** (Help) > **Command Line Tools**.

    ```sh
    curl -L https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-4.21/openshift-client-linux.tar.gz \
      | sudo tar -xz -C /usr/local/bin oc kubectl
    oc version --client
    ```

    Install PyYAML with pip:

    ```sh
    python3 -m pip install pyyaml
    ```
- At least one worker node with an RBLN NPU.
- Access from the cluster to the Red Hat Operator catalogs (`certified-operators`, `redhat-operators`). For
  clusters without internet access, see [Disconnected Environments](#disconnected-environments).
- An RBLN Portal account, to pull the RBLN Driver and RSMD images.

## Automatic Configuration

Run `automation/playbooks/openshift-stack.yaml`. It configures only the stack on the existing cluster and is
separate from the Kubernetes playbook `install.yaml`. It runs the following steps in order:

1. Installs the Red Hat NFD Operator and creates a `NodeFeatureDiscovery`
   (`automation/manifests/openshift/nfd-*.yaml`).
2. Subscribes to the RBLN NPU Operator with OLM (`automation/manifests/openshift/olm.yaml` and
   `stacks.rbln-npu-operator.olm` in the values file).
3. Creates the `repo-rbln` image pull secret, if RBLN Portal credentials are passed
   (`rbln_registry_username`).
4. Creates an `RBLNClusterPolicy` and one `RBLNDriver` per NPU family from the default CRs of the installed
   Operator (CSV), with the image registries, repositories, and tags from `stacks.card.<family>` in the values
   file.
5. Installs cert-manager if `enable_cert_manager` is `true`, and Istio if `enable_istio` is `true`
   (`addons.yaml`).

```sh
cd automation/playbooks
ansible-playbook openshift-stack.yaml -e @../values/rcns_openshift_values_<OpenShift version>.yaml \
  -e kubeconfig=<path to the OpenShift kubeconfig> \
  -e rbln_registry_username=<RBLN Portal account> -e rbln_registry_password=<password>
```

- The playbook does not connect to the nodes over SSH, so the hosts file does not need to be edited.
- If NFD is already running in the cluster, add `-e enable_nfd=false`.
- If credentials are not passed, the `repo-rbln` secret must already exist in `rbln-system`. Otherwise, the
  configuration stops.
- Passing the password in an encrypted vars file, rather than on the command line, is recommended.

> [!NOTE]
> In batch 2026.10.0, automatic configuration does not apply kernel boot parameters. If the hardware needs
> them, apply them before running the playbook, as described in
> [Configure Kernel Boot Parameters](#2-configure-kernel-boot-parameters).

When the playbook finishes, check the status as described in
[Validating the Configuration](#validating-the-configuration).

## Manual Configuration

### 1. Prepare Node Feature Discovery and Check NPU Detection

The RBLN NPU Operator finds NPU nodes by the PCI labels that NFD adds. If NFD is not already running in the
cluster, install the Red Hat NFD Operator and create a `NodeFeatureDiscovery` with the same manifests as
automatic configuration; otherwise, skip to the NPU detection check below.

```bash
oc apply -f automation/manifests/openshift/nfd-operator.yaml
oc wait -n openshift-nfd --for=jsonpath='{.status.installedCSV}' subscription/nfd --timeout=600s
NFD_CSV=$(oc get subscription nfd -n openshift-nfd -o jsonpath='{.status.installedCSV}')
oc wait -n openshift-nfd --for=jsonpath='{.status.phase}'=Succeeded "csv/$NFD_CSV" --timeout=600s
oc apply -f automation/manifests/openshift/nfd-cr.yaml
```

The NFD Operator can also be installed from the Software Catalog in the web console (OperatorHub in
OpenShift 4.19). See
[Installing the Node Feature Discovery Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/specialized_hardware_and_driver_enablement/psap-node-feature-discovery-operator)
in the Red Hat documentation.

Check that the NPU nodes are detected. RBLN NPUs use PCI vendor ID `1eff`. Depending on the NFD configuration,
the label can include the PCI class, so query both forms:

```bash
oc get nodes -l feature.node.kubernetes.io/pci-1eff.present=true
oc get nodes -l feature.node.kubernetes.io/pci-1200_1eff.present=true
```

One of the two commands must list the NPU worker nodes.

### 2. Configure Kernel Boot Parameters

If the hardware needs kernel boot parameters, apply them with a MachineConfig. For the values, see
[Configuring Kernel Parameters on OpenShift](https://docs.rbln.ai/latest/software/system_management/openshift/kernel_args_tuning.html)
in the RBLN SDK documentation. Applying a MachineConfig reboots the nodes one at a time.

### 3. Create the Image Pull Secret

Create the `rbln-system` namespace and store the RBLN Portal credentials in the `repo-rbln` image pull secret.
Both the RBLN Driver and RSMD pull their images with this secret.

```bash
oc create namespace rbln-system --dry-run=client -o yaml | oc apply -f -
export REGISTRY_USER='<RBLN_PORTAL_USERNAME>'
export REGISTRY_PASSWORD='<RBLN_PORTAL_PASSWORD>'
oc create secret docker-registry repo-rbln \
  --docker-server=repo.rebellions.ai \
  --docker-username="$REGISTRY_USER" \
  --docker-password="$REGISTRY_PASSWORD" \
  -n rbln-system
unset REGISTRY_PASSWORD
```

### 4. Install the RBLN NPU Operator (OLM Subscription)

Apply `olm.yaml`, which automatic configuration also uses. It creates the `rbln-system` namespace, the
OperatorGroup, and a Subscription. The Subscription follows the `stable` channel of the `certified-operators`
catalog, uses `Manual` approval, and starts at the version in `startingCSV` (the same version as
`components.rbln-operator` in the values file).

Steps 5 and 6 use the `CSV` and `VALUES` shell variables set in this step, so run steps 4 through 6 in one
shell session.

```bash
VALUES=automation/values/rcns_openshift_values_<OpenShift version>.yaml   # for example, 4.21
oc apply -f automation/manifests/openshift/olm.yaml
TARGET=$(oc get subscription rbln-npu-operator -n rbln-system -o jsonpath='{.spec.startingCSV}')
echo "$TARGET"   # same version as components.rbln-operator in the values file
```

Check that the first InstallPlan created by OLM contains the target version, then approve it:

```bash
oc wait -n rbln-system --for=jsonpath='{.status.installPlanRef.name}' subscription/rbln-npu-operator --timeout=600s
PLAN=$(oc get subscription rbln-npu-operator -n rbln-system -o jsonpath='{.status.installPlanRef.name}')
oc get installplan "$PLAN" -n rbln-system -o jsonpath='{.spec.clusterServiceVersionNames}{"\n"}'   # must include $TARGET
oc patch installplan "$PLAN" -n rbln-system --type merge -p '{"spec":{"approved":true}}'
```

Wait until the Operator (CSV) reaches `Succeeded`, and check that the installed version matches the target:

```bash
oc wait -n rbln-system --for=jsonpath='{.status.phase}'=Succeeded "csv/$TARGET" --timeout=600s
CSV=$(oc get subscription rbln-npu-operator -n rbln-system -o jsonpath='{.status.installedCSV}')
[ "$CSV" = "$TARGET" ] && echo "installed version: $CSV"
```

When a newer version becomes available in the channel, OLM creates a new InstallPlan, and the upgrade does not
start until an administrator approves it. Approve the InstallPlan only after the new version is validated in
RCNS.

If the Operator installation fails, check `oc describe subscription rbln-npu-operator -n rbln-system` and the
InstallPlan status. If the catalog does not have the version, list the available versions with
`oc get packagemanifest rbln-npu-operator -n openshift-marketplace`.

### 5. Create the RBLNClusterPolicy

Create the `RBLNClusterPolicy` from the default CR in the `alm-examples` annotation of the installed Operator's
CSV. The default CR ships with the Operator, so its API fields and operand images (Device Plugin, Metrics
Exporter, and others) match the Operator version. Automatic configuration uses the same defaults.

```bash
oc get csv "$CSV" -n rbln-system \
  -o jsonpath='{.metadata.annotations.alm-examples}' > alm-examples.json
jq -e 'map(select(.kind == "RBLNClusterPolicy"))[0] // error("No policy example")' alm-examples.json \
  > clusterpolicy.json
oc apply -f clusterpolicy.json
```

To change the defaults, for example to enable DRA allocation or VM passthrough, edit `clusterpolicy.json`
before applying it. For the fields, see
[Installing the RBLN NPU Operator on OpenShift](https://docs.rbln.ai/latest/software/system_management/openshift/about_openshift.html)
in the RBLN SDK documentation.

### 6. Create an RBLNDriver per NPU Family

The following commands take the default `rbln-driver-<family>` CR from the `alm-examples` annotation of the
installed Operator's CSV. In each CR, they set the image registry, repository, and tag from
`stacks.card.<family>` in the values file, and set the image pull secret to `repo-rbln` (step 3).

```bash
python3 - "$VALUES" > card.json <<'PY'
import json, sys, yaml
print(json.dumps(yaml.safe_load(open(sys.argv[1]))["stacks"]["card"]))
PY

for FAMILY in $(jq -r 'keys[]' card.json); do
  jq -e --arg name "rbln-driver-$FAMILY" \
    'map(select(.kind == "RBLNDriver" and .metadata.name == $name))[0] // error("no " + $name + " in bundle")' \
    alm-examples.json \
  | jq --slurpfile card card.json --arg family "$FAMILY" '
      ($card[0][$family]) as $c
      | .spec.registry = $c.image.registry
      | .spec.image = $c.image.repository
      | .spec.version = ($c.image.tag | tostring)
      | .spec.imagePullPolicy = ($c.image.pullPolicy // "IfNotPresent")
      | .spec.imagePullSecrets = ["repo-rbln"]
      | .spec.smd.registry = $c.smd.registry
      | .spec.smd.image = $c.smd.repository' > "driver-$FAMILY.json"
done
```

Before applying them, check the following fields in each generated `driver-<family>.json`.

| Field | What to check |
|---|---|
| `spec.registry`, `spec.image`, `spec.version` | The image must exist and support the NPU family and the RHCOS kernel of the worker nodes |
| `spec.imagePullSecrets` | Must include `repo-rbln` from step 3 |
| `spec.smd.registry`, `spec.smd.image` | The RSMD repository must be reachable (RSMD uses the RBLN Driver tag) |

```bash
for FAMILY in $(jq -r 'keys[]' card.json); do
  oc apply -f "driver-$FAMILY.json"
done
```

An `RBLNDriver` for an NPU family that is not present in the cluster has no target nodes and deploys nothing.

To use a private mirror registry, change the registry and repository under `stacks.card.<family>` in the
values file to the mirror, then run step 6 again.

<a id="verification"></a>

## Validating the Configuration

Validate automatic and manual configuration the same way. Wait until the `RBLNClusterPolicy` and every
`RBLNDriver` report `ready`:

```bash
oc wait --for=jsonpath='{.status.state}'=ready rblnclusterpolicy --all --timeout=600s
oc get rblndrivers \
  -o 'custom-columns=NAME:.metadata.name,STATE:.status.state,VERSION:.spec.version,READY:.status.readyNodes,DESIRED:.status.desiredNodes'
oc get pods -n rbln-system
```

`VERSION` must match `stacks.card.<family>.image.tag` in the values file, and `READY` must equal `DESIRED`.
Example output for a cluster with one ATOM™ worker:

```text
NAME               STATE   VERSION   READY   DESIRED
rbln-driver-atom   ready   3.4.1     1       1
```

On nodes that run container workloads with the Device Plugin, check the allocatable NPUs of each node:

```bash
oc get nodes -l rebellions.ai/npu.present=true \
  -o 'custom-columns=NAME:.metadata.name,FAMILY:.metadata.labels.rebellions\.ai/npu\.family,ALLOCATABLE:.status.allocatable.rebellions\.ai/npu'
```

`ALLOCATABLE` must equal the number of NPUs installed in the node. Example output for an ATOM™ worker with four
NPUs:

```text
NAME            FAMILY   ALLOCATABLE
ocp-wk-npu-01   atom     4
```

To run workloads, see [Run an NPU Pod](https://docs.rbln.ai/latest/software/system_management/kubernetes/tutorials/first_npu.html)
in the RBLN SDK documentation.

## Disconnected Environments

In a disconnected (air-gapped) environment, mirror the catalogs and images to an internal registry, and then
follow the procedure above.

- For automatic configuration, in `stacks.rbln-npu-operator.olm` of the values file, set `source` to the name
  of the mirror `CatalogSource` and `catalog` to the mirror catalog image. If `catalog` is set, the playbook
  creates that `CatalogSource` in `openshift-marketplace`.
- For manual configuration, set the Subscription `source` in step 4 to the name of the mirror `CatalogSource`,
  and change the image registries and repositories in step 6 to the mirror.
- Leave `enable_cert_manager` and `enable_istio` set to `false`. The playbook installs cert-manager and Istio
  from public Helm chart repositories.

For the mirroring procedure, see
[Installing the RBLN NPU Operator in a Disconnected or Air-gapped Environment](https://docs.rbln.ai/latest/software/system_management/openshift/airgapped_install.html)
in the RBLN SDK documentation.

## Uninstalling the RBLN NPU Operator

Delete the `RBLNDriver` and `RBLNClusterPolicy` resources, then remove the Operator. This procedure does not
remove NFD or the OpenShift cluster.

```bash
oc delete rblndriver --all
oc delete rblnclusterpolicy --all
CSV=$(oc get subscription rbln-npu-operator -n rbln-system -o jsonpath='{.status.installedCSV}')
oc delete subscription rbln-npu-operator -n rbln-system
oc delete csv "$CSV" -n rbln-system
oc delete operatorgroup rbln-npu-operator -n rbln-system
oc delete namespace rbln-system
```

For details, see the **OpenShift / OLM** tab in
[Uninstalling the RBLN NPU Operator](https://docs.rbln.ai/latest/software/system_management/kubernetes/uninstall_npu_operator.html)
in the RBLN SDK documentation.
