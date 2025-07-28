# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Use an ARM64-compatible box for VMware Fusion
  config.vm.box = "gyptazy/ubuntu22.04-arm64"
  
  # Configure VMware Fusion provider
  config.vm.provider "vmware_fusion" do |vmware|
    # VM memory in MB
    vmware.memory = 2048
    
    # Number of CPUs
    vmware.cpus = 2
    
    # Enable GUI (set to false for headless)
    vmware.gui = true
    
    # VM name in VMware
    vmware.vmx["displayName"] = "Ubuntu-ARM64-Vagrant"
    
    # Enable nested virtualization if needed
    # vmware.vmx["vhv.enable"] = "TRUE"
    
    # Optimize for ARM architecture
    vmware.vmx["ethernet0.virtualDev"] = "vmxnet3"
  end
  
  # Shared folders - Remove the type specification to use default
  config.vm.synced_folder ".", "/vagrant"
  
  # Copy license file from host to VM
  config.vm.provision "file", source: "./vault.hclic", destination: "/tmp/vault.hclic"
  
  # Provisioning script
  config.vm.provision "shell", inline: <<-SHELL
    # Wait for unattended-upgrades to finish
    echo "Waiting for unattended-upgrades to complete..."
    while sudo fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do
      sleep 5
    done
    
    # Disable unattended-upgrades during provisioning
    sudo systemctl stop unattended-upgrades
    sudo systemctl disable unattended-upgrades
    
    # Update package list
    sudo apt-get update
    
    # Install essential packages
    sudo apt-get install -y curl wget git vim htop jq

    
    # Re-enable unattended-upgrades
    sudo systemctl enable unattended-upgrades
    sudo systemctl start unattended-upgrades
    
    # Install Vault Enterprise
    echo "Installing HashiCorp Vault Enterprise..."
    
    # Add HashiCorp GPG key and repository
    wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
    echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
    
    # Update package list and install Vault Enterprise
    sudo apt-get update
    sudo apt-get install -y vault-enterprise
    
    # Create vault user and directories
    sudo mkdir -p /opt/vault/data
    sudo mkdir -p /etc/vault.d
    sudo chown -R vault:vault /opt/vault/data
    sudo chown -R vault:vault /etc/vault.d
    
    # Move license file to proper location with correct permissions
    sudo cp /tmp/vault.hclic /etc/vault.d/vault.hclic
    sudo chown vault:vault /etc/vault.d/vault.hclic
    sudo chmod 640 /etc/vault.d/vault.hclic
    
    # Create Vault configuration
    sudo tee /etc/vault.d/vault.hcl > /dev/null <<EOF
storage "raft" {
  path = "/opt/vault/data"
  node_id = "vault-node-1"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

api_addr = "http://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
ui = true
disable_mlock = true

# Add your license
license_path = "/etc/vault.d/vault.hclic"
EOF

    # Create systemd service
    sudo tee /etc/systemd/system/vault.service > /dev/null <<EOF
[Unit]
Description=HashiCorp Vault
Documentation=https://www.vaultproject.io/docs/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault.d/vault.hcl
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
Type=notify
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP \$MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitInterval=60
StartLimitBurst=3
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
EOF
    
    # Enable and start Vault service
    sudo systemctl daemon-reload
    sudo systemctl enable vault
    sudo systemctl start vault
    
    # Set environment variables
    echo 'export VAULT_ADDR="http://127.0.0.1:8200"' >> /home/vagrant/.bashrc
    export VAULT_ADDR="http://127.0.0.1:8200"
    
    # Wait for Vault to be ready
    echo "Waiting for Vault to be ready..."
    while ! curl -s http://127.0.0.1:8200/v1/sys/health >/dev/null 2>&1; do
      echo "Waiting for Vault service to start..."
      sleep 5
    done
    
    # Check if Vault is already initialized
    if vault status 2>/dev/null | grep -q "Initialized.*false"; then
      echo "Initializing Vault..."
      
      # Initialize Vault and capture output
      vault operator init -key-shares=5 -key-threshold=3 -format=json > /tmp/vault-init.json
      
      # Extract unseal keys and root token
      UNSEAL_KEY_1=$(cat /tmp/vault-init.json | jq -r '.unseal_keys_b64[0]')
      UNSEAL_KEY_2=$(cat /tmp/vault-init.json | jq -r '.unseal_keys_b64[1]')
      UNSEAL_KEY_3=$(cat /tmp/vault-init.json | jq -r '.unseal_keys_b64[2]')
      ROOT_TOKEN=$(cat /tmp/vault-init.json | jq -r '.root_token')
      
      # Save keys and token to files for easy access
      sudo cp /tmp/vault-init.json /etc/vault.d/vault-init.json
      sudo chown vault:vault /etc/vault.d/vault-init.json
      sudo chmod 600 /etc/vault.d/vault-init.json
      
      # Also save to vagrant home for easy access
      cp /tmp/vault-init.json /home/vagrant/vault-init.json
      chown vagrant:vagrant /home/vagrant/vault-init.json
      
      echo "Vault initialized successfully!"
      echo "Unseal keys and root token saved to /home/vagrant/vault-init.json"
      
      # Unseal Vault
      echo "Unsealing Vault..."
      vault operator unseal $UNSEAL_KEY_1
      vault operator unseal $UNSEAL_KEY_2
      vault operator unseal $UNSEAL_KEY_3
      
      echo "Vault unsealed successfully!"
      
      # Display important information
      echo "=========================================="
      echo "Vault Enterprise Setup Complete!"
      echo "=========================================="
      echo "Vault UI: http://localhost:8200"
      echo "Root Token: $ROOT_TOKEN"
      echo "Unseal keys saved in: /home/vagrant/vault-init.json"
      echo "=========================================="
      
      # Create a convenient script for re-unsealing if needed
      sudo tee /usr/local/bin/unseal-vault > /dev/null <<UNSEAL_SCRIPT
#!/bin/bash
export VAULT_ADDR="http://127.0.0.1:8200"
UNSEAL_KEY_1=\$(cat /home/vagrant/vault-init.json | jq -r '.unseal_keys_b64[0]')
UNSEAL_KEY_2=\$(cat /home/vagrant/vault-init.json | jq -r '.unseal_keys_b64[1]')
UNSEAL_KEY_3=\$(cat /home/vagrant/vault-init.json | jq -r '.unseal_keys_b64[2]')
vault operator unseal \$UNSEAL_KEY_1
vault operator unseal \$UNSEAL_KEY_2
vault operator unseal \$UNSEAL_KEY_3
echo "Vault unsealed!"
UNSEAL_SCRIPT
      
      sudo chmod +x /usr/local/bin/unseal-vault
      
    else
      echo "Vault is already initialized."
      
      # Check if Vault is sealed and unseal it
      if vault status 2>/dev/null | grep -q "Sealed.*true"; then
        echo "Vault is sealed. Attempting to unseal..."
        
        if [ -f "/home/vagrant/vault-init.json" ]; then
          UNSEAL_KEY_1=$(cat /home/vagrant/vault-init.json | jq -r '.unseal_keys_b64[0]')
          UNSEAL_KEY_2=$(cat /home/vagrant/vault-init.json | jq -r '.unseal_keys_b64[1]')
          UNSEAL_KEY_3=$(cat /home/vagrant/vault-init.json | jq -r '.unseal_keys_b64[2]')
          
          vault operator unseal $UNSEAL_KEY_1
          vault operator unseal $UNSEAL_KEY_2
          vault operator unseal $UNSEAL_KEY_3
          
          echo "Vault unsealed successfully!"
        else
          echo "Unseal keys not found. Please unseal manually."
        fi
      else
        echo "Vault is already unsealed."
      fi
    fi
    
    echo "VM setup completed successfully!"
  SHELL
end
