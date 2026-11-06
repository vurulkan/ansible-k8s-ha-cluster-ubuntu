# Kubernetes HA Cluster Deployment on Ubuntu Using Ansible

This document describes how to deploy a highly available (HA) Kubernetes cluster on Ubuntu using Ansible and this repository.

The goal is to provide a repeatable, production‑oriented procedure that can be used by operations and platform teams.

---
## 1. Overview

The Ansible playbooks in this repository perform the following high‑level steps:

- Prepare all Kubernetes nodes (masters and workers) with required OS, kernel and container runtime settings.
- Deploy a highly available Kubernetes API endpoint using HAProxy and Keepalived on dedicated load balancer nodes.
- Initialize the Kubernetes control plane on the first master node.
- Install the Calico CNI plugin for pod networking.
- Join additional master nodes to the cluster (control‑plane HA).
- Join worker nodes to the cluster.
- Optionally install Longhorn distributed block storage with Helm.

All automation is orchestrated from an Ansible control host.

---
## 2. Prerequisites

Before running the playbooks, ensure the following:

- **Operating System**
  - Ubuntu 24.04 or 26.04 on all master, worker and load balancer nodes.
  - Ubuntu 22.04 also works if `haproxy_branch` is set to `3.0` (see Section 2.1).
  - All OS updates applied on every node.

- **User and Privileges**
  - A non‑root user with `sudo` privileges on all nodes.
  - Passwordless SSH access from the Ansible control host to all target nodes (via SSH key).

- **Hostnames**
  - Each server has a meaningful and unique hostname configured (for example: `mk8scp1`, `mk8scp2`, `mk8scp3`, `mk8swk1`, etc.).
  - Hostnames must be set **before** running any playbooks, because hostname‑based references are used by the scripts and configurations.

- **Ansible Control Host**
  - Ansible installed.
  - All target servers defined in the Ansible `inventory` file in this repository.

### 2.1 Configuration Checklist Before You Begin

Before executing any playbooks, review and adjust the following configuration items:

- `roles/kubernetes_cluster/defaults/main.yml` – component versions
  - `k8s_version` – Kubernetes minor version (e.g. `v1.37`). It is used in the Kubernetes APT repository URL; the latest patch release of that minor version is installed.
  - `calico_version` – Calico release (e.g. `v3.33.0`).
  - `haproxy_branch` – HAProxy branch installed from the `ppa:vbernat/haproxy-<branch>` PPA (default `3.2`, LTS). Make sure the PPA publishes packages for your Ubuntu release (`3.2`: 24.04 and 26.04; use `3.0` for 22.04).
  - `longhorn_version` – Longhorn Helm chart version (e.g. `1.13.0`). Longhorn 1.13 requires Kubernetes v1.34 or later.

- `roles/kubernetes_cluster/templates/keepalived.conf.j2`
  - Set `virtual_router_id` to a value that is **unique within the same subnet/L2 domain**. Having another VRRP instance with the same ID on the same subnet can lead to unstable or unexpected behavior.
  - After selecting the VRRP ID for this environment, document it and share it with the customer so that any future deployments can avoid ID conflicts on the same network.

- `roles/kubernetes_cluster/defaults/main.yml`
  - Update `keepalive_ip`, `interface_name`, `pod_network_cidr` and master node addresses (`master1`, `master2`, `master3`) to reflect your environment.

- `inventory`
  - Ensure all nodes are assigned to the correct groups:
    - `[preinstall]` – all master and worker nodes.
    - `[master]` – master nodes.
    - `[loadbalancer]` – load balancer nodes (HAProxy + Keepalived).
    - `[worker]` and `[worker_init]` – worker nodes.
    - `[cluster_init]` – the initial master node (`master1`).
    - `[master_init]` – additional master nodes except `master1`. These nodes are joined one by one; keeping all of them active at the same time in this group can cause token‑related errors during the join‑master‑to‑master phase.
    - `[longhorn]` – nodes that will store Longhorn volumes (only needed if you install Longhorn).

---
## 3. Ansible Control Host Setup

Perform the following steps on the Ansible control host (for example, on `mk8scp1` or a dedicated management server).

### 3.1 Update `/etc/hosts`

On the Ansible control host, edit `/etc/hosts` and add all master and worker nodes, including logical aliases for masters:

```text
10.42.0.11 mk8scp1 master1
10.42.0.12 mk8scp2 master2
10.42.0.13 mk8scp3 master3
10.42.0.14 mk8swk1
...
```

- Include all master and worker nodes.
- For master nodes, add aliases such as `master1`, `master2`, `master3` to be used by Ansible tasks (for example, `delegate_to: master1`).

### 3.2 Install Ansible

On the Ansible control host, install Ansible:

```bash
apt-add-repository ppa:ansible/ansible
apt update
apt install ansible -y
```

> Note: Ansible is a command‑line tool and does **not** run as a systemd service; there is no need to start or enable any `ansible` service.

### 3.3 Configure SSH Key‑Based Authentication

On the Ansible control host:

1. Generate an SSH key (if not already present):
   ```bash
   ssh-keygen
   ```
2. Copy the public key to each target server:
   ```bash
   ssh-copy-id example_user@server_ip
   ssh-copy-id example_user@server_ip
   ssh-copy-id example_user@server_ip
   ```

Use the same user that will be used by Ansible to connect to the servers.

### 3.4 Test Ansible Connectivity

From the repository directory on the Ansible control host, run:

```bash
ansible -i inventory all -m ping --become
```

You should receive `SUCCESS` / `pong` responses from all hosts. For example:

```text
mk8scp1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
...
```

If all nodes respond successfully, Ansible installation and basic connectivity are correctly configured.

### 3.5 Notes on Users and `become`

When running Ansible commands, the effective user is important:

- Use the same user for Ansible that was used when copying the SSH key to the servers.
- To execute commands with elevated privileges, use the `--become` flag.

Examples:

```bash
# Run a command as the current user with privilege escalation
ansible all -i inventory -a "cat /etc/hosts" --become

# Run as a specific become user
ansible preinstall -i inventory -a "cat /etc/hosts" --become --become-user=example_user

# If sudo prompts for a password
ansible preinstall -i inventory -a "cat /etc/hosts" --become --ask-become-pass
```

---
## 4. Repository Structure and Key Files

Important files and their purpose:

- `inventory`  
  Ansible inventory file defining all nodes and host groups (masters, workers, load balancers, etc.).

- `run.yaml`  
  Main playbook that imports the `kubernetes_cluster` role for a given host group:
  ```yaml
  - hosts: "{{ machine }}"
    tasks:
      - name: Deploy Kubernetes Cluster
        import_role:
          name: kubernetes_cluster
  ```
  The `machine` variable is passed on the command line to target a specific group.

- `roles/kubernetes_cluster/defaults/main.yml`  
  Default variables used by the role, for example:
  - `keepalive_ip` – virtual IP for the Kubernetes API load balancer
  - `interface_name` – network interface used by Keepalived/HAProxy
  - `pod_network_cidr` – pod network CIDR (e.g. `192.168.0.0/16`)
  - `master1`, `master2`, `master3` – IP addresses or host aliases for master nodes
  - `k8s_version`, `calico_version`, `haproxy_branch`, `longhorn_version` – component versions (see Section 2.1)

- `roles/kubernetes_cluster/tasks/*.yaml`  
  Task files that implement each phase of the deployment:
  - `preinstall.yaml` – OS preparation, container runtime, Kubernetes packages, sysctl, modules, etc.
  - `loadbalancer.yaml` – HAProxy and Keepalived installation and configuration.
  - `init_kubernetes.yaml` – Cluster initialization on the first master with `kubeadm init`.
  - `network.yaml` – Calico CNI deployment.
  - `join_master_to_master.yaml` – Joining additional master nodes to the control plane.
  - `join_worker_to_master.yaml` – Joining worker nodes to the cluster.
  - `longhorn.yaml` – Longhorn node prerequisites, Helm installation and the Longhorn Helm chart.

---
## 5. Inventory Groups

The `inventory` file uses the following host groups:

- `[preinstall]` – All master and worker nodes. Used for common OS and Kubernetes prerequisites.
- `[master]` – All master nodes.
- `[loadbalancer]` – Load balancer nodes running HAProxy and Keepalived.
- `[worker]` – Worker nodes.
- `[cluster_init]` – The primary master node (e.g. `master1`) used to bootstrap the cluster.
- `[worker_init]` – Worker nodes to be joined to the cluster.
- `[master_init]` – Additional master nodes (excluding `master1`).  
  These nodes are joined sequentially to the cluster.
- `[longhorn]` – Nodes that will store Longhorn volumes. Leave it empty if you do not install Longhorn.

Ensure the inventory groups accurately reflect your environment before running any playbooks.

---
## 6. Pre-Installation Configuration

This section summarizes configuration items that are already described in detail in **Section 2.1 – Configuration Checklist Before You Begin**.

Before executing any playbooks, ensure you have reviewed and updated:

- Core role defaults (`roles/kubernetes_cluster/defaults/main.yml`)
- Inventory groups and host assignments (`inventory`)
- Component versions (`k8s_version`, `calico_version`, `haproxy_branch`, `longhorn_version` in `roles/kubernetes_cluster/defaults/main.yml`)
- Keepalived VRRP configuration (`roles/kubernetes_cluster/templates/keepalived.conf.j2`)

No additional settings are required here beyond what is covered in Section 2.1.

---
## 7. Deployment Steps

All commands in this section are executed from the repository directory on the Ansible control host.

Assumptions:

- Ansible connects using a user with `sudo` privileges.
- `sudo` prompts for a password; therefore `--ask-become-pass` is used.  
  If your environment uses passwordless sudo, you can omit this flag.

### 7.1 Pre‑Installation on All Nodes

Run:

```bash
ansible-playbook -i inventory -e 'machine=preinstall' --tag preinstall --become --ask-become-pass run.yaml
```

This playbook:

- Updates and upgrades APT packages.
- Disables UFW and AppArmor (where applicable).
- Installs prerequisite utilities.
- Configures kernel modules and sysctl parameters required by Kubernetes.
- Installs `containerd.io` (containerd 2.x) from the Docker repository and configures it with the systemd cgroup driver.
- Adds Docker and Kubernetes APT repositories and keys.
- Installs `kubelet`, `kubeadm`, and `kubectl`.
- Enables and configures `kubelet`.

### 7.2 Load Balancer Deployment

Run:

```bash
ansible-playbook -i inventory --tag loadbalancer -e 'machine=loadbalancer' --become --ask-become-pass run.yaml
```

This playbook:

- Adds the `ppa:vbernat/haproxy-<haproxy_branch>` PPA and installs HAProxy from it, together with Keepalived.
- Applies required `sysctl` settings.
- Deploys the following configuration files:
  - `/etc/keepalived/check_apiserver.sh`
  - `/etc/keepalived/keepalived.conf`
  - `/etc/haproxy/haproxy.cfg`
- Enables and starts the `keepalived` and `haproxy` services.
- Disables swap on the load balancer nodes.

> **Operational Note:**  
> After the initial deployment, update the Keepalived configuration on the secondary load balancer (LB2) so that:
> - `state` is set to `BACKUP`.
> - `priority` is set to `99` (or lower than the primary).  
> The primary load balancer (LB1) should remain `MASTER` with a higher priority (for example `100`).

### 7.3 Initialize the First Master (Cluster Bootstrap)

Run:

```bash
ansible-playbook -i inventory --tag master1 -e 'machine=cluster_init' --become --ask-become-pass run.yaml
```

This playbook:

- Creates and populates `/etc/kubernetes/kubeadm-config.yaml` with:
  - `controlPlaneEndpoint` pointing to the load balancer virtual IP (`keepalive_ip:6443`).
  - Pod network configuration (`pod_network_cidr`).
  - Kubelet configuration including cgroup driver and resource reservations.
- Executes `kubeadm init` using the generated configuration.
- Extracts and stores the join command and token.
- Configures the local kubeconfig for the Ansible user (copying `/etc/kubernetes/admin.conf` to `$HOME/.kube/config`).

### 7.4 Deploy the Pod Network (Calico)

Run:

```bash
ansible-playbook -i inventory --tag network -e 'machine=cluster_init' --become --ask-become-pass run.yaml
```

This playbook:

- Installs the Calico operator and custom resources using `kubectl` and the official manifests.

After the network installation:

- Verify that all system pods are running:
  ```bash
  kubectl get pod -A -o wide
  ```
- Verify that the first master node is in `Ready` state:
  ```bash
  kubectl get nodes -o wide
  ```

Wait until pods reach `Running` state (typically 3–4 minutes) before proceeding.

### 7.5 Join Additional Masters

Run:

```bash
ansible-playbook -i inventory --tag master_init -e 'machine=master_init' --become --ask-become-pass run.yaml
```

This playbook:

- Uploads control plane certificates from `master1`.
- Generates a join command and certificate key.
- Joins additional master nodes to the existing control plane.

Operational considerations:

- In the `inventory` file, `[master_init]` should contain **one** master at a time, with others commented out.
- Run the playbook once per additional master node, updating `[master_init]` each time.

After each run, confirm the new master is `Ready`:

```bash
kubectl get nodes -o wide
```

### 7.6 Join Worker Nodes

Run:

```bash
ansible-playbook -i inventory --tag worker_init -e 'machine=worker_init' --become --ask-become-pass run.yaml
```

This playbook:

- Generates a join command on `master1` using `kubeadm token create --print-join-command`.
- Executes the join command on all nodes in the `[worker_init]` group.
- Optionally retrieves and displays the node status via `kubectl get nodes`.

After completion, verify that all worker nodes appear as `Ready`:

```bash
kubectl get nodes -o wide
```

### 7.7 Install Longhorn (Optional)

[Longhorn](https://longhorn.io) provides distributed block storage for the cluster. Install it after all nodes have joined and Calico is running.

1. Add the nodes that will store Longhorn volumes to the `[longhorn]` group in `inventory`.

2. Install the node prerequisites:

   ```bash
   ansible-playbook -i inventory --tag longhorn -e 'machine=longhorn' --become --ask-become-pass run.yaml
   ```

   This playbook, on every node in `[longhorn]`:

   - Installs `open-iscsi`, `nfs-common`, `cryptsetup` and `dmsetup`.
   - Loads the `iscsi_tcp` kernel module and adds it to `/etc/modules-load.d/` so it is loaded at boot.
   - Enables and starts `iscsid`.

3. Install Helm and Longhorn:

   ```bash
   ansible-playbook -i inventory --tag helm -e 'machine=master' --become --ask-become-pass run.yaml
   ```

   This playbook:

   - Installs Helm on **all master nodes** from the official Helm APT repository.
   - Installs the Longhorn chart (`longhorn_version`) into the `longhorn-system` namespace **from `master1` only**. It uses `helm upgrade --install`, so it can be re-run safely.

4. Verify on `master1`:

   ```bash
   kubectl -n longhorn-system get pod
   ```

   All pods should reach `Running` state.

Notes:

- Volume data is stored under `/var/lib/longhorn` on each node by default.
- The Longhorn UI needs a separate Ingress and has no authentication by default.
- Longhorn's `longhornctl check preflight` tool can be used to validate the nodes before installation.

---
## 8. Notes and Recommendations

- **Versioning**
  - All component versions are set in `roles/kubernetes_cluster/defaults/main.yml`; the task files contain no hard‑coded versions.
  - A value can be overridden for a single run with `-e`, for example `-e k8s_version=v1.36`.
  - Changing `k8s_version` does **not** upgrade an existing cluster. Kubernetes packages are locked with `apt-mark hold`, so the change only affects newly installed nodes. Use `kubeadm upgrade` to upgrade a cluster.
  - Changing `longhorn_version` and re‑running the `helm` tag **upgrades** Longhorn (`helm upgrade --install`). Longhorn only supports upgrading from the previous minor version; read the release notes first.

- **Idempotency**
  - The playbooks are designed to be idempotent where practical, but some steps (e.g. `kubeadm init`) are inherently single‑run operations. Re‑running those tasks on an already initialized control plane may fail and should be avoided unless performing a clean rebuild.

- **Host Naming and Delegation**
  - Ensure that aliases such as `master1`, `master2`, `master3` defined in `/etc/hosts` match the usage in Ansible tasks (e.g., `delegate_to: master1`) to avoid connectivity issues.

- **Swap and Kernel Settings**
  - Kubernetes requires swap to be disabled and specific kernel parameters to be set. These are handled by `preinstall.yaml`, but any manual changes should respect these constraints.

---
## 9. Support and Maintenance

For ongoing operations:

- Regularly monitor the health of:
  - Kubernetes control plane components.
  - HAProxy and Keepalived services on load balancers.
  - Cluster nodes (`kubectl get nodes`) and system pods (`kubectl get pod -A`).
  - Longhorn components (`kubectl -n longhorn-system get pod`), if Longhorn is installed.
- Apply security updates to the underlying OS and review Kubernetes, Calico and Longhorn release notes before upgrading repositories or manifests.

This document should serve as a baseline guide for deploying and operating a highly available Kubernetes cluster using Ansible on Ubuntu. Adjust the configuration and procedures as needed to align with your organization’s standards and policies.
