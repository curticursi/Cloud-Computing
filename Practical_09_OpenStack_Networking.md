# Practical 09 — Set Up Networking in OpenStack (Neutron)

---

## 📌 Objective

Configure OpenStack Neutron to provide networking for instances by creating private and public networks, attaching a router to connect them, and assigning floating IPs to instances for external access.

---

## 🧠 Conceptual Background (Know-How)

### What is OpenStack Neutron?

**OpenStack Neutron** (formerly Quantum) is the networking-as-a-service component of OpenStack. It provides connectivity between the interfaces of other OpenStack services — most notably Nova instances.

### Neutron vs AWS VPC

```
OpenStack Neutron ↔ AWS VPC Concepts:

┌──────────────────────┬──────────────────────┐
│  OpenStack Neutron   │       AWS VPC        │
├──────────────────────┼──────────────────────┤
│ Private Network      │ Private Subnet       │
│ Public Network       │ Public Subnet + IGW  │
│ Router               │ Route Table + NAT GW │
│ Floating IP          │ Elastic IP           │
│ Security Group       │ Security Group       │
│ VLAN / VXLAN         │ Subnet segmentation  │
│ Provider Network     │ VPC CIDR Block       │
└──────────────────────┴──────────────────────┘
```

### Neutron Networking Architecture

```
                       External Network (Internet / Physical)
                                      │
                    ┌─────────────────▼─────────────────┐
                    │       Public Network               │
                    │   (ext-net: 192.168.1.0/24)        │
                    │   Provider / Floating IPs          │
                    └─────────────────┬─────────────────┘
                                      │
                    ┌─────────────────▼─────────────────┐
                    │            Router                  │
                    │  (connects private ↔ public)       │
                    │  Gateway: external interface       │
                    └──────────┬──────────────┬──────────┘
                               │              │
             ┌─────────────────▼──┐  ┌────────▼────────────────┐
             │  Private Network   │  │  Another Private Network │
             │ 10.0.0.0/24        │  │ 172.16.0.0/24            │
             │                    │  │                          │
             │  ┌──┐ ┌──┐ ┌──┐   │  │  ┌──┐ ┌──┐              │
             │  │VM│ │VM│ │VM│   │  │  │VM│ │VM│              │
             │  └──┘ └──┘ └──┘   │  │  └──┘ └──┘              │
             └────────────────────┘  └──────────────────────────┘
```

### Floating IPs

A **Floating IP** is a publicly routable IP address from the external network pool that can be associated with a VM's private IP. This is equivalent to an **Elastic IP** in AWS.

```
                     Floating IP: 192.168.1.200
                           │
                     ┌─────▼──────┐
                     │   Router   │
                     │  (1:1 NAT) │
                     └─────┬──────┘
                           │
                     Fixed IP: 10.0.0.5
                     ┌─────▼──────┐
                     │     VM     │
                     └────────────┘

External traffic to 192.168.1.200 → Router → NAT → VM at 10.0.0.5
```

### Neutron Network Types

| Type | Use Case |
|---|---|
| **Local** | Connectivity only within the same hypervisor (testing) |
| **Flat** | All VMs on same network, no VLANs |
| **VLAN** | IEEE 802.1Q VLANs — segment traffic by VLAN ID |
| **VXLAN** | Overlay network — scales beyond VLAN limitations (most common) |
| **GRE** | Generic Routing Encapsulation tunnels |

---

## 🛠️ Step-by-Step Guide

### Prerequisites
- OpenStack running (from Practical 07)
- Admin credentials: `source /opt/stack/openrc admin admin`
- At least one running instance (from Practical 08)

---

### Step A — Create a Private Network

#### Via CLI

```bash
# Source admin credentials
source /opt/stack/openrc admin admin

# Create a private network
openstack network create \
  --project CloudLab-Project \
  --description "Private network for CloudLab instances" \
  private-net

# Create a subnet for the private network
openstack subnet create \
  --network private-net \
  --subnet-range 10.0.0.0/24 \
  --gateway 10.0.0.1 \
  --dns-nameserver 8.8.8.8 \
  --dns-nameserver 8.8.4.4 \
  --ip-version 4 \
  private-subnet

# Verify
openstack network show private-net
openstack subnet show private-subnet
```

**Expected output:**
```
+------------------+--------------------------------------+
| Field            | Value                                |
+------------------+--------------------------------------+
| admin_state_up   | UP                                   |
| cidr             | 10.0.0.0/24                         |
| gateway_ip       | 10.0.0.1                             |
| ip_version       | 4                                    |
| name             | private-subnet                       |
| network_id       | xxxx-xxxx-xxxx-xxxx                  |
+------------------+--------------------------------------+
```

#### Via Horizon Dashboard

```
Horizon → Project → Network → Networks → Create Network

Network tab:
  Network Name:    private-net
  Admin State:     UP
  ✅ Create Subnet

Subnet tab:
  Subnet Name:     private-subnet
  Network Address: 10.0.0.0/24
  IP Version:      IPv4
  Gateway IP:      10.0.0.1
  ✅ Enable Gateway

Subnet Details tab:
  Enable DHCP:     ✅
  DNS Name Servers: 8.8.8.8
                    8.8.4.4

→ Create
```

---

### Create a Public (External) Network (Admin only)

The public network simulates an external/internet-facing network. In DevStack, this is often pre-configured. Let's create one if it doesn't exist:

```bash
# Must be done as admin
source /opt/stack/openrc admin admin

# Create external network (provider network)
openstack network create \
  --external \
  --provider-network-type flat \
  --provider-physical-network public \
  ext-net

# Create a subnet for external network
openstack subnet create \
  --network ext-net \
  --subnet-range 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --no-dhcp \
  --allocation-pool start=192.168.1.100,end=192.168.1.200 \
  --dns-nameserver 8.8.8.8 \
  ext-subnet

# Verify
openstack network list --external
```

> 💡 In DevStack, the external network is usually pre-created as `public` with floating IPs in the `172.24.4.0/24` range. Check with: `openstack network list --external`

---

### Step B — Attach a Router to Connect Networks

#### Create the Router

```bash
# Switch to project user context
source /opt/stack/openrc demo demo
# or
source /opt/stack/openrc admin admin

# Create a router
openstack router create \
  --description "Router connecting private to public" \
  lab-router

# Set the external gateway (connect to public network)
openstack router set \
  --external-gateway ext-net \
  lab-router

# Add interface to private subnet
openstack router add subnet \
  lab-router \
  private-subnet

# Verify router configuration
openstack router show lab-router
openstack router port list lab-router
```

**Router verification output:**
```
+----------------------------------+---------------------------+
| Field                            | Value                     |
+----------------------------------+---------------------------+
| external_gateway_info            | {network_id: xxxx,        |
|                                  |  ip: 192.168.1.101}       |
| interfaces_info                  | [{subnet_id: xxx,         |
|                                  |  ip: 10.0.0.1}]           |
| name                             | lab-router                |
| status                           | ACTIVE                    |
+----------------------------------+---------------------------+
```

#### Via Horizon Dashboard

```
Horizon → Project → Network → Routers → Create Router
  Router Name:     lab-router
  Admin State:     UP
  External Network: ext-net
→ Create Router

Then add interface:
  Click lab-router → Interfaces tab → Add Interface
    Subnet:    private-subnet
    IP Address: (leave blank for auto)
  → Submit
```

---

### Step C — Assign Floating IPs to Instances

#### Allocate a Floating IP from the External Pool

```bash
# Allocate a floating IP from the external network
openstack floating ip create ext-net

# Output:
+-------------+--------------------------------------+
| Field       | Value                                |
+-------------+--------------------------------------+
| floating_ip | 192.168.1.105                        |
| id          | f1e2d3c4-xxxx-xxxx-xxxx-xxxxxxxxxxxx |
| status      | DOWN                                 |
+-------------+--------------------------------------+
```

#### Associate Floating IP with an Instance

```bash
# List your instances and their fixed IPs
openstack server list

# Associate floating IP with instance
openstack server add floating ip \
  my-first-instance \
  192.168.1.105

# Verify
openstack server show my-first-instance | grep addresses
# Output: addresses | private-net=10.0.0.5, 192.168.1.105
```

#### Via Horizon Dashboard

```
Horizon → Project → Compute → Instances →
  Find my-first-instance → Actions dropdown → Associate Floating IP
  
  IP Address:  Select floating IP (or + to allocate new)
  Port:        Select the instance port
→ Associate
```

---

## 🔍 Verification and Testing

### Test Connectivity

```bash
# Ping the floating IP from outside OpenStack (your host machine)
ping 192.168.1.105

# SSH to the instance via floating IP
ssh -i ~/.ssh/lab-keypair.pem ubuntu@192.168.1.105

# From inside the VM, test internet access (via router)
ping 8.8.8.8
curl ifconfig.me

# Test private network communication between VMs
# From VM1, ping VM2's private IP
ping 10.0.0.6
```
<img width="601" height="366" alt="image" src="https://github.com/user-attachments/assets/35b99cbb-653c-4446-995a-8855621c83f1" />

### Network Topology Visualization

```
Horizon → Project → Network → Network Topology

This shows a visual graph of:
  [ext-net] ── [lab-router] ── [private-net] ── [VM1, VM2...]
```

### Diagnostic Commands

```bash
# List all networks
openstack network list

# List all subnets
openstack subnet list

# List all routers
openstack router list

# Show router details and ports
openstack router show lab-router
openstack router port list lab-router

# List floating IPs
openstack floating ip list

# Check DHCP agent
openstack network agent list | grep dhcp

# Check L3 agent
openstack network agent list | grep l3

# Trace routing in Neutron namespace
# (as root on the compute/network node)
sudo ip netns list
# Look for: qrouter-XXXX (router namespace) and qdhcp-XXXX (DHCP namespace)

sudo ip netns exec qrouter-XXXX ip route
sudo ip netns exec qrouter-XXXX ping 8.8.8.8
```

---

## 📊 Complete Network Architecture (After This Practical)

```
Internet / Physical Network
         │
         │ (192.168.1.0/24)
┌────────▼─────────────────────────────────────────────┐
│                  ext-net (Public)                    │
│                192.168.1.0/24                        │
│          Floating IP Pool: .100-.200                 │
└────────────────────┬─────────────────────────────────┘
                     │ Gateway: 192.168.1.101
              ┌──────▼───────┐
              │  lab-router  │
              │   (Neutron)  │
              └──────┬───────┘
                     │ Internal: 10.0.0.1
┌────────────────────▼─────────────────────────────────┐
│                private-net                           │
│              10.0.0.0/24                             │
│  ┌─────────────────────┐  ┌─────────────────────┐    │
│  │ my-first-instance   │  │   other-instance    │    │
│  │ Fixed:  10.0.0.5    │  │ Fixed: 10.0.0.6     │    │
│  │ Float:  192.168.1.105│  │                     │    │
│  └─────────────────────┘  └─────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

---

## ✅ Learning Outcomes

After completing this practical, you should be able to:

- [ ] Understand Neutron's role in OpenStack networking
- [ ] Create private and public (external) networks in OpenStack
- [ ] Create subnets with DHCP and DNS configuration
- [ ] Create and configure a Neutron router
- [ ] Connect private networks to the external network via a router
- [ ] Allocate and assign floating IPs to instances
- [ ] Use the Network Topology visualization in Horizon
- [ ] Debug network issues using Neutron namespaces and CLI tools

---

## 🔐 Security Considerations

1. **Security Groups** act as the per-instance firewall — always restrict to needed ports only
2. **Network segmentation** keeps different tenant VMs isolated at the network level
3. **Provider networks** should only be managed by the admin
4. Use **Neutron FWaaS** (Firewall-as-a-Service) for perimeter firewall rules at the subnet/router level

---

## 📚 Further Reading

- [OpenStack Neutron Documentation](https://docs.openstack.org/neutron/latest/)
- [Neutron Architecture Deep Dive](https://docs.openstack.org/neutron/latest/admin/intro-os-networking.html)
- [Floating IPs in OpenStack](https://docs.openstack.org/neutron/latest/admin/intro-network-components.html)
