# Ansible Role: install_kubernetes

This role installs a Kubernetes cluster using `kubeadm`.

## Requirements

- Ubuntu-based systems.
- An inventory file with `master_nodes` and `worker_nodes` groups.

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

- `kube_version`: The version of Kubernetes to install.
  Default: `1.30.0`

- `pod_network_cidr`: The CIDR range for the pod network.
  Default: `"192.168.0.0/16"` (compatible with Calico)

- `cni_manifest_url`: The URL for the CNI plugin manifest.
  Default: `"https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml"`

- `remove_control_plane_taint`: Whether to remove the `NoSchedule` taint from the control-plane node, allowing pods to be scheduled on it.
  Default: `true`

## Dependencies

None.

## Example Playbook

```yaml
- hosts: all
  become: true
  roles:
     - role: install_kubernetes
```
