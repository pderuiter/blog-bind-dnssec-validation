# blog-bind-dnssec-validation

Companion code for the blog post
[Enabling DNSSEC Validation on BIND Resolvers](https://blog.bsdserver.nl/enabling-dnssec-validation-on-bind-resolvers/).

It builds a BIND recursive resolver that validates DNSSEC, exactly as the post's
"Complete Resolver Configuration" describes:

- `dnssec-validation auto` (built-in root trust anchor with RFC 5011 updates), or the explicit
  `trust-anchors` block with both root KSKs (20326 and 38696)
- optional permanent negative trust anchors (`validate-except`)
- EDNS buffer size 1232, `minimal-responses`, `prefetch`
- separate `named.log` and `dnssec.log` channels in `/var/log/named`
- the `/usr/local/bin/check-dnssec.sh` health check script

## Requirements

- Terraform 1.6+ and a Proxmox VE cluster with an API token (only if you want Terraform to create the VM)
- Ansible core 2.16+ on your workstation
- A target running Ubuntu 24.04 (tested) or RHEL/Rocky 9 (paths follow the post's RHEL layout:
  `/etc/named.conf`, `/var/named`)
- Outbound DNS (UDP/TCP 53) from the resolver to the internet

## Quick start

```bash
# 1. Create the VM (optional, skip if you already have a server)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Configure BIND
cd ../ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml

# 3. Run the post's checks (AD flag, SERVFAIL for dnssec-failed.org, check script, trust anchors)
ansible-playbook ../tests/verify.yml
```

Without Terraform, copy `ansible/inventory.example.yml` to `ansible/inventory.yml` and edit it.

## Variables

| Variable | Default | Description |
| --- | --- | --- |
| `bind_dnssec_validation_trusted_networks` | localhost, localnets, 192.168.0.0/16, 10.0.0.0/8 | Members of `acl "trusted"` (recursion and cache access) |
| `bind_dnssec_validation_mode` | `auto` | `auto` uses the built-in trust anchor, `yes` writes the explicit `trust-anchors` block |
| `bind_dnssec_validation_except` | `[]` | Domains for `validate-except` (permanent negative trust anchors) |
| `bind_dnssec_validation_edns_udp_size` | `1232` | `edns-udp-size` and `max-udp-size` |
| `bind_dnssec_validation_log_dir` | `/var/log/named` | Directory for `named.log` and `dnssec.log` |
| `bind_dnssec_validation_install_check_script` | `true` | Install `/usr/local/bin/check-dnssec.sh` |
| `bind_install_source` | `distro` | `distro` (BIND 9.18 on Ubuntu 24.04) or `isc` (ppa:isc/bind, BIND 9.20) |

## Layout

- `terraform/` creates Ubuntu 24.04 VMs on Proxmox VE (bpg/proxmox provider) and outputs an Ansible inventory
- `ansible/roles/bind_install` installs BIND
- `ansible/roles/bind_dnssec_validation` writes the resolver configuration
- `tests/verify.yml` runs the functional checks from the post

## License

MIT
