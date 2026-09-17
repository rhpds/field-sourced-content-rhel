# RHEL Field Sourced Content

Template project repository for the RHDP **RHEL Field Asset** catalog item. Fork or copy this repo as a starting point for your own project.

A project repo is a mono-repo: Ansible automation (structured as a collection) and optionally Showroom lab content, all in one place. Order the catalog item, point it at your repo, and the platform provisions a bastion plus RHEL nodes. The bastion runs your playbook in a Podman execution environment and **does not wait for it to succeed** -- broken automation is yours to debug.

> For OpenShift-based field content, see `field-sourced-content-template`.

## How an order works

1. Choose node count (1--10), node size (small/medium/large), and RHEL version (9 or 10).
2. Leave **Existing content repo?** unchecked if you only need VMs.
3. To run automation, check it and provide:
   - Your project repo URL (must contain `galaxy.yml` at the root)
   - Git revision (branch, tag, or commit)
   - **Playbook entrypoint** -- path to your playbook inside the collection (this template uses `playbooks/deploy.yml`)
4. The platform writes inventory on the bastion, pulls the execution environment, and starts `field-content.service`.
5. The RHDP order succeeds once the container is **running**. Galaxy install and `ansible-playbook` continue in the background.

Watch progress:

```bash
sudo systemctl status field-content
sudo journalctl -u field-content -f
sudo podman logs -f field-content
```

## Repository layout

```
your-project/
├── galaxy.yml                 # required -- makes this an installable collection
├── playbooks/
│   └── deploy.yml             # your playbook (name it whatever you want)
├── roles/
│   ├── example_setup/         # baseline on every node
│   ├── example_httpd/         # per-node web server example
│   └── example_cockpit/       # Cockpit on nodes + bastion proxy config
├── requirements.yml           # optional -- extra collections installed at runtime
├── site.yml                   # optional Antora playbook for Showroom
├── ui-config.yml              # optional Showroom UI config
└── content/                   # optional Showroom (Antora) sources
```

If you add `requirements.yml` at the collection root or under `playbooks/`, the runner installs those collections before running your playbook.

## Inventory

The platform writes `/opt/field-content/inventory` on the bastion:

```ini
[bastions]
bastion ansible_host=127.0.0.1

[nodes]
node1 ansible_host=...
node2 ansible_host=...

[all:vars]
ansible_user=cloud-user
ansible_ssh_private_key_file=/ssh/id_rsa
ansible_become=true
```

Bastions connect via SSH to 127.0.0.1 (the EE container runs with `--network=host`). This lets your playbook manage services on the bastion with `ansible.builtin.service` and other modules that need a real init system.

Target `nodes` for workload. Platform extra-vars are injected automatically (`guid`, `subdomain`, `field_asset.*`).

## Extra-vars

```yaml
guid: "<guid>"
subdomain: "<apps domain>"
field_asset:
  node_count: <int>
  node_size: "<small|medium|large>"
  rhel_version: "<rhel9|rhel10>"
  enable_bastion_proxy: <bool>
  route_app: "app-<guid>.<subdomain>"
  route_app_secure: "app-secure-<guid>.<subdomain>"
```

## Playbook entrypoint

The order form field is a path **inside the installed collection**, not a filesystem path on the bastion. This template uses `playbooks/deploy.yml`. Key patterns in the example:

- `hosts: nodes` -- every workload node, never the bastion
- `groups['nodes'][0]` -- first node only (always exists)
- `groups['nodes'][1] if groups['nodes'] | length > 1 else []` -- second node, or skip the play
- Empty host list `[]` skips a play cleanly. Do not use `when:` on a Play -- it is not a valid play keyword.

## Through the bastion

Workload nodes have no public routes. Two reserved ports on the bastion provide external access:

| Public URL | Bastion port |
|---|---|
| `http://app-<guid>.<subdomain>` | **8080** (HTTP) |
| `https://app-secure-<guid>.<subdomain>` | **8443** (HTTPS edge; listen HTTP on 8443) |

### Bastion proxy

Check **Install proxy on bastion?** on the order form and the platform installs Nginx as a reverse proxy on ports 8080 and 8443. The Nginx server block includes `/etc/nginx/proxy.d/*.conf` — an empty directory where your playbook drops location configs for the services you install on the nodes.

This template demonstrates the pattern with the `example_cockpit` role:

1. Installs Cockpit on all nodes with a per-node `UrlRoot`
2. Delegates to the bastion to write an Nginx location config into `proxy.d/`
3. Reloads Nginx

The result: `https://app-secure-<guid>.<subdomain>/node1/` proxies to Cockpit on node1, `/node2/` to node2, and so on. Use this as a pattern for routing to any service on any port.

If you do not check the proxy box, the ports are still routed — you can listen on them however you like from your own playbook.

### Showroom

Check **Deploy Showroom?** to build a lab guide from the same repo. Showroom serves at `https://bastion-<guid>.<subdomain>` (port 80). Do not bind port 80.

At the repo root (or the directory set as **Showroom path**), provide `site.yml`, `ui-config.yml`, and `content/`. See the [Showroom UI docs](https://github.com/rhpds/showroom_template_nookbag/blob/main/content/modules/ROOT/pages/ui-config.adoc).

Showroom can block the order. The Ansible runner cannot.

## Credentials

The platform creates `lab-user` with a password on the bastion and all workload nodes. These credentials work for SSH, Cockpit, and sudo. The password is shown on the RHDP order page.

SSH between hosts is preconfigured — from the bastion, just `ssh <node-name>`.

## Private repositories

Check **Private git repo?** and paste a token with read access.

## What will not fail the RHDP order

Once the container is running, these are yours to debug:

- Repository does not exist or token is wrong
- Repo is not a valid collection
- Playbook path is wrong
- Playbook syntax or task failures

The order **does** fail if the repo URL is missing, the EE cannot be pulled, or the container never reaches Running.

## Local testing

```bash
ANSIBLE_ROLES_PATH=roles ansible-playbook -i my-inventory.ini playbooks/deploy.yml
```
