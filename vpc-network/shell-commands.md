# VPC Networks & Subnets: Complete Operations & Verification Manual

This document is an operational reference manual for Google Cloud Virtual Private Cloud (VPC) Networks, Subnets, Firewall Rules, and IP Address Management.

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
6. [Category 6: Static Internal & External IP Address Allocation](#category-6-static-internal--external-ip-address-allocation)
7. [Category 7: Exhaustive Failure Diagnosis & Resolution Matrix](#category-7-exhaustive-failure-diagnosis--resolution-matrix)

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

This scenario demonstrates handling subnet IP exhaustion on a small `/29` subnet (8 total IPs, 4 reserved by GCP, leaving 4 available host IPs). When 4 VMs consume all available IPs, a 5th VM creation fails with `IP_SPACE_EXHAUSTED`. Expanding the subnet to `/23` (512 total IPs) live allows the 5th VM to launch without taking down any of the 4 running VMs.

#### Step 1: Create Small /29 Subnet (4 Usable Host IPs)
```bash
gcloud compute networks subnets create demo-small-subnet \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --range=10.10.0.0/29
```

#### Step 2: Launch 4 VMs to Exhaust Subnet IP Space (Consumes 10.10.0.2 to 10.10.0.5)
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

#### Expected Terminal Output:
```text
Updated [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/subnetworks/demo-small-subnet].
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

#### How to Verify Configuration & Zero-Downtime Correctness:
```bash
# Verify all 5 VMs are active and running without downtime
gcloud compute instances list --filter="subnet:demo-small-subnet"
```

#### Expected Verification Output:
```text
NAME  ZONE           MACHINE_TYPE  INTERNAL_IP  STATUS
vm-1  us-central1-a  e2-micro      10.10.0.2    RUNNING
vm-2  us-central1-a  e2-micro      10.10.0.3    RUNNING
vm-3  us-central1-a  e2-micro      10.10.0.4    RUNNING
vm-4  us-central1-a  e2-micro      10.10.0.5    RUNNING
vm-5  us-central1-a  e2-micro      10.10.0.6    RUNNING
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
NAME                  REGION       NETWORK              RANGE        STACK_TYPE  IPV6_ACCESS_TYPE
prod-dualstack-subnet  us-central1  gcd-prod-custom-vpc  10.2.0.0/24  IPV4_IPV6   EXTERNAL
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks subnets describe prod-dualstack-subnet \
    --region=us-central1 \
    --format="yaml(stackType, ipv6AccessType, externalIpv6Prefix)"
```

#### Expected Verification Output:
```yaml
externalIpv6Prefix: 2600:1901:0:81a2::/64
ipv6AccessType: EXTERNAL
stackType: IPV4_IPV6
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
NAME                NETWORK              DIRECTION  PRIORITY  ALLOW     DENY  DISABLED
allow-internal-ssh  gcd-prod-custom-vpc  INGRESS    1000      tcp:22,icmp     False
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute firewall-rules describe allow-internal-ssh --format="yaml(name, allowed, sourceRanges, targetTags)"
```

#### Expected Verification Output:
```yaml
allowed:
- IPProtocol: tcp
  ports:
  - '22'
- IPProtocol: icmp
name: allow-internal-ssh
sourceRanges:
- 10.1.0.0/20
targetTags:
- ssh-enabled
```

---

## Category 6: Static Internal & External IP Address Allocation

### 1. Reserve Static Regional Internal IP Address

```bash
gcloud compute addresses create internal-db-ip \
    --region=us-central1 \
    --subnet=prod-subnet-us-central1 \
    --addresses=10.1.0.50
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/addresses/internal-db-ip].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute addresses describe internal-db-ip --region=us-central1 --format="yaml(name, address, addressType, status)"
```

#### Expected Verification Output:
```yaml
address: 10.1.0.50
addressType: INTERNAL
name: internal-db-ip
status: RESERVED
```

---

## Category 7: Exhaustive Failure Diagnosis & Resolution Matrix

| Error Code / Status | Root Cause | Diagnosis Command | Immediate Resolution Command |
| :--- | :--- | :--- | :--- |
| **`IP_SPACE_EXHAUSTED`** | All available host IPs in subnet CIDR block consumed by running instances. | `gcloud compute instances list --filter="subnet:SUBNET"` | Expand subnet live: `gcloud compute networks subnets expand-ip-range SUBNET --prefix-length=NEW_MASK` (e.g. `/29` to `/23`). |
| **`INVALID_CIDR_RANGE` / Overlap** | Subnet IP range overlaps with another subnet in the same VPC or peered network. | `gcloud compute networks subnets list --network=VPC_NAME` | Assign non-overlapping RFC 1918 CIDR block (e.g., `10.2.0.0/24`). |
| **`CANNOT_SHRINK_SUBNET`** | Attempted to expand range with larger prefix mask (e.g. `/20` to `/24`). | `gcloud compute networks subnets describe SUBNET` | Subnet expansion is one-way. Pass smaller prefix number (e.g., `/20` or `/16`). |
| **`AUTO_MODE_CONVERSION_FAILED`** | Custom subnets already exist or API parameters malformed. | `gcloud compute networks describe VPC` | Verify VPC is currently Auto mode before calling `switch-mode --mode=custom`. |
| **`RESERVED_IP_COLLISION`** | Attempted to assign `.0`, `.1`, `N-2`, or `N-1` address to a host. | `gcloud compute addresses describe IP` | Use available IP in range (`.2` through `N-3`). `.0`, `.1`, `N-2`, and `N-1` are reserved by GCP. |
| **`SUBNET_EXPANSION_EXCEEDED`** | Auto-mode subnet expansion requested beyond `/16` limit. | `gcloud compute networks subnets describe SUBNET` | Convert Auto Mode network to Custom Mode before expanding subnet beyond `/16`. |
| **`FIREWALL_RULE_CONFLICT`** | Lower priority rule overriding ingress traffic rules. | `gcloud compute firewall-rules list --filter="network=VPC_NAME"` | Adjust priority numerical value (lower integer = higher precedence, e.g., `--priority=100`). |
| **`IPV6_NOT_SUPPORTED_ON_AUTO`** | Attempted dual-stack setup on Auto Mode network. | `gcloud compute networks describe VPC` | Switch network to Custom Mode before enabling `--stack-type=IPV4_IPV6`. |
