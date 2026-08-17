# AGENTS.md

Personal home-lab Ansible repo. It provisions a mixed-architecture Kubernetes cluster and supporting infrastructure on a small home network (192.168.0.0/24):

- `ubuntu01`–`ubuntu05`, `ubuntu06` (192.168.0.25–29, .30): x86_64, `ubuntu01` is the primary control plane
- `jetson` (192.168.0.32), `spark` (192.168.0.33): aarch64 k8s followers
- `pi` (192.168.0.89): Raspberry Pi (`inventory/pis.yaml`)
- `pxe01` (192.168.0.50): PXE boot server (`inventory/pxe.yaml`)

Run everything from the repo root; `ansible.cfg` sets `inventory = inventory`, `roles_path = roles`, `forks = 12`, and `any_errors_fatal = true` (one failed host aborts the whole run).

## Commands

```bash
# PI first-boot provisioning (fresh Pi, password auth; needs sshpass on the control node)
ansible-playbook -i inventory/pis.yaml playbooks/setuppi.yaml

# Users/keys on every host in an inventory
ansible-playbook -i inventory/all.yaml playbooks/user_accounts.yaml

# Fresh k8s cluster
ansible-playbook -i inventory/k8s.yaml playbooks/install-k8s.yaml

# k8s upgrade (bump kubernetes_version in inventory/group_vars/all.yaml FIRST, one minor version at a time)
ansible-playbook -i inventory/k8s.yaml playbooks/k8s/upgrade-k8s.yaml

# Full cluster teardown
ansible-playbook -i inventory/k8s.yaml playbooks/reset-k8s.yaml
```

- `venv/` (gitignored) holds the working control-node venv with `ansible` 8.5.0 / core 2.15.9 installed. Root `requirements.txt` is a full `pip freeze` of that Ubuntu environment, not a curated dependency list.
- Lint: `pre-commit run --all-files` (yamllint via `.yamllint.yml`, plus JSON/key checks). `yamllint -c .yamllint.yml <file>` works standalone.
- There is NO CI validation of playbooks (no syntax-check, no ansible-lint). The only workflow (`.github/workflows/deploy.yml`) runs `mkdocs gh-deploy` on push to main to publish `documentation/`. Validate manually: `ansible-playbook --syntax-check <playbook>`.
- The README mentions extracting mitogen into `/opt` for a speedup strategy; the corresponding `ansible.cfg` lines are commented out and nothing else references it.

## Layout

- `inventory/` — four inventory files + `group_vars/all.yaml` (the only group_vars file; holds `users`, `kubernetes_version`, `kubernetes_cluster_label`, and `kubeadm_config_options`).
- `playbooks/` — thin play files, mostly `hosts:` + `roles:`; `playbooks/k8s/` holds the lifecycle chain stages, each of which is itself imported in order by `install-k8s.yaml` / `k8s/upgrade-k8s.yaml` / `reset-k8s.yaml`.
- `roles/` — one directory per role, conventional layout (`tasks/`, `defaults/`, `templates/`, `vars/`). Only 6 roles have `defaults/`; most hardcode values in tasks.
- `documentation/` — mkdocs-material site, auto-deployed to GitHub Pages.

## Inventory groups & targeting

`inventory/k8s.yaml` defines: `k8s` (all 8 cluster nodes), `k8s_leaders` (**only `ubuntu01`**), `k8s_followers` (the other 7), and `k8s_workers` (a `children:` alias of `k8s_followers`, targeted by the worker upgrade step). `inventory/all.yaml` is the same hosts flat under `all`, **minus `ubuntu06`** (only present in `k8s.yaml`), plus a `monitoring` group (commented-out host) for `install-loki`/`install-mimir`.

- Several playbooks use `hosts: all` or `hosts: ubuntu01` — these only make sense with the inventory file you pass via `-i`. `setuppi.yaml` targets `all`, so it must always be run with `-i inventory/pis.yaml` or it will apply Pi-specific setup (disabling SSH password auth, restarting sshd) to the whole cluster.
- `k8s_workers` is a `children:` alias of `k8s_followers` in `inventory/k8s.yaml` (the worker upgrade step targets it); `upgrade-workers.yaml` honors `worker_upgrade_strategy` via a parenthesized ternary, so `serial` upgrades one worker at a time with a `pause` between them.
- `k8s_leaders[1:]` is empty in the current inventory, so `kubeadm-join-controlplane.yaml` and `upgrade-control-plane-additional.yaml` are no-ops until a second leader is added.
- `monitoring` exists in `inventory/all.yaml` but has no uncommented host, so `install-loki.yaml` / `install-mimir.yaml` still match zero hosts until a node is assigned to it.
- Playbooks with hardcoded `ubuntu01` (`kubeadm-init-controlplane`, `install-cilium`, `install-rook`, `install-argo`, `install-metallb`) assume node 1 is the primary.

## K8s lifecycle patterns

- **Install chain** (`install-k8s.yaml`, order is meaningful): k8s-binaries → disable-swap → install-cri → kubeadm-prep → kubeadm-init-controlplane → install-cilium → kubeadm-join-controlplane → kubeadm-join-workers. The metallb/rook/argo imports are commented out; note `install-metallb.yaml` exists but **`roles/install-metallb` does not** — uncommenting it will fail.
- **HA bootstrap**: `kubeadm-init-controlplane` runs `kubeadm init` on ubuntu01, then `fetch`es the PKI (ca, sa key, front-proxy, etcd) + `admin.conf` to `/tmp/kubeadm-ha/` on the *Ansible controller*; `kubeadm-join-controlplane` pushes that dir out to additional leaders. The fetches have no `when:` guard, so they re-run (and fail if init was skipped) on re-runs.
- **Join tokens** are generated at runtime via `kubeadm token create --print-join-command`, delegated to `k8s_leaders[0]` — nothing is stored.
- **Idempotency**: init skips if `/etc/kubernetes/admin.conf` exists; joins skip if `kubelet.conf` exists; `k8s-binaries` uses `get_url` without `force` and a kubelet restart handler that only fires if kubelet is running (the role never starts kubelet — kubeadm does). **Upgrade roles have no skip guards** and use `get_url force: yes`.
- **All `kubectl` drain/uncordon/health tasks are `delegate_to: "{{ groups['k8s_leaders'][0] }}"`** because only the leader has a kubeconfig; keep that pattern when adding upgrade tasks.
- **Preflight** (`upgrade-preflight-checks`) fails on missing cluster, downgrade, or >1 minor version jump (`max_version_jump`); it only *warns* if nodes aren't all Ready, and its `min_ready_nodes_percent` default is unused by tasks.

## Variables to edit before running

- `inventory/group_vars/all.yaml`: `kubernetes_version` / `kubeadm_version` (currently v1.31.3), `kubernetes_cluster_label`, and `kubeadm_config_options` (hardcoded `controlPlaneEndpoint: 192.168.0.25`, `podSubnet: 10.244.0.0/16`; a large audit-log block is commented out — `audit.yaml` is templated to `/opt/kubernetes/audit.yaml` but never wired into the apiserver).
- `roles/install-cilium/defaults/main.yaml`: `cilium_chart_version` + `cilium_cli_version` + `cilium_cli_checksums` — bumping the chart version without bumping/verifying the CLI checksums breaks the run.
- `roles/upgrade-workers/defaults/main.yaml`: `worker_upgrade_strategy`, `worker_upgrade_delay`.

## Gotchas

- **`rook-pre-reqs` is destructive**: `wipefs -a /dev/sda3` + `ceph-volume lvm zap /dev/sda3` on every `k8s_followers` host (including the aarch64 nodes), with a hardcoded device and a Ubuntu-20.04-pinned `ceph-osd` version.
- **Architecture is half-handled**. Mixed-arch is the norm here (aarch64 nodes in the cluster), yet:
  - `install-loki` / `install-mimir` default `*_architecture: arm64` with matching pinned SHAs — the opposite problem; both must be overridden for x86_64.
  - `pxe` is x86-only (`pxe-service=x86PC`, amd64 ISOs) and its vars live in `vars/` (not `defaults/`), so they can only be overridden via extra-vars/host vars.
  - `install-node-exporter` and `k8s-binaries` are arch-aware via `<name>_arch_mapping` + assert (amd64/arm64/armv7/armv6 for node-exporter).
  - Arch mappings are duplicated and inconsistent: `k8s-binaries/defaults` (complete), `install-cilium/defaults` (x86_64/aarch64 only), and inline copies in the three upgrade playbooks.
- **Secrets are ad hoc**: no ansible-vault anywhere (the `.yamllint.yml` ignore for `group_vars/*/vault.yml` is a leftover). `roles/setuppi/vars/main.yaml` commits `ansible_password: raspberry`; `roles/create-secrets` is orphaned (empty `tasks/main.yaml`, no playbook references it) and its gitignored plaintext `image-pull-secret.yaml` is the unsealed twin of the committed sealed secret. Join tokens are the one well-handled case (runtime-generated).
- **`install-cilium` is not idempotent**: `git clone --depth 1` at the chart tag into `/tmp/cilium-git`, checksum-verified CLI download, unconditional `cilium install`; the 600-line `templates/cilium.yaml` is unused (CLI-driven install).
- **`install-cri`** gates docker install on a `docker --version` probe (read-only tasks use `failed_when: false` / `changed_when: false` throughout — follow that convention); the `disable-swap` `swapoff -a` task has no `changed_when`, so it always reports changed.
- **Dotfiles/vim ordering**: `setprompt` (clones `jgavinray/dotfiles` at a pinned commit) must run before `vim.yaml`/`vundle` (which source `~/.vimrc`), and both assume `setuppi` already installed `vim`+`git`.
- **`pxe` role is only half-wired**: the ISO download task is commented out, `files/dnsmasq.conf` is never copied, and its `/tftpboot` path disagrees with the (uncopied) dnsmasq `tftp-root`.
- **Hardcoded identities**: `users: [jgavinray]` in group_vars and PXE boot URL `http://192.168.0.50/...`. (`kubeadm-init-controlplane` parameterized its kubeconfig owner as `kubeconfig_user`, defaulting to `ansible_user_id` — see `roles/kubeadm-init-controlplane/README.md`.)
- `kubeadm-reset` uses `ignore_errors` on `kubeadm reset -f` and deletes `/var/lib/etcd`, `/etc/kubernetes`, etc. on all of `k8s` — it is the only role relying on `ignore_errors` for the core operation.

## Conventions

- `become` is set at play level for k8s playbooks, per-task in user/PI roles.
- Role var names are domain-prefixed (`k8s_*`, `cilium_*`, `loki_*`, `mimir_*`); shared cluster vars are unprefixed in `group_vars/all.yaml`.
- Explicit `changed_when:`/`failed_when:` on every `shell`/read-only task is the norm in the k8s roles; task comments in some roles have sloppy numbering (e.g. "8/7") — cosmetic, ignore.
- yamllint enforces indentation/spacing/trailing whitespace strictly; `document-start`, `truthy`, and comments are warnings only (the repo mixes `---` start markers and `yes`/`true`).
