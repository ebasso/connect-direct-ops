# Connect:Direct Tekton Automation

Tekton pipelines for automating IBM Connect:Direct configuration management in Kubernetes.

## Overview

This repository provides Tekton resources to automate updates to Connect:Direct configuration files (`netmap.cfg` and `userfile.cfg`) running in Kubernetes.

## Prerequisites

- Kubernetes cluster with Tekton Pipelines installed
- kubectl configured to access your cluster
- Connect:Direct deployed and running in namespace: `sterling-cdnode01-dev01`
- Existing PVC: `s0-ibm-connect-direct-pvc`

## Quick Start

### 1. Deploy Resources

```bash
# Set your namespace
NAMESPACE=sterling-cdnode01-dev01

# Deploy service account and RBAC
kubectl apply -f tekton/resources/serviceaccount.yaml -n $NAMESPACE

# Deploy tasks
kubectl apply -f tekton/tasks/ -n $NAMESPACE

# Deploy pipelines
kubectl apply -f tekton/pipelines/ -n $NAMESPACE
```

### 2. Create ConfigMap

Copy the example files from `examples/` directory, do the changes and create the configmap:

```bash
cp examples/*.cfg custom/.

kubectl create configmap cd-userfiles \
  --from-file=netmap.cfg=custom/netmap.cfg \
  --from-file=userfile.cfg=custom/userfile.cfg \
  --from-file=netmap_full.cfg=custom/netmap_full.cfg \
  --from-file=userfile_full.cfg=custom/userfile_full.cfg \
  -n sterling-cdnode01-dev01
```

### 3. Run Pipelines

```bash
NAMESPACE=sterling-cdnode01-dev01
# Update netmap.cfg
kubectl create -f tekton/runs/update-netmap-pipelinerun.yaml -n $NAMESPACE

# Update userfile.cfg
kubectl create -f tekton/runs/update-userfile-pipelinerun.yaml -n $NAMESPACE
```

### 4. Monitor Execution

```bash
NAMESPACE=sterling-cdnode01-dev01
# View pipeline runs
kubectl get pipelineruns -n $NAMESPACE

# View logs (using Tekton CLI)
tkn pipelinerun logs -f --last -n $NAMESPACE

# Or using kubectl
kubectl logs -l tekton.dev/pipelineRun --all-containers=true -n $NAMESPACE
```

## Configuration Files

### Example Files

The `examples/` directory contains sample configuration files:

- **examples/netmap.cfg** - Sample netmap entries to append to existing configuration
- **examples/userfile.cfg** - Sample userfile entries to append to existing configuration
- **examples/netmap_full.cfg** - Complete netmap configuration to replace the entire file 
- **examples/userfile_full.cfg** - Complete userfile configuration to replace the entire file 

### Update Modes

The pipelines support two update modes:

#### Append Mode (Default)
- **Use case**: Add new nodes/users to existing configuration
- **Files to use**: `netmap.cfg` and `userfile.cfg`
- **Behavior**: Appends new entries to the end of existing configuration files

```yaml
- name: update-mode
  value: "append"
```

#### Replace Mode
- **Use case**: Replace entire configuration file with new content
- **Files to use**: `netmap_full.cfg` and `userfile_full.cfg`
- **Behavior**: Backs up existing file and replaces it completely with new configuration

```yaml
- name: update-mode
  value: "replace"
```


## What to Change

### 1. Namespace

If using a different namespace, change it in the kubectl commands:

```bash
# Use your namespace
NAMESPACE=your-namespace

kubectl apply -f tekton/tasks/ -n $NAMESPACE
kubectl apply -f tekton/pipelines/ -n $NAMESPACE
kubectl create -f tekton/runs/update-netmap-pipelinerun.yaml -n $NAMESPACE
```

### 2. PipelineRun

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: update-netmap-run-
spec:
  pipelineRef:
    name: update-netmap-pipeline
  params:
    - name: node-name
      value: "CDNODE01"
    - name: update-mode
      value: "replace"
    - name: pvc-name
      value: "s0-ibm-connect-direct-pvc"
    - name: run-cfgcheck
      value: "true"
    - name: image
      value: "cp.icr.io/cp/ibm-connectdirect/cdu:6.4.0.4-iFix011-2026-01-13"
  serviceAccountName: tekton-pipeline-sa
```

* **PVC Name**: same PVC of your C:D Pod
* **Node Name**: yours C:D node name.
* **Update Mode**: `append` or `replace` (see "Update Modes" section above)
* **Run Cfgcheck**: `true` or `false` (see "Cfgcheck" section above)
* **Image**: same image of your C:D Pod

## Directory Structure

```
connect-direct-ops/
├── examples/
│   ├── netmap.cfg                   # Append mode: entries to add to existing netmap
│   ├── userfile.cfg                 # Append mode: entries to add to existing userfile
│   ├── netmap_full.cfg          # Replace mode: complete netmap configuration
│   └── userfile_full.cfg        # Replace mode: complete userfile configuration
├── tekton/
│   ├── tasks/
│   │   ├── update-netmap-task.yaml      # Task to update netmap.cfg
│   │   └── update-userfile-task.yaml    # Task to update userfile.cfg
│   ├── pipelines/
│   │   ├── update-netmap-pipeline.yaml  # Pipeline for netmap updates
│   │   └── update-userfile-pipeline.yaml # Pipeline for userfile updates
│   ├── runs/
│   │   ├── update-netmap-pipelinerun.yaml   # PipelineRun for netmap
│   │   └── update-userfile-pipelinerun.yaml # PipelineRun for userfile
│   └── resources/
│       ├── serviceaccount.yaml          # RBAC configuration
│       ├── pvc.yaml                     # PVC reference (for documentation)
│       └── configmap-example.yaml       # Example ConfigMaps
├── .gitignore
├── LICENSE
└── README.md
```

## How It Works


1. **Validate ConfigMap**
   - Verifies ConfigMap `cd-userfiles` exists
   - Checks for required configuration keys (`netmap.cfg` or `userfile.cfg`)

2. **Create Backup**
   - Do a copy of file as: `<filename>.backup.YYYYMMDD_HHMMSS`

3. **Update Configuration**
   - **Append Mode**: Adds new entries to end of existing file
   - **Replace Mode**: Replaces entire file with new content
   - Mode is automatically detected based on ConfigMap keys

4. **Verify Update**
   - Confirms file was updated successfully
   - Checks file size and modification timestamp
   - Validates file is readable

## Common Operations

### Update ConfigMap

To update configurations, edit your files and update the ConfigMap:

```bash
# Update ConfigMap
kubectl create configmap cd-userfiles \
  --from-file=netmap.cfg=examples/netmap.cfg \
  --from-file=userfile.cfg=examples/userfile.cfg \
  -n sterling-cdnode01-dev01 \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Run with Custom Parameters

Create a custom PipelineRun with different settings:

cp tekton/runs/update-netmap-pipelinerun.yaml custom-netmap-pipelinerun.yaml


## Cleanup

### Remove All Tekton Resources

To completely remove all Tekton resources (pipelines, tasks, runs) from a namespace:

```bash
# Set your namespace
NAMESPACE=sterling-cdnode01-dev01

# Delete all Tekton resources in one command
kubectl delete pipelineruns,taskruns,pipelines,tasks --all -n $NAMESPACE

# Also delete ConfigMap and service account
kubectl delete configmap cd-userfiles -n $NAMESPACE
kubectl delete -f tekton/resources/serviceaccount.yaml -n $NAMESPACE
```

## Best Practices

1. **Choose the right mode**
   - Use **append mode** (`netmap.cfg`/`userfile.cfg`) when adding new nodes/users
   - Use **replace mode** (`netmap_full.cfg`/`userfile_full.cfg`) when restructuring entire configuration

2. **Edit example files first** - Modify example files with your configuration before creating ConfigMaps

3. **Test in dev first** - Always verify changes in development namespace before production

4. **Keep backups enabled** - Set `backup-enabled: "true"` in production environments

5. **Version control** - Keep your configuration files in git for tracking changes

6. **Monitor pipeline runs** - Check logs after each execution to verify success

7. **Verify ConfigMap content** - Use `kubectl describe configmap` to confirm correct content before running pipelines

8. **Clean old runs** - Regularly delete completed PipelineRuns to avoid clutter

9. **Document changes** - Keep notes on what was changed and why in your git commits

## Disclaimer

**USE AT YOUR OWN RISK**

This automation tool is provided "as is" without warranty of any kind, either express or implied. The author assumes no responsibility for any damages, data loss, or issues that may arise from using this software.

**Important:**
- Always test in a development environment first
- Ensure you have proper backups before running in production
- Verify all configurations match your environment
- Review and understand the code before executing
- The author is not liable for any consequences of using this tool

By using this software, you acknowledge that you do so at your own risk and agree to take full responsibility for any outcomes.

## License

See [LICENSE](LICENSE) file for details.
