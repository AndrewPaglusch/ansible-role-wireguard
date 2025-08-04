# Ansible WireGuard Role

A simple and opinionated Ansible role for configuring WireGuard point-to-point tunnels.

## Features

- Installs WireGuard packages
- Creates WireGuard configuration files with proper permissions
- Manages WireGuard systemd services
- Supports both inbound-only and outbound peer configurations

## Limitations

- No keys are generated for you. You must provide them
- Only a single peer can be specified, so this role is really only for connecting two servers or sites
- No fancy DKMS or kernel modules are managed. Your kernel must support Wireguard.

## Requirements

- Ansible 2.9+
- Target systems with systemd support
- WireGuard available in package repositories

## Role Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `wireguard_address` | Interface IP address with CIDR | `"172.19.42.1/30"` |
| `wireguard_private_key` | Private key for this interface | `"{{ vault_private_key }}"` |
| `wireguard_peer_public_key` | Public key of the peer | `"{{ vault_peer_public_key }}"` |
| `wireguard_peer_allowed_ips` | Allowed IPs from peer | `"172.19.42.2/32"` |

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `wireguard_interface` | Interface name | `wg0` |
| `wireguard_port` | UDP listen port | `51820` |
| `wireguard_peer_keepalive` | Keepalive interval in seconds | `25` |
| `wireguard_peer_endpoint` | Peer endpoint (hostname:port) | `""` (unset) |

**Note:** The `wireguard_peer_endpoint` is optional. When unset, the peer will only listen for inbound connections. When set, the peer will actively connect to the specified endpoint.

## Example Playbooks

### Simple Point-to-Point Tunnel

This example sets up a tunnel between two hosts:

```yaml
---
# Configure first peer
- hosts: host01.example.com
  become: yes
  roles:
    - role: wireguard
      vars:
        wireguard_address: "10.100.0.1/30"
        wireguard_private_key: "{{ host01_private_key }}"
        wireguard_peer_public_key: "{{ host02_public_key }}"
        wireguard_peer_allowed_ips: "10.100.0.2/32"

# Configure second peer (with outbound connection to host01)
- hosts: host02.example.com
  become: yes
  roles:
    - role: wireguard
      vars:
        wireguard_address: "10.100.0.2/30"
        wireguard_private_key: "{{ host02_private_key }}"
        wireguard_peer_endpoint: "host01.example.com:51820"
        wireguard_peer_public_key: "{{ host01_public_key }}"
        wireguard_peer_allowed_ips: "10.100.0.1/32"
```

## Key Management

This role does not generate keys. You must provide pre-generated keys via variables. It's recommended to store the keys using Ansible Vault.

Generate keys using:
```bash
# Generate private key
wg genkey

# Generate public key from private key
echo "PRIVATE_KEY_HERE" | wg pubkey
```
