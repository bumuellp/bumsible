# bumuellp.bumsible

Reusable Ansible collection providing atomic, environment-agnostic roles and composable playbooks for Linux host hardening, WireGuard full-mesh networking, LXC nested container prerequisites, and K3s clusters.

## Installation

Add `bumuellp.bumsible` to your `requirements.yml`:

```yaml
collections:
  - name: https://github.com/bumuellp/bumsible.git
    type: git
    version: main
```

Install with:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Roles

| Role | Purpose |
| --- | --- |
| `bumuellp.bumsible.host_security` | SSH hardening, admin/login user provisioning, and opt-in UFW firewall rules (auto-skipped inside containers). |
| `bumuellp.bumsible.linux_baseline` | Baseline OS package installation and configurable kernel `sysctl` drop-ins. |
| `bumuellp.bumsible.wireguard_mesh` | Full-mesh WireGuard (`wg0`) key generation, peer discovery across an inventory group, and conditional kernel module loading. |
| `bumuellp.bumsible.lxc_nested_containers` | Early-boot systemd unit inside LXC containers for `/dev/kmsg` -> `/dev/console` symlink, `mount --make-rshared /`, and optional device node inspection. |
| `bumuellp.bumsible.k3s` | Unified K3s installer supporting `k3s_mode: server` and `k3s_mode: agent`, token discovery, optional `LimitMEMLOCK=infinity`, and kubeconfig fetch. |

Each role defines its full variable contract in `meta/argument_specs.yml`.

## Reusable Playbooks

You can import the collection's built-in playbooks directly from your infrastructure repository or compose the roles in your own playbooks:

- `bumuellp.bumsible.harden_host` — applies `host_security` and `linux_baseline`.
- `bumuellp.bumsible.wireguard_mesh` — configures `wireguard_mesh` across the `wireguard_mesh` group (or `-e target_hosts=...`).
- `bumuellp.bumsible.k3s_cluster` — prepares `k3s_cluster` nodes (`linux_baseline` + `lxc_nested_containers`), bootstraps `k3s_servers`, and joins `k3s_agents`.

Example composition in an infrastructure repo:

```yaml
- name: Harden hosts
  ansible.builtin.import_playbook: bumuellp.bumsible.harden_host
```

### Optional pre/post hooks

Each playbook accepts optional task files via extra vars, included in `pre_tasks` (after fact gathering, before roles) and `post_tasks`:

```bash
ansible-playbook bumuellp.bumsible.harden_host -e bumsible_pre_tasks_file=my_pre.yml -e bumsible_post_tasks_file=my_post.yml
```

In `k3s_cluster`, the pre-hook runs on the first play (`k3s_cluster` node preparation) and the post-hook runs on the final `k3s_cluster` play (covering both servers and agents).

### Access patterns for restricted login users

- **Jump hosts**: set `ansible_ssh_common_args='-o ProxyJump=user@bastion:port'` in inventory; no playbook changes needed.
- **Login user without sudo**: set `ansible_become_method=su` (and `ansible_become_user=root`) for those hosts and supply the root password via `-K` or Ansible Vault. Alternatively, set `ssh_allow_admin_login=true` and connect as `admin_user` with `sudo`.
- **Boolean variables in INI inventories**: use `True`/`False`, because lowercase `true`/`false` are parsed as strings.
