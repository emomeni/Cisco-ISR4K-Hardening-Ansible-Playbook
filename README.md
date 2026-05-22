# Cisco ISR 4000 Series - CIS/NSA Hardening Playbook

Ansible playbook that applies CIS Benchmark (v4.x) and NSA/CISA hardening controls to Cisco ISR 4000 series routers running IOS XE.

## What This Playbook Does

| Section | Controls | Tag |
|---------|----------|-----|
| Service hardening | Disable BOOTP, PAD, finger, CDP, LLDP, HTTP, small servers, source routing | `services` |
| Passwords | Enable secret (scrypt), min-length 14, remove plaintext password | `passwords` |
| AAA / TACACS+ | AAA new-model, TACACS+ group with local fallback, login blocking | `aaa` |
| SSH | SSHv2 only, 4096-bit RSA keys, timeout, retries, VTY ACL | `ssh` |
| Interfaces | Disable proxy-arp, redirects, unreachables, directed-broadcast on all interfaces | `interfaces` |
| SNMP | Remove v1/v2c communities, configure SNMPv3 (SHA + AES128), restrict with ACL | `snmp` |
| Banners | Login, MOTD, and EXEC banners | `banners` |
| NTP | Authenticated NTP with trusted keys | `ntp` |
| Logging | Remote syslog, buffered logging, source-interface, disable console logging | `logging` |
| CoPP | Rate-limit ICMP, SSH, SNMP, routing protocols, drop everything else | `copp` |

## Prerequisites

- **Ansible** >= 2.14
- **Python** >= 3.10
- **Collections** (install before first run):

```bash
ansible-galaxy collection install cisco.ios ansible.netcommon
```

- SSH reachability from the Ansible control node to all target routers
- A local user account on each router (fallback for TACACS+ outages)
- A pre-generated scrypt (`type 9`) hash for the enable secret

### Generating the enable secret hash

On any IOS XE device:

```
Router(config)# enable algorithm-type scrypt secret YourStrongPasswordHere
Router# show running-config | include enable secret
```

Copy the full `$9$...` hash into `group_vars/all.yml` as the `enable_secret_hash` value.

## Directory Structure

```
cisco-isr4k-hardening/
  inventory/
    hosts.yml                  # Inventory with connection vars
  group_vars/
    all.yml                    # Variables (vault-encrypt this)
  cisco_isr4k_hardening.yml    # Main playbook
  README.md                    # This file
```

## Setup

### 1. Edit the inventory

Open `inventory/hosts.yml` and replace the example hostnames and IPs with your routers:

```yaml
isr4k-core-01:
  ansible_host: 10.0.100.1
```

### 2. Fill in variables

Edit `group_vars/all.yml`. Replace every `CHANGE_ME` placeholder with real values:

- `vault_ansible_user` / `vault_ansible_password` - SSH credentials
- `vault_enable_secret` - enable mode password
- `enable_secret_hash` - scrypt hash (see above)
- `tacacs_server_ip` / `tacacs_key` - your TACACS+ server
- `snmp_auth_pass` / `snmp_priv_pass` - SNMPv3 credentials
- `ntp_key_values` - NTP authentication keys
- `mgmt_acl_permit_subnet` - your management network
- `syslog_server_ip` - your syslog collector
- `management_interface` - the interface used for management traffic

### 3. Encrypt the variables file

```bash
ansible-vault encrypt group_vars/all.yml
```

You'll set a vault password. Remember it - you'll need it every time you run the playbook.

## Running the Playbook

### Dry run (check mode) - see what would change without touching the router

```bash
ansible-playbook -i inventory/hosts.yml cisco_isr4k_hardening.yml \
  --ask-vault-pass --check --diff
```

### Full run - apply all hardening controls

```bash
ansible-playbook -i inventory/hosts.yml cisco_isr4k_hardening.yml \
  --ask-vault-pass
```

### Run specific sections only (using tags)

```bash
# SSH hardening only
ansible-playbook -i inventory/hosts.yml cisco_isr4k_hardening.yml \
  --ask-vault-pass --tags ssh

# AAA and TACACS+ only
ansible-playbook -i inventory/hosts.yml cisco_isr4k_hardening.yml \
  --ask-vault-pass --tags aaa

# CoPP only
ansible-playbook -i inventory/hosts.yml cisco_isr4k_hardening.yml \
  --ask-vault-pass --tags copp

# Multiple tags
ansible-playbook -i inventory/hosts.yml cisco_isr4k_hardening.yml \
  --ask-vault-pass --tags "ssh,aaa,logging"
```

### Target a single router

```bash
ansible-playbook -i inventory/hosts.yml cisco_isr4k_hardening.yml \
  --ask-vault-pass --limit isr4k-core-01
```

### Use a vault password file instead of prompting

```bash
echo 'your-vault-password' > .vault_pass
chmod 600 .vault_pass
ansible-playbook -i inventory/hosts.yml cisco_isr4k_hardening.yml \
  --vault-password-file .vault_pass
```

Add `.vault_pass` to `.gitignore`. Don't commit it.

## Available Tags

| Tag | What it covers |
|-----|---------------|
| `services` | BOOTP, PAD, finger, CDP, LLDP, HTTP, small servers |
| `passwords` | Enable secret, min-length, remove plaintext |
| `aaa` | AAA new-model, TACACS+, login blocking |
| `tacacs` | TACACS+ server configuration only |
| `ssh` | SSHv2, keys, timeouts, VTY hardening |
| `vty` | VTY line configuration only |
| `console` | Console line configuration only |
| `interfaces` | Proxy-arp, redirects, unreachables, directed-broadcast |
| `snmp` | Remove v1/v2c, configure SNMPv3 |
| `banners` | Login, MOTD, EXEC banners |
| `ntp` | NTP servers and authentication |
| `logging` | Syslog, buffer, timestamps |
| `copp` | Control Plane Policing class-maps, policy-map, ACLs |
| `cis` | All CIS Benchmark controls |
| `nsa` | All NSA/CISA controls |
| `verify` | Post-run verification checks only |
| `acl` | All ACL-related tasks |

## Idempotency

Every task is safe to re-run. The `cisco.ios.ios_config` module compares desired state against the running config and only pushes changes when needed.

The RSA key generation task (`CIS 4.1.4`) uses `changed_when` logic to detect whether keys already exist at the requested bit length.

The SNMPv3 user task uses `ios_command` because IOS XE doesn't store `snmp-server user` in the running config. It will re-apply on each run. This is the standard approach - the router accepts the duplicate command without error.

## Post-run Verification

The playbook runs four verification checks automatically:

1. SSHv2 is active
2. AAA new-model is present in running config
3. CoPP policy is applied to the control plane
4. NTP status (displayed for manual review)

Run verification checks alone:

```bash
ansible-playbook -i inventory/hosts.yml cisco_isr4k_hardening.yml \
  --ask-vault-pass --tags verify
```

## Customisation

### Adjusting CoPP rate limits

All CoPP rates are in `group_vars/all.yml`. The defaults are conservative starting points:

| Class | Rate (bps) | Burst |
|-------|-----------|-------|
| ICMP | 8000 | 1500 |
| SSH | 64000 | 8000 |
| SNMP | 32000 | 4000 |
| Routing | 128000 | 16000 |
| Default | 8000 | 1000 |

Tune based on your traffic patterns. Monitor with `show policy-map control-plane` after deployment.

### Adding routing protocol authentication

This playbook doesn't configure routing protocol authentication (OSPF MD5/SHA, BGP TCP-AO, EIGRP key chains) because those are topology-specific. Add them as separate tasks or a dedicated role.

### Extending to other platforms

This playbook targets IOS XE specifically. For IOS (classic), NX-OS, or IOS XR, you'll need platform-specific modules and adjusted commands.

## Troubleshooting

**"Target does not appear to be a Cisco IOS XE device"** - The pre-check assertion failed. Verify the device model and image name in the error message. If it's a valid ISR 4000 but the string matching is off, adjust the assert conditions in the pre-tasks.

**SSH connection timeout** - Check that SSH is already enabled on the target. This playbook hardens SSH - it doesn't enable it from scratch. You need basic SSH access before running the playbook.

**TACACS+ lockout** - If TACACS+ is unreachable after applying AAA, the `local` fallback in the authentication line lets you log in with local credentials. Always maintain a working local account.

**SNMPv3 user already exists** - IOS XE may show a warning. The command is still accepted. This is expected behaviour.

## References

- [CIS Cisco IOS Benchmark v4.1.1](https://www.cisecurity.org/benchmark/cisco)
- [NSA/CISA Network Infrastructure Security Guide](https://media.defense.gov/2022/Jun/15/2003018261/-1/-1/0/CTR_NSA_NETWORK_INFRASTRUCTURE_SECURITY_GUIDE_20220615.PDF)
- [Cisco IOS XE Hardening Guide](https://www.cisco.com/c/en/us/support/docs/ip/access-lists/13608-21.html)
- [cisco.ios Ansible Collection Docs](https://docs.ansible.com/ansible/latest/collections/cisco/ios/index.html)
