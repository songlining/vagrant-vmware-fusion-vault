# HashiCorp Vault Enterprise 3-Node Cluster with Vagrant & VMware Fusion

This project sets up a 3-node HashiCorp Vault Enterprise cluster using Vagrant and VMware Fusion on ARM64 architecture (Apple Silicon).

## Overview

The setup creates a highly available Vault cluster with:
- **3 Vault Enterprise nodes** in Raft storage mode
- **Automatic initialization and unsealing**
- **Load balancing and failover capabilities**
- **Web UI access** via port forwarding

## Architecture
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│    vault-1      │    │    vault-2      │    │    vault-3      │
│ 192.168.56.10   │◄──►│ 192.168.56.11   │◄──►│ 192.168.56.12   │
│   (Leader)      │    │   (Follower)    │    │   (Follower)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
▲
│ Port Forward 8200:8200
▼
┌─────────────────┐
│   Host Machine  │
│ localhost:8200  │
└─────────────────┘


## Prerequisites

- **VMware Fusion** (for ARM64/Apple Silicon)
- **Vagrant** with VMware provider
- **Vault Enterprise License** (`vault.hclic` file)
- **macOS** with Apple Silicon

## Quick Start

### 1. Setup License
```bash
# Place your Vault Enterprise license file in the project root
cp /path/to/your/vault.hclic .
```

### 2. Start the Cluster
```bash
vagrant up
```

This will:
- Create 3 Ubuntu 22.04 ARM64 VMs
- Install Vault Enterprise on each node
- Configure Raft storage backend
- Automatically initialize the cluster (5 key shares, 3 key threshold)
- Unseal all nodes automatically

### 3. Access Vault

**Web UI**: http://localhost:8200

**CLI Access**:
```bash
export VAULT_ADDR="http://localhost:8200"
vault status
```

**Root Token**: Found in `vault-init.json`
```bash
cat vault-init.json | jq -r '.root_token'
```

## Cluster Configuration

| Node | Hostname | IP Address | Role | Port Forwarding |
|------|----------|------------|------|----------------|
| 1 | vault-1 | 192.168.56.10 | Leader | 8200 → 8200 |
| 2 | vault-2 | 192.168.56.11 | Follower | - |
| 3 | vault-3 | 192.168.56.12 | Follower | - |

### VM Specifications
- **OS**: Ubuntu 22.04 ARM64
- **Memory**: 2GB per VM
- **CPUs**: 2 per VM
- **Storage**: Raft (local disk)

## Key Features

### Automatic Initialization
- **Key Shares**: 5 total unseal keys
- **Key Threshold**: 3 keys required to unseal
- **Auto-unseal**: All nodes unsealed automatically on startup

### High Availability
- **Raft Consensus**: Leader election and data replication
- **Failover**: Automatic leader election if primary fails
- **Load Balancing**: Requests forwarded to leader automatically

### Security
- **Shamir's Secret Sharing**: Distributed unseal keys
- **Enterprise License**: Included via license file
- **Network Isolation**: Private network (192.168.56.0/24)

## Management Commands

### Cluster Operations
```bash
# Check cluster status
vault status
vault operator raft list-peers

# View cluster health
vault operator raft autopilot state
```

### VM Management
```bash
# Stop all VMs
vagrant halt

# Start all VMs
vagrant up

# Restart cluster
vagrant reload

# Destroy cluster (WARNING: Data loss)
vagrant destroy

# Check VM status
vagrant status
```

### Individual Node Management
```bash
# Manage specific nodes
vagrant halt vault-1
vagrant up vault-2
vagrant ssh vault-3
```

## File Structure
vault-vmware-vagrant/
├── Vagrantfile          # Main configuration
├── README.md           # This file
├── .gitignore          # Git ignore rules
├── vault.hclic         # Vault Enterprise license (not in git)
└── vault-init.json     # Initialization data (auto-generated)

## Troubleshooting

### Common Issues

**License Missing**:
```bash
# Ensure vault.hclic exists in project root
ls -la vault.hclic
```

**Vault Sealed**:
```bash
# Manual unseal (if needed)
vault operator unseal $(cat vault-init.json | jq -r '.unseal_keys_b64[0]')
vault operator unseal $(cat vault-init.json | jq -r '.unseal_keys_b64[1]')
vault operator unseal $(cat vault-init.json | jq -r '.unseal_keys_b64[2]')
```

**Connection Issues**:
```bash
# Check if VMs are running
vagrant status

# Check Vault service status
vagrant ssh vault-1 -c "sudo systemctl status vault"
```

### Logs
```bash
# View Vault logs
vagrant ssh vault-1 -c "sudo journalctl -u vault -f"

# Check system logs
vagrant ssh vault-1 -c "sudo tail -f /var/log/syslog"
```

### Manual Unsealing
If automatic unsealing fails:
```bash
# SSH into any node
vagrant ssh vault-1

# Run the unseal script
sudo /usr/local/bin/unseal-vault

# Or manually unseal
export VAULT_ADDR="http://192.168.56.10:8200"
vault operator unseal # Enter unseal key when prompted
```

## Security Notes

⚠️ **Important**: This setup is for **development/testing only**

- TLS is disabled (`tls_disable = 1`)
- Root token and unseal keys are stored in plaintext
- VMs use default credentials
- No firewall restrictions

For production use:
- Enable TLS encryption
- Use auto-unseal mechanisms (AWS KMS, Azure Key Vault, etc.)
- Implement proper key management
- Configure network security
- Use HashiCorp Cloud Platform (HCP) Vault

## Advanced Usage

### Accessing Individual Nodes
```bash
# Connect to specific nodes directly
export VAULT_ADDR="http://192.168.56.10:8200"  # vault-1
export VAULT_ADDR="http://192.168.56.11:8200"  # vault-2
export VAULT_ADDR="http://192.168.56.12:8200"  # vault-3
```

### Cluster Maintenance
```bash
# Remove a node from cluster (advanced)
vault operator raft remove-peer vault-node-2

# Add a node back to cluster
vault operator raft join http://192.168.56.10:8200
```

### Performance Monitoring
```bash
# Check Raft metrics
vault read sys/metrics

# Monitor cluster performance
vault read sys/health
```

## License

Requires a valid HashiCorp Vault Enterprise license. The `vault.hclic` file is excluded from version control for security.

## Support

For issues with:
- **Vault**: [HashiCorp Vault Documentation](https://www.vaultproject.io/docs)
- **Vagrant**: [Vagrant Documentation](https://www.vagrantup.com/docs)
- **VMware Fusion**: [VMware Fusion Documentation](https://docs.vmware.com/en/VMware-Fusion/)

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with `vagrant up`
5. Submit a pull request

---

**Note**: This project is designed for ARM64 architecture (Apple Silicon). For Intel-based systems, modify the `config.vm.box` in the Vagrantfile to use an appropriate x86_64 box.