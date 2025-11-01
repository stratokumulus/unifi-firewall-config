# Unifi Cloud Gateway Firewall Configuration

This Ansible playbook automates the configuration of firewall rules on a Unifi Cloud Gateway using a data-driven approach. Rules are defined in YAML format for easy maintenance and updates.

## Overview

The playbook creates a secure firewall ruleset that:
- Implements a default deny policy for inter-VLAN traffic
- Allows the default network to access all other networks
- Grants specific IoT devices controlled access based on their requirements
- Maintains established and related connections

**Key Features:**
- **Data-driven design**: All rules defined in YAML format
- **Easy maintenance**: Add/remove/modify rules by editing `vars/firewall_rules.yml`
- **Idempotent**: Can be run multiple times safely
- **Flexible**: Supports all Unifi firewall rule parameters

## Prerequisites

### Required Software
- Ansible 2.9 or higher
- Python 3.6 or higher
- Access to Unifi Cloud Gateway with the API key of a **Super Admin** (needed for rules creation)

**IMPORTANT**: Regular "Admin" users can read via API but cannot create/modify rules.

### Required Files
```
.
├── configure_firewall.yml          # Main playbook
├── vars/
│   └── firewall_rules.yml          # Firewall rules definition
```

## Network Architecture

### VLANs Defined

| VLAN ID | Name | Subnet | Purpose |
|---------|------|--------|---------|
| 1 | default | 192.168.1.0/24 | Main trusted network |
| 2 | lab | 192.168.2.0/24 | Homelab network |
| 10 | iot | 192.168.10.0/24 | IoT devices |
| 11 | cameras | 192.168.11.0/24 | Security cameras |
| 100 | untrusted | 192.168.100.0/24 | Isolated, untrustred network |

### Special Host Rules

| Hostname | IP Address | Access Level | Description |
|----------|------------|--------------|-------------|
| automation-hub | 192.168.10.3 | All VLANs | Home automation hub |
| robot1 | 192.168.10.60 | Internet only | Robot vacuum |
| robot2 | 192.168.10.61 | Internet only | Robot mower |

## Configuration Structure

### Main Playbook

The `configure_firewall.yml` playbook:

1. Loads rules from `vars/firewall_rules.yml`
2. Removes existing ANSIBLE-managed rules
3. Creates all rules defined in the configuration
4. Provides detailed output and summaries

### Rules Definition File

The `vars/firewall_rules.yml` file contains:
- **VLANs**: Network definitions for reference
- **Special Hosts**: Host-specific access requirements
- **Firewall Rules**: List of all firewall rules to create

## Rule Definition Format

Each rule in `vars/firewall_rules.yml` follows this structure:

```yaml
firewall_rules:
  - name: "ANSIBLE-Rule-Name"              # Required: Rule name (must start with ANSIBLE-)
    description: "Rule description"         # Optional: Human-readable description
    ruleset: "LAN_IN"                       # Required: LAN_IN, LAN_OUT, WAN_IN, WAN_OUT
    rule_index: 2000                        # Required: Priority (lower = higher priority)
    enabled: true                           # Optional: Enable/disable (default: true)
    action: "accept"                        # Required: accept, drop, reject
    protocol: "all"                         # Optional: all, tcp, udp, icmp (default: all)

    # Optional source/destination
    src_address: "192.168.1.0/24"          # Source IP or CIDR
    dst_address: "192.168.2.0/24"          # Destination IP or CIDR
    src_port: "1024-65535"                  # Source port(s)
    dst_port: "80,443"                      # Destination port(s)

    # Optional VLAN references
    src_networkconf_id: "1"                 # Source VLAN ID
    dst_networkconf_id: "10"                # Destination VLAN ID

    # Optional connection states
    state_established: true                 # Allow established connections
    state_related: true                     # Allow related connections
    state_new: false                        # Allow new connections
    state_invalid: false                    # Allow invalid connections

    # Optional logging
    logging: true                           # Enable logging for this rule
```

### Supported Parameters

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `name` | string | Rule name (must start with "ANSIBLE-") | "ANSIBLE-Allow-SSH" |
| `description` | string | Human-readable description | "Allow SSH from admin network" |
| `ruleset` | string | Rule direction | "LAN_IN", "LAN_OUT", "WAN_IN", "WAN_OUT" |
| `rule_index` | integer | Priority (lower = higher) | 2000 |
| `enabled` | boolean | Enable/disable rule | true, false |
| `action` | string | What to do with matching traffic | "accept", "drop", "reject" |
| `protocol` | string | Network protocol | "all", "tcp", "udp", "icmp" |
| `src_address` | string | Source IP/CIDR | "192.168.1.0/24" |
| `dst_address` | string | Destination IP/CIDR | "10.0.0.0/8" |
| `src_port` | string | Source port(s) | "1024-65535", "22,80,443" |
| `dst_port` | string | Destination port(s) | "80,443" |
| `src_networkconf_id` | string | Source VLAN ID | "1", "20" |
| `dst_networkconf_id` | string | Destination VLAN ID | "21", "100" |
| `state_established` | boolean | Match established connections | true, false |
| `state_related` | boolean | Match related connections | true, false |
| `state_new` | boolean | Match new connections | true, false |
| `state_invalid` | boolean | Match invalid connections | true, false |
| `logging` | boolean | Log matching traffic | true, false |
| `icmp_typename` | string | ICMP type | "echo-request", "echo-reply" |

## Installation and Setup

### 1. Clone or Download Files

```bash
git clone <your-repo-url>
cd unifi-firewall-config
```

### 2. Configure Environment Variables

Set the following environment variables:

```bash
export UNIFI_CONTROLLER="192.168.1.1"    # IP/hostname of Unifi controller
export UNIFI_USERNAME="admin"             # Admin username
export UNIFI_PASSWORD="your_password"     # Admin password
export UNIFI_SITE="default"               # Site name (optional, defaults to "default")
export UNIFI_PORT="443"                   # HTTPS port (optional, defaults to "443")
```

### 3. Review and Customize Rules

Edit `vars/firewall_rules.yml` to match your requirements:

```bash
vim vars/firewall_rules.yml
```

### 4. Run the Playbook

```bash
ansible-playbook configure_firewall.yml
```

## Usage Examples

### Basic Execution

```bash
# Set environment variables
export UNIFI_CONTROLLER="192.168.1.1"
export UNIFI_API_TOKEN="Blahblahblah"

# Run the playbook
ansible-playbook configure_firewall.yml
```

### Using Ansible Vault for Credentials

Create a vault file:

```bash
# Create vault file
ansible-vault create vault.yml
```

Add credentials:
```yaml
---
vault_unifi_api_token: "Blahblahblah"
```

Modify the playbook to use vault variables:
```yaml
vars:
  unifi_username: "{{ vault_unifi_api_token }}"

```

Run with vault:
```bash
ansible-playbook configure_firewall.yml --ask-vault-pass
```

### Check Mode (Dry Run)

```bash
ansible-playbook configure_firewall.yml --check
```

### Verbose Output

```bash
ansible-playbook configure_firewall.yml -vvv
```

## Adding New Rules

To add a new firewall rule, edit `vars/firewall_rules.yml` and add a new entry to the `firewall_rules` list:

### Example 1: Allow SSH from Admin Network

```yaml
firewall_rules:
  # ... existing rules ...

  - name: "ANSIBLE-Allow-SSH-from-Admin"
    description: "Allow SSH access from default network to lab"
    ruleset: "LAN_IN"
    rule_index: 2100
    enabled: true
    action: "accept"
    protocol: "tcp"
    src_address: "192.168.1.0/24"
    dst_address: "192.168.2.0/24"
    dst_port: "22"
    logging: false
```

### Example 2: Allow Camera RTSP from Lab

```yaml
  - name: "ANSIBLE-Lab-to-Cameras-RTSP"
    description: "Allow lab network to access camera RTSP streams"
    ruleset: "LAN_IN"
    rule_index: 2101
    enabled: true
    action: "accept"
    protocol: "tcp"
    src_address: "192.168.2.0/24"
    dst_address: "192.168.21.0/24"
    dst_port: "554,8554"
    logging: false
```

### Example 3: Allow DNS from IoT Network

```yaml
  - name: "ANSIBLE-IoT-to-DNS"
    description: "Allow IoT devices to access DNS server"
    ruleset: "LAN_IN"
    rule_index: 2102
    enabled: true
    action: "accept"
    protocol: "udp"
    src_address: "192.168.10.0/24"
    dst_address: "192.168.1.1"
    dst_port: "53"
    logging: false
```

### Example 4: Block Specific Host

```yaml
  - name: "ANSIBLE-Block-Suspicious-Device"
    description: "Block suspicious IoT device from internet"
    ruleset: "LAN_OUT"
    rule_index: 2103
    enabled: true
    action: "drop"
    protocol: "all"
    src_address: "192.168.10.100"
    logging: true
```

After adding rules, simply re-run the playbook:

```bash
ansible-playbook configure_firewall.yml
```

## Modifying Existing Rules

### Temporarily Disable a Rule

Set `enabled: false` in the rule definition:

```yaml
  - name: "ANSIBLE-Robot1-Internet"
    description: "Allow robot1 to access internet"
    enabled: false  # Temporarily disabled
    # ... rest of rule ...
```

### Change Rule Priority

Modify the `rule_index` value (remember: lower = higher priority):

```yaml
  - name: "ANSIBLE-HomeAssistant-to-All"
    rule_index: 1999  # Changed from 2002 to run earlier
```

### Remove a Rule

Simply delete or comment out the rule in `vars/firewall_rules.yml`:

```yaml
# Removed - no longer needed
#  - name: "ANSIBLE-Old-Rule"
#    ruleset: "LAN_IN"
#    ...
```

## Firewall Rules Logic

My current configuration implements the following hierarchy (evaluated in order):

1. **Allow Established/Related** (Priority: 2000)
   - Permits return traffic for existing connections
   - Essential for bidirectional communication

2. **Allow Default Network to All** (Priority: 2001)
   - Source: 192.168.1.0/24 (default VLAN)
   - Destination: All networks
   - Reason: Admin network needs full access

3. **Allow Home Assistant to All VLANs** (Priority: 2002)
   - Source: 192.168.20.3
   - Destination: All networks
   - Reason: Home automation requires device access

4. **Block Robots to RFC1918** (Priority: 2003-2004)
   - Source: 192.168.20.60, 192.168.20.61
   - Destination: Private networks (this requires to create a group, query the API to get the group id, and put this is dsp_firewallgroup_ids. Or, create one rule per private subnet :) )
   - Action: Drop
   - Reason: Restrict to internet-only

5. **Allow Robots to Internet** (Priority: 2005-2006)
   - Source: 192.168.20.60, 192.168.20.61
   - Destination: Any (after RFC1918 block)
   - Reason: Cloud connectivity required. Some robots require access to a cloud account to work (yuk :( next robot : valetudo enabled, with a local map only))

6. **Deny Inter-VLAN Traffic** (Priority: 2999)
   - Source: Any RFC1918 address (see comment for point #4)
   - Destination: Any RFC1918 address
   - Action: Drop with logging
   - Reason: Default deny policy

## Security Considerations

### Best Practices

1. **Default Deny Policy**: All inter-VLAN traffic is denied unless explicitly allowed
2. **Least Privilege**: Devices receive only the minimum access required
3. **Logging**: Important deny rules log traffic for auditing
4. **Stateful Firewall**: Established and related connections are tracked
5. **Rule Ordering**: Priority is critical - more specific rules should have lower indexes
6. **Documentation**: Use the `description` field to explain why each rule exists

### Security Implications

- **Default Network**: Acts as management/admin network with full access
- **Home Assistant**: Broad access required; consider additional application-level security
- **Robots**: Internet-only access prevents lateral movement
- **Cameras**: Isolated unless explicitly allowed
- **Airgap Lab**: Should remain isolated for sensitive testing

### Recommended Additional Rules

Consider adding these rules for enhanced security:

```yaml
# Allow NTP for time synchronization
- name: "ANSIBLE-Allow-NTP"
  description: "Allow all networks to access NTP"
  ruleset: "LAN_IN"
  rule_index: 2010
  action: "accept"
  protocol: "udp"
  dst_port: "123"

# Allow DHCP
- name: "ANSIBLE-Allow-DHCP"
  description: "Allow DHCP traffic"
  ruleset: "LAN_IN"
  rule_index: 2011
  action: "accept"
  protocol: "udp"
  src_port: "68"
  dst_port: "67"

# Rate limit ICMP
- name: "ANSIBLE-Rate-Limit-ICMP"
  description: "Allow ICMP but rate limited"
  ruleset: "LAN_IN"
  rule_index: 2012
  action: "accept"
  protocol: "icmp"
  # Note: Rate limiting configured in Unifi UI
```

## Troubleshooting

### Connection Issues

```bash
# Test controller connectivity
curl -k -X GET 'https://192.168.1.1/proxy/network/api/s/default/rest/firewallrule' -H 'X-API-KEY: <put your API token here>' -H 'Accept: application/json'

# Verify credentials
ansible-playbook configure_firewall.yml -vvv
```

### Rule Verification

1. Log into Unifi Controller web interface
2. Navigate to Settings > Security > Firewall
3. Look for rules prefixed with "ANSIBLE-"
4. Verify rule order and configuration

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| Authentication failed | Invalid credentials | Verify you're using the right token and that the environment variable or vault is properly set |
| Connection refused | Wrong controller IP/port | Check UNIFI_CONTROLLER and UNIFI_PORT |
| 403 Forbidden on rule creation | Insufficient API permissions | The key must be created by a user that can create firewall rules in the webUI |
| Rule creation failed | Invalid rule syntax | Check vars/firewall_rules.yml for syntax errors |
| Rules not appearing | Wrong site name | Verify UNIFI_SITE matches your site name |
| SSL certificate error | Self-signed certificate | Playbook uses `validate_certs: false` |

### Debugging Tips

1. **Enable verbose output:**
   ```bash
   ansible-playbook configure_firewall.yml -vvv 2>&1 | tee ansible.log
   ```

2. **Validate YAML syntax:**
   ```bash
   ansible-playbook configure_firewall.yml --syntax-check
   ```

3. **Check rule file syntax:**
   ```bash
   python3 -c "import yaml; yaml.safe_load(open('vars/firewall_rules.yml'))"
   ```

4. **Test rule logic:**
   ```bash
   # From source network, test connectivity
   ping <destination_ip>
   curl -v <destination_ip>:<port>
   ```

5. **Check Unifi logs:**
   ```bash
   # SSH to controller
   tail -f /var/log/unifi/server.log
   ```

### Rules Not Working as Expected

1. **Check rule order**: Lower rule_index has higher priority
2. **Check for overlapping rules**: More specific rules should come first
3. **Enable logging**: Set `logging: true` to see what's being blocked
4. **Verify stateful rules**: Ensure established/related rule is first
5. **Check Unifi UI**: Manually verify rules appear correctly

## Maintenance

### Updating Rules

1. Edit `vars/firewall_rules.yml`
2. Re-run the playbook
3. Existing ANSIBLE-* rules are automatically removed and recreated

```bash
vim vars/firewall_rules.yml
ansible-playbook configure_firewall.yml
```

### Backing Up Configuration

Before making changes, backup your Unifi configuration:

### Removing All Managed Rules

To remove all ANSIBLE-managed rules, delete all rules from `vars/firewall_rules.yml` and run:

```bash
# Make firewall_rules an empty list
echo "firewall_rules: []" > vars/firewall_rules_empty.yml

# Run with empty rules
ansible-playbook configure_firewall.yml -e @vars/firewall_rules_empty.yml
```

Or manually delete via Unifi UI by searching for "ANSIBLE-" prefix.

## Testing

### Test Plan

1. **Verify Default Network Access**
   ```bash
   # From 192.168.1.x host
   ping 192.168.2.1    # Should succeed
   ping 192.168.20.1   # Should succeed
   ping 192.168.21.1   # Should succeed
   ```

2. **Verify IoT Isolation**
   ```bash
   # From 192.168.20.x host (not home-assistant)
   ping 192.168.1.1    # Should fail
   ping 192.168.2.1    # Should fail
   ```

3. **Verify Home Assistant Access**
   ```bash
   # From 192.168.20.3
   ping 192.168.1.1    # Should succeed
   ping 192.168.2.1    # Should succeed
   curl 192.168.21.100 # Should succeed
   ```

4. **Verify Robot Internet-Only Access**
   ```bash
   # From 192.168.20.60 or 192.168.20.61
   ping 8.8.8.8        # Should succeed
   ping 192.168.1.1    # Should fail
   ping 192.168.2.1    # Should fail
   ```

5. **Verify Logging**
   ```bash
   # Check Unifi logs for blocked traffic
   # Settings > System > Logs > Events
   # Filter by "Firewall"
   ```

### Automated Testing

Create a test script:

```bash
#!/bin/bash
# test_firewall.sh

echo "Testing firewall rules..."

# Test from various networks
for src in 192.168.1.10 192.168.20.3 192.168.20.60; do
  echo "Testing from $src"
  ssh user@$src "ping -c 1 192.168.1.1"
  ssh user@$src "ping -c 1 8.8.8.8"
done
```

## Advanced Usage

### Rule Templates

Create reusable rule templates in `vars/firewall_rules.yml`:

```yaml
# Define template anchor
rule_templates:
  allow_web: &allow_web
    protocol: "tcp"
    dst_port: "80,443"
    action: "accept"

firewall_rules:
  # Use template
  - name: "ANSIBLE-Lab-Web-Access"
    <<: *allow_web
    ruleset: "LAN_IN"
    rule_index: 2200
    src_address: "192.168.2.0/24"
```

### Environment-Specific Configurations

Maintain different rule sets for different environments:

```bash
# Production rules
ansible-playbook configure_firewall.yml -e @vars/firewall_rules_prod.yml

# Development rules
ansible-playbook configure_firewall.yml -e @vars/firewall_rules_dev.yml
```

### Integration with CI/CD

```yaml
# .gitlab-ci.yml or .github/workflows/deploy.yml
deploy_firewall:
  script:
    - ansible-playbook configure_firewall.yml
  only:
    changes:
      - vars/firewall_rules.yml
```

## File Reference

### Project Structure

```
unifi-firewall-config/
├── configure_firewall.yml           # Main Ansible playbook
├── vars/
│   └── firewall_rules.yml           # Firewall rules definition
├── README.md                        # This file
```

### vars/firewall_rules.yml

Primary configuration file containing:
- Network definitions (vlans, subnets)
- Special host definitions
- Complete firewall rules list

### configure_firewall.yml

Main playbook that:
- Loads configuration from vars/
- Authenticates to Unifi Controller
- Manages firewall rules lifecycle
- Provides detailed logging

## License

This playbook is provided as-is for education purposes only. 
