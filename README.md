# 🧑‍💼Jenkins+Agent Installation for Debian Family. 

---

## 🔧 Part 1: Install Jenkins LTS on Debian 13 (Controller)

### Step 1: Install Java 21 (Required First!)
> ⚠️ **Critical**: Install Java **before** Jenkins to avoid service startup failures.

```bash
# Update package repositories
sudo apt update

# Install OpenJDK 21 and font dependencies
sudo apt install fontconfig openjdk-21-jre -y

# Verify Java installation
java -version
```

✅ Expected output:
```
openjdk 21.0.x 2025-xx-xx
OpenJDK Runtime Environment (build 21.0.x+xx-Debian-1)
OpenJDK 64-Bit Server VM (build 21.0.x+xx-Debian-1, mixed mode, sharing)
```

### Step 2: Add Jenkins LTS Repository

```bash
# Download and install the Jenkins GPG key
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

# Add the Jenkins LTS repository
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update package index
sudo apt update
```

### Step 3: Install Jenkins LTS

```bash
# Install Jenkins
sudo apt install jenkins -y
```

### Step 4: Start and Enable Jenkins Service

```bash
# Enable Jenkins to start on boot
sudo systemctl enable jenkins

# Start Jenkins service
sudo systemctl start jenkins

# Verify service status
sudo systemctl status jenkins
```

✅ You should see: `Active: active (running)`

### Step 5: Configure Firewall (if applicable)

```bash
# Allow Jenkins default port (8080) through UFW
sudo ufw allow 8080/tcp
sudo ufw reload

# OR for firewalld (if used):
# sudo firewall-cmd --permanent --add-port=8080/tcp
# sudo firewall-cmd --reload
```

### Step 6: Access Jenkins Web UI

Open your browser and navigate to:
```
http://<your-debian13-ip>:8080
```

---

## 🔐 Part 2: Post-Installation Setup Wizard

### Step 1: Unlock Jenkins

1. On the "Unlock Jenkins" page, retrieve the initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
2. Copy the alphanumeric password and paste it into the web UI
3. Click **Continue**

### Step 2: Install Plugins

- **Recommended**: Click **"Install suggested plugins"** for a standard setup
- **Advanced**: Choose **"Select plugins to install"** for custom plugin selection

⏳ Wait for installation to complete (may take 5-10 minutes)

### Step 3: Create First Admin User

1. Fill in the administrator user details:
   - Username, Password, Full Name, Email
2. Click **Save and Finish**
3. Click **Start using Jenkins** when ready

✅ Jenkins controller is now operational!

---

## 👷‍♂️ +Add a Node (Builder/Worker Agent-Linux via SSH)

### Prerequisites on Agent Machine
- Linux machine (Debian/Ubuntu/RHEL/CentOS)
- SSH server running and accessible from controller
- Dedicated user for Jenkins (recommended: `jenkins`)

### Log in to the Jenkins Controller as the jenkins user: Node (⚠️on Controller)
```
# Switch to jenkins user (if you're root/admin)
sudo su - jenkins
cd ~
```
Prepare SSH Key Pair 

```bash
# Generate SSH key for Jenkins agent communication
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# View the public key (you'll need this)
# On Controller: cat ~/.ssh/id_rsa.pub → copy output
cat ~/.ssh/id_rsa.pub
```
⚠️Copy public Key from Controller: cat ~/.ssh/id_rsa.pub → copy output, we need this to connect with the worker node.

### Step 1: Configure Agent Machine (⚠️on Agent Node)

```bash
#Java 21 installed
sudo apt install openjdk-21-jre
```
```

# On the AGENT machine, create jenkins user
sudo useradd -m -s /bin/bash jenkins

```
```
# Switch to jenkins user (if you're root/admin)
sudo su - jenkins
cd ~
```
Paste the Public key we copied from the controller using `cat ~/.ssh/id_rsa.pub → copy output` in `<PASTE_PUBLIC_KEY>`
```
mkdir -p ~/.ssh
echo "<PASTE_PUBLIC_KEY>" >> ~/.ssh/authorized_keys

# Set proper permissions
sudo chmod 700 /home/jenkins/.ssh
sudo chmod 600 /home/jenkins/.ssh/authorized_keys
sudo chown -R jenkins:jenkins /home/jenkins/.ssh

# Ensure Java is installed on agent
sudo apt update && sudo apt install openjdk-21-jre -y
java -version  # Verify
```
Verify Passwordless SSH
Test the connection from the Controller:
```
ssh jenkins@<NODE_HOST>
```
### Step 2: Add SSH Credential in Jenkins

1. In Jenkins UI: **Manage Jenkins** → **Credentials** → **System** → **Global credentials** → **Add Credentials**
2. Configure:
   - **Kind**: `SSH Username with private key`
   - **ID**: `jenkins-agent-ssh`
   - **Description**: `SSH Key for Linux Agent`
   - **Username**: `jenkins`
   - **Private Key**: Select **"Enter directly"** → **Add** → paste content of `~/.ssh/authorized_keys`
   - **Passphrase**: (leave empty if you didn't set one)
3. Click **Create**

### Step 3: Create New Agent Node in Jenkins

1. Navigate to: **Manage Jenkins** → **Nodes** → **New Node**
2. Configure the agent:

| Field | Value/Example |
|-------|--------------|
| **Node name** | `debian-agent-01` |
| **Type** | `Permanent Agent` → **Create** |
| **Description** | `Debian 13 Build Agent` |
| **Remote root directory** | `/home/jenkins` |
| **Labels** | `linux debian agent` (space-separated) |
| **Usage** | `Use this node as much as possible` |
| **Launch method** | `Launch agents via SSH` |
| **Host** | `<agent-machine-IP-or-hostname>` |
| **Credentials** | Select `jenkins-agent-ssh` |
| **Host Key Verification Strategy** | `Non verifying Verification Strategy` (for testing) or `Known hosts file Verification Strategy` (production) |
| **Availability** | `Keep this agent online as much as possible` |

3. Click **Save**

### Step 4: Launch and Verify Agent

1. In **Nodes** list, click on your new agent (`debian-agent-01`)
2. Click **Relaunch agent** if status shows "Offline"
3. Click **Log** to view connection logs
4. ✅ Success message: `Agent successfully connected and online`

### Step 5: Test Agent Assignment

1. Create a new **Freestyle Project**
2. In **General** section: Check **"Restrict where this project can be run"**
3. Enter label: `linux` (or your custom label)
4. Add a build step: **Execute shell** with command:
   ```bash
   echo "Running on agent: $(hostname)"
   java -version
   ```
5. Save and **Build Now**
6. Check **Console Output** to confirm it ran on the agent

---

## 🔧 Optional: Change Jenkins Port (if 8080 is occupied)

```bash
# Edit Jenkins systemd override
sudo systemctl edit jenkins

# Add these lines:
[Service]
Environment="JENKINS_PORT=8081"

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

---

## 🛡️ Security Best Practices

1. **Change default port** from 8080 in production
2. **Enable HTTPS** using a reverse proxy (Nginx/Apache) with Let's Encrypt
3. **Restrict SSH access** to controller IP only on agent machines
4. **Use known_hosts verification** for SSH instead of "Non verifying"
5. **Regularly update** Jenkins and plugins via **Manage Jenkins** → **Plugins**
6. **Backup** `/var/lib/jenkins` regularly

---

## 📦 Useful Commands Reference

```bash
# Jenkins service management
sudo systemctl status jenkins
sudo systemctl restart jenkins
sudo systemctl stop jenkins
sudo journalctl -u jenkins -f  # View live logs

# Jenkins configuration files
/var/lib/jenkins/          # JENKINS_HOME (jobs, config, plugins)
/etc/default/jenkins       # Environment variables (Debian)
/lib/systemd/system/jenkins.service  # Systemd unit file

# Agent troubleshooting
# On controller: Check agent logs via Jenkins UI → Node → Log
# On agent: Check SSH auth logs
sudo tail -f /var/log/auth.log  # Debian/Ubuntu
```

---

## 🔄 Updating Jenkins LTS

```bash
# Update via apt (recommended)
sudo apt update
sudo apt upgrade jenkins

# Restart service if needed
sudo systemctl restart jenkins
```

> 💡 Jenkins LTS releases are updated approximately every 12 weeks. Always review release notes before upgrading in production.

---

✅ **You now have a fully configured Jenkins LTS controller on Debian 13 with a remote Linux agent ready for distributed builds!** 
