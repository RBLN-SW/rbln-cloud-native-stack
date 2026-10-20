<div align="center">

<img alt="Rebellions" src="https://raw.githubusercontent.com/RBLN-SW/optimum-rbln/dev/assets/rbln_logo_light.png#gh-light-mode-only" width="50%">
<img alt="Rebellions" src="https://raw.githubusercontent.com/RBLN-SW/optimum-rbln/dev/assets/rbln_logo_dark.png#gh-dark-mode-only" width="50%">

# RBLN Cloud Native Stack (RCNS)

*A validated software stack for Kubernetes and OpenShift on RBLN NPUs, with Ansible playbooks that take bare nodes or an existing cluster to a running RBLN NPU Operator.*

[![Documentation](https://img.shields.io/badge/docs-latest-8B5CF6?style=flat-square)](https://docs.rbln.ai/latest/software/system_management/cloud_native/overview.html)
[![Support Matrix](https://img.shields.io/badge/RCNS-Support%20Matrix-10B981?style=flat-square)](https://docs.rbln.ai/latest/supports/version_matrix.html#rbln-cloud-native-stack-rcns)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.34%20%7C%201.35%20%7C%201.36-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](#component-matrix)
[![OpenShift](https://img.shields.io/badge/OpenShift-4.19%20%7C%204.20%20%7C%204.21-EE0000?style=flat-square&logo=redhatopenshift&logoColor=white)](guides/openshift/configure.md)

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%20%7C%2024.04-E95420?style=flat-square&logo=ubuntu&logoColor=white)](#component-matrix)
[![RHEL](https://img.shields.io/badge/RHEL-9.6%20%7C%209.8%20%7C%2010.0%20%7C%2010.2%20(Rebel100%E2%84%A2)-EE0000?style=flat-square&logo=redhat&logoColor=white)](#component-matrix)

</div>

---

## Introduction

RBLN Cloud Native Stack (RCNS) is a validated collection of software for running cloud native workloads on
RBLN NPUs. It is based on Ubuntu or RHEL, Kubernetes, a container runtime, Helm, and the RBLN NPU Operator. It
covers what an NPU cluster needs: Kubernetes cluster setup, container runtime configuration, RBLN NPU Operator
deployment, and container support for the RBLN SDK.

To set up RCNS, follow one of the [guides](#getting-started), which use the
[Ansible playbooks](automation/playbooks/). For component descriptions, such as the RBLN NPU Operator and the
RBLN Container Toolkit, see the
[Cloud Native overview](https://docs.rbln.ai/latest/software/system_management/cloud_native/overview.html) in
the RBLN SDK documentation.

## Objective

RCNS is a validated reference architecture. Common uses:

RCNS provides a mutually validated reference configuration for reliable component compatibility. With RCNS, you can:
- Define production specifications: Use the component matrix as the baseline standard for production deployments.
- Evaluate updates safely: Benchmark and validate features and performance before adopting a new NPU server or RBLN SDK version.
- Standardize workloads: Build and test NPU-accelerated container applications within a consistent, verified environment.
- Accelerate deployment: Quickly stand up a Kubernetes environment with the RBLN NPU Operator using the provided installation guides and playbooks.

> [!NOTE]
> The cluster that the Ubuntu install guide creates has no high availability and is intended only for test and
> PoC environments. A production deployment needs its own high availability, security, and operations setup.
> The RBLN components themselves are validated on the environments in the component matrix.

## Version Policy

RCNS is released in batches.

| Version | Format | Example | Meaning |
|---|---|---|---|
| Batch | `YYYY.MM.PATCH` | 2026.10.0 | The release unit: the whole matrix. |
| RCNS version | `v0.MINOR.PATCH` | v0.12.0 | One column of the matrix. Each minor number maps to one Kubernetes minor version. |

A batch is released in one of two ways.

- **Patch release of an existing batch.** The RCNS versions and the component list stay the same; only
  component patch versions go up. The batch patch number goes up (2026.10.0 → 2026.10.1), and so does the
  patch number of each RCNS version whose components changed (v0.12.0 → v0.12.1). If a change affects the
  compatibility of RBLN components, the release notes describe it and provide either a manual migration
  procedure or migration code.
- **New batch.** A new batch is released when an RCNS version for a new Kubernetes minor version is added. The
  new RCNS version gets the next minor number (for example, v0.13.0). The batch also includes new releases of
  the RBLN and stack components, plus any newly added components; the remaining RCNS versions move to these
  releases and their patch numbers go up. The matrix keeps only the three most recent RCNS versions, so the
  oldest is dropped.

> [!IMPORTANT]
> In the initial batch, 2026.10.x, even a patch release may break backward compatibility, and automatic
> migration is not supported.

## Component Matrix

Each column is one RCNS version: a recommended combination of components validated to work together. For how
versions are numbered, see [Version Policy](#version-policy).

### RCNS Batch 2026.10.0 (Release Date: October 15, 2026)

<table>
<thead>
<tr><th>Component</th><th>RCNS v0.10.0</th><th>RCNS v0.11.0</th><th>RCNS v0.12.0</th></tr>
</thead>
<tbody>
<tr><td>Supported OS</td><td>Ubuntu 22.04, 24.04<br>RHEL 9.6, 9.8, 10.0, 10.2<sup>R</sup></td><td>Ubuntu 22.04, 24.04<br>RHEL 9.6, 9.8, 10.0, 10.2<sup>R</sup></td><td>Ubuntu 22.04, 24.04<br>RHEL 9.6, 9.8, 10.0, 10.2<sup>R</sup></td></tr>
<tr><td>Kubernetes</td><td>v1.34.12</td><td>v1.35.9</td><td>v1.36.4</td></tr>
<tr><td>Container Runtime</td><td>containerd v2.3.4<br>CRI-O v1.34.12</td><td>containerd v2.3.4<br>CRI-O v1.35.0</td><td>containerd v2.3.4<br>CRI-O v1.36.3</td></tr>
<tr><td>Kubespray</td><td>v2.32.0</td><td>v2.32.0</td><td>v2.32.0</td></tr>
<tr><td>CNI</td><td>Calico v3.31.5</td><td>Calico v3.31.5</td><td>Calico v3.31.5</td></tr>
<tr><td>Helm</td><td>v3.18.4</td><td>v3.18.4</td><td>v3.18.4</td></tr>
<tr><td>Serving Stack</td><td>Istio v1.28.0<br>cert-manager v1.17.0</td><td>Istio v1.28.0<br>cert-manager v1.17.0</td><td>Istio v1.28.0<br>cert-manager v1.17.0</td></tr>
<tr><td>RBLN Driver</td><td>v3.5.1<sup>R</sup><br>v3.4.1<sup>A</sup></td><td>v3.5.1<sup>R</sup><br>v3.4.1<sup>A</sup></td><td>v3.5.1<sup>R</sup><br>v3.4.1<sup>A</sup></td></tr>
<tr><td>RBLN NPU Operator</td><td>v0.6.0</td><td>v0.6.0</td><td>v0.6.0</td></tr>
<tr><td>NPU Support</td><td>RBLN-CR13<br>RBLN-CA22, RBLN-CA25</td><td>RBLN-CR13<br>RBLN-CA22, RBLN-CA25</td><td>RBLN-CR13<br>RBLN-CA22, RBLN-CA25</td></tr>
</tbody>
<tfoot>
<tr><td colspan="4"><sup>R</sup> Rebel100™ (RBLN-CR13). <sup>A</sup> ATOM™ (RBLN-CA22, RBLN-CA25). Apart from the NPU Support row, unmarked values apply to both NPU families.</td></tr>
</tfoot>
</table>

Supported OS lists the operating systems validated on worker nodes. Control plane nodes use Ubuntu.

The RBLN NPU Operator installs its own operands (Device Plugin, Driver Manager, Metrics Exporter, and others) at
their default versions, so the matrix does not list them.

For the full support scope, including the RBLN SDK, see the
[Support Matrix](https://docs.rbln.ai/latest/supports/version_matrix.html#rbln-cloud-native-stack-rcns) in the
RBLN SDK documentation.

Red Hat OpenShift is not a column in the matrix, because each OpenShift release brings its own operating
system, container runtime, and Kubernetes version. The RBLN components of RCNS are validated on OpenShift 4.19
through 4.21. See the [OpenShift configuration guide](guides/openshift/configure.md).

## Getting Started

### Prerequisites

- Internet access for the Kubernetes install playbooks. For an OpenShift cluster without internet access,
  see [Disconnected Environments](guides/openshift/configure.md#disconnected-environments).
- An RBLN Portal account, to pull the RBLN Driver and RBLN System Management Daemon (RSMD) images.

This repository does not distribute container images; it provides only configuration files, playbooks, and
guides.

### Installation

Choose the guide for the target environment.

| Environment | Guide |
|---|---|
| Kubernetes (Ubuntu control plane nodes; Ubuntu or RHEL worker nodes) | [RBLN Cloud Native Stack (RCNS) - Install Guide for Ubuntu Server](guides/ubuntu/install.md) |
| Red Hat OpenShift (existing cluster) | [RBLN Cloud Native Stack (RCNS) - Configuration Guide for Red Hat OpenShift](guides/openshift/configure.md) |

## Getting Help

- Read the [RBLN SDK documentation](https://docs.rbln.ai/latest/).
- Check supported versions in the [Support Matrix](https://docs.rbln.ai/latest/supports/version_matrix.html#rbln-cloud-native-stack-rcns).
- See what changed in each release in the [Release Notes](https://docs.rbln.ai/latest/supports/release_note.html).
- Report bugs by opening an issue in this repository with the bug report template.
- Report security vulnerabilities privately through [Contact Us](https://docs.rbln.ai/latest/supports/contact_us.html), not in a
  public issue.
