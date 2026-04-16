# Practical 08 — Launch Your First Instance on OpenStack

---

## 📌 Objective

Create a virtual machine (VM) using OpenStack by creating a project, assigning roles to users, uploading a cloud image to Glance, defining a flavor, and launching an instance via the Horizon dashboard or CLI.

---

## 🧠 Conceptual Background (Know-How)

### OpenStack VM Launch Flow

```
 Before launching a VM, you need:

 1. Project (Tenant)  → Logical container for resources
         │
 2. User + Role       → Who can access what in the project
         │
 3. Image (Glance)    → OS disk image (like AWS AMI)
         │
 4. Flavor            → Hardware spec (CPU, RAM, disk)
         │
 5. Network (Neutron) → Which network the VM connects to
         │
 6. Security Group    → Firewall rules for the VM
         │
 7. Key Pair          → SSH key for authentication
         │
         ▼
 8. Launch Instance → Nova schedules on hypervisor → VM running
```

### OpenStack Projects and Users

**Projects** (also called tenants) group and isolate resources. Each user can belong to multiple projects with different roles.

| Role | Permissions |
|---|---|
| admin | Full access across all projects |
| member | Manage own resources within a project |
| reader | Read-only access within a project |

### OpenStack Image Service (Glance)

| Format | Description |
|---|---|
| qcow2 | QEMU Copy-On-Write — most common for OpenStack |
| raw | Raw disk image |
| vmdk | VMware format |
| iso | CD/DVD ISO format |

### OpenStack Flavors

```
Flavor: m1.small
  ├── vCPUs: 1
  ├── RAM:   2 GB
  └── Disk:  20 GB root disk

Equivalent to EC2 instance types (t2.micro, m5.large, etc.)
```

---

## 🛠️ Step-by-Step Guide

### Prerequisites
- OpenStack running (from Practical 07) ✅ — confirmed by Horizon Instances page
- Logged in to Horizon as **admin**
- CLI access: `source /opt/stack/openrc admin admin`

> 💡 **You're already past the hardest part.** Seeing the Horizon dashboard with the Instances page means DevStack installed successfully. This practical picks up from there.

---

### Step A — Create a Project and Assign Roles to Users

> ⚠️ **Important:** Always work inside a dedicated project, not the default `admin` project. This keeps lab resources isolated and reflects real-world multi-tenant usage.

#### Via Horizon Dashboard

**Create a New Project:**
```
Horizon → Identity → Projects → Create Project

  Name:        CloudLab-Project
  Description: Student cloud computing lab environment
  Enabled:     ✅

Quotas tab (optional):
  Instances:    10
  VCPUs:        20
  RAM (MB):     51200
  Floating IPs: 5

→ Create Project
```

<img width="1919" height="910" alt="image" src="https://github.com/user-attachments/assets/16495e08-7020-4f57-a4e8-4113d64ce04d" />

**Create a New User:**
```
Horizon → Identity → Users → Create User

  Username:        student01
  Email:           student01@lab.local
  Password:        Student@123
  Primary Project: CloudLab-Project
  Role:            member

<img width="1917" height="895" alt="image" src="https://github.com/user-attachments/assets/dd68b14e-025a-478c-89b0-3c60fa4adc58" />

→ Create User
```

**Assign Role to User in Project:**
```
Horizon → Identity → Projects → CloudLab-Project →
  Edit → Project Members tab →
  Add student01 → Role: member
→ Save
```

#### Via CLI

```bash
source /opt/stack/openrc admin admin

# Create project
openstack project create \
  --description "Student cloud computing lab" \
  --enable \
  CloudLab-Project

# Create user
openstack user create \
  --project CloudLab-Project \
  --password "Student@123" \
  --email student01@lab.local \
  --enable \
  student01

# Assign member role
openstack role add \
  --project CloudLab-Project \
  --user student01 \
  member

# Verify
openstack project list
openstack user list
openstack role assignment list --project CloudLab-Project
```
<img width="1245" height="520" alt="image" src="https://github.com/user-attachments/assets/d168aab4-eada-41bb-a13d-a77bded9c99d" />

---

### Step B — Switch to Your Project Context

Before creating resources, switch Horizon to the correct project. Otherwise everything lands in the admin project.

**Via Horizon:**
```
Top navigation bar → click "admin" project dropdown (top-left, next to the OpenStack logo)
→ Select CloudLab-Project
```

The breadcrumb at the top should now show `Project / Compute / Instances` under CloudLab-Project.

**Via CLI:**
```bash
# Source credentials scoped to your project
source /opt/stack/openrc student01 Student@123

# Or stay as admin but scoped to the new project
export OS_PROJECT_NAME=CloudLab-Project
openstack token issue    # verify the project in the output
```

---

### Step C — Upload an Image to Glance

#### Option 1 — Use Cirros (Already Pre-loaded by DevStack)

```bash
openstack image list
# Should show: cirros-0.6.2-x86_64-disk  →  status: active
```

If Cirros is listed, **you can skip to Step D** — Cirros is a 12 MB minimal test image perfect for verifying instance launch works.

#### Option 2 — Upload Ubuntu Cloud Image (~600 MB)

```bash
cd /tmp

# Download Ubuntu 22.04 minimal cloud image
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Verify it is a QCOW2 image
file jammy-server-cloudimg-amd64.img
# Expected: QEMU QCOW2 Image (v3)
```

**Upload via CLI:**
```bash
openstack image create \
  --container-format bare \
  --disk-format qcow2 \
  --file /tmp/jammy-server-cloudimg-amd64.img \
  --public \
  --min-ram 512 \
  --min-disk 8 \
  "Ubuntu 22.04 LTS"

# Verify — status must be 'active' before using it
openstack image list
```

**Upload via Horizon:**
```
Horizon → Project → Compute → Images → Create Image

  Image Name:    Ubuntu 22.04 LTS
  File:          Browse → jammy-server-cloudimg-amd64.img
  Format:        QCOW2
  Architecture:  x86_64
  Min Disk (GB): 8
  Min RAM (MB):  512
  Visibility:    Public

<img width="974" height="793" alt="image" src="https://github.com/user-attachments/assets/7ce8d77c-8699-4944-9e1e-8ba574089ec8" />

→ Create Image
```

Wait for status to change from `Saving` to `Active` before proceeding.

---

### Step D — Define a Flavor

#### Check Existing Flavors First

```bash
openstack flavor list

# DevStack pre-loads these:
# m1.tiny   → 1 vCPU,  512 MB RAM,  1 GB disk
# m1.small  → 1 vCPU,  2 GB RAM,   20 GB disk
# m1.medium → 2 vCPU,  4 GB RAM,   40 GB disk
# m1.large  → 4 vCPU,  8 GB RAM,   80 GB disk
# m1.xlarge → 8 vCPU, 16 GB RAM,  160 GB disk
```

If `m1.tiny` or `m1.small` is listed you can **skip flavor creation** and use those directly.

#### Create a Custom Flavor (Optional)

```bash
openstack flavor create \
  --vcpus 1 \
  --ram 1024 \
  --disk 10 \
  --public \
  lab.small

# Verify
openstack flavor show lab.small
```
<img width="678" height="426" alt="image" src="https://github.com/user-attachments/assets/cf050c69-f585-4b56-8c95-aef90938aee1" />

**Via Horizon:**
```
Horizon → Admin → Compute → Flavors → Create Flavor

  Name:      lab.small
  vCPUs:     1
  RAM (MB):  1024
  Root Disk: 10
  Public:    ✅

→ Create Flavor
```
<img width="1919" height="737" alt="image" src="https://github.com/user-attachments/assets/bc264848-2dd3-4c74-8773-1a6739d6a19f" />

---

### Step E — Launch an Instance

#### 1. Create a Security Group

```bash
# Create the group
openstack security group create \
  --description "Allow SSH and ICMP" \
  lab-sg
<img width="1013" height="603" alt="image" src="https://github.com/user-attachments/assets/5c33376f-db2c-4d56-a54e-609fe0de2ad9" />

# Allow SSH (port 22)
openstack security group rule create \
  --protocol tcp \
  --dst-port 22 \
  --remote-ip 0.0.0.0/0 \
  lab-sg
<img width="1009" height="647" alt="image" src="https://github.com/user-attachments/assets/6de6f7ea-9e23-4ae9-8c4b-0ef9e582ba03" />

# Allow ICMP (ping)
openstack security group rule create \
  --protocol icmp \
  --remote-ip 0.0.0.0/0 \
  lab-sg
<img width="1009" height="647" alt="image" src="https://github.com/user-attachments/assets/2ef42295-954a-4e9f-84ef-b8d453686298" />

# Allow HTTP (port 80)
openstack security group rule create \
  --protocol tcp \
  --dst-port 80 \
  --remote-ip 0.0.0.0/0 \
  lab-sg
<img width="1009" height="647" alt="image" src="https://github.com/user-attachments/assets/f22c84d4-76f7-45c6-a471-0979ceed3545" />

```

#### 2. Create a Key Pair

```bash
# Generate new key pair
mkdir -p ~/.ssh
chmod 700 ~/.ssh
openstack keypair create lab-keypair > ~/.ssh/lab-keypair.pem
chmod 400 ~/.ssh/lab-keypair.pem

# OR import your existing public key
openstack keypair create \
  --public-key ~/.ssh/id_rsa.pub \
  lab-keypair
```

#### 3. Get the Network ID

```bash
openstack network list
# Note the ID of 'private' or 'shared' network
<img width="798" height="155" alt="image" src="https://github.com/user-attachments/assets/70e5dbe7-9bfa-44af-a261-4e54b0ac121c" />

NETWORK_ID=$(openstack network show private -f value -c id)
echo $NETWORK_ID    # confirm it printed a UUID
```

#### 4. Launch via CLI

```bash
openstack server create \
  --flavor m1.tiny \
  --image cirros-0.6.2-x86_64-disk \
  --network $NETWORK_ID \
  --security-group lab-sg \
  --key-name lab-keypair \
  --wait \
  my-first-instance
```
<img width="849" height="644" alt="image" src="https://github.com/user-attachments/assets/f385819c-d708-472d-b778-00b66fedc292" />

> 💡 Use `cirros` for your first launch — it's already loaded and boots in under 30 seconds. Switch to Ubuntu once you confirm the workflow works.

#### 5. Launch via Horizon Dashboard

```
Horizon → Project → Compute → Instances → Launch Instance

Step 1 — Details:
  Instance Name: my-first-instance
  Count:         1

Step 2 — Source:
  Boot Source:        Image
  Create New Volume:  No
  Click ↑ on cirros (or Ubuntu 22.04 LTS) to move to Allocated

Step 3 — Flavor:
  Click ↑ on m1.tiny (or lab.small) to allocate

Step 4 — Networks:
  Click ↑ on private (or shared) to allocate

Step 5 — Security Groups:
  Click ↑ on lab-sg to allocate
  (optionally remove 'default')

Step 6 — Key Pair:
  Click ↑ on lab-keypair to allocate

→ Launch Instance
```

The instance will appear in the Instances table. Watch the **Status** column change:

```
Spawning → Build → Active ✅
```

---

## 🔍 Verification

```bash
# Check instance status
openstack server list
<img width="940" height="119" alt="image" src="https://github.com/user-attachments/assets/1de626ca-3e18-412f-a2b0-bfadfe88bfb4" />

# Expected:
+--------------------------------------+------------------+--------+---------------------+
| ID                                   | Name             | Status | Networks            |
+--------------------------------------+------------------+--------+---------------------+
| a1b2c3d4-xxxx-xxxx-xxxx-xxxxxxxxxxxx | my-first-instance| ACTIVE | private=10.11.12.5  |
+--------------------------------------+------------------+--------+---------------------+

# View full details
openstack server show my-first-instance
<img width="898" height="642" alt="image" src="https://github.com/user-attachments/assets/4d97b8f9-c948-4ee9-a5ea-5a9d9542231b" />

# View boot log
openstack console log show my-first-instance | tail -20
<img width="644" height="422" alt="image" src="https://github.com/user-attachments/assets/caae9bf1-1bbb-4727-938e-c33923af2921" />

# Get VNC console URL (open in browser)
openstack console url show --novnc my-first-instance

```

**In Horizon** — click the instance name to open its detail page, then click **Console** tab to get an in-browser terminal.

---

## 🚨 Common Issues

| Problem | Fix |
|---|---|
| Status stays at `BUILD` > 5 min | Check `openstack console log show my-first-instance` for boot errors |
| Status shows `ERROR` | Run `openstack server show` — check `fault` field for reason |
| No networks available in launch wizard | Create a network first (Practical 09), or use DevStack's pre-created `private` network |
| `Image not found` | Image status is not `active` yet — wait for upload to complete |
| `Flavor not found` | Run `openstack flavor list` and use an exact name from the output |
| VNC console says "Connection failed" | Run `sudo systemctl restart devstack@n-novnc.service` |

---

## ✅ Learning Outcomes

After completing this practical, you should be able to:

- [ ] Switch Horizon context between projects
- [ ] Create projects and assign user roles
- [ ] Upload OS images to Glance and verify `active` status
- [ ] List and create flavors
- [ ] Create security groups with SSH, ICMP, and HTTP rules
- [ ] Create a key pair and set correct file permissions
- [ ] Launch an instance via both Horizon and CLI
- [ ] Read instance status and interpret console logs
- [ ] Access the instance via VNC console in the browser

---

## 📚 Further Reading

- [OpenStack Nova (Compute)](https://docs.openstack.org/nova/latest/)
- [OpenStack Glance (Image Service)](https://docs.openstack.org/glance/latest/)
- [Ubuntu Cloud Images](https://cloud-images.ubuntu.com/)
- [OpenStack CLI Reference](https://docs.openstack.org/python-openstackclient/latest/)
