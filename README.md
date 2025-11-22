Ansible Lab Setup: Kali Master Managing ParrotOS & CentOS NodesThis repository documents the process of setting up an Ansible environment using Kali Linux as the master (control node) to manage diverse Linux distributions. It starts with a Parrot OS node and expands to include a CentOS node, demonstrating Ansible's distribution-agnostic capabilities.Lab Environment OverviewRoleOSIP AddressNotesMaster NodeKali Linux192.168.0.150Where Ansible is installed and run.Managed Node 1Parrot OS192.168.0.122Debian-based node.Managed Node 2CentOS192.168.0.XXXRedHat-based node (Replace XXX with actual IP).Phase 1: Initial Setup (Kali Master & ParrotOS Node)Step 1: Install Ansible on Kali Linux (Master)On your Kali Linux machine (192.168.0.150):Bash# Update package lists
sudo apt update

# Install Ansible
sudo apt install ansible -y

# Verify installation
ansible --version
Step 2: Configure SSH AccessAnsible uses SSH to communicate with nodes. We need to establish passwordless SSH access from the master to the node.On Kali Linux (Master):Generate an SSH key pair (if you don't have one already):Bashssh-keygen -t rsa -b 4096
# Press Enter to accept default location and empty passphrase for passwordless access
Copy the public key to the Parrot OS node:Note: Replace user below with your actual username on Parrot OS.Bashssh-copy-id user@192.168.0.122
You will be prompted for the Parrot OS user's password once.Test the connection. If successful, you will log in without a password:Bashssh user@192.168.0.122
Step 3: Configure Initial Ansible InventoryCreate a directory for your project and define your managed nodes.Bashmkdir -p ~/ansible-project
cd ~/ansible-project
nano inventory.ini
Add the following content to inventory.ini (adjusting the ansible_user):Ini, TOML[parrot_nodes]
parrot-node1 ansible_host=192.168.0.122 ansible_user=user

[parrot_nodes:vars]
ansible_python_interpreter=/usr/bin/python3
Step 4: Test Connection and PrerequisitesRun an Ansible ping test:Bashansible -i inventory.ini all -m ping
You should see a "SUCCESS" => "pong" response.Ensure Python 3 is on Parrot OS (192.168.0.122):Ansible requires Python on the managed node.Bash# Check version
python3 --version

# If missing, install it:
sudo apt update && sudo apt install python3 -y
Step 5: Run Your First Ansible Commands (Ad-Hoc)Try basic commands from the Kali master:Bash# Check uptime
ansible -i inventory.ini all -m command -a "uptime"

# Get disk space
ansible -i inventory.ini all -m command -a "df -h"
Phase 2: Scaling Up - Adding a CentOS NodeWe will now add a CentOS machine to act as a second node. Ansible can manage different Linux flavors simultaneously.Step 1: Set Up SSH Access to CentOS NodeOn your Kali Linux (Master):Replace centos_username and 192.168.0.XXX with your CentOS node's details.Bashssh-copy-id centos_username@192.168.0.XXX

# Test connection
ssh centos_username@192.168.0.XXX
⚠️ Troubleshooting CentOS SSH IssuesIf you copied the ID but are still asked for a password when connecting to CentOS, it is usually a permissions issue on the CentOS node.On the CentOS Node (192.168.0.XXX), run these commands to fix permissions:Bash# Set correct permissions for .ssh directory (owner read/write/execute only)
chmod 700 ~/.ssh

# Set correct permissions for authorized_keys (owner read/write only)
chmod 600 ~/.ssh/authorized_keys

# Ensure your home directory isn't too open
chmod 755 ~

# Verify ownership (everything should be owned by your user)
ls -la ~/.ssh/
Try your passwordless SSH entry again from Kali; it should work now.Step 2: Update Inventory for Multi-OS SupportWe will update the inventory file to group nodes by their distribution family (Debian vs. RedHat).Edit your inventory: nano ~/ansible-project/inventory.iniIni, TOML# Parrot OS Node
[parrot_nodes]
parrot-node1 ansible_host=192.168.0.122 ansible_user=parrot_user

# CentOS Node (Update IP and User)
[centos_nodes]
centos-node1 ansible_host=192.168.0.XXX ansible_user=centos_username

# --- Groups based on OS Family ---

# Group all Debian-based systems
[debian_based]
parrot-node1

# Group all RedHat-based systems
[redhat_based]
centos-node1

# Group all nodes together
[all_nodes:children]
parrot_nodes
centos_nodes

# --- Variables ---
[all_nodes:vars]
ansible_python_interpreter=/usr/bin/python3
Step 3: Test Connections to New GroupsUse the -K flag to prompt for the sudo password required for privilege escalation.Bash# Test all nodes together
ansible -i inventory.ini all_nodes -m ping -K

# Test specific groups
ansible -i inventory.ini debian_based -m ping -K
ansible -i inventory.ini redhat_based -m ping -K
Pro Tip: If your nodes have different sudo passwords, you can define them in the inventory file inline:centos-node1 ansible_host=... ansible_become_pass=your_centos_passwordPhase 3: Running Multi-Distribution PlaybooksSince Parrot uses apt and CentOS uses yum/dnf, we need distribution-aware playbooks.Create a Multi-Distribution PlaybookCreate multi-node-playbook.yml:YAML---
# --- Tasks for Debian Family (ParrotOS) ---
- name: Configure Debian-based systems
  hosts: debian_based
  become: yes
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install basic tools (apt)
      apt:
        name: [vim, curl, wget, htop]
        state: present

# --- Tasks for RedHat Family (CentOS) ---
- name: Configure RedHat-based systems
  hosts: redhat_based
  become: yes
  tasks:
    - name: Update yum cache
      yum:
        update_cache: yes

    - name: Install basic tools (yum)
      yum:
        name: [vim, curl, wget, htop]
        state: present

# --- Common Tasks for All Nodes ---
- name: Common tasks for all nodes
  hosts: all_nodes
  become: yes
  tasks:
    - name: Create a test directory
      file:
        path: /tmp/ansible-test
        state: directory
        mode: '0755'

    - name: Create a managed file info tag
      copy:
        content: "Managed by Ansible from Kali Linux\n"
        dest: /tmp/ansible-test/info.txt
        mode: '0644'
Run the PlaybookBashansible-playbook -i inventory.ini multi-node-playbook.yml -K
This playbook will run apt tasks only on ParrotOS, yum tasks only on CentOS, and the file creation tasks on both.
