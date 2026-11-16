# DS - AD Proxy

> **Disclaimer:** This project is provided as-is for testing and demonstration purposes only. It is not officially supported in any way. No guarantees are made regarding its functionality, reliability, or suitability for any environment. The author assumes no responsibility or liability for any issues, damages, or consequences arising from its use. Use at your own risk.

Three AD domains (`external.example.com`, `internal.example.com`, `company.example.com`) fronted by a 389 Directory Server instance that chains LDAP queries to each domain over LDAPS. No IdM required.

The proxy presents a single `dc=example,dc=com` tree with one sub-suffix per AD domain:

```
dc=example,dc=com               (proxy local suffix)
├── dc=external,dc=example,dc=com   ──► chains to external.example.com:636
├── dc=internal,dc=example,dc=com   ──► chains to internal.example.com:636
└── dc=company,dc=example,dc=com    ──► chains to company.example.com:636
```

## Configuration

Edit [`vars.yml`](vars.yml) and [`inventory`](inventory) before running.

**Required before every deployment:** set `ad_domain_controller_ip` for each AD server entry in `vars.yml` — these IPs change with every lab build and are intentionally left as `CHANGE_ME` placeholders.

Key settings in `vars.yml`:

| Variable | Description |
|---|---|
| `ad_users` / `ad_groups` | Users and groups to create on each AD domain |
| `ad_servers[]` | List of AD domains with FQDN, NetBIOS name, admin credentials, and DC IP |
| `ad_servers[].ad_domain_controller_ip` | **Must be set** to the current IP of each domain controller |
| `ds_instance` / `ds_port` | 389 DS instance name and LDAP port |
| `ds_suffix` | Base suffix for the proxy DIT (`dc=example,dc=com`) |
| `ds_search_account` | Service account used by LDAP applications to search the proxy |
| `ds_proxy_bind_password` | Password for the `svc-ldapproxy` account created on each AD domain |

## What gets deployed

| Component | Details |
|---|---|
| AD domain controllers | Three independent forests, each with users (`jdoe`, `sysadmin`, `dba`) and group `linux-admins` |
| LDAPS | Self-signed certificates on each DC, trusted by the proxy |
| Proxy service account | `svc-ldapproxy` created on each AD domain for chaining bind |
| 389 DS instance | `slapd-adproxy` on port `3911`, with search account `svc-cm` |

## Run order

> **Prerequisite:** `configure-hosts.yml` must be run first. It populates `/etc/hosts` on both the Ansible controller and the DS proxy so that all AD domain FQDNs resolve. Every other playbook checks for this and will abort with instructions if it has not been done.

```bash
# 1. Set ad_domain_controller_ip in vars.yml, then populate /etc/hosts
ansible-playbook -i inventory configure-hosts.yml

# 2. Deploy the three AD domain controllers
ansible-playbook -i inventory ad/deploy-domain-controller.yml

# 3. Create AD users and groups on each domain
ansible-playbook -i inventory ad/create-users-groups.yml

# 4. Enable LDAPS on each domain controller (self-signed certs)
ansible-playbook -i inventory ad/enable-ldaps.yml

# 5. Create the svc-ldapproxy bind account on each AD domain
ansible-playbook -i inventory ad/create-proxy-account.yml

# 6. Deploy the 389 DS proxy instance with chaining
ansible-playbook -i inventory ds/deploy-proxy-instance.yml

# 7. Test connectivity through the proxy
ansible-playbook -i inventory ds/test-proxy.yml
```

## Troubleshooting

If LDAPS is not working on a domain controller:

```bash
ansible-playbook -i inventory ad/diagnose-ldaps.yml
```

This checks boot time, AD DS certificate events (ID 1220), and certificate details.

## Cleanup

Removes the 389 DS instance and the `svc-ldapproxy` accounts from each AD domain:

```bash
ansible-playbook -i inventory cleanup.yml
```

## Connecting an LDAP application

| Parameter | Value |
|---|---|
| LDAP Server URI | `ldap://<ds-proxy-fqdn>:3911` |
| Bind DN | `uid=svc-cm,dc=example,dc=com` |
| Bind Password | `SvcCmPass123!` |
| User Search Base | `dc=example,dc=com` |
| User Search Scope | `SUBTREE` |
| User Search Filter | `(userPrincipalName=%(user)s)` |
| Group Type | `MemberDNGroupType` |
| Connection Options | `OPT_REFERRALS: 0` |

Using `userPrincipalName` (e.g., `jdoe@external.example.com`) lets users specify which domain they are authenticating against when the same `sAMAccountName` exists in multiple domains.

## Playbooks

| Playbook | Description |
|---|---|
| `configure-hosts.yml` | Adds AD domain entries to `/etc/hosts` on the Ansible controller and DS proxy host |
| `check-hosts.yml` | Pre-flight check imported by all other playbooks; aborts if `/etc/hosts` is not configured |
| `ad/deploy-domain-controller.yml` | Installs AD DS, creates forests, promotes to DC, configures timezone and NTP |
| `ad/create-users-groups.yml` | Creates AD users and groups defined in `vars.yml` |
| `ad/enable-ldaps.yml` | Creates self-signed cert, trusts it, reboots DC, verifies LDAPS |
| `ad/diagnose-ldaps.yml` | Read-only LDAPS troubleshooting (events, certs, boot time) |
| `ad/create-proxy-account.yml` | Creates `svc-ldapproxy` on each AD domain and verifies auth |
| `ds/deploy-proxy-instance.yml` | Full proxy lifecycle: instance, certs, entries, chaining |
| `ds/test-proxy.yml` | Cross-domain search, service account bind, wrong password rejection |
| `cleanup.yml` | Removes the DS instance and proxy accounts from AD |

AD and hosts tasks (`ad/deploy-domain-controller.yml`, `ad/create-users-groups.yml`, `check-hosts.yml`, `configure-hosts.yml`) use shared task files from [`common/`](../common/).
