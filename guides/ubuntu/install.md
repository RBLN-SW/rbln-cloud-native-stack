# RBLN Cloud Native Stack (RCNS) - Install Guide for Ubuntu Server

A single Ansible playbook installs RCNS on servers with RBLN NPUs. It creates a Kubernetes cluster and
installs Node Feature Discovery (NFD), the RBLN NPU Operator, and its dependencies. It then configures the
Operator with an `RBLNClusterPolicy` and an `RBLNDriver` per NPU family. On an existing Kubernetes cluster,
only the RBLN NPU stack can be installed (see
[Install on an Existing Cluster](#3-install-on-an-existing-cluster-install_k8s-false)).

The procedure is the same for every RCNS version; only the component versions differ, and a
[values file](#choosing-an-rcns-version) selects them.

Control plane nodes use Ubuntu. Worker nodes can use Ubuntu or RHEL. For OpenShift, see
[RBLN Cloud Native Stack (RCNS) - Configuration Guide for Red Hat OpenShift](../openshift/configure.md).

## Choosing an RCNS Version

Choose one RCNS version and use its values file (`automation/values/rcns_values_<version>.yaml`) for every
step. All the versions this guide uses are in the values file.

| Values file key | Description |
|---|---|
| `kubernetes_version` | Kubernetes version |
| `containerd_version`, `crio_version` | Container runtime versions (`container_runtime` selects which one is used) |
| `clm.kubespray` | Kubespray tag used to create the cluster |
| `stacks.card.<family>.image.tag` | RBLN Driver version per NPU family (`rebel100`, `atom`) |
| `components` | RBLN NPU Operator version and the cert-manager and Istio Helm chart versions |
| `stacks.rbln-npu-operator.chart` | RBLN NPU Operator Helm chart repository and version (`repoURL`, `version`) |
| `kernel_pin_versions` | Validated kernel for each node OS (see [Kernel Versions](#kernel-versions)) |

For the components of each RCNS version, see the [Component Matrix](../../README.md#component-matrix) in the
repository README.

## Prerequisites

- Node operating systems:
  - Control plane nodes: Ubuntu 22.04 or 24.04
  - Worker nodes: Ubuntu 22.04 or 24.04; RHEL 9.6, 9.8, or 10.0; or RHEL 10.2 (Rebel100™)
- Internet access from the nodes and a workstation. The nodes must reach the OS package repositories, the
  Kubernetes package repositories, and the container registries.
- A workstation that can reach the nodes over SSH and has the following tools:
  - Git (to clone this repository and Kubespray), Python 3 with venv, Helm, and `kubectl`
  - Ansible (ansible-core 2.15 or later) and the required collections:

    ```sh
    python3 -m venv .venv && . .venv/bin/activate   # on Ubuntu/Debian, install the python3-venv package first
    python3 -m pip install 'ansible-core>=2.15'
    ansible-galaxy collection install -r automation/playbooks/requirements.yml
    ansible --version
    ```
- An RBLN Portal account, to pull the RBLN Driver and RBLN System Management Daemon (RSMD) images.
- For RHEL worker nodes, a Red Hat subscription, because the validated kernels are in the EUS channel. If the
  `RHSM_ORG_ID` and `RHSM_ACTIVATION_KEY` environment variables are set on the workstation, node setup
  registers each node with the activation key. Otherwise, the nodes must already be registered. Uninstalling
  RCNS does not unregister them.
- A RoCE NIC driver on each node, if NPUs on different nodes exchange data over RDMA (RoCE) NICs, for example
  for multi-node `rbln-ccl` collective communication or KV cache transfer in prefill/decode disaggregation.
  Install the driver by following the NIC vendor's documentation. The reference nodes use the Broadcom
  `bnxt_en` and `bnxt_re` drivers.

## Preparing the Hosts File

List every node in the hosts file, `automation/playbooks/hosts`, with its IP address and SSH user. List the
control plane nodes under `[master]` and the worker nodes under `[nodes]`.

```
[master]
192.0.2.10 ansible_user=ubuntu
[nodes]
192.0.2.11 ansible_user=ubuntu
192.0.2.12 ansible_user=cloud-user
```

- SSH key authentication is recommended. Load the key into ssh-agent, or set `ANSIBLE_PRIVATE_KEY_FILE=<key>`.
  If the `--private-key` option is used instead, the cluster creation step (Kubespray) does not receive the
  key.
- For password authentication, append `ansible_ssh_pass=... ansible_become_pass=...` to the node's line
  (requires sshpass).
- For a node with several NICs, append `ip=<node IP>` to set the address that Kubernetes uses for the node.
- To run workloads on a control plane node, list it under `[nodes]` as well.
- Do not add Kubespray groups (`kube_control_plane` and others) to the inventory.
  `automation/clm/kubespray/kubespray-groups.yaml` maps `master` and `nodes` to them.

### Kernel Versions

Node setup pins each node to a validated kernel for which an RBLN Driver image exists, then reboots the node.
`kernel_pin_versions` in the values file picks the kernel for each node OS automatically. RCNS v0.10.0 through
v0.12.0 use the same validated kernels.

| Node OS | Validated kernel |
|---|---|
| Ubuntu 22.04 | `6.8.0-138-generic` |
| Ubuntu 24.04 | `6.8.0-139-generic` |
| RHEL 9.6 | `5.14.0-570.141.1.el9_6` |
| RHEL 9.8 | `5.14.0-687.48.1.el9_8` |
| RHEL 10.0 | `6.12.0-55.104.1.el10_0` |
| RHEL 10.2 (Rebel100™) | `6.12.0-211.55.1.el10_2` |

To pin a different kernel on one node, append `kernel_pin_version=<kernel>` to its line in the hosts file. An
RBLN Driver image must exist for that kernel.

## Installing RCNS

`install.yaml` runs the following steps in order. Flags in the values file control which steps run.

1. Node setup (`node-setup.yaml`): pins the validated kernel. On Ubuntu, it also installs the required packages;
   on RHEL, it sets SELinux to permissive, disables firewalld, and stops NetworkManager from managing the CNI
   interfaces.
2. Kubernetes cluster creation (`k8s-install.yaml`): when `install_k8s` is `true`, creates the cluster with
   Kubespray.
3. Image pull secret: if RBLN Portal credentials are passed, creates the `repo-rbln` secret in the
   `rbln-system` namespace.
4. RBLN NPU Operator installation (`npu-operator.yaml`): installs the RBLN NPU Operator Helm chart from
   `stacks.rbln-npu-operator.chart` in the values file (`repoURL` and `version`; the version matches
   `components.rbln-operator`). The chart also creates the following resources:
   - `RBLNClusterPolicy`, which defines the operands (Device Plugin, Metrics Exporter, and others) and NFD.
     Its common settings come from `automation/manifests/k8s/npu-operator-values.yaml`. The NPU flags in the
     values file (`enable_npu_dra`, `enable_npu_vm_passthrough`, `enable_nfd`, `enable_metrics_exporter`, and
     others) are applied on top of them.
   - One `RBLNDriver` per NPU family, using the image registry, repository, and tag from `stacks.card.<family>`
     in the values file. Each `RBLNDriver` selects its nodes by the `rebellions.ai/npu.family` node label and
     pulls the RBLN Driver and RSMD images with the `repo-rbln` image pull secret.
5. Optional stacks (`addons.yaml`): installs cert-manager if `enable_cert_manager` is `true`, and Istio if
   `enable_istio` is `true`.
6. Validation (optional): when `rcns_validation` is `true`, runs `validation.yaml` at the end.

Unless a step says otherwise, run the commands from the `automation/playbooks` directory. In the commands
below, `<version>` is the chosen RCNS version (for example, `0.12.0`).

### 1. Prepare the Values File

Copy the values file for the RCNS version and edit its flags, or override individual flags on the command
line with `-e`. To keep the defaults, skip the copy and pass `../values/rcns_values_<version>.yaml` wherever
the commands below use `my-values.yaml`.

```sh
cd automation/playbooks
cp ../values/rcns_values_<version>.yaml my-values.yaml
```

| Flag | Default | Description |
|---|---|---|
| `install_k8s` | `true` | Create a Kubernetes cluster with Kubespray. Set to `false` for an existing cluster. |
| `container_runtime` | `containerd` | Container runtime (`containerd` \| `crio`). Used when `install_k8s` is `true`. |
| `rbln_registry_username` | `""` | RBLN Portal username. If set, the playbook creates the `repo-rbln` image pull secret. Otherwise, [create the secret manually](#create-the-image-pull-secret-manually). |
| `rbln_registry_password` | `""` | RBLN Portal password. Do not put it in the file; pass it with `-e` or an encrypted vars file. |
| `enable_npu_operator` | `true` | Install the RBLN NPU Operator (required). |
| `enable_npu_dra` | `false` | Allocate NPUs with DRA instead of the Device Plugin (Kubernetes 1.34 or later). |
| `enable_npu_vm_passthrough` | `false` | Enable VM passthrough (Sandbox Device Plugin, VFIO Manager). |
| `enable_driver_auto_upgrade` | `false` | Upgrade nodes one at a time when the RBLN Driver image changes. |
| `enable_nfd` | `true` | Install Node Feature Discovery, which the NPU family labels require. Set to `false` if NFD is already running in the cluster. |
| `enable_metrics_exporter` | `true` | Install the Metrics Exporter. |
| `enable_cert_manager` | `false` | Install cert-manager. |
| `enable_istio` | `false` | Install Istio base and istiod. |
| `rcns_validation` | `false` | Run `validation.yaml` at the end of the installation. |

To use a private mirror registry, change the registry and repository of the RBLN Driver (`image`) and
RSMD (`smd`) images under `stacks.card.<family>`.

### 2. Install with a New Cluster (`install_k8s: true`)

```sh
export ANSIBLE_PRIVATE_KEY_FILE=<node SSH key>     # or use ssh-agent
ansible-playbook -i hosts install.yaml -e @my-values.yaml \
  -e rbln_registry_username='<RBLN_PORTAL_USERNAME>' -e rbln_registry_password='<RBLN_PORTAL_PASSWORD>'
```

- The cluster creation step clones Kubespray into `automation/clm/kubespray/kubespray` and runs it. Progress is
  logged to `automation/clm/kubespray/kubespray-install.log`.
- The control plane VIP (kube-vip) is disabled by default, so clients reach the cluster through the first
  control plane node. If there are multiple control plane nodes, uncomment the VIP example in
  `automation/clm/kubespray/vars/kubespray-vars_<version>.yaml` and replace `192.0.2.10` with an address
  reserved on the network.
- The kubeconfig is written to `automation/clm/kubespray/artifacts/admin.conf`, which later steps use by
  default.
- If a rerun stops because the control plane never becomes Ready (the CNI is not installed), run Kubespray
  once with `--tags network` (see the Kubespray commands below), then rerun.

To create only the cluster, run `k8s-install.yaml` directly. If `automation/clm/kubespray/kubespray` is cloned
at a tag other than `clm.kubespray`, delete it and rerun.

```sh
ansible-playbook -i hosts k8s-install.yaml -e @my-values.yaml [-e container_runtime=crio]
```

To run Kubespray manually, run the following commands from `automation/clm/kubespray`. These are the same
commands that `k8s-install.yaml` runs.

```sh
git clone --depth 1 --branch <clm.kubespray> https://github.com/kubernetes-sigs/kubespray.git
python3 -m venv .venv && . .venv/bin/activate && python3 -m pip install -r kubespray/requirements.txt
cd kubespray && ansible-playbook -i ../../../playbooks/hosts -i ../kubespray-groups.yaml -b \
  -e @../vars/kubespray-vars_<version>.yaml \
  -e container_manager=<containerd|crio> -e <containerd|crio>_version=<version from the values file> \
  -e artifacts_dir="$PWD/../artifacts" cluster.yml
```

### 3. Install on an Existing Cluster (`install_k8s: false`)

The cluster must run the Kubernetes version that the RCNS version specifies, and its nodes must be listed in
the hosts file.

Node setup processes the nodes one at a time; it pins the kernel and then reboots the node. If the nodes are
running workloads, add `-e kernel_pin_reboot_enabled=false` to defer the reboot, then drain and reboot each
node manually, one at a time. After a node reboots, run the command again; the run verifies the running kernel
and, on RHEL, removes old kernels.

```sh
ansible-playbook -i hosts install.yaml -e @my-values.yaml -e install_k8s=false \
  -e kubeconfig=<kubeconfig of the existing cluster> [-e kernel_pin_reboot_enabled=false] \
  -e rbln_registry_username='<RBLN_PORTAL_USERNAME>' -e rbln_registry_password='<RBLN_PORTAL_PASSWORD>'
```

### Create the Image Pull Secret Manually

If the credential flags are not passed, create the secret before installing. Without it, the installation
continues with only a warning, but the RBLN Driver image cannot be pulled and the NPU nodes do not become
ready.

Point `kubectl` at the target cluster first. The playbook's `-e kubeconfig=...` does not reach `kubectl` in
the shell, so without `KUBECONFIG` the secret may be created in a different cluster.

```sh
export KUBECONFIG=<kubeconfig of the target cluster>   # new cluster: automation/clm/kubespray/artifacts/admin.conf
kubectl create namespace rbln-system --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret docker-registry repo-rbln --docker-server=repo.rebellions.ai \
  --docker-username='<RBLN_PORTAL_USERNAME>' --docker-password='<RBLN_PORTAL_PASSWORD>' -n rbln-system
```

## Validating the Installation

Validation compares each item of the installed stack with the values file of the RCNS version. It reports the
target and installed version of each item, then gives an overall verdict. On a cluster without NPUs, the NPU
items are reported but do not count as failures.

Pass the same values file (`my-values.yaml`) used for the installation and, for an existing cluster,
`-e kubeconfig=<path>`. The uninstall commands take the same arguments.

```sh
ansible-playbook validation.yaml -e @my-values.yaml [-e kubeconfig=<path>]
```

Example output for RCNS v0.12.0 on three nodes, one of which is an ATOM™ worker with four NPUs (image list
shortened):

```text
TASK [Report Target vs Installed] **********************************************
ok: [localhost] => {
    "msg": [
        "cluster reachable      : ok",
        "nodes Ready            : 3/3",
        "kubelet    target v1.36.4 / installed: ['v1.36.4'] / mismatch: []",
        "runtime    target containerd 2.3.4 · crio 1.36.3 / installed: ['containerd://2.3.4']",
        "kernel     installed: ['6.8.0-138-generic'] (report only — the pin depends on the environment)",
        "operator   target rbln-npu-operator-chart-0.6.0 / installed: rbln-npu-operator-chart-0.6.0",
        "pods(rbln-system) not running: []",
        "warning events (last 5)  : []",
        "images(rbln-system)    : ['docker.io/rebellions/rbln-npu-operator:v0.6.0', ...]",
        "components target      : {'rbln-operator': '0.6.0', 'cert-manager': '1.17.0', 'istio': '1.28.0'}",
        "NPU allocatable        : 4 — per node ['worker-1=4']",
        "NPU nodes (driver pods ∪ labels) : ['worker-1'] / allocatable 0 : []"
    ]
}

TASK [Validation verdict] ******************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "RCNS validation passed — the stack matches the values file (rcns_version 0.12.0)."
}
```

Validation fails if a node is not Ready, a version differs from its target, a pod in `rbln-system` is not
running, or an NPU node has no allocatable NPUs. With `enable_npu_dra: true`, validation skips the
allocatable NPU count check.

## Uninstalling RCNS

Uninstalling takes three steps, in the reverse order of the installation. Restoring the nodes removes packages
and directories, so do not run step 3 on the nodes of a running cluster.

```sh
# 1) Remove the stacks (optional stacks, then the NPU Operator)
ansible-playbook uninstall.yaml --tags stacks -e @my-values.yaml [-e kubeconfig=<path>]

# 2) Remove the cluster: only a cluster created with install_k8s. Skip this for an existing cluster
cd ../clm/kubespray && . .venv/bin/activate && cd kubespray
ansible-playbook -i ../../../playbooks/hosts -i ../kubespray-groups.yaml \
  -b -e @../vars/kubespray-vars_<version>.yaml reset.yml -e reset_confirmation=yes
deactivate && cd ../../../playbooks

# 3) Restore the nodes (unpin packages, clean up packages and directories)
ansible-playbook -i hosts uninstall.yaml --tags nodes -e @my-values.yaml
```

For cert-manager and Istio, removing the stacks deletes only the releases that RCNS installed. Releases with
the same names that existed before RCNS, and the `istio-system` and `cert-manager` namespaces, are kept. If a
deletion fails, the command fails.

Restoring the nodes supports both Ubuntu and RHEL. It removes the kernel pin (the Ubuntu hold or the RHEL
versionlock) and deletes the Kubernetes and container runtime packages and leftover directories. The pinned
kernel stays installed so that the node still boots. Settings that node setup or Kubespray changed, such as
SELinux, firewalld, the NetworkManager exclusion of CNI interfaces, swap, and base packages, are not reverted.
