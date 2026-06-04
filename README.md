# sno-iac

Infrastructure as Code for Single Node OpenShift (SNO) clusters. Provides a pull/deploy workflow to export a running SNO cluster's configuration to YAML and restore it to a new or replacement cluster.

## Overview

This project follows the same pull/deploy IaC pattern used in `aap-iac` and `satellite-iac`:

- **Pull** — connect to a running SNO cluster, export its configuration to portable YAML files, and commit them to GitHub
- **Deploy** — read the exported YAML files and apply them to a target cluster in the correct dependency order

Exported files are stored under `iac-exports/` and committed to this repository so they can be version-controlled and used to reproduce the cluster configuration on demand.

## Playbooks

### `pull_sno_config.yml`

Connects to a running SNO cluster and exports its configuration to `iac-exports/`. At the end of the run the exported files are committed and pushed to this GitHub repository.

**What is exported:**

| Directory | Contents |
|---|---|
| `cluster/` | ClusterVersion, OAuth, APIServer, Proxy, Scheduler, Ingress, Image, Console, DNS, FeatureGate, image registry operator config |
| `machine-config/` | MachineConfigs and MachineConfigPools |
| `operators/` | CatalogSources, OperatorGroups, Subscriptions, installed CSV inventory, HyperConverged, LVMCluster, AAP CRs (AnsibleAutomationPlatform, AutomationController, AutomationHub, EDA) |
| `storage/` | StorageClasses, VolumeSnapshotClasses, PersistentVolumes |
| `networking/` | IngressControllers, cluster-scoped NetworkAttachmentDefinitions, NMState instance, NodeNetworkConfigurationPolicies |
| `rbac/` | Custom ClusterRoles, ClusterRoleBindings, OpenShift Users, htpasswd secret (bcrypt hashes) |
| `namespaces/` | Namespace and Project definitions, plus per-namespace resources: Deployments, StatefulSets, DaemonSets, CronJobs, Services, Routes, ConfigMaps, NetworkPolicies, PVCs, NetworkAttachmentDefinitions, VirtualMachines, DataVolumes, and a Secrets inventory (names/types only — no secret data) |

**What is NOT exported:**

- Secret data (passwords, tokens, keys) — only secret names and types are recorded
- PersistentVolume disk contents — PVs are listed but data requires snapshot/replication to migrate
- Cluster-instance-specific credentials injected at runtime

**Required variables (pass via AAP survey or Extra Vars):**

| Variable | Description |
|---|---|
| `survey_url` | API URL of the source SNO cluster (e.g. `https://api.cluster.example.com:6443`) |
| `survey_token` | Bearer token with cluster-admin access |
| `survey_github_token` | GitHub personal access token with write access to this repository |

**Example execution:**
```bash
ansible-playbook playbooks/pull_sno_config.yml \
  -e "survey_url=https://api.cluster.example.com:6443" \
  -e "survey_token=<token>" \
  -e "survey_github_token=<github-token>"
```

---

### `deploy_sno_config.yml`

Reads the exported YAML files from `iac-exports/` and applies them to a target SNO cluster. Resources are applied in strict dependency order so that each resource's prerequisites exist before it is created.

**Deployment order:**

1. **htpasswd secret** — applied first because the OAuth config references it by name
2. **Cluster config singletons** — OAuth (identity providers), APIServer, Proxy, Scheduler, Image, Console, FeatureGate applied as patches; DNS and Ingress are skipped because their specs contain cluster-instance-specific values (base domain, ingressDomain)
3. **Image registry operator config** — restores storage backend, managementState, and custom routes
4. **Custom RBAC** — ClusterRoles, ClusterRoleBindings, Users
5. **Networking infrastructure** — NMState instance (waits for Available), NodeNetworkConfigurationPolicies, cluster-scoped NetworkAttachmentDefinitions, IngressControllers
6. **Storage** — StorageClasses and VolumeSnapshotClasses (required before PVCs and operators that reference a storageClassName)
7. **Namespaces and Projects** — required before any namespace-scoped resources
8. **OLM operators** — OperatorGroups, then Subscriptions; waits for all Subscriptions to reach `AtLatestKnown` before proceeding
9. **Operator custom resources** — HyperConverged (waits for Available before VMs are created), then AAP CRs
10. **Per-namespace workloads** — applied across all exported namespace directories in order: ConfigMaps → NetworkAttachmentDefinitions → NetworkPolicies → PVCs → Deployments/StatefulSets/DaemonSets/CronJobs → Services → Routes → DataVolumes → VirtualMachines
11. **MachineConfigs** — disabled by default; triggers a SNO node reboot when enabled

**Required variables:**

| Variable | Description |
|---|---|
| `survey_url` | API URL of the target SNO cluster |
| `survey_token` | Bearer token with cluster-admin access |
| `survey_apply_machine_configs` | Set to `true` to apply MachineConfigs (triggers node reboot). Default: `false` |

**Example execution:**
```bash
ansible-playbook playbooks/deploy_sno_config.yml \
  -e "survey_url=https://api.new-cluster.example.com:6443" \
  -e "survey_token=<token>"
```

**Post-deploy manual steps:**

After the playbook completes, the following must be done manually before workloads are fully operational:

1. **Recreate secrets** — review each namespace's `iac-exports/namespaces/<ns>/secrets_inventory.yml` and recreate any required secrets. Workloads that depend on missing secrets will fail to start.
2. **Restore VM disk data** — PVCs are created but empty. Restore disk contents via storage snapshots or volume replication before starting VirtualMachines.
3. **Apply MachineConfigs** — if custom node configuration is required, re-run the deploy playbook with `survey_apply_machine_configs=true` during a scheduled maintenance window.

## Export Structure

```
iac-exports/
├── cluster/                    # Cluster-level config singletons
├── machine-config/             # MachineConfigs and MachineConfigPools
├── networking/                 # NMState, NNCPs, NADs, IngressControllers
├── operators/                  # OLM resources and operator CRs
├── rbac/                       # ClusterRoles, ClusterRoleBindings, Users, htpasswd
├── storage/                    # StorageClasses, VolumeSnapshotClasses, PVs
└── namespaces/
    ├── namespaces.yml          # All user namespace definitions
    ├── projects.yml            # OpenShift Project wrappers
    └── <namespace>/            # One directory per user namespace
        ├── configmaps.yml
        ├── cronjobs.yml
        ├── data_volumes.yml
        ├── daemonsets.yml
        ├── deployments.yml
        ├── network_attachment_definitions.yml
        ├── network_policies.yml
        ├── pvcs.yml
        ├── routes.yml
        ├── secrets_inventory.yml   # Names/types only — no secret data
        ├── services.yml
        ├── statefulsets.yml
        └── virtual_machines.yml
```

## Collections Required

```bash
ansible-galaxy collection install kubernetes.core
ansible-galaxy collection install -r collections/requirements.yml
```
