# cilium

Cilium configuration for the Talos cluster built by `lab-proxmox`.
This directory is meant to become a git submodule shared with the ArgoCD repository, so it must stay
self-contained: no references to files outside this directory.

## Files
- `version.yaml`: Helm repo, chart name and chart version (the single pin)
- `values.yaml`: Helm values. Contains the Talos-specific settings and the pod CIDR; anything else
  (custom images, features) is added here and rolled out by ArgoCD
- `gateway-api.yaml`: Gateway API CRD version (standard channel) matching the pinned Cilium release

## Who reads it
- **Bootstrap (`lab-proxmox`, `kubernetes/bootstrap.sh`)**: on a fresh cluster it runs
  `helm install cilium <chart> --repo <repo> --version <version> --values values.yaml` so that ArgoCD
  has a network to run on. It never upgrades or reconfigures an existing installation.
- **ArgoCD (`lab-argo`)**: owns every change after bootstrap. This directory is checked out at `cilium/`;
  the app-of-apps chart reads `version.yaml` (through the `apps/cilium` symlink) to fill the chart pin and
  renders a multi-source Application with `releaseName: cilium` in `kube-system` and
  `$values/cilium/values.yaml`.

## Contract
- `ipam.operator.clusterPoolIPv4PodCIDRList` must match the pod subnet of the Talos machine config
  (`pod_subnet` in lab-proxmox `terraform/environments/k8s`).
- Keep `cgroup.hostRoot: /sys/fs/cgroup` with `cgroup.autoMount.enabled: false` and the agent capabilities
  without `SYS_MODULE`, as the Talos Cilium guide prescribes (Talos mounts cgroup2 itself and does not let
  pods load kernel modules).
- `kubeProxyReplacement: true` with `k8sServiceHost: localhost` / `k8sServicePort: 7445` (KubePrism) is part
  of the node contract: the Talos machine config deploys no kube-proxy because this is true. Turning it off
  later requires enabling kube-proxy in the machine config first (`kubeProxyReplacement` cannot be toggled on
  a running cluster without breaking existing connections).
- Bump `version.yaml` one minor version at a time (Cilium only supports consecutive-minor upgrades) and
  move the submodule pointer in the ArgoCD repo in the same change, so a freshly bootstrapped cluster
  and ArgoCD agree on the version.
- Deleting the ArgoCD Application must not delete Cilium: do not add the resources finalizer.
- `gatewayAPI.enabled: true` requires the Gateway API CRDs before the Cilium operator starts. The
  bootstrap installs the version in `gateway-api.yaml` before Cilium; ArgoCD syncs the same CRDs in an
  earlier wave than Cilium. Bump `gateway-api.yaml` only to a version the pinned Cilium release supports.
- Metrics ports and Grafana dashboards are enabled here, but the scrape configuration (PodMonitors) lives in
  the ArgoCD repo. Keep `prometheus.serviceMonitor.enabled` (and the operator/Hubble/Envoy equivalents) off:
  the bootstrap installs Cilium before the Prometheus Operator CRDs exist.
