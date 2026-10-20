# kernel-pin

Pins each node to the kernel version validated in the RCNS matrix. The NPU driver images are built per
kernel, so the driver loads only on a pinned kernel.

- Ubuntu: installs the kernel packages, holds them with `dpkg` selections, and sets the GRUB default entry.
- RHEL: installs the kernel packages, sets the default boot entry with `grubby`, and locks the version with
  `dnf versionlock`. The kernel must be available in the EUS channel of the subscription.

`node-setup.yaml` picks the kernel for each node OS from `kernel_pin_versions` in the values file.
To pin a different kernel on one node, set `kernel_pin_version` on that line in `hosts`.

| Variable | Default | Description |
|---|---|---|
| `kernel_pin_version` | from the values file | Kernel to pin, in `uname -r` format (RHEL: the NVR without `.x86_64`) |
| `kernel_pin_reboot_enabled` | `true` in `node-setup.yaml` | Reboot into the pinned kernel. Set to `false` on running nodes, then drain and reboot them manually |
