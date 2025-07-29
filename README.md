# Enterprise Jira Deployment on AWS: Bastion-Hardened, Docker-Orchestrated, and Ansible-Automated

🔐  traffic (default)


# Prepare Bastion Host

 ssh -i bastion-key.pem ec2-user@<Bastion_Public_IP>
sudo yum update -y
sudo hostnamectl set-hostname bastion-host


# Harden SSH Access

 sudo nano /etc/ssh/sshd_config
# Set:
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers ec2-user
sudo systemctl restart sshd



# 👤 Step 0.1: Create IAM UsersStep 0: Launch and Configure Bastion Host for Secure Access
# 🟩 Purpose
Acts as a secure jump server to access private EC2 instances


# 🛠 Instructions

# Launch EC2 Instance

AMI: Amazon Linux 2

Instance type: t2.micro

Key pair: bastion-key.pem

Network:

  - VPC: Same as the rest of the deployment

  - Subnet: Public Subnet

  - Auto-assign Public IP: Enabled

Security Group:

  - Inbound:

     - SSH (22): My IP (Restrict access)

 - Outbound:

# All with Console Access for SSH Provisioning
# 🟩 Purpose
Allow team members to manage their own access securely using IAM console credentials.
# 🛠 Instructions

  **Use Terraform to Automate IAM User Creation**

Create a Terraform file (e.g., iam_users.tf):
variable "users" {
  description = "List of team members to create"
  type        = list(string)
  default     = ["alice", "bob", "carol"]
}

resource "aws_iam_user" "team" {
  for_each = toset(var.users)
  name     = each.key
}

resource "aws_iam_user_login_profile" "team" {
  for_each                = aws_iam_user.team
  user                    = each.value.name
  password_reset_required = true
  password                = "TempPassw0rd@123"  # Set securely or generate
}

resource "aws_iam_group" "read_only" {
  name = "TeamReadOnlyGroup"
}

resource "aws_iam_group_policy_attachment" "readonly_policy" {
  group      = aws_iam_group.read_only.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}

resource "aws_iam_user_group_membership" "team" {
  for_each = aws_iam_user.team
  user     = each.value.name
  groups   = [aws_iam_group.read_only.name]
}

Run:
terraform init
terraform apply

# Distribute Console Login Info

Share the IAM user's:

Console login URL

Username & Temporary Password


# Team Member Uploads SSH Public Key to Admin (via S3 or email)


# Admin Adds SSH Key to Bastion Host

sudo useradd -m username
sudo mkdir -p /home/username/.ssh
sudo nano /home/username/.ssh/authorized_keys
# Paste the public key
sudo chown -R username:username /home/username/.ssh
sudo chmod 700 /home/username/.ssh
sudo chmod 600 /home/username/.ssh/authorized_keys

  **Test SSH Access**

 ssh -i ~/.ssh/team_member_key.pem username@<Bastion_Public_IP>

# 🧱 Step 1: Launch Application EC2 Instances (Ansible & Workers)
# 🛠 Configuration
Launch 3 instances:

ansible (Control Node)

worker-node1

worker-node2

AMI: Amazon Linux 2

Instance type: t2.micro

Key pair: ansible-key.pem

Subnet: Private Subnet

Auto-assign Public IP: Disabled

Security Group:

Inbound:

SSH (22): From Bastion SG

Outbound: All



# 🖥️ Step 2: Rename the EC2 Instances
For each node via Bastion:
ssh -i ansible-key.pem -J ec2-user@<Bastion_IP> ec2-user@<Private_IP>
sudo su -
echo "control" > /etc/hostname  # Or worker-node1, worker-node2
sudo reboot


# 👤 Step: Create Ansible User on All Nodes
sudo su -
useradd ansible
passwd ansible

# Enable password login
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
vi /etc/ssh/sshd_config  # Uncomment PermitRootLogin yes
sudo systemctl restart sshd

# Grant sudo
visudo
ansible ALL=(ALL) NOPASSWD:ALL


# 🔑 Step: SSH Keypair Setup (Control → Workers)
On Ansible control node:
sudo su - ansible
ssh-keygen -t rsa
chmod 700 ~/.ssh
ssh-copy-id ansible@worker-node1
ssh-copy-id ansible@worker-node2


# 🧰 Step: Install Ansible on Control Node
sudo amazon-linux-extras install ansible2 -y
ansible --version


# 🐳 Step: Install Docker on Worker Nodes
From control node:
ssh ansible@worker-node1
sudo amazon-linux-extras install docker -y
sudo service docker start
sudo systemctl enable docker
sudo docker run hello-world

Repeat for worker-node2

# 📋 Step: Update Ansible Inventory
sudo nano /etc/ansible/hosts

[webservers]
worker-node1
worker-node2


# 📦 Step: Create Docker Compose & Ansible Playbook
a. Docker Compose File
mkdir ~/jira-docker
nano ~/jira-docker/docker-compose.yml

Paste the Jira + PostgreSQL docker-compose content
b. Ansible Playbook
mkdir ~/ansible-playbooks
cd ~/ansible-playbooks
nano deploy_jira.yml

Paste playbook that installs Docker, copies the compose file, and runs docker-compose up

# 🚀 Step: Deploy Jira
cd ~/ansible-playbooks
ansible-playbook deploy_jira.yml


# 🔐 Step: SSH Config for Team Members
Edit ~/.ssh/config on each team member's laptop:
Host bastion
  HostName <Bastion_Public_IP>
  User ec2-user
  IdentityFile ~/.ssh/bastion-key.pem

Host control
  HostName <Private_IP_of_Control>
  User ansible
  IdentityFile ~/.ssh/ansible-key.pem
  ProxyJump bastion

Host worker1
  HostName <Private_IP_of_Worker1>
  User ansible
  IdentityFile ~/.ssh/ansible-key.pem
  ProxyJump bastion

Host worker2
  HostName <Private_IP_of_Worker2>
  User ansible
  IdentityFile ~/.ssh/ansible-key.pem
  ProxyJump bastion

Test with:
ssh control


✅ Final Outcome

✅ Secure, centralized Bastion Host access

✅ IAM Console-based team member provisioning (automated via Terraform)

✅ Modular deployment using Docker and Ansible

✅ No public IPs on application servers

https://docs.google.com/document/d/1BWTS9oJ4VojQMKNX8cBVSJBzQj73K5BngprIdJcRxCY/edit?usp=sharing
