# RHEL Field Sourced Content

Template for field-developed Ansible that runs on RHDP **RHEL Field Asset** (CNV VMs), not OpenShift.

Order the catalog item, optionally point it at this repository (or a fork), and the platform provisions a bastion plus N RHEL nodes. If you attached a git repo, the bastion starts your collection in a Podman execution environment and **does not wait for the playbook to succeed**. Broken automation is yours to debug.

OpenShift Field Sourced Content is a different catalog item and a different template (`field-sourced-content-template`). This repository is the RHEL equivalent.

## How an order works

1. You choose node count, one node size, and RHEL 9 or 10.
2. You may leave the git repo blank. You get VMs only.
3. If you provide a git repo, it must be an **Ansible collection** (`galaxy.yml` at the root).
4. The order form asks for a **playbook entrypoint** relative to the collection root. That path can be any playbook filename you choose. This template uses `playbooks/deploy.yml` so it is not confused with Antora's `site.yml`.
5. The platform writes inventory on the bastion, pulls an execution environment, and starts `field-content.service`.
6. The RHDP service is successful once that container is **running**. Galaxy install and `ansible-playbook` continue in the background.

Watch it:

```bash
sudo systemctl status field-content
sudo journalctl -u field-content -f
sudo podman logs -f field-content
```

## Repository layout

This repo is a collection. Copy it and keep the same shape:

```
field-sourced-content-rhel/
├── galaxy.yml                 # required — ansible-galaxy installs this repo
├── playbooks/
│   └── deploy.yml             # example playbook (name yours whatever you want)
├── roles/
│   ├── example_setup/         # baseline on every node
│   ├── example_httpd/         # per-node example, driven by play vars
│   └── example_squid/         # optional bastion listeners on 8080/8443
├── site.yml                   # optional Antora playbook for Showroom
├── ui-config.yml              # optional Showroom tabs and layout
└── content/                   # optional Showroom (Antora) sources
```

Do not invent a stub file listing roles. Write a playbook. The playbook is how Ansible composes roles.

## Inventory the platform writes

On the bastion, `/opt/field-content/inventory`:

```ini
[bastions]
bastion ansible_host=...

[nodes]
node1 ansible_host=...
node2 ansible_host=...

[all:vars]
ansible_user=cloud-user
ansible_ssh_private_key_file=/ssh/id_rsa
ansible_become=true
```

Target `nodes` for workload, `bastions` only if you mean to change the control node.

Platform extra-vars (always injected; you can ignore them):

```yaml
guid: <guid>
field_asset:
  node_count: 3
  node_size: medium
  rhel_version: rhel9
  install_squid: false
```

## HTTPS through the bastion

Workload nodes are not given public Routes. SSH to a node is hop-through from the bastion (`ssh <name>`). HTTPS is the same idea: two reserved ports on the **bastion**, each with a public **HTTPS** URL and the cluster certificate.

| Public URL (HTTPS) | Listen on the bastion |
|---|---|
| `https://app-<guid>.<subdomain>` | **8080** |
| `https://app2-<guid>.<subdomain>` | **8443** |

TLS stops at the Route. Listen HTTP on those ports.

When **Deploy Showroom?** is checked, Showroom occupies bastion **443** (`https://bastion-<guid>.<subdomain>`). Leave 443 alone. Port 80 is unused by this catalog.

**Install proxy on bastion?** runs `example_squid` on `bastions`. It installs Squid, writes `/etc/squid/squid.conf` (listens on 8080 and 8443, denies everything else), and starts the service. Edit that file to add your `cache_peer` lines. The checkbox only does something if your playbook includes the role; this template's `playbooks/deploy.yml` does.

## Playbook entrypoint

The catalog field is a path **inside the installed collection**, not a path you make up on the bastion. After `ansible-galaxy collection install git+<your-repo>`, the runner executes:

```text
<collection_dir>/<entrypoint>
```

This template's example playbook is `playbooks/deploy.yml`. On the order form, set **Playbook entrypoint** to that path — or to whatever you renamed it.

The example is four plays so you can see how targeting works:

```yaml
- name: Baseline setup on every workload node
  hosts: nodes
  become: true
  roles:
    - example_setup

- name: HTTP site on the first workload node
  hosts: "{{ groups['nodes'][0] }}"
  become: true
  vars:
    example_httpd_site_role: primary
  roles:
    - example_httpd

- name: HTTP site on the second workload node, when the order has one
  hosts: "{{ groups['nodes'][1] if groups['nodes'] | length > 1 else [] }}"
  become: true
  vars:
    example_httpd_site_role: replica
  roles:
    - example_httpd

- name: Sample proxy on the bastion
  hosts: bastions
  become: true
  roles:
    - example_squid
  when: field_asset.install_squid | default(false) | bool
```

`hosts: nodes` is every workload VM and never the bastion. `groups['nodes'][0]` is the first node (the catalog always creates at least one). The third play's `hosts:` is the second node, or `[]` when the order has only one — Ansible then skips that play. Same role, different `example_httpd_site_role`. Do not use `hosts: nodes[1]`; a missing subscript is an error, not a skip. The proxy play runs on `bastions` when the order-form checkbox is on.

If you add `requirements.yml` at the collection root or under `playbooks/`, the runner installs it before the playbook.

## Optional Showroom

The same git repo can hold a Showroom lab guide. At the repository root (or the directory you set as **Showroom path**) you need:

- `site.yml` — Antora playbook
- `ui-config.yml` — tabs and layout for the right-hand pane (Wetty terminal in this template)
- `content/` — Antora sources (`antora.yml`, modules, pages)

`ui-config.yml` is required by Showroom. It lives next to `site.yml`, not under `content/`. See the [Showroom UI configuration docs](https://github.com/rhpds/showroom_template_nookbag/blob/main/content/modules/ROOT/pages/ui-config.adoc).

Check **Deploy Showroom?** on the order form. If that project is not at the repo root, set **Showroom path** to the directory that contains those three.

Showroom is allowed to block the order (existing Showroom role behavior). The Ansible runner is not.

## Private repositories

Check **Private git repo?** and paste a token with read access. The runner rewrites the URL as `https://x-access-token:<token>@...` inside the already-started job.

## What will not fail the RHDP order

Once the container is running, these are your problem, not the platform's:

- Repository does not exist or token is wrong
- Repo is not a valid collection
- Playbook path is wrong
- Playbook syntax or task failures
- Hosts unreachable from a bad `hosts:` line

The order **does** fail if the URL is missing/malformed when you checked the repo box, the EE image cannot be pulled, inventory cannot be written, or the container never reaches Running.

## Local check

Use a throwaway inventory with groups `bastions` and `nodes`. Do not point this at production hosts.

```bash
ANSIBLE_ROLES_PATH=roles ansible-playbook -i my-inventory.ini playbooks/deploy.yml
```

## Related

- Catalog item (devs): `agd_v2/rhel-field-asset-cnv` in AgnosticV
- Runner role: `agnosticd.cloud_vm_workloads.vm_workload_field_content`
- OpenShift sibling: https://github.com/rhpds/field-sourced-content-template
- Design notes: https://gist.github.com/stencell/9cb5315693aeda56066af9c72bc7b287
