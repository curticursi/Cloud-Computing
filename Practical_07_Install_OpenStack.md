# Practical 07 — Install OpenStack

-----

## 📌 Objective

Set up a local OpenStack environment for practicing private cloud infrastructure deployment and management, with specific troubleshooting for Ubuntu derivatives like Pop\!\_OS.

-----

## 🧠 Conceptual Background (Know-How)

### What is OpenStack?

**OpenStack** is an open-source cloud operating system that controls large pools of compute, storage, and networking resources throughout a data center. It is the most widely used platform for building private clouds — essentially giving you your own AWS-like environment on your own hardware.

### OpenStack vs AWS

```text
┌──────────────────────────────────────────────────────┐
│                  OpenStack ↔ AWS Service Mapping     │
├─────────────────────┬────────────────────────────────┤
│    OpenStack        │             AWS                │
├─────────────────────┼────────────────────────────────┤
│ Nova                │ EC2 (Compute)                  │
│ Neutron             │ VPC (Networking)               │
│ Cinder              │ EBS (Block Storage)            │
│ Swift               │ S3 (Object Storage)            │
│ Glance              │ AMI / EC2 Image Service        │
│ Keystone            │ IAM (Identity)                 │
│ Horizon             │ AWS Management Console         │
│ Heat                │ CloudFormation                 │
│ Ceilometer          │ CloudWatch                     │
│ Barbican            │ Secrets Manager                │
└─────────────────────┴────────────────────────────────┘
```

### OpenStack Architecture

```text
                        ┌──────────────────┐
                        │     Horizon      │  ← Web Dashboard (UI)
                        │  (Dashboard)     │
                        └────────┬─────────┘
                                 │
                        ┌────────▼─────────┐
                        │    Keystone      │  ← Identity & Auth
                        │  (Identity)      │
                        └────────┬─────────┘
                                 │
           ┌─────────────────────┼─────────────────────┐
           │                     │                     │
   ┌───────▼──────┐    ┌─────────▼────┐    ┌───────────▼──┐
   │    Nova      │    │   Neutron    │    │    Glance    │
   │  (Compute)   │    │ (Networking) │    │   (Images)   │
   └───────┬──────┘    └─────────┬────┘    └──────────────┘
           │                     │
   ┌───────▼──────┐    ┌─────────▼────┐    ┌──────────────┐
   │   Cinder     │    │    Swift     │    │     Heat     │
   │   (Block     │    │   (Object    │    │(Orchestration│
   │   Storage)   │    │   Storage)   │    │              │
   └──────────────┘    └──────────────┘    └──────────────┘
```

### OpenStack Release Names (Reference)

| Release | Branch | Status |
| :--- | :--- | :--- |
| **Dalmatian** | `stable/2024.2` | Current stable ✅ |
| **Caracal** | `stable/2024.1` | May be removed from upstream |
| **Bobcat** | `stable/2023.2` | Older stable |
| **Antelope** | `stable/2023.1` | Older stable |
| **master** | `master` | Development (unstable) |

> ⚠️ **Branch note:** DevStack branch names change as new releases land. Always check available branches before cloning.

### Installation Options

1.  **DevStack (Recommended for learning):** All-in-one script-based installation. Runs on a single VM or bare metal server. Resets on reboot (development only).
2.  **MicroStack:** Ubuntu Snap — quickest. Good for quick demos.
3.  **Packstack:** RHEL/CentOS Puppet-based installer.
4.  **Kolla-Ansible:** Docker-container based deployment. Production-grade, multi-node.
5.  **Manual Installation:** Component by component. Best for deep understanding.

-----

## 🛠️ Installation Guide

### System Requirements

**Minimum for DevStack (All-in-One):**

  * **CPU:** 4 cores (8 recommended)
  * **RAM:** 8 GB minimum (16 GB recommended)
  * **Disk:** 50 GB free
  * **OS:** Ubuntu 22.04 LTS / 24.04 LTS (or derivatives like Pop\!\_OS)
  * **Network:** Internet access for downloading packages

**Hypervisor Check (Nested Virtualization):**

```bash
# Check if KVM is available (Output > 0 means KVM is supported)
egrep -c '(vmx|svm)' /proc/cpuinfo

# Enable nested virtualization (Intel CPU)
sudo modprobe -r kvm_intel
sudo modprobe kvm_intel nested=1
echo "options kvm-intel nested=1" | sudo tee /etc/modprobe.d/kvm-intel.conf
```

-----

### Method 1 — MicroStack (Quickest, Ubuntu / Snap only)

*Note: Requires `snapd` to be installed on Pop\!\_OS (`sudo apt install snapd`).*

```bash
# Install MicroStack via snap
sudo snap install microstack --beta

# Initialize (all-in-one)
sudo microstack init --auto --control

# Get admin password
sudo snap get microstack config.credentials.keystone-password

#By default, MicroStack configures a virtual router and assigns the dashboard to a specific local IP address. Usually, this is:
http://10.20.20.1
```
<img width="1037" height="731" alt="image" src="https://github.com/user-attachments/assets/e8eadc0f-3b66-43fa-814b-dc6c1f426a6f" />
<img width="566" height="41" alt="image" src="https://github.com/user-attachments/assets/65aee02a-355d-4c44-98eb-329d4fdf082b" />

-----

### Method 2 — DevStack (Recommended for Learning)

#### 1\. Prepare the System

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git python3-pip
```

#### 2\. Create the Stack User (Required)

DevStack requires a dedicated user with passwordless sudo access.

```bash
# Create dedicated user
sudo useradd -s /bin/bash -d /opt/stack -m stack

# Give passwordless sudo
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack

# Switch to stack user
sudo su - stack
```

#### 3\. Clone DevStack

Always check for the latest stable branch before cloning.

```bash
# Clone the repository
git clone https://opendev.org/openstack/devstack
cd devstack

# List available stable branches
git branch -r | grep stable

# Checkout the latest stable branch (e.g., master or 2024.2)
git checkout master
```

#### 4\. 🛑 FIX: The "Bulletproof" Distro Override for Pop\!\_OS

DevStack rigidly checks for "Ubuntu" or "Debian". If you are on an Ubuntu derivative like Pop\!\_OS, the script will crash because it fails to map the package manager correctly. You must hardcode the OS variables.

1.  Open the internal functions file:
    ```bash
    nano /opt/stack/devstack/functions-common
    ```
2.  Press `Ctrl+W` to search for `typeset -xr os_VENDOR`. This is where DevStack locks in the variables.
3.  Immediately **above** that block of `typeset` commands, add this override:
    ```bash
    # --- FORCE UBUNTU NOBLE OVERRIDE ---
    os_VENDOR="Ubuntu"
    os_PACKAGE="deb"
    os_CODENAME="noble"
    # -----------------------------------
    ```
4.  Save (`Ctrl+O`, `Enter`) and Exit (`Ctrl+X`).
5.  Ensure the logging directory exists so the script doesn't crash on output:
    ```bash
    sudo mkdir -p /opt/stack/logs
    sudo chown -R stack:stack /opt/stack/logs
    ```

#### 5\. Find Your Network Interface Name

```bash
ip addr show | grep -E "^[0-9]+:" | awk '{print $2}' | sed 's/://'
# Note down your primary interface (e.g., eth0, ens33, enp0s3)
```

\<img width="824" height="543" alt="image" src="[https://github.com/user-attachments/assets/6315ec3f-34ce-4dac-bd89-916eba240d2b](https://github.com/user-attachments/assets/6315ec3f-34ce-4dac-bd89-916eba240d2b)" /\>

#### 6\. Create the Configuration File (`local.conf`)

Create the configuration file inside the `/opt/stack/devstack` directory:

```bash
cat > local.conf <<'EOF'
[[local|localrc]]
# ── Admin Passwords ────────────────────────────────
ADMIN_PASSWORD=secret123
DATABASE_PASSWORD=secret123
RABBIT_PASSWORD=secret123
SERVICE_PASSWORD=secret123

# ── Network Settings ───────────────────────────────
# HOST_IP is auto-detected; set explicitly if detection fails:
# HOST_IP=192.168.x.x

FLOATING_RANGE=192.168.1.224/27
FIXED_RANGE=10.11.12.0/24
FIXED_NETWORK_SIZE=256
FLAT_INTERFACE=ens33    # <-- REPLACE WITH YOUR INTERFACE NAME FROM STEP 5

# ── Log Settings ───────────────────────────────────
LOGFILE=/opt/stack/logs/stack.sh.log
VERBOSE=True
LOG_COLOR=True

# ── Image Settings ─────────────────────────────────
# Cirros is a tiny test image (~12 MB) — perfect for DevStack
IMAGE_URLS="http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img"
EOF
```

#### 7\. Run the DevStack Installer

```bash
./stack.sh
```

> ⏳ This takes **20–45 minutes**. Watch progress in a second terminal:
>
> ```bash
> tail -f /opt/stack/logs/stack.sh.log
> ```

**When complete, you will see:**

```text
Horizon is now available at http://<HOST_IP>/dashboard
Username: admin  |  Password: secret123
```

-----
<img width="1919" height="910" alt="image" src="https://github.com/user-attachments/assets/bbaa309e-8c7e-4a16-a813-48d736226e4c" />

## 🔍 Post-Installation Verification

```bash
# Source admin credentials to use OpenStack CLI
source /opt/stack/openrc admin admin

# Verify all services registered
openstack service list

# Check compute nodes
openstack compute service list

# Check network agents
openstack network agent list

# List images (Cirros should be pre-loaded)
openstack image list
```

**Expected service list output:**

```text
+----+----------+----------+
| ID | Name     | Type     |
+----+----------+----------+
|  1 | keystone | identity |
|  2 | glance   | image    |
|  3 | nova     | compute  |
|  4 | neutron  | network  |
|  5 | cinder   | volume   |
+----+----------+----------+
```
<img width="1919" height="910" alt="image" src="https://github.com/user-attachments/assets/ba7b5bb1-3996-4074-b4c5-5a00dcf09ddf" />

-----

## 🚨 Common Issues and Fixes

  * **Issue:** `die 509 'Unable to determine DISTRO'` or `Support for noble is incomplete`
      * **Fix:** You missed Step 4. You must apply the hardcoded Ubuntu override in `functions-common` before running the installer.
  * **Issue:** `/opt/stack/logs/error.log: No such file or directory`
      * **Fix:** Manually create the logs folder and assign ownership to the stack user: `sudo mkdir -p /opt/stack/logs && sudo chown -R stack:stack /opt/stack/logs`.
  * **Issue:** `stack.sh` fails partway through
      * **Fix:** DevStack is mostly idempotent. Resolve the printed error and re-run `./stack.sh`.
  * **Issue:** Services not running after system reboot
      * **Fix:** DevStack is a development environment and *does not persist* its runtime state across reboots. Run `cd /opt/stack/devstack && ./rejoin-stack.sh`.
  * **Issue:** "Connection refused" on the Horizon dashboard
      * **Fix:** Check service status with `sudo systemctl status devstack@*`. Restart Apache if necessary: `sudo systemctl restart apache2`.

-----

## ✅ Learning Outcomes

  - [ ] Explain what OpenStack is and how it compares to AWS.
  - [ ] Map OpenStack components to AWS equivalents (Nova=EC2, Glance=AMI, etc.).
  - [ ] Navigate Linux distribution checks in bash scripts and bypass hardcoded limits.
  - [ ] Install OpenStack using DevStack on an Ubuntu-derivative OS.
  - [ ] Create and configure the `local.conf` deployment file.
  - [ ] Source credentials and use the OpenStack CLI to verify running services.
