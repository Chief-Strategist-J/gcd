# VPC Networks & Subnets: Complete Operations & Verification Manual

This document is an operational reference manual for Google Cloud Virtual Private Cloud (VPC) Networks, Subnets, Firewall Rules, IP Address Management, Bring Your Own IP (BYOIP), Virtual Routers, and Cloud DNS.

Every command snippet includes:
1. **Command to Execute**
2. **Expected Terminal Output (What to read & look for in terminal)**
3. **How to Verify Configuration Correctness & Expected Verification Output**

---

## Table of Contents
1. [Category 1: VPC Network Creation & Mode Management](#category-1-vpc-network-creation--mode-management)
2. [Category 2: Regional Subnetwork Provisioning & Custom CIDR Allocation](#category-2-regional-subnetwork-provisioning--custom-cidr-allocation)
3. [Category 3: Zero-Downtime Subnet IP Range Expansion](#category-3-zero-downtime-subnet-ip-range-expansion)
4. [Category 4: Dual-Stack IPv4 & IPv6 Subnet Provisioning](#category-4-dual-stack-ipv4--ipv6-subnet-provisioning)
5. [Category 5: VPC Firewall Rules & Baseline Security Policies](#category-5-vpc-firewall-rules--baseline-security-policies)
6. [Category 6: Static Internal, External & Ephemeral IP Management](#category-6-static-internal-external--ephemeral-ip-management)
7. [Category 7: Bring Your Own IP (BYOIP) & Internal DNS Verification](#category-7-bring-your-own-ip-byoip--internal-dns-verification)
8. [Category 8: Alias IP Ranges & OS NAT Transparency Inspection](#category-8-alias-ip-ranges--os-nat-transparency-inspection)
9. [Category 9: Cloud DNS Managed Zones & Record Sets](#category-9-cloud-dns-managed-zones--record-sets)
10. [Category 10: Custom VPC Routes & Virtual Router Management](#category-10-custom-vpc-routes--virtual-router-management)
11. [Category 11: Advanced Stateful Ingress & Egress Firewall Policies](#category-11-advanced-stateful-ingress--egress-firewall-policies)
12. [Category 12: Exhaustive Failure Diagnosis & Resolution Matrix](#category-12-exhaustive-failure-diagnosis--resolution-matrix)

---

## Category 1: VPC Network Creation & Mode Management

### 1. Provision Custom Mode VPC Network (Production Standard)

```bash
gcloud compute networks create gcd-prod-custom-vpc \
    --subnet-mode=custom \
    --bgp-routing-mode=global \
    --mtu=1460
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/global/networks/gcd-prod-custom-vpc].
NAME                SUBNET_MODE  BGP_ROUTING_MODE  IPV4_RANGE  GATEWAY_IPV4
gcd-prod-custom-vpc  CUSTOM       GLOBAL
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks describe gcd-prod-custom-vpc --format="yaml(name, autoCreateSubnetworks, routingConfig, mtu)"
```

#### Expected Verification Output:
```yaml
autoCreateSubnetworks: false
mtu: 1460
name: gcd-prod-custom-vpc
routingConfig:
  routingMode: GLOBAL
```

---

### 2. Convert Existing Auto Mode VPC Network to Custom Mode (Irreversible)

```bash
gcloud compute networks switch-mode gcd-dev-auto-vpc --mode=custom
```

#### Expected Terminal Output:
```text
Switching network to custom mode...done.
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks describe gcd-dev-auto-vpc --format="value(autoCreateSubnetworks)"
```

#### Expected Verification Output:
```text
False
```

---

## Category 2: Regional Subnetwork Provisioning & Custom CIDR Allocation

### 1. Provision Regional Custom Subnetwork

```bash
gcloud compute networks subnets create prod-subnet-us-central1 \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --range=10.1.0.0/24 \
    --enable-private-ip-google-access
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/subnetworks/prod-subnet-us-central1].
NAME                     REGION       NETWORK              RANGE
prod-subnet-us-central1  us-central1  gcd-prod-custom-vpc  10.1.0.0/24
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks subnets describe prod-subnet-us-central1 \
    --region=us-central1 \
    --format="yaml(name, ipCidrRange, gatewayAddress, privateIpGoogleAccess)"
```

#### Expected Verification Output:
```yaml
gatewayAddress: 10.1.0.1
ipCidrRange: 10.1.0.0/24
name: prod-subnet-us-central1
privateIpGoogleAccess: true
```

---

## Category 3: Zero-Downtime Subnet IP Range Expansion

### 1. Standard Subnet Range Expansion (e.g. /24 to /20)

```bash
gcloud compute networks subnets expand-ip-range prod-subnet-us-central1 \
    --region=us-central1 \
    --prefix-length=20
```

#### Expected Terminal Output:
```text
Updated [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/subnetworks/prod-subnet-us-central1].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks subnets describe prod-subnet-us-central1 \
    --region=us-central1 \
    --format="value(ipCidrRange)"
```

#### Expected Verification Output:
```text
10.1.0.0/20
```

---

### 2. Practical End-to-End Lab Scenario: IP Space Exhaustion on /29 & Live Expansion to /23

#### Step 1: Create Small /29 Subnet (4 Usable Host IPs)
```bash
gcloud compute networks subnets create demo-small-subnet \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --range=10.10.0.0/29
```

#### Step 2: Launch 4 VMs to Exhaust Subnet IP Space
```bash
gcloud compute instances create vm-1 vm-2 vm-3 vm-4 \
    --zone=us-central1-a \
    --machine-type=e2-micro \
    --subnet=demo-small-subnet
```

#### Step 3: Attempt to Launch 5th VM Instance (Fails Due to IP Exhaustion)
```bash
gcloud compute instances create vm-5 \
    --zone=us-central1-a \
    --machine-type=e2-micro \
    --subnet=demo-small-subnet
```

#### Expected Terminal Output (IP Exhaustion Failure):
```text
ERROR: (gcloud.compute.instances.create) Could not fetch resource:
- IP space of subnetwork 'demo-small-subnet' in region 'us-central1' is exhausted.
```

#### Step 4: Expand Subnet Live from /29 to /23 (Zero VM Downtime for VMs 1-4)
```bash
gcloud compute networks subnets expand-ip-range demo-small-subnet \
    --region=us-central1 \
    --prefix-length=23
```

#### Step 5: Retry Launching 5th VM Instance (Succeeds Immediately)
```bash
gcloud compute instances create vm-5 \
    --zone=us-central1-a \
    --machine-type=e2-micro \
    --subnet=demo-small-subnet
```

#### Expected Terminal Output (Success):
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/zones/us-central1-a/instances/vm-5].
NAME  ZONE           MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
vm-5  us-central1-a  e2-micro                   10.10.0.6    34.122.10.55   RUNNING
```

---

## Category 4: Dual-Stack IPv4 & IPv6 Subnet Provisioning

### 1. Create Dual-Stack Subnet with External IPv6 Access

```bash
gcloud compute networks subnets create prod-dualstack-subnet \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --range=10.2.0.0/24 \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=EXTERNAL
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/subnetworks/prod-dualstack-subnet].
```

---

## Category 5: VPC Firewall Rules & Baseline Security Policies

### 1. Create Ingress Firewall Rule for SSH & Health Checks

```bash
gcloud compute firewall-rules create allow-internal-ssh \
    --network=gcd-prod-custom-vpc \
    --allow=tcp:22,icmp \
    --direction=INGRESS \
    --priority=1000 \
    --source-ranges=10.1.0.0/20 \
    --target-tags=ssh-enabled
```

#### Expected Terminal Output:
```text
Creating firewall rule...done.
```

---

## Category 6: Static Internal, External & Ephemeral IP Management

### 1. Reserve Static Regional External IP Address

```bash
gcloud compute addresses create prod-web-static-ip \
    --region=us-central1 \
    --network-tier=PREMIUM
```

---

### 2. Practical Lab Scenario: VM Stop/Start Lifecycle & Ephemeral External IP Mutation

#### Step 1: Provision Test VM with Ephemeral External IP
```bash
gcloud compute instances create lifecycle-demo-vm \
    --zone=us-central1-a \
    --machine-type=e2-micro \
    --subnet=prod-subnet-us-central1
```

#### Step 2: Stop VM Instance (Triggers 90-Second Grace Period)
```bash
gcloud compute instances stop lifecycle-demo-vm --zone=us-central1-a
```

#### Step 3: Inspect Stopped VM IP State (Internal Retained, External Released)
```bash
gcloud compute instances describe lifecycle-demo-vm --zone=us-central1-a \
    --format="yaml(status, networkInterfaces[0].networkIP, networkInterfaces[0].accessConfigs)"
```

#### Expected Verification Output:
```yaml
networkIP: 10.1.0.2
status: TERMINATED
```

#### Step 4: Restart VM & Verify Mutated Ephemeral External IP
```bash
gcloud compute instances start lifecycle-demo-vm --zone=us-central1-a
gcloud compute instances describe lifecycle-demo-vm --zone=us-central1-a \
    --format="yaml(status, networkInterfaces[0].networkIP, networkInterfaces[0].accessConfigs[0].natIP)"
```

#### Expected Verification Output:
```yaml
accessConfigs:
- natIP: 35.202.88.19
networkIP: 10.1.0.2
status: RUNNING
```

---

## Category 7: Bring Your Own IP (BYOIP) & Internal DNS Verification

### 1. Verify Network-Scoped Internal DNS Resolution Inside VM

```bash
gcloud compute ssh vm-1 --zone=us-central1-a --command="dig +short vm-2.us-central1-a.c.YOUR_PROJECT.internal"
```

#### Expected Terminal Output:
```text
10.10.0.3
```

---

## Category 8: Alias IP Ranges & OS NAT Transparency Inspection

### 1. Inspect OS Network Interfaces Inside VM (Demonstrating NAT Transparency)

```bash
# Execute ip addr show inside VM OS
gcloud compute ssh lifecycle-demo-vm --zone=us-central1-a --command="ip addr show eth0"
```

#### Expected Terminal Output (Guest OS ONLY Sees Internal IP 10.1.0.2):
```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc mq state UP group default qlen 1000
    link/ether 42:01:0a:01:00:02 brd ff:ff:ff:ff:ff:ff
    inet 10.1.0.2/32 brd 10.1.0.2 scope global eth0
       valid_lft forever preferred_lft forever
```
* Note: The external IP `35.202.88.19` does NOT appear on `eth0`. It is mapped transparently via 1:1 NAT at the VPC hypervisor layer.

---

### 2. Provision VM Instance with Alias IP Range for Container Hosting

```bash
gcloud compute instances create container-host-vm \
    --zone=us-central1-a \
    --machine-type=n2-standard-2 \
    --subnet=prod-subnet-us-central1 \
    --aliases="10.1.0.64/28"
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/zones/us-central1-a/instances/container-host-vm].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute instances describe container-host-vm --zone=us-central1-a \
    --format="yaml(networkInterfaces[0].aliasIpRanges)"
```

#### Expected Verification Output:
```yaml
aliasIpRanges:
- ipCidrRange: 10.1.0.64/28
```

---

## Category 9: Cloud DNS Managed Zones & Record Sets

### 1. Provision Authoritative Cloud DNS Public Managed Zone

```bash
gcloud dns managed-zones create company-public-zone \
    --dns-name="example.com." \
    --description="Production Public DNS Zone" \
    --visibility=public
```

#### Expected Terminal Output:
```text
Created [https://dns.googleapis.com/dns/v1/projects/YOUR_PROJECT/managedZones/company-public-zone].
```

---

### 2. Add DNS A-Record Pointing to VM External IP

```bash
gcloud dns record-sets create "app.example.com." \
    --zone=company-public-zone \
    --type=A \
    --ttl=300 \
    --rrdatas="35.202.88.19"
```

#### Expected Terminal Output:
```text
Created [https://dns.googleapis.com/dns/v1/projects/YOUR_PROJECT/managedZones/company-public-zone/rrsets/app.example.com./A].
```

---

## Category 10: Custom VPC Routes & Virtual Router Management

### 1. Create Custom Static Route Overriding Default Routing

```bash
gcloud compute routes create route-to-internal-appliance \
    --network=gcd-prod-custom-vpc \
    --destination-range=172.16.0.0/12 \
    --next-hop-instance=firewall-appliance-vm \
    --next-hop-instance-zone=us-central1-a \
    --priority=800
```

#### Expected Terminal Output:
```text
Creating route...done.
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute routes describe route-to-internal-appliance --format="yaml(name, destRange, priority, nextHopInstance)"
```

#### Expected Verification Output:
```yaml
destRange: 172.16.0.0/12
name: route-to-internal-appliance
priority: 800
```

---

## Category 11: Advanced Stateful Ingress & Egress Firewall Policies

### 1. Provision Strict Egress Firewall Rule Target by Service Account

```bash
gcloud compute firewall-rules create block-database-egress-to-internet \
    --network=gcd-prod-custom-vpc \
    --action=DENY \
    --direction=EGRESS \
    --priority=500 \
    --destination-ranges=0.0.0.0/0 \
    --target-service-accounts="sa-database@YOUR_PROJECT.iam.gserviceaccount.com"
```

#### Expected Terminal Output:
```text
Creating firewall rule...done.
```

---

## Category 12: Exhaustive Failure Diagnosis & Resolution Matrix

| Error Code / Status | Root Cause | Diagnosis Command | Immediate Resolution Command |
| :--- | :--- | :--- | :--- |
| **`EXTERNAL_IP_NOT_IN_OS_IFCONFIG`** | Expected external IP to show inside VM OS `ifconfig`. | `gcloud compute ssh VM --command="ip addr show"` | Working as intended. GCP uses 1:1 hypervisor NAT. External IP is mapped transparently outside the VM OS. |
| **`ALIAS_IP_CIDR_OVERLAP`** | Alias IP range conflicts with primary VM IP or another alias range. | `gcloud compute instances describe VM \| grep aliasIpRanges` | Assign secondary CIDR range from subnet that is not allocated to other VMs. |
| **`ROUTE_DESTINATION_MISMATCH`** | Next hop instance specified in custom route is not in the same VPC or zone. | `gcloud compute routes describe ROUTE_NAME` | Ensure `--next-hop-instance` is running and attached to the target VPC network. |
| **`FIREWALL_IMPLIED_DENY_INGRESS`** | Inbound connection blocked because no explicit Ingress Allow rule exists. | `gcloud compute firewall-rules list --filter="network=VPC_NAME"` | Create Ingress Allow rule matching source CIDR, protocol, and port (Priority < 65535). |
| **`UNASSIGNED_STATIC_IP_SURCHARGE`** | Static external IP reserved but not attached to running VM, triggering penalty billing rate. | `gcloud compute addresses list --filter="status=RESERVED AND users:*"` | Attach static IP to active VM or release address: `gcloud compute addresses delete IP_NAME`. |
| **`BYOIP_PREFIX_TOO_SMALL`** | Attempted BYOIP PAP creation on subnet prefix smaller than `/24` (e.g. `/25` or `/28`). | `gcloud compute public-advertised-prefixes describe PAP` | BYOIP requires `/24` minimum prefix for global BGP Anycast routing. Use block $\ge$ `/24`. |
| **`INTERNAL_DNS_CROSS_VPC_FAILED`** | Attempted internal DNS lookup for VM in another VPC network. | `gcloud compute ssh VM --command="dig HOST.c.PROJECT.internal"` | Internal DNS is strictly scoped to a single VPC network. Set up Cloud DNS Private Zones for cross-VPC DNS. |
| **`IP_SPACE_EXHAUSTED`** | All available host IPs in subnet CIDR block consumed by running instances. | `gcloud compute instances list --filter="subnet:SUBNET"` | Expand subnet live: `gcloud compute networks subnets expand-ip-range SUBNET --prefix-length=NEW_MASK` (e.g. `/29` to `/23`). |
| **`INVALID_CIDR_RANGE` / Overlap** | Subnet IP range overlaps with another subnet in the same VPC or peered network. | `gcloud compute networks subnets list --network=VPC_NAME` | Assign non-overlapping RFC 1918 CIDR block (e.g., `10.2.0.0/24`). |
| **`CANNOT_SHRINK_SUBNET`** | Attempted to expand range with larger prefix mask (e.g. `/20` to `/24`). | `gcloud compute networks subnets describe SUBNET` | Subnet expansion is one-way. Pass smaller prefix number (e.g., `/20` or `/16`). |
| **`AUTO_MODE_CONVERSION_FAILED`** | Custom subnets already exist or API parameters malformed. | `gcloud compute networks describe VPC` | Verify VPC is currently Auto mode before calling `switch-mode --mode=custom`. |
| **`RESERVED_IP_COLLISION`** | Attempted to assign `.0`, `.1`, `N-2`, or `N-1` address to a host. | `gcloud compute addresses describe IP` | Use available IP in range (`.2` through `N-3`). `.0`, `.1`, `N-2`, and `N-1` are reserved by GCP. |
