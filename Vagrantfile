# -*- mode: ruby -*-
# vi: set ft=ruby :

# Cluster configuration
CLUSTER_NODES = [
  { name: "vault-1", ip: "192.168.56.10", node_id: "vault-node-1" },
  { name: "vault-2", ip: "192.168.56.11", node_id: "vault-node-2" },
  { name: "vault-3", ip: "192.168.56.12", node_id: "vault-node-3" }
]

Vagrant.configure("2") do |config|
  # Use an ARM64-compatible box for VMware Fusion
  config.vm.box = "gyptazy/ubuntu22.04-arm64"
  
  # Configure each node
  CLUSTER_NODES.each_with_index do |node, index|
    config.vm.define node[:name] do |vm_config|
      # VM configuration
      vm_config.vm.hostname = node[:name]
      vm_config.vm.network "private_network", ip: node[:ip]
      
      # Port forwarding for API access (only for first node)
      if index == 0
        vm_config.vm.network "forwarded_port", guest: 8200, host: 8200
      end
      
      # Configure VMware Fusion provider
      vm_config.vm.provider "vmware_fusion" do |vmware|
        vmware.memory = 2048
        vmware.cpus = 2
        vmware.gui = false  # Set to true if you want GUI
        vmware.vmx["displayName"] = "#{node[:name]}-Vault-Cluster"
        vmware.vmx["ethernet0.virtualDev"] = "vmxnet3"
      end
      
      # Shared folders
      vm_config.vm.synced_folder ".", "/vagrant"
      
      # Copy license file
      vm_config.vm.provision "file", source: "./vault.hclic", destination: "/tmp/vault.hclic"
      
      # Provisioning script
      vm_config.vm.provision "shell", inline: <<-SHELL
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
        echo "Installing HashiCorp Vault Enterprise on #{node[:name]}..."
        
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
        
        # Move license file to proper location
        sudo cp /tmp/vault.hclic /etc/vault.d/vault.hclic
        sudo chown vault:vault /etc/vault.d/vault.hclic
        sudo chmod 640 /etc/vault.d/vault.hclic
        
        # Create Vault configuration
        sudo tee /etc/vault.d/vault.hcl > /dev/null <<EOF
storage "raft" {
  path = "/opt/vault/data"
  node_id = "#{node[:node_id]}"
  
  # Autopilot configuration
  autopilot {
    cleanup_dead_servers = true
    last_contact_threshold = "10s"
    max_trailing_logs = 1000
    server_stabilization_time = "10s"
    disable_upgrade_migration = false
  }
  
  retry_join {
    leader_api_addr = "http://192.168.56.10:8200"
  }
  retry_join {
    leader_api_addr = "http://192.168.56.11:8200"
  }
  retry_join {
    leader_api_addr = "http://192.168.56.12:8200"
  }
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

api_addr = "http://#{node[:ip]}:8200"
cluster_addr = "https://#{node[:ip]}:8201"
ui = true
disable_mlock = true

# License configuration
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
        echo 'export VAULT_ADDR="http://#{node[:ip]}:8200"' >> /home/vagrant/.bashrc
        export VAULT_ADDR="http://#{node[:ip]}:8200"
        
        echo "#{node[:name]} setup completed!"
      SHELL
      
      # Initialize cluster only on the first node
      if index == 0
        vm_config.vm.provision "shell", inline: <<-SHELL
          echo "Initializing Vault cluster on #{node[:name]}..."
          
          # Wait for Vault to be ready
          export VAULT_ADDR="http://#{node[:ip]}:8200"
          while ! curl -s http://#{node[:ip]}:8200/v1/sys/health >/dev/null 2>&1; do
            echo "Waiting for Vault service to start..."
            sleep 5
          done
          
          # Wait a bit more for cluster to stabilize
          sleep 10
          
          # Check if Vault is already initialized
          if vault status 2>/dev/null | grep -q "Initialized.*false"; then
            echo "Initializing Vault cluster..."
            
            # Initialize Vault
            vault operator init -key-shares=5 -key-threshold=3 -format=json > /tmp/vault-init.json
            
            # Extract unseal keys and root token
            UNSEAL_KEY_1=$(cat /tmp/vault-init.json | jq -r '.unseal_keys_b64[0]')
            UNSEAL_KEY_2=$(cat /tmp/vault-init.json | jq -r '.unseal_keys_b64[1]')
            UNSEAL_KEY_3=$(cat /tmp/vault-init.json | jq -r '.unseal_keys_b64[2]')
            ROOT_TOKEN=$(cat /tmp/vault-init.json | jq -r '.root_token')
            
            # Save initialization data
            sudo cp /tmp/vault-init.json /etc/vault.d/vault-init.json
            sudo chown vault:vault /etc/vault.d/vault-init.json
            sudo chmod 600 /etc/vault.d/vault-init.json
            
            # Copy to vagrant home and shared folder
            cp /tmp/vault-init.json /home/vagrant/vault-init.json
            cp /tmp/vault-init.json /vagrant/vault-init.json
            chown vagrant:vagrant /home/vagrant/vault-init.json
            
            # Unseal the leader node
            echo "Unsealing leader node..."
            vault operator unseal $UNSEAL_KEY_1
            vault operator unseal $UNSEAL_KEY_2
            vault operator unseal $UNSEAL_KEY_3
            
            echo "=========================================="
            echo "Vault Cluster Initialized!"
            echo "=========================================="
            echo "Leader Node: #{node[:name]} (#{node[:ip]}:8200)"
            echo "Root Token: $ROOT_TOKEN"
            echo "Unseal keys saved in: /vagrant/vault-init.json"
            echo "=========================================="
            
            # Create unseal script for all nodes
            sudo tee /usr/local/bin/unseal-vault > /dev/null <<UNSEAL_SCRIPT
#!/bin/bash
if [ -f "/vagrant/vault-init.json" ]; then
  UNSEAL_KEY_1=\$(cat /vagrant/vault-init.json | jq -r '.unseal_keys_b64[0]')
  UNSEAL_KEY_2=\$(cat /vagrant/vault-init.json | jq -r '.unseal_keys_b64[1]')
  UNSEAL_KEY_3=\$(cat /vagrant/vault-init.json | jq -r '.unseal_keys_b64[2]')
  
  export VAULT_ADDR="http://#{node[:ip]}:8200"
  vault operator unseal \$UNSEAL_KEY_1
  vault operator unseal \$UNSEAL_KEY_2
  vault operator unseal \$UNSEAL_KEY_3
  echo "Vault unsealed!"
else
  echo "Unseal keys not found at /vagrant/vault-init.json"
fi
UNSEAL_SCRIPT
            
            sudo chmod +x /usr/local/bin/unseal-vault
            
          else
            echo "Vault is already initialized."
          fi
        SHELL
      else
        # For follower nodes, wait and then unseal
        vm_config.vm.provision "shell", inline: <<-SHELL
          echo "Setting up follower node #{node[:name]}..."
          
          # Wait for leader to be initialized
          sleep 30
          
          export VAULT_ADDR="http://#{node[:ip]}:8200"
          
          # Wait for Vault to be ready
          while ! curl -s http://#{node[:ip]}:8200/v1/sys/health >/dev/null 2>&1; do
            echo "Waiting for Vault service to start..."
            sleep 5
          done
          
          # Unseal this node if keys are available
          if [ -f "/vagrant/vault-init.json" ]; then
            echo "Unsealing follower node #{node[:name]}..."
            
            UNSEAL_KEY_1=$(cat /vagrant/vault-init.json | jq -r '.unseal_keys_b64[0]')
            UNSEAL_KEY_2=$(cat /vagrant/vault-init.json | jq -r '.unseal_keys_b64[1]')
            UNSEAL_KEY_3=$(cat /vagrant/vault-init.json | jq -r '.unseal_keys_b64[2]')
            
            vault operator unseal $UNSEAL_KEY_1
            vault operator unseal $UNSEAL_KEY_2
            vault operator unseal $UNSEAL_KEY_3
            
            echo "#{node[:name]} unsealed and joined cluster!"
          else
            echo "Waiting for cluster initialization..."
          fi
          
          # Create unseal script
          sudo tee /usr/local/bin/unseal-vault > /dev/null <<UNSEAL_SCRIPT
#!/bin/bash
if [ -f "/vagrant/vault-init.json" ]; then
  UNSEAL_KEY_1=\$(cat /vagrant/vault-init.json | jq -r '.unseal_keys_b64[0]')
  UNSEAL_KEY_2=\$(cat /vagrant/vault-init.json | jq -r '.unseal_keys_b64[1]')
  UNSEAL_KEY_3=\$(cat /vagrant/vault-init.json | jq -r '.unseal_keys_b64[2]')
  
  export VAULT_ADDR="http://#{node[:ip]}:8200"
  vault operator unseal \$UNSEAL_KEY_1
  vault operator unseal \$UNSEAL_KEY_2
  vault operator unseal \$UNSEAL_KEY_3
  echo "Vault unsealed!"
else
  echo "Unseal keys not found at /vagrant/vault-init.json"
fi
UNSEAL_SCRIPT
          
          sudo chmod +x /usr/local/bin/unseal-vault
        SHELL
      end
    end
  end
end
