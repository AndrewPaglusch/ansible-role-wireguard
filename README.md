# Ansible WireGuard Role

A simple and opinionated Ansible role for configuring WireGuard point-to-point tunnels.

## Features

- Installs WireGuard packages
- Creates WireGuard configuration files with proper permissions
- Manages WireGuard systemd services
- Supports both inbound-only and outbound peer configurations
- Supports multiple peers for hub-and-spoke or mesh configurations

## Limitations

- No keys are generated for you. You must provide them
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
| `wireguard_peers` | List of peer configurations | See examples below |

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `wireguard_interface` | Interface name | `wg0` |
| `wireguard_port` | UDP listen port | `51820` |
| `wireguard_peer_keepalive` | Default keepalive interval in seconds | `25` |
| `wireguard_skip_install` | Skip package installation of `wireguard` and `wireguard-tools` | `false` |
| `wireguard_postup` | List of PostUp commands with optional comments | `[]` (empty list) |
| `wireguard_postdown` | List of PostDown commands with optional comments | `[]` (empty list) |

### Peer Configuration

Each peer in the `wireguard_peers` list supports:

| Field | Description | Required | Example |
|-------|-------------|----------|---------|
| `public_key` | Public key of the peer | Yes | `"CLIENT_PUBLIC_KEY_HERE"` |
| `allowed_ips` | Allowed IPs from this peer | Yes | `"10.0.0.2/32"` |
| `name` | Optional comment for the peer | No | `"client1"` |
| `endpoint` | Peer endpoint (hostname:port) | No | `"client.example.com:51820"` |
| `keepalive` | Keepalive interval (overrides default) | No | `30` |

**Note:** The `endpoint` is optional. When unset, the peer will only listen for inbound connections. When set, this host will actively connect to the specified endpoint.

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
        wireguard_peers:
          - name: "host02"
            public_key: "{{ host02_public_key }}"
            allowed_ips: "10.100.0.2/32"

# Configure second peer (with outbound connection to host01)
- hosts: host02.example.com
  become: yes
  roles:
    - role: wireguard
      vars:
        wireguard_address: "10.100.0.2/30"
        wireguard_private_key: "{{ host02_private_key }}"
        wireguard_peers:
          - name: "host01"
            public_key: "{{ host01_public_key }}"
            allowed_ips: "10.100.0.1/32"
            endpoint: "host01.example.com:51820"
        wireguard_postup:
          - comment: "Allow forwarding through WireGuard interface"
            command: "iptables -A FORWARD -i %i -j ACCEPT"
          - comment: "Enable NAT for WireGuard traffic"
            command: "iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE"
        wireguard_postdown:
          - command: "iptables -D FORWARD -i %i -j ACCEPT"
          - command: "iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE"
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
