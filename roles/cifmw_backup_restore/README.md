# cifmw_backup_restore

Automate OpenStack on OpenShift backup and restore operations using OADP
(OpenShift API for Data Protection) and Velero. The role supports three
actions: **backup**, **restore**, and **cleanup**.

- **backup** — creates Galera database dumps, optionally backs up OVN NB/SB
  databases onto their PVCs, then creates Velero backups of labeled PVCs
  (via CSI snapshots) and cluster resources.
- **restore** — performs an ordered Velero restore sequence (PVCs,
  foundation, infrastructure, control plane, Galera, optional OVN file restore,
  full control plane resume, Neutron–OVN sync, dataplane, EDPM), with
  Neutron–OVN verification and sync (**log** mode, then **repair**) run before
  the EDPM deployment to prevent data plane outage (matching user guide Step 10).
- **cleanup** — tears down dataplane and control-plane resources so the
  namespace is ready for a fresh restore.

## Privilege escalation

None. All cluster operations are performed through `oc` against the target
OpenShift cluster. The `e2e.yml` and `main.yml` entry points run them with
`KUBECONFIG` set to `cifmw_openshift_kubeconfig` when it is defined, so they
follow the cluster the framework targets (for example a hosted cluster).
Otherwise the ambient `KUBECONFIG` is used. The `openshift_adp` and
`deploy_minio` roles do the same only when called through these entry points.

## Parameters

### Common

* `cifmw_backup_restore_action`: (String) Action to perform. Must be one of `backup`, `restore`, or `cleanup`. Defaults to `""` (role will fail if unset).
* `cifmw_backup_restore_namespace`: (String) Target OpenStack namespace. Defaults to `openstack`.
* `cifmw_backup_restore_oadp_namespace`: (String) Namespace where Velero/OADP is running. Defaults to `openshift-adp`.
* `cifmw_backup_restore_auto_ack`: (Boolean) Skip interactive pause prompts when `true`. Defaults to `false`.
* `cifmw_backup_restore_ovn_db`: (Boolean) When `true` (default), the **backup** path labels OVN NB/SB PVCs and runs `ovsdb-client` backup before the OADP PVC backup, and the **restore** path runs OVN NB/SB file restore after Galera (when timestamped files exist on the PVC) before resuming the full control plane. Set to `false` to skip both; `neutron-ovn-db-sync` still runs (before EDPM) when OVN files were not backed up.
* `cifmw_backup_restore_ovn_db_ready_timeout`: (String) Timeout for `oc wait` on OVN database pods during OVN backup/restore. Defaults to `5m`.

### Backup

* `cifmw_backup_restore_galera_backup_timeout`: (String) Timeout for `oc wait` on Galera backup jobs. Defaults to `10m`.
* `cifmw_backup_restore_galera_storage_class`: (String) StorageClass for Galera backup PVCs. Empty string uses the cluster default. Defaults to `""`.
* `cifmw_backup_restore_galera_storage_request`: (String) Size of the Galera backup PVC. Defaults to `5Gi`.
* `cifmw_backup_restore_galera_transfer_storage_request`: (String) Size of the Galera transfer storage PVC. Defaults to `5Gi`.
* `cifmw_backup_restore_oadp_backup_timeout`: (String) Timeout for OADP PVC and resource backup completion. Defaults to `30m`.
* `cifmw_backup_restore_storage_location`: (String) Velero `BackupStorageLocation` name. Defaults to `velero-1`.
* `cifmw_backup_restore_backup_ttl`: (String) TTL for Velero backups. Defaults to `720h`.
* `cifmw_backup_restore_data_node`: (String) Worker that runs the Galera dump jobs and the Velero data movers. Empty uses the worker recorded on the dump PVCs, else the first ready worker, and only on KubeVirt hosted clusters; set it to turn pinning on elsewhere. Defaults to `""`.
* `cifmw_backup_restore_snapshot_move_data`: (Boolean) Enable Velero snapshot data mover. When `true`, cleanup also deletes labeled PVCs. Defaults to `true`.

### Restore

* `cifmw_backup_restore_backup_timestamp`: (String) Timestamp suffix that identifies the backup to restore (e.g. `20260311-081234`). **Required** when `cifmw_backup_restore_action` is `restore`.
* `cifmw_backup_restore_restore_timeout`: (Integer) Seconds to wait for each Velero Restore to reach a terminal phase. Defaults to `900`.
* `cifmw_backup_restore_infra_ready_timeout`: (String) Timeout for `oc wait` on `OpenStackControlPlaneInfrastructureReady`. Defaults to `20m`.
* `cifmw_backup_restore_ctlplane_ready_timeout`: (String) Timeout for `oc wait` on control plane `Ready` after removing the deployment-stage annotation. Defaults to `10m`.
* `cifmw_backup_restore_strict_restore`: (Boolean) Fail on Velero `PartiallyFailed` status when `true`; only warn when `false`. Defaults to `true`.
* `cifmw_backup_restore_restore_content`: (String) Content flag passed to `restore_galera` (`--content`). Defaults to `data`.
* `cifmw_backup_restore_edpm_deploy_timeout`: (String) Timeout for `oc wait` on the post-restore EDPM deployment. Defaults to `40m`.
* `cifmw_backup_restore_pin_pvcs`: (Boolean) Enable PVC-to-node pinning during restore for WaitForFirstConsumer storage classes. Defaults to `false`.
* `cifmw_backup_restore_extra_resource_modifier_rules`: (List) Extra Velero resource modifier rules appended to the restore ConfigMap, for example to give the restored control plane different IPs. Each rule has `conditions` and any of `patches` (JSON Patch), `mergePatches` or `strategicPatches`; `patchData` may be a mapping or a string. Defaults to `[]`.
* **Neutron–OVN** sync runs after the control plane is ready but before EDPM deployment, following [user guide Step 10](https://github.com/openstack-k8s-operators/dev-docs/blob/main/backup-restore/user-guide.md#step-10-verify-and-sync-neutron-to-ovn): `neutron-ovn-db-sync-util` runs in `log` mode first (`neutron-dist.conf`, `neutron.conf`, `neutron.conf.d`). **Repair** runs if `cifmw_backup_restore_ovn_db` is `false` (no OVN NB/SB file backup was taken), or if log-mode stdout/stderr contains a `WARNING` line—Neutron reports drift that way while still exiting 0. If OVN file backup/restore was enabled and log output has no `WARNING` lines, repair is skipped as redundant.

### End-to-end orchestration (e2e.yml)

* `cifmw_backup_restore_install_deps`: (Boolean) Install MinIO, OADP, and (only when `cifmw_backup_restore_run_backup` is `true`) GaleraBackup CRs. MinIO is skipped when `cifmw_openshift_adp_s3_url` is set, in which case `cifmw_openshift_adp_s3_access_key` and `cifmw_openshift_adp_s3_secret_key` must be provided. Defaults to `true`.
* `cifmw_backup_restore_create_workload`: (Boolean) Create a test VM with floating IP before backup. Defaults to `true`.
* `cifmw_backup_restore_run_backup`: (Boolean) Run the backup step. Defaults to `true`.
* `cifmw_backup_restore_run_cleanup`: (Boolean) Run the cleanup step. Defaults to `true`.
* `cifmw_backup_restore_run_restore`: (Boolean) Run the restore step. Defaults to `true`.
* `cifmw_backup_restore_run_post_tempest`: (Boolean) Run tempest validation after restore. Defaults to `false`.
* `cifmw_backup_restore_test_swift_data`: (Boolean) Upload a random file to Swift before backup and verify it can be downloaded after restore. Catches Swift data loss caused by xattr issues (OSPRH-29818). Defaults to `true`.
* `cifmw_backup_restore_swift_test_container`: (String) Swift container name for the test object. Defaults to `backup-test-container`.
* `cifmw_backup_restore_swift_test_object`: (String) Object name for the test file. Defaults to `backup-test-object`.
* `cifmw_backup_restore_swift_test_file_size_bytes`: (Integer) Size of the random test file in bytes. Defaults to `1048576` (1 MiB).

### Cleanup

* `cifmw_backup_restore_cleanup_ctlplane`: (Boolean) Delete control-plane resources during cleanup. Defaults to `true`.
* `cifmw_backup_restore_cleanup_ignore_static_pvs`: (Boolean) Cleanup fails when PVCs in the namespace use a StorageClass with the `kubernetes.io/no-provisioner` provisioner (such as `local-storage`), because deleting them leaves their pre-created PVs `Released` and the restored PVCs can never bind. Set to `true` only if you recycle those PVs yourself. Defaults to `false`.
* `cifmw_backup_restore_cleanup_dataplane`: (Boolean) Delete dataplane resources during cleanup. Defaults to `true`.

## Storage requirements

* **Snapshot-capable CSI storage is needed for data.** PVC contents are moved
  with Velero's CSI snapshot data mover. Volumes on a class without CSI
  snapshots, such as `local-storage`, are skipped
  (`backup.velero.io/skipped-no-csi-pv: true`), so their contents (for example
  OVN DB files and Glance images on such PVCs) are not in the backup. Galera
  data is restored from the dump on the `GaleraBackup` PVC, which must be on a
  snapshot-capable class.
* **In-place cleanup and restore needs dynamic provisioning.** See
  `cifmw_backup_restore_cleanup_ignore_static_pvs`.
* **KubeVirt hosted clusters on node-local infra storage.** With a
  `WaitForFirstConsumer` infra class, a volume is created on the node of the
  first worker VM that uses it, and a CSI snapshot of it can only be cloned and
  attached on that node. Pods that land on a worker VM on another node fail,
  for example with `timeout on preparing data upload`. On hosted clusters
  (`cifmw_deploy_hosted_cluster`), or whenever `cifmw_backup_restore_data_node`
  is set, the backup therefore pins the Galera dump Jobs to one worker with a
  `nodeSelector`, and the Velero data movers to the same worker through the
  DPA's `spec.configuration.nodeAgent.loadAffinity` (restarting the node
  agents). The worker defaults to the one recorded in the
  `cifmw.openstack.org/data-node` annotation on the `mysql-backup-*` PVCs,
  else the first ready worker, and is recorded there after the dumps. The
  dump volumes cannot move, so if they already exist on a different worker,
  set `cifmw_backup_restore_data_node` to that worker. Limits: the daily
  `GaleraBackup` cron jobs are not pinned, and the `GaleraRestore` pod is
  created by the operator from a CR with no scheduling fields, so a restore
  into a hosted cluster can still fail to attach the restored dump volume.

## Examples

### Running a backup

```YAML
- hosts: localhost
  tasks:
    - name: Backup OpenStack
      ansible.builtin.include_role:
        name: cifmw_backup_restore
      vars:
        cifmw_backup_restore_action: backup
        cifmw_backup_restore_namespace: openstack
        cifmw_backup_restore_auto_ack: true
```

### Restoring from a backup

```YAML
- hosts: localhost
  tasks:
    - name: Restore OpenStack
      ansible.builtin.include_role:
        name: cifmw_backup_restore
      vars:
        cifmw_backup_restore_action: restore
        cifmw_backup_restore_backup_timestamp: "20260311-081234"
        cifmw_backup_restore_auto_ack: true
```

### Restoring into a different cluster

Back up on the source cluster with an S3 endpoint both clusters can reach,
then run restore-only against the destination cluster (its
`cifmw_openshift_kubeconfig`, or its `KUBECONFIG` when that is not set). The
destination needs the same `openstack-operator` version as the backup, and
must not have a running control plane in the target namespace. Prepare it by
deploying with `cifmw_deploy_architecture_stopper` set to a stage before the
control plane is created (for example
`-e cifmw_deploy_architecture_stopper=pre_kustomize_stage_<ID>`; see the
`kustomize_deploy` role), so the operators, MetalLB and NADs exist but no
`OpenStackControlPlane` does.

```YAML
# Source cluster
- hosts: localhost
  tasks:
    - name: Backup to the shared S3
      ansible.builtin.import_role:
        name: cifmw_backup_restore
        tasks_from: e2e.yml
      vars:
        cifmw_openshift_adp_s3_url: https://s3.example.com
        cifmw_openshift_adp_s3_access_key: "{{ s3_access_key }}"
        cifmw_openshift_adp_s3_secret_key: "{{ s3_secret_key }}"
        cifmw_backup_restore_run_cleanup: false
        cifmw_backup_restore_run_restore: false

# Destination cluster
- hosts: localhost
  tasks:
    - name: Restore from the shared S3
      ansible.builtin.import_role:
        name: cifmw_backup_restore
        tasks_from: e2e.yml
      vars:
        cifmw_openshift_adp_s3_url: https://s3.example.com
        cifmw_openshift_adp_s3_access_key: "{{ s3_access_key }}"
        cifmw_openshift_adp_s3_secret_key: "{{ s3_secret_key }}"
        cifmw_openshift_adp_bsl_access_mode: ReadOnly
        cifmw_backup_restore_create_workload: false
        cifmw_backup_restore_run_backup: false
        cifmw_backup_restore_run_cleanup: false
        cifmw_backup_restore_backup_timestamp: "20260311-081234"
```

To keep the bucket outside both clusters, deploy MinIO on a management
cluster with `KUBECONFIG=<management kubeconfig> ansible-playbook
playbooks/deploy_minio.yaml`. It prints the `minio-api` route, which is the
value for `cifmw_openshift_adp_s3_url` (`https://<route host>`), and the
credentials default to `minio` / `minio123`. The management cluster has to
exist first, so on a fresh KubeVirt hosted environment deploy it, then run the
backup step on its own, for example with
`post_deployment.sh --tags backup-restore -e cifmw_run_backup_restore_test=true`
and the S3 variables.

Velero on the destination imports the source's `Backup` objects from the
bucket after about a minute; the restore pre-flight retries for up to about
three minutes while waiting for them.

### Giving the restored control plane different IPs

When the source control plane stays up, the restored one needs its own
addresses on the shared networks. A backup embeds them in the restored
`OpenStackControlPlane` (`metallb.universe.tf/loadBalancerIPs` annotations)
and the `dns-svc` `DNSData`. Velero has no find-and-replace, so each path is
patched explicitly:

```YAML
cifmw_backup_restore_extra_resource_modifier_rules:
  - conditions:
      groupResource: openstackcontrolplanes.core.openstack.org
    mergePatches:
      - patchData:
          spec:
            keystone:
              template:
                override:
                  service:
                    internal:
                      metadata:
                        annotations:
                          metallb.universe.tf/loadBalancerIPs: 172.17.0.180
```

Velero skips resources that already exist, and the restore templates do not
set `existingResourcePolicy`. The destination therefore keeps its own
`NetworkAttachmentDefinition`s, so its pod address ranges come from how the
destination was deployed and a rule patching a NAD that already exists has no
effect. Likewise leave the data plane entries (`IPSet`, `Reservation`,
`OpenStackDataPlaneNodeSet` and the EDPM `DNSData`) untouched: the data plane
hosts keep their addresses. The destination cluster needs MetalLB pools
covering the new addresses.

### Cleaning up before a restore

```YAML
- hosts: localhost
  tasks:
    - name: Cleanup namespace
      ansible.builtin.include_role:
        name: cifmw_backup_restore
      vars:
        cifmw_backup_restore_action: cleanup
        cifmw_backup_restore_auto_ack: true
        cifmw_backup_restore_cleanup_ctlplane: true
        cifmw_backup_restore_cleanup_dataplane: true
```
