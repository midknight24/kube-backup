## Kube-backup

#### What is kube-backup?

kube-backup is a scaffolding tool that generates a set of yamls based on [Argo Workflows](https://argoproj.github.io/workflows/) that backups PVCs (using restic) in kubernetes application. 

During the backup process, kube-backup will first stop all related pods, then backup all related PVCs, and finally resume all workloads.

#### Prerequisites

- kubernetes cluster
- argo workflow installed
- [ytt](https://carvel.dev/ytt/docs/v0.52.x/) 0.52.1

#### Installation

- change `data.yaml` according to application specific parameters
- run ytt to generate yamls

```bash
ytt -f templates/ --data-values-file data.yaml  --output-files ./output
```

- install rbac related resources

```bash
kubectl create -f ./output/rbac
```

- install generated argo workflow templates

```bash
kubectl create -f ./output/workflow-template
```

#### Usage

- to run backup, create a backup workflow

```bash
kubectl create -f ./output/workflow/manage-backup.yml
```

- to restore a backup, first get the tag of the snapshot to be restored

```bash
kubectl create -f ./output/workflow/show-snapshots.yml
```

then run the following command to connect to argo workflow webui:

```bash
kubectl -n argo port-forward service/argo-server 2746:2746
```

open <http://locahost:2746> in browser and locate the log of showsnapshots workflow pod, here is how the log should look like:

```
ID        Time                 Host               Tags                  Paths
-------------------------------------------------------------------------------------------------
4a9df45d  2025-09-30 01:39:39  autoquiz-test-pvc  2025-09-30T01:38:46Z  /source/autoquiz-test-pvc
16c8fc19  2025-09-30 01:41:40  autoquiz-test-pvc  2025-09-30T01:40:50Z  /source/autoquiz-test-pvc
646eff32  2025-09-30 08:43:54  autoquiz-test-pvc  2025-09-30T08:43:01Z  /source/autoquiz-test-pvc
-------------------------------------------------------------------------------------------------
3 snapshots
time="2025-09-30T08:48:30 UTC" level=info msg="sub-process exited" argo=true error="<nil>"

```

All PVC use their own restic repos, but their snapshots will share the same tag if generated in the same workflow.

Say we want to restore all pvcs to tag `2025-09-30T01:40:50Z`：

change the `backup-id` value in `output/workflow/manage-restore` to `2025-09-30T01:40:50Z`, then run

```bash
kubectl create -f ./output/workflow/manage-restore.yml
```

check the logs in argo workflow to see if it's successful or not.







