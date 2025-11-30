# kubeadm-init-controlplane role

This role initializes a Kubernetes control plane node and prepares kubeconfig files for interactive use.

## Configurable variables

- `kubeconfig_user` (default: `ansible_user_id`): User account that owns the downloaded `admin.conf` kubeconfig for kubectl usage.
- `kubeconfig_user_home` (default: `/root` for `root`, otherwise `/home/<kubeconfig_user>`): Home directory for `kubeconfig_user`.

### Override example

To place the kubeconfig under a different account, set `kubeconfig_user` when running the playbook:

```yaml
- hosts: masters
  roles:
    - role: kubeadm-init-controlplane
      vars:
        kubeconfig_user: devops
```
