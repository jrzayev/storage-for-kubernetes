# Storage for Kubernetes (Pacemaker + GFS2)

Shared storage for Kubernetes workers using a Pacemaker cluster and GFS2. Built with [@tima722](https://github.com/tima722). After evaluating existing options, this approach was chosen because others did not fit our infrastructure; Pacemaker plus GFS2 met our requirements.

## Requirements

- Kubernetes worker nodes must have (or be given) a **shared disk** on VMware.
- On the machine where you run the setup:
  - **Python** 3.6
  - **passlib** (`pip install passlib`)
  - **Ansible** 2.8 or newer

## Run

From the project root, run and answer all prompts:

```bash
python3.6 initial-vmware.py
```

When asked, provide workers and IPs in the same order, for example:

- **Workers:** `test1 test2 test3`
- **IPs:** `192.168.1.2 192.168.1.3 192.168.1.4`

Hostnames and IPs must match by position.

## Optional: Add shared disk via vCenter

If you answer **yes** to adding a shared disk, the script will:

1. Ask for vCenter IP, ESXi IP, credentials, datastore, datacenter, disk size, and head worker.
2. Generate Ansible vars and run `vsphere-site.yaml`, which:
   - Powers off the workers
   - Adds the disk to the head worker
   - Configures VMX (SCSI bus sharing) on ESXi
   - Powers the workers back on

If you already attached the shared disk yourself, answer **no** and continue; the playbook will use the existing disk.

## What gets configured

- **Pacemaker** cluster across the workers (pcs, corosync, pacemaker).
- **Fencing** via VMware SOAP (vCenter) so failed nodes can be fenced.
- **DLM** and **CLVM** for clustered locking and LVM.
- **GFS2** on the shared LVM LV, mounted at a folder you specify (e.g. `/clusterfs`).

Kubernetes can use this mount via `hostPath` (see `example_sc_pv_pvc.yml`). The PV uses `hostPath.path: "/clusterfs"` and `ReadWriteMany`; point this path to the GFS2 mount folder you chose during setup.

## Tested

- CentOS 7.6  
- Red Hat Enterprise Linux 7.6  

## Project layout

- `initial-vmware.py` – interactive setup; writes inventory and vars, runs Ansible.
- `run.yml` – main playbook (runs `pacemaker-ansible-role` on VMs).
- `vsphere-site.yaml` – playbook for vCenter/ESXi (power off, add disk, VMX, power on).
- `pacemaker-ansible-role/` – Pacemaker, fencing, DLM, CLVM, GFS2 configuration (Red Hat 7).
- `add-share-disk/` – roles for adding a shared disk via vCenter/ESXi.
- `example_sc_pv_pvc.yml` – sample StorageClass, PV, and PVC for Kubernetes.
