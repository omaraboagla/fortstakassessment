**For part 2, I chose WSL2 as my local Linux VM, used WSL2 Ubuntu 24.04 as the target Linux "VM" on my Windows 11 machine as it functions like a real remote Linux Server with its own IP and runs a full Ubuntu OS

**Set up WSL2 VM, in windows terminal:

wsl --install -d Ubuntu

returned "A distribution with the supplied name already exists." which means WSL was already installed.

**Created a devops user in WSL2. Creating devops as the primary Ansible target user, with sudo privileges for Docker installation:

in WSL2 Ubuntu terminal:
sudo adduser devops
sudo usermod -aG sudo devops

**Enabled passwordless sudo for devops to ensure Ansible can sudo without being blocked by password prompts:

sudo visudo (to access the file)

**at the end of the file I added:

devops ALL=(ALL) NOPASSWD:ALL

**I Checked and found WSL2 IP Address:

ip addr | grep inet
returned: inet 192.168.118.152/20 brd 192.168.127.255 scope global eth0
so used 192.168.118.152 as the IP Address for Ansible to target.


**I Installed Ansible in the Control Machine (WSL terminal), Ansible was installed on the WSL terminal (same as control machine) — which can SSH into the target VM (also WSL2 in this case) by:

sudo apt update
sudo apt install ansible -y

**Had to install sshpass for password based SSH in Ansible:

sudo apt install sshpass -y

**I created project folder on Ansible control machine:

mkdir ~/ansible-docker-vm
cd ~ansible-docker-vm

**Created inventory.ini:
nano inventory.ini

contains:
[dev]
192.168.118.152 ansible_user=devops ansible_ssh_pass=YOUR_PASSWORD ansible_python_interpreter=/usr/bin/python3

**Faced error: 

sshpass does not support host key check

**Manually SSH’d from control machine to the target once, typed yes to trust the host and added fingerprint to ~/.ssh/known_hosts so that Ansible could SSH non-interactively.:

ssh devops@192.168.118.152

**Created Ansible Playbook install-docker.yml:

nano install-docker.yml

**install-docker.yml comtains:

---
- name: Setup Docker on remote Linux VM
  hosts: dev
  become: true
  tasks:
    - name: Install required packages
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        update_cache: yes

    - name: Add Docker GPG key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Add Docker APT repo
      apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        state: present

    - name: Install Docker
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
        update_cache: yes
        state: latest

    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: true

    - name: Add user to docker group
      user:
        name: devops
        groups: docker
        append: yes

**Ran the playbook:

ansible-playbook -i inventory.ini install-docker.yml

**When playbook run, I saw:

Gathering facts
Installed all Docker dependencies
Added official Docker repo and key
Installed Docker
Added devops user to the docker group
Started and enabled Docker service


**To verify that part 2 has been completed successfully and that Ansible was used to configure the machine and install Docker, Ansible has run from my local machine against the VM:

SSH'd into VM again:

ssh devops@192.168.118.152

Checked Docker:

docker --version
returns: Docker version 28.3.2, build 578ccf6

docker ps
returns:
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES