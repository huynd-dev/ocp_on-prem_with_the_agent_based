# 1. Prerequisites — Bastion Host Preparation

Everything below runs on the bastion node, before any OpenShift client tool is installed.

## OS tuning

Raise file-descriptor/process limits, tune kernel networking sysctls, and sync the timezone.
The full script is at [`../scripts/tune-os.sh`](../scripts/tune-os.sh):

```bash
sudo ./scripts/tune-os.sh
```

What it does:

- Clears any existing `nofile` / `nproc` / `memlock` limits in `/etc/security/limits.conf` and
  replaces them with higher values suited to running an image registry and OpenShift tooling.
- Sets the system timezone to `Asia/Ho_Chi_Minh` and enables NTP.
- Loads the `overlay` and `br_netfilter` kernel modules (required by container runtimes).
- Writes `/etc/sysctl.d/k8s.conf` with connection-tracking, backlog and inotify tunables sized
  for a registry/cluster host, then applies them with `sysctl --system`.

Adjust the timezone and limits to your environment before running it on hosts outside this project.

## Install the OpenShift client tools

See [04-cluster-install.md § Download the OpenShift client tools](04-cluster-install.md#download-the-openshift-client-tools).

---
Next: [2. Registry Setup](02-registry-setup.md)
