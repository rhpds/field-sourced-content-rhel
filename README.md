# RHEL Field Sourced Content

Template for field-developed Ansible that runs on RHDP **RHEL Field Asset** (CNV VMs), not OpenShift.

Order the catalog item, optionally point it at this repository (or a fork), and the platform provisions a bastion plus N RHEL nodes. If you attached a git repo, the bastion starts your collection in a Podman execution environment and **does not wait for the playbook to succeed**. Broken automation is yours to debug.

OpenShift Field Sourced Content is a different catalog item and a different template (`field-sourced-content-template`). This repository is the RHEL equivalent.

## How an order works

1. You choose node count, one node size, and RHEL 9 or 10.
2. You may leave the git repo blank. You get VMs only.
3. If you provide a git repo, it must be an **Ansible collection** (`galaxy.yml` at the root).
4. The order form also asks for a **playbook entrypoint** relative to the collection root. The default is `playbooks/site.yml`.
5. The platform writes inventory on the bastion, pulls an execution environment, and starts `field-content.service`.
6. The RHDP service is successful once that container is **running**. Galaxy install and `ansible-playbook` continue in the background.

Watch it:

```bash
systemctl status field-content
journalctl -u field-content -f
podman logs -f field-content
```

## Repository layout

This repo is a collection. Copy it and keep the same shape:

```
field-sourced-content-rhel/
├── galaxy.yml                 # required — ansible-galaxy installs this repo
├── playbooks/
│   └── site.yml               # default order-form entrypoint
├── roles/
│   └── example_setup/         # your roles
├── site.yml                   # optional Antora playbook for Showroom
└── content/                   # optional Showroom (Antora) sources
```

Do not invent a stub file listing roles. Write a playbook. The playbook is how Ansible composes roles.

## Inventory the platform writes

On the bastion, `/opt/field-content/inventory`:

```ini
[bastions]
bastion ansible_host=...

[nodes]
node0 ansible_host=...
node1 ansible_host=...

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
```

## Playbook entrypoint

The catalog field is a path **inside the installed collection**, not a path you make up on the bastion. After `ansible-galaxy collection install git+<your-repo>`, the runner executes:

```text
<collection_dir>/<entrypoint>
```

For this repository the entrypoint is `playbooks/site.yml`:

```yaml
- name: Example field sourced content
  hosts: nodes
  become: true
  roles:
    - example_setup
```

If you add `requirements.yml` at the collection root or under `playbooks/`, the runner installs it before the playbook.

## Optional Showroom

The same git repo can hold an Antora lab guide (`site.yml` + `content/` at the repo root, as in this template). Check **Deploy Showroom?** on the order form. If your Showroom project is not at the repo root, set **Showroom path** to that directory.

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
ANSIBLE_ROLES_PATH=roles ansible-playbook -i my-inventory.ini playbooks/site.yml
```

## Related

- Catalog item (devs): `agd_v2/rhel-field-asset-cnv` in AgnosticV
- Runner role: `agnosticd.cloud_vm_workloads.vm_workload_field_content`
- OpenShift sibling: https://github.com/rhpds/field-sourced-content-template
- Design notes: https://gist.github.com/stencell/9cb5315693aeda56066af9c72bc7b287
