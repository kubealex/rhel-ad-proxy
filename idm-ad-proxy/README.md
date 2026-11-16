# IDM - AD Proxy

> **Disclaimer:** This project is provided as-is for testing and demonstration purposes only. It is not officially supported in any way. No guarantees are made regarding its functionality, reliability, or suitability for any environment. The author assumes no responsibility or liability for any issues, damages, or consequences arising from its use. Use at your own risk.

Three AD domains (`external.example.com`, `internal.example.com`, `company.example.com`) with IdM trusting each one. IdM acts as a unified authentication endpoint that can resolve users from any trusted domain via the `cn=users,cn=compat` tree, without needing POSIX group mappings on the IdM side.

## Configuration

Edit [`vars.yml`](vars.yml) and [`inventory`](inventory) before running.

**Required before every deployment:** set `ad_domain_controller_ip` for each AD server entry in `vars.yml` — these IPs change with every lab build and are intentionally left as `CHANGE_ME` placeholders.

Key settings in `vars.yml`:

| Variable | Description |
|---|---|
| `idm_domain` / `idm_domain_server` | IdM domain and server FQDN |
| `idm_admin_password` | IdM admin password |
| `ad_users` / `ad_groups` | Users and groups to create on each AD domain |
| `ad_servers[]` | List of AD domains with FQDN, NetBIOS name, admin credentials, and DC IP |
| `ad_servers[].ad_domain_controller_ip` | **Must be set** to the current IP of each domain controller |

## What gets deployed

| Component | Details |
|---|---|
| AD domain controllers | Three independent forests (`external`, `internal`, `company`), each with users (`jdoe`, `sysadmin`, `dba`) and group `linux-admins` |
| IdM trusts | Two-way Kerberos trust between IdM and each AD domain |

No external/POSIX groups are created on the IdM side since users do not log into RHEL directly in this scenario.

## Run order

> **Prerequisite:** `configure-hosts.yml` must be run first. It populates `/etc/hosts` on both the Ansible controller and the IdM server so that all AD domain FQDNs resolve. Every other playbook checks for this and will abort with instructions if it has not been done.

```bash
# 1. Set ad_domain_controller_ip in vars.yml, then populate /etc/hosts
ansible-playbook -i inventory configure-hosts.yml

# 2. Deploy the three AD domain controllers
ansible-playbook -i inventory ad/deploy-domain-controller.yml

# 3. Create AD users and groups on each domain
ansible-playbook -i inventory ad/create-users-groups.yml

# 4. Establish Kerberos trusts from IdM to all three AD domains
ansible-playbook -i inventory idm/establish-ad-trust.yml
```

## Cleanup

Removes all three trusts and their DNS forward zones from IdM:

```bash
ansible-playbook -i inventory cleanup.yml
```

## Verification

```bash
# Test proxy connectivity: compat tree searches, SSSD resolution, Kerberos auth
ansible-playbook -i inventory idm/test-proxy.yml

# Verify trust infrastructure only
ansible-playbook -i inventory idm/verify-trust.yml
```

## Connecting an LDAP application

| Parameter | Value |
|---|---|
| LDAP Server URI | `ldap://<idm-server-fqdn>` |
| Bind DN | `uid=admin,cn=users,cn=accounts,dc=rh-lab,dc=labs` |
| User Search Base | `cn=users,cn=compat,dc=rh-lab,dc=labs` |
| User Search Scope | `SUBTREE` |
| User Search Filter | `(uid=%(user)s)` |

## Playbooks

| Playbook | Description |
|---|---|
| `configure-hosts.yml` | Adds AD domain entries to `/etc/hosts` on the Ansible controller and IdM server |
| `check-hosts.yml` | Pre-flight check imported by all other playbooks; aborts if `/etc/hosts` is not configured |
| `ad/deploy-domain-controller.yml` | Installs AD DS, creates forests, promotes to DC, adds DNS forwarder to IdM, configures timezone and NTP |
| `ad/create-users-groups.yml` | Creates AD users and groups defined in `vars.yml` |
| `idm/establish-ad-trust.yml` | Creates DNS forward zones, disables DNSSEC, establishes two-way trusts |
| `idm/test-proxy.yml` | Tests proxy: compat tree searches, SSSD resolution for all users, Kerberos auth, wrong password rejection |
| `idm/verify-trust.yml` | Verifies trust infrastructure, compat tree lookups, Kerberos cross-realm auth |
| `cleanup.yml` | Removes all trusts and DNS forward zones |

AD and hosts tasks (`ad/`, `check-hosts.yml`, `configure-hosts.yml`) use shared task files from [`common/`](../common/).
