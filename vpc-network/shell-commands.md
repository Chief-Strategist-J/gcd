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
12. [Category 12: Cloud VPN Provisioning (Classic VPN, HA VPN, BGP Cloud Router, AWS Interop & GCP-to-GCP HA VPN)](#category-12-cloud-vpn-provisioning-classic-vpn-ha-vpn-bgp-cloud-router-aws-interop--gcp-to-gcp-ha-vpn)
13. [Category 13: Dedicated, Partner & Cross-Cloud Interconnect Operations](#category-13-dedicated-partner--cross-cloud-interconnect-operations)
14. [Category 14: Exhaustive Failure Diagnosis & Resolution Matrix](#category-14-exhaustive-failure-diagnosis--resolution-matrix)
15. [Category 15: Master End-to-End Sequential Deployment Commands](#category-15-master-end-to-end-sequential-deployment-commands)

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
```yaml
False
```

---

### 3. Explore & Delete Default Network & Verify VM Instance Creation Failure Without VPC

```bash
# Step 1: Delete default firewall rules
gcloud compute firewall-rules delete default-allow-icmp default-allow-internal default-allow-rdp default-allow-ssh --quiet

# Step 2: Delete default VPC network
gcloud compute networks delete default --quiet

# Step 3: Attempt to launch VM without VPC network (Expected Failure)
gcloud compute instances create test-no-vpc-vm --zone=us-central1-a
```

#### Expected Terminal Output:
```text
Deleted [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/global/firewallrules/default-allow-icmp].
Deleted [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/global/networks/default].
ERROR: (gcloud.compute.instances.create) Could not fetch resource:
- No more networks available in project.
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks list
```

#### Expected Verification Output:
```text
Listed 0 items.
```

---

### 4. Enable Required Network Management & IAP APIs

```bash
gcloud services enable iap.googleapis.com networkmanagement.googleapis.com
```

#### Expected Terminal Output:
```text
Operation "operations/acf.123456789" finished successfully.
```

#### How to Verify Configuration Correctness:
```bash
gcloud services list --enabled --filter="name:(iap.googleapis.com OR networkmanagement.googleapis.com)"
```

#### Expected Verification Output:
```text
NAME                                TITLE
iap.googleapis.com                  Identity-Aware Proxy API
networkmanagement.googleapis.com    Network Management API
```

---

### 5. Create Auto Mode VPC Network for Prototyping

```bash
gcloud compute networks create mynetwork --subnet-mode=auto
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/global/networks/mynetwork].
NAME       SUBNET_MODE  BGP_ROUTING_MODE  IPV4_RANGE  GATEWAY_IPV4
mynetwork  AUTO         REGIONAL
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks subnets list --network=mynetwork --format="table(name, region, ipCidrRange)"
```

#### Expected Verification Output:
```text
NAME       REGION           RANGE
mynetwork  us-central1      10.128.0.0/20
mynetwork  europe-west1     10.132.0.0/20
mynetwork  asia-east1       10.140.0.0/20
```

---

### 6. Provision Additional Custom VPC Networks (managementnet & privatenet) with Custom Subnets

```bash
# Create custom VPC networks
gcloud compute networks create managementnet --subnet-mode=custom
gcloud compute networks create privatenet --subnet-mode=custom

# Create custom subnet for managementnet
gcloud compute networks subnets create managementsubnet-us \
    --network=managementnet \
    --region=us-central1 \
    --range=10.240.0.0/20

# Create custom subnets for privatenet across regions
gcloud compute networks subnets create privatesubnet-us \
    --network=privatenet \
    --region=us-central1 \
    --range=172.16.0.0/24

gcloud compute networks subnets create privatesubnet-notus \
    --network=privatenet \
    --region=us-east1 \
    --range=172.20.0.0/20
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/global/networks/managementnet].
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/global/networks/privatenet].
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/subnetworks/managementsubnet-us].
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/subnetworks/privatesubnet-us].
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-east1/subnetworks/privatesubnet-notus].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks subnets list --sort-by=NETWORK --format="table(network, name, region, ipCidrRange)"
```

#### Expected Verification Output:
```text
NETWORK        NAME                 REGION       RANGE
managementnet  managementsubnet-us  us-central1  10.240.0.0/20
mynetwork      mynetwork            us-central1  10.128.0.0/20
privatenet     privatesubnet-notus  us-east1     172.20.0.0/20
privatenet     privatesubnet-us     us-central1  172.16.0.0/24
```

---

### 7. Cross-VPC Multi-Network Connectivity Audit (Internal IP Isolation vs External IP Access)

```bash
# Test 1: Ping External IP of VM in different VPC (Succeeds via public IP / firewall ICMP allow)
gcloud compute ssh mynet-us-vm --zone=us-central1-a --tunnel-through-iap --command="ping -c 3 34.122.10.55"

# Test 2: Ping Internal IP of VM in same VPC, different region (Succeeds via global VPC internal routing)
gcloud compute ssh mynet-us-vm --zone=us-central1-a --tunnel-through-iap --command="ping -c 3 10.132.0.2"

# Test 3: Ping Internal IP of VM in different VPC in same physical zone (Fails - 100% packet loss)
gcloud compute ssh mynet-us-vm --zone=us-central1-a --tunnel-through-iap --command="ping -c 3 10.240.0.2"
```

#### Expected Terminal Output:
```text
--- 34.122.10.55 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms

--- 10.132.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms

--- 10.240.0.2 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2048ms
```

#### How to Verify Configuration Correctness:
```bash
# Confirm that isolated VPCs require VPC Peering or Cloud VPN for internal IP communication
gcloud compute network-peerings list
```

#### Expected Verification Output:
```text
Listed 0 items.
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

### 2. Provision Identity-Aware Proxy (IAP) Ingress Firewall Rule (35.235.240.0/20)

```bash
gcloud compute firewall-rules create allow-iap-ssh \
    --network=mynetwork \
    --direction=INGRESS \
    --priority=1000 \
    --action=ALLOW \
    --rules=tcp:22 \
    --source-ranges=35.235.240.0/20 \
    --target-tags=iap-gce
```

#### Expected Terminal Output:
```text
Creating firewall rule...done.
NAME          NETWORK    DIRECTION  PRIORITY  ALLOW   DENY  DISABLED
allow-iap-ssh mynetwork  INGRESS    1000      tcp:22        False
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute firewall-rules describe allow-iap-ssh --format="yaml(name, sourceRanges, allowed, targetTags)"
```

#### Expected Verification Output:
```yaml
allowed:
- IPProtocol: tcp
  ports:
  - '22'
name: allow-iap-ssh
sourceRanges:
- 35.235.240.0/20
targetTags:
- iap-gce
```

---

### 3. Provision Multi-Protocol Combined Ingress Firewall Rules (ICMP, SSH, RDP)

```bash
# Provision firewall rule allowing ICMP, TCP 22 (SSH), and TCP 3389 (RDP) for privatenet
gcloud compute firewall-rules create privatenet-allow-icmp-ssh-rdp \
    --direction=INGRESS \
    --priority=1000 \
    --network=privatenet \
    --action=ALLOW \
    --rules=icmp,tcp:22,tcp:3389 \
    --source-ranges=0.0.0.0/0
```

#### Expected Terminal Output:
```text
Creating firewall rule...done.
NAME                           NETWORK     DIRECTION  PRIORITY  ALLOW                 DENY  DISABLED
privatenet-allow-icmp-ssh-rdp  privatenet  INGRESS    1000      icmp,tcp:22,tcp:3389        False
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute firewall-rules list --sort-by=NETWORK --format="table(network, name, priority, allow)"
```

#### Expected Verification Output:
```text
NETWORK        NAME                           PRIORITY  ALLOW
managementnet  managementnet-allow-icmp-ssh  1000      tcp:22,tcp:3389,icmp
privatenet     privatenet-allow-icmp-ssh-rdp  1000      icmp,tcp:22,tcp:3389
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

### 3. Promote Existing Ephemeral External IP Address to Static External IP

```bash
gcloud compute addresses create promoted-static-ip \
    --region=us-central1 \
    --addresses=35.202.88.19
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/addresses/promoted-static-ip].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute addresses describe promoted-static-ip --region=us-central1 --format="yaml(name, address, status)"
```

#### Expected Verification Output:
```yaml
address: 35.202.88.19
name: promoted-static-ip
status: IN_USE
```

---

### 4. Audit & Release Unassigned Static External IPs to Eliminate Surcharge Billing

```bash
gcloud compute addresses list \
    --filter="status=RESERVED AND users:*" \
    --format="table(name, region, address, status)"
```

#### Expected Terminal Output:
```text
NAME                 REGION       ADDRESS        STATUS
abandoned-legacy-ip  us-central1  34.122.10.55   RESERVED
unused-test-ip       us-central1  35.202.99.11   RESERVED
```

#### How to Verify Configuration Correctness & Release Unassigned IPs:
```bash
gcloud compute addresses delete abandoned-legacy-ip unused-test-ip --region=us-central1 --quiet
```

#### Expected Verification Output:
```text
Deleted [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/addresses/abandoned-legacy-ip].
Deleted [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/addresses/unused-test-ip].
```

---

### 5. Provision Private VM Instance Without External IP (--no-address)

```bash
gcloud compute instances create private-backend-db \
    --zone=us-central1-a \
    --machine-type=n2-standard-4 \
    --subnet=prod-subnet-us-central1 \
    --no-address
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/zones/us-central1-a/instances/private-backend-db].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute instances describe private-backend-db --zone=us-central1-a \
    --format="yaml(name, networkInterfaces[0].accessConfigs)"
```

#### Expected Verification Output:
```yaml
name: private-backend-db
```

---

### 6. Custom Static Internal IP Assignment Within Subnet Range

```bash
gcloud compute instances create custom-ip-vm \
    --zone=us-central1-a \
    --machine-type=e2-micro \
    --subnet=prod-subnet-us-central1 \
    --private-network-ip=10.1.0.25
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/zones/us-central1-a/instances/custom-ip-vm].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute instances describe custom-ip-vm --zone=us-central1-a \
    --format="value(networkInterfaces[0].networkIP)"
```

#### Expected Verification Output:
```text
10.1.0.25
```

---

## Category 7: Bring Your Own IP (BYOIP) & Internal DNS Verification

### 1. Provision BYOIP Public Advertised Prefix (PAP) (/24 Minimum Block)

```bash
gcloud compute public-advertised-prefixes create my-company-byoip-pap \
    --dns-verification-ip=198.51.100.1 \
    --range=198.51.100.0/24
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/global/publicAdvertisedPrefixes/my-company-byoip-pap].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute public-advertised-prefixes describe my-company-byoip-pap --format="yaml(name, ipCidrRange, status)"
```

#### Expected Verification Output:
```yaml
ipCidrRange: 198.51.100.0/24
name: my-company-byoip-pap
status: INITIAL
```

---

### 2. Verify Network-Scoped Internal DNS Resolution Inside VM

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

## Category 12: Private Google Access & Egress Cost Optimization

### 1. Enable Private Google Access on Subnet to Eliminate External Egress Charges

When VMs without external IPs connect to Google APIs (Cloud Storage, BigQuery, Maps), Private Google Access routes traffic internally across Google's private backbone for $0.00/GB egress cost.

```bash
gcloud compute networks subnets update prod-subnet-us-central1 \
    --region=us-central1 \
    --enable-private-ip-google-access
```

#### Expected Terminal Output:
```text
Updated [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/subnetworks/prod-subnet-us-central1].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks subnets describe prod-subnet-us-central1 \
    --region=us-central1 \
    --format="value(privateIpGoogleAccess)"
```

#### Expected Verification Output:
```text
True
```

---

### 2. Audit Intra-Zone External IP Communication Leaks

```bash
gcloud compute instances list --format="table(name, zone, networkInterfaces[0].networkIP, networkInterfaces[0].accessConfigs[0].natIP)"
```

#### Expected Terminal Output:
```text
NAME               ZONE          PRIMARY_IP  EXTERNAL_IP
web-frontend-vm-1  us-central1-a 10.1.0.2    35.202.88.19
backend-api-vm-2   us-central1-a 10.1.0.3    34.122.10.55
```

#### How to Verify & Remediate:
```bash
# Test internal DNS connectivity to ensure internal communication is used instead of external IPs
gcloud compute ssh web-frontend-vm-1 --zone=us-central1-a --command="curl -I http://backend-api-vm-2.us-central1-a.c.YOUR_PROJECT.internal"
```

#### Expected Verification Output:
```text
HTTP/1.1 200 OK
```

---

## Category 12: Cloud VPN Provisioning (Classic VPN, HA VPN, BGP Cloud Router, AWS Interop & GCP-to-GCP HA VPN)

### 1. Classic VPN Gateway Provisioning (99.9% SLA, Target Gateway, ESP / UDP Ports & MTU 1460)

```bash
# Step 1: Reserve regional static external IP address
gcloud compute addresses create classic-vpn-ip --region=us-central1

# Step 2: Create Target Classic VPN Gateway attached to custom VPC
gcloud compute target-vpn-gateways create classic-vpn-gw \
    --network=gcd-prod-custom-vpc \
    --region=us-central1

# Step 3: Create forwarding rules for ESP, UDP 500, and UDP 4500
gcloud compute forwarding-rules create classic-fr-esp \
    --region=us-central1 \
    --ip-protocol=ESP \
    --address=classic-vpn-ip \
    --target-vpn-gateway=classic-vpn-gw

gcloud compute forwarding-rules create classic-fr-udp500 \
    --region=us-central1 \
    --ip-protocol=UDP \
    --ports=500 \
    --address=classic-vpn-ip \
    --target-vpn-gateway=classic-vpn-gw

gcloud compute forwarding-rules create classic-fr-udp4500 \
    --region=us-central1 \
    --ip-protocol=UDP \
    --ports=4500 \
    --address=classic-vpn-ip \
    --target-vpn-gateway=classic-vpn-gw

# Step 4: Create Classic IPsec VPN Tunnel (Peer MTU <= 1460 bytes)
gcloud compute vpn-tunnels create classic-tunnel-1 \
    --peer-address=203.0.113.5 \
    --shared-secret=MySecretPass123 \
    --target-vpn-gateway=classic-vpn-gw \
    --region=us-central1 \
    --ike-version=2 \
    --local-traffic-selector=10.1.0.0/16 \
    --remote-traffic-selector=192.168.1.0/24

# Step 5: Create static route directing remote subnet traffic into VPN tunnel
gcloud compute routes create route-to-onprem \
    --destination-range=192.168.1.0/24 \
    --network=gcd-prod-custom-vpc \
    --next-hop-vpn-tunnel=classic-tunnel-1 \
    --next-hop-vpn-tunnel-region=us-central1
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/targetVpnGateways/classic-vpn-gw].
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/vpnTunnels/classic-tunnel-1].
NAME              REGION       GATEWAY         VPN_INTERFACE
classic-tunnel-1  us-central1  classic-vpn-gw
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute vpn-tunnels describe classic-tunnel-1 --region=us-central1 --format="yaml(status, detailedStatus, peerIp)"
```

#### Expected Verification Output:
```yaml
detailedStatus: Tunnel is up and operating normally.
peerIp: 203.0.113.5
status: ESTABLISHED
```

---

### 2. High Availability (HA) VPN Gateway Provisioning (99.99% SLA, Dual Interfaces `if0`/`if1` & BGP Cloud Router)

```bash
# Step 1: Create HA VPN Gateway (Google Cloud auto-allocates two regional external IPs for if0 and if1)
gcloud compute vpn-gateways create ha-vpn-gw-01 \
    --network=gcd-prod-custom-vpc \
    --region=us-central1

# Step 2: Create Cloud Router to handle dynamic BGP routing
gcloud compute routers create vpn-cloud-router \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --asn=65001

# Step 3: Create External Peer VPN Gateway resource representing two on-premises devices (TWO_IPS_REDUNDANCY)
gcloud compute external-vpn-gateways create onprem-peer-gateway \
    --interfaces=0=203.0.113.10,1=203.0.113.11 \
    --redundancy-type=TWO_IPS_REDUNDANCY

# Step 4: Create HA VPN Tunnel 0 (Interface 0 -> Peer Interface 0)
gcloud compute vpn-tunnels create ha-tunnel-0 \
    --vpn-gateway=ha-vpn-gw-01 \
    --interface=0 \
    --peer-external-gateway=onprem-peer-gateway \
    --peer-external-gateway-interface=0 \
    --shared-secret=SecretPassA123 \
    --router=vpn-cloud-router \
    --region=us-central1

# Step 5: Create HA VPN Tunnel 1 (Interface 1 -> Peer Interface 1)
gcloud compute vpn-tunnels create ha-tunnel-1 \
    --vpn-gateway=ha-vpn-gw-01 \
    --interface=1 \
    --peer-external-gateway=onprem-peer-gateway \
    --peer-external-gateway-interface=1 \
    --shared-secret=SecretPassB123 \
    --router=vpn-cloud-router \
    --region=us-central1

# Step 6: Configure BGP Router Interfaces with Link-Local Addresses (169.254.0.0/16 range)
gcloud compute routers add-interface vpn-cloud-router \
    --interface-name=router-if-0 \
    --ip-address=169.254.0.1 \
    --mask-length=30 \
    --vpn-tunnel=ha-tunnel-0 \
    --region=us-central1

gcloud compute routers add-interface vpn-cloud-router \
    --interface-name=router-if-1 \
    --ip-address=169.254.1.1 \
    --mask-length=30 \
    --vpn-tunnel=ha-tunnel-1 \
    --region=us-central1

# Step 7: Configure BGP Peers on Cloud Router
gcloud compute routers add-bgp-peer vpn-cloud-router \
    --peer-name=bgp-peer-0 \
    --interface=router-if-0 \
    --peer-ip-address=169.254.0.2 \
    --peer-asn=65002 \
    --region=us-central1

gcloud compute routers add-bgp-peer vpn-cloud-router \
    --peer-name=bgp-peer-1 \
    --interface=router-if-1 \
    --peer-ip-address=169.254.1.2 \
    --peer-asn=65002 \
    --region=us-central1
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/vpnGateways/ha-vpn-gw-01].
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/global/externalVpnGateways/onprem-peer-gateway].
Creating VPN tunnels...done.
```

#### How to Verify Configuration Correctness:
```bash
# Verify BGP session status across both tunnels
gcloud compute routers get-status vpn-cloud-router --region=us-central1 --format="yaml(bgpPeerStatus)"
```

#### Expected Verification Output:
```yaml
bgpPeerStatus:
- ipAddress: 169.254.0.1
  name: bgp-peer-0
  numLearnedRoutes: 5
  peerIpAddress: 169.254.0.2
  state: ESTABLISHED
- ipAddress: 169.254.1.1
  name: bgp-peer-1
  numLearnedRoutes: 5
  peerIpAddress: 169.254.1.2
  state: ESTABLISHED
```

---

### 3. HA VPN to AWS Interop Topology (4 Tunnels, ECMP Routing & External Gateway)

```bash
# Step 1: Create External Peer VPN Gateway representing AWS Virtual Private Gateways (FOUR_IPS_REDUNDANCY)
gcloud compute external-vpn-gateways create aws-peer-gateway \
    --interfaces=0=52.93.1.10,1=52.93.1.11,2=52.93.2.10,3=52.93.2.11 \
    --redundancy-type=FOUR_IPS_REDUNDANCY

# Step 2: Create 4 HA VPN Tunnels targeting AWS gateway interfaces
gcloud compute vpn-tunnels create aws-tunnel-0 \
    --vpn-gateway=ha-vpn-gw-01 --interface=0 \
    --peer-external-gateway=aws-peer-gateway --peer-external-gateway-interface=0 \
    --shared-secret=AWSPassword123 --router=vpn-cloud-router --region=us-central1

gcloud compute vpn-tunnels create aws-tunnel-1 \
    --vpn-gateway=ha-vpn-gw-01 --interface=0 \
    --peer-external-gateway=aws-peer-gateway --peer-external-gateway-interface=1 \
    --shared-secret=AWSPassword123 --router=vpn-cloud-router --region=us-central1

gcloud compute vpn-tunnels create aws-tunnel-2 \
    --vpn-gateway=ha-vpn-gw-01 --interface=1 \
    --peer-external-gateway=aws-peer-gateway --peer-external-gateway-interface=2 \
    --shared-secret=AWSPassword123 --router=vpn-cloud-router --region=us-central1

gcloud compute vpn-tunnels create aws-tunnel-3 \
    --vpn-gateway=ha-vpn-gw-01 --interface=1 \
    --peer-external-gateway=aws-peer-gateway --peer-external-gateway-interface=3 \
    --shared-secret=AWSPassword123 --router=vpn-cloud-router --region=us-central1
```

---

### 4. GCP VPC-to-VPC Interconnect via HA VPN (Two GCP HA VPN Gateways Connected)

```bash
# Step 1: Create HA VPN Gateway in VPC-A and VPC-B
gcloud compute vpn-gateways create ha-gw-vpc-a --network=gcd-prod-custom-vpc --region=us-central1
gcloud compute vpn-gateways create ha-gw-vpc-b --network=gcd-dev-auto-vpc --region=us-central1

# Step 2: Extract auto-allocated external IPs for both gateways
export IP_A_IF0=$(gcloud compute vpn-gateways describe ha-gw-vpc-a --region=us-central1 --format="value(vpnInterfaces[0].ipAddress)")
export IP_A_IF1=$(gcloud compute vpn-gateways describe ha-gw-vpc-a --region=us-central1 --format="value(vpnInterfaces[1].ipAddress)")
export IP_B_IF0=$(gcloud compute vpn-gateways describe ha-gw-vpc-b --region=us-central1 --format="value(vpnInterfaces[0].ipAddress)")
export IP_B_IF1=$(gcloud compute vpn-gateways describe ha-gw-vpc-b --region=us-central1 --format="value(vpnInterfaces[1].ipAddress)")

# Step 3: Create Cloud Routers for both VPCs
gcloud compute routers create router-vpc-a --network=gcd-prod-custom-vpc --region=us-central1 --asn=65010
gcloud compute routers create router-vpc-b --network=gcd-dev-auto-vpc --region=us-central1 --asn=65020

# Step 4: Create interconnecting VPN tunnels (if0 -> if0 and if1 -> if1)
gcloud compute vpn-tunnels create tunnel-a-to-b-0 \
    --vpn-gateway=ha-gw-vpc-a --interface=0 \
    --peer-gcp-gateway=ha-gw-vpc-b --peer-interface=0 \
    --shared-secret=InterconnectSecret123 --router=router-vpc-a --region=us-central1

gcloud compute vpn-tunnels create tunnel-a-to-b-1 \
    --vpn-gateway=ha-gw-vpc-a --interface=1 \
    --peer-gcp-gateway=ha-gw-vpc-b --peer-interface=1 \
    --shared-secret=InterconnectSecret456 --router=router-vpc-a --region=us-central1
```

---

### 5. Multi-Region HA VPN Lab Blueprint & Global Dynamic Routing Mode (`vpc-demo` <-> `on-prem` with Failover Testing)

This complete operational blueprint deploys two simulated networks (`vpc-demo` with multi-region subnets and `on-prem`), configures HA VPN gateways, enables Global Dynamic Routing, tests inter-region ping, and validates HA tunnel failover.

```bash
# Step 1: Create Networks (vpc-demo and on-prem)
gcloud compute networks create vpc-demo --subnet-mode=custom
gcloud compute networks create on-prem --subnet-mode=custom

# Step 2: Create Subnets (Multi-region in vpc-demo)
gcloud compute networks subnets create vpc-demo-subnet1 --network=vpc-demo --range=10.1.1.0/24 --region=us-central1
gcloud compute networks subnets create vpc-demo-subnet2 --network=vpc-demo --range=10.2.1.0/24 --region=us-west1
gcloud compute networks subnets create on-prem-subnet1 --network=on-prem --range=192.168.1.0/24 --region=us-central1

# Step 3: Create Firewall Rules
gcloud compute firewall-rules create vpc-demo-allow-custom --network=vpc-demo --allow=tcp:0-65535,udp:0-65535,icmp --source-ranges=10.0.0.0/8
gcloud compute firewall-rules create vpc-demo-allow-ssh-icmp --network=vpc-demo --allow=tcp:22,icmp
gcloud compute firewall-rules create on-prem-allow-custom --network=on-prem --allow=tcp:0-65535,udp:0-65535,icmp --source-ranges=192.168.0.0/16
gcloud compute firewall-rules create on-prem-allow-ssh-icmp --network=on-prem --allow=tcp:22,icmp
gcloud compute firewall-rules create vpc-demo-allow-subnets-from-on-prem --network=vpc-demo --allow=tcp,udp,icmp --source-ranges=192.168.1.0/24
gcloud compute firewall-rules create on-prem-allow-subnets-from-vpc-demo --network=on-prem --allow=tcp,udp,icmp --source-ranges=10.1.1.0/24,10.2.1.0/24

# Step 4: Provision Compute VM Instances
gcloud compute instances create vpc-demo-instance1 --machine-type=e2-medium --zone=us-central1-a --subnet=vpc-demo-subnet1
gcloud compute instances create vpc-demo-instance2 --machine-type=e2-medium --zone=us-west1-a --subnet=vpc-demo-subnet2
gcloud compute instances create on-prem-instance1 --machine-type=e2-medium --zone=us-central1-b --subnet=on-prem-subnet1

# Step 5: Provision HA VPN Gateways & Cloud Routers
gcloud compute vpn-gateways create vpc-demo-vpn-gw1 --network=vpc-demo --region=us-central1
gcloud compute vpn-gateways create on-prem-vpn-gw1 --network=on-prem --region=us-central1
gcloud compute routers create vpc-demo-router1 --region=us-central1 --network=vpc-demo --asn=65001
gcloud compute routers create on-prem-router1 --region=us-central1 --network=on-prem --asn=65002

# Step 6: Create HA VPN Tunnels (Interface 0 -> Interface 0 and Interface 1 -> Interface 1)
gcloud compute vpn-tunnels create vpc-demo-tunnel0 --peer-gcp-gateway=on-prem-vpn-gw1 --region=us-central1 --ike-version=2 --shared-secret=SharedSecretKey123 --router=vpc-demo-router1 --vpn-gateway=vpc-demo-vpn-gw1 --interface=0
gcloud compute vpn-tunnels create vpc-demo-tunnel1 --peer-gcp-gateway=on-prem-vpn-gw1 --region=us-central1 --ike-version=2 --shared-secret=SharedSecretKey123 --router=vpc-demo-router1 --vpn-gateway=vpc-demo-vpn-gw1 --interface=1
gcloud compute vpn-tunnels create on-prem-tunnel0 --peer-gcp-gateway=vpc-demo-vpn-gw1 --region=us-central1 --ike-version=2 --shared-secret=SharedSecretKey123 --router=on-prem-router1 --vpn-gateway=on-prem-vpn-gw1 --interface=0
gcloud compute vpn-tunnels create on-prem-tunnel1 --peer-gcp-gateway=vpc-demo-vpn-gw1 --region=us-central1 --ike-version=2 --shared-secret=SharedSecretKey123 --router=on-prem-router1 --vpn-gateway=on-prem-vpn-gw1 --interface=1

# Step 7: Configure BGP Router Interfaces & Peers (Link-Local IPs 169.254.0.0/16)
gcloud compute routers add-interface vpc-demo-router1 --interface-name=if-tunnel0-to-on-prem --ip-address=169.254.0.1 --mask-length=30 --vpn-tunnel=vpc-demo-tunnel0 --region=us-central1
gcloud compute routers add-bgp-peer vpc-demo-router1 --peer-name=bgp-on-prem-tunnel0 --interface=if-tunnel0-to-on-prem --peer-ip-address=169.254.0.2 --peer-asn=65002 --region=us-central1
gcloud compute routers add-interface vpc-demo-router1 --interface-name=if-tunnel1-to-on-prem --ip-address=169.254.1.1 --mask-length=30 --vpn-tunnel=vpc-demo-tunnel1 --region=us-central1
gcloud compute routers add-bgp-peer vpc-demo-router1 --peer-name=bgp-on-prem-tunnel1 --interface=if-tunnel1-to-on-prem --peer-ip-address=169.254.1.2 --peer-asn=65002 --region=us-central1

gcloud compute routers add-interface on-prem-router1 --interface-name=if-tunnel0-to-vpc-demo --ip-address=169.254.0.2 --mask-length=30 --vpn-tunnel=on-prem-tunnel0 --region=us-central1
gcloud compute routers add-bgp-peer on-prem-router1 --peer-name=bgp-vpc-demo-tunnel0 --interface=if-tunnel0-to-vpc-demo --peer-ip-address=169.254.0.1 --peer-asn=65001 --region=us-central1
gcloud compute routers add-interface on-prem-router1 --interface-name=if-tunnel1-to-vpc-demo --ip-address=169.254.1.2 --mask-length=30 --vpn-tunnel=on-prem-tunnel1 --region=us-central1
gcloud compute routers add-bgp-peer on-prem-router1 --peer-name=bgp-vpc-demo-tunnel1 --interface=if-tunnel1-to-vpc-demo --peer-ip-address=169.254.1.1 --peer-asn=65001 --region=us-central1

# Step 8: Update VPC to Global Dynamic Routing Mode (Enables Cross-Region HA VPN Routing)
gcloud compute networks update vpc-demo --bgp-routing-mode=GLOBAL

# Step 9: Verify Connectivity from on-prem to multi-region subnets via SSH ping
gcloud compute ssh on-prem-instance1 --zone=us-central1-b --command="ping -c 4 10.1.1.2"
gcloud compute ssh on-prem-instance1 --zone=us-central1-b --command="ping -c 4 10.2.1.2"

# Step 10: Test HA Tunnel Failover (Delete Tunnel 0 and verify packet delivery over Tunnel 1)
gcloud compute vpn-tunnels delete vpc-demo-tunnel0 --region=us-central1 --quiet
gcloud compute ssh on-prem-instance1 --zone=us-central1-b --command="ping -c 4 10.1.1.2"
```

#### Expected Terminal Output:
```text
Updated [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/global/networks/vpc-demo].
PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
64 bytes from 10.1.1.2: icmp_seq=1 ttl=62 time=2.01 ms
64 bytes from 10.1.1.2: icmp_seq=2 ttl=62 time=1.71 ms
--- 10.1.1.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
```

#### Complete Lab Resource Teardown Master Command:
```bash
gcloud compute vpn-tunnels delete on-prem-tunnel0 --region=us-central1 --quiet && \
gcloud compute vpn-tunnels delete vpc-demo-tunnel1 --region=us-central1 --quiet && \
gcloud compute vpn-tunnels delete on-prem-tunnel1 --region=us-central1 --quiet && \
gcloud compute routers remove-bgp-peer vpc-demo-router1 --peer-name=bgp-on-prem-tunnel0 --region=us-central1 --quiet && \
gcloud compute routers remove-bgp-peer vpc-demo-router1 --peer-name=bgp-on-prem-tunnel1 --region=us-central1 --quiet && \
gcloud compute routers remove-bgp-peer on-prem-router1 --peer-name=bgp-vpc-demo-tunnel0 --region=us-central1 --quiet && \
gcloud compute routers remove-bgp-peer on-prem-router1 --peer-name=bgp-vpc-demo-tunnel1 --region=us-central1 --quiet && \
gcloud compute routers delete on-prem-router1 --region=us-central1 --quiet && \
gcloud compute routers delete vpc-demo-router1 --region=us-central1 --quiet && \
gcloud compute vpn-gateways delete vpc-demo-vpn-gw1 --region=us-central1 --quiet && \
gcloud compute vpn-gateways delete on-prem-vpn-gw1 --region=us-central1 --quiet && \
gcloud compute instances delete vpc-demo-instance1 --zone=us-central1-a --quiet && \
gcloud compute instances delete vpc-demo-instance2 --zone=us-west1-a --quiet && \
gcloud compute instances delete on-prem-instance1 --zone=us-central1-b --quiet && \
gcloud compute firewall-rules delete vpc-demo-allow-custom --quiet && \
gcloud compute firewall-rules delete on-prem-allow-subnets-from-vpc-demo --quiet && \
gcloud compute firewall-rules delete on-prem-allow-ssh-icmp --quiet && \
gcloud compute firewall-rules delete on-prem-allow-custom --quiet && \
gcloud compute firewall-rules delete vpc-demo-allow-subnets-from-on-prem --quiet && \
gcloud compute firewall-rules delete vpc-demo-allow-ssh-icmp --quiet && \
gcloud compute networks subnets delete vpc-demo-subnet1 --region=us-central1 --quiet && \
gcloud compute networks subnets delete vpc-demo-subnet2 --region=us-west1 --quiet && \
gcloud compute networks subnets delete on-prem-subnet1 --region=us-central1 --quiet && \
gcloud compute networks delete vpc-demo --quiet && \
gcloud compute networks delete on-prem --quiet
```
```

---

## Category 13: Cloud Interconnect & Peering Services (Dedicated Interconnect, Partner Interconnect, Direct Peering & VLAN Attachments)

### 1. Dedicated Interconnect Provisioning (Direct Physical 10G/100G Circuits, Layer 2 RFC 1918 Private IP Access)

```bash
# Step 1: Request Dedicated Interconnect Circuit at Google Colocation Location
gcloud compute interconnects create dedicated-interconnect-01 \
    --interconnect-type=DEDICATED \
    --link-type=LINK_TYPE_ETHERNET_10G_LR \
    --requested-link-count=1 \
    --location=equinix-sv1 \
    --admin-enabled \
    --description="Primary 10G Dedicated Interconnect Circuit to On-Prem Data Center"

# Step 2: Provision Cloud Router for Interconnect BGP Route Exchange
gcloud compute routers create interconnect-router-uscentral1 \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --asn=65001

# Step 3: Provision VLAN Attachment (InterconnectAttachment) on Dedicated Interconnect
gcloud compute interconnects attachments create dedicated-vlan-attachment-01 \
    --interconnect=dedicated-interconnect-01 \
    --router=interconnect-router-uscentral1 \
    --region=us-central1 \
    --vlan=100 \
    --candidate-subnets=169.254.10.0/29

# Step 4: Configure BGP Interface and Peer on Cloud Router
gcloud compute routers add-interface interconnect-router-uscentral1 \
    --interface-name=if-dedicated-vlan \
    --ip-address=169.254.10.1 \
    --mask-length=29 \
    --interconnect-attachment=dedicated-vlan-attachment-01 \
    --region=us-central1

gcloud compute routers add-bgp-peer interconnect-router-uscentral1 \
    --peer-name=bgp-peer-dedicated \
    --interface=if-dedicated-vlan \
    --peer-ip-address=169.254.10.2 \
    --peer-asn=65002 \
    --region=us-central1
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/global/interconnects/dedicated-interconnect-01].
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/interconnectAttachments/dedicated-vlan-attachment-01].
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute interconnects attachments describe dedicated-vlan-attachment-01 --region=us-central1 --format="yaml(state, operationalStatus)"
```

#### Expected Verification Output:
```yaml
operationalStatus: ACTIVE
state: ACTIVE
```

---

### 2. Partner Interconnect Provisioning (Layer 2 / Layer 3 Connection via Service Provider)

```bash
# Step 1: Create Partner Interconnect Attachment (Generates Pairing Key)
gcloud compute interconnects attachments create partner-vlan-attachment-01 \
    --edge-availability-domain=AVAILABILITY_DOMAIN_1 \
    --router=interconnect-router-uscentral1 \
    --region=us-central1 \
    --type=PARTNER

# Step 2: Retrieve Pairing Key to provide to Partner (e.g. Equinix Fabric / Megaport)
gcloud compute interconnects attachments describe partner-vlan-attachment-01 \
    --region=us-central1 \
    --format="value(pairingKey)"
```

#### Expected Terminal Output:
```text
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/interconnectAttachments/partner-vlan-attachment-01].
7b4c9e12-3a5f-4d11-8e99-0a1b2c3d4e5f/us-central1/1
```

---

### 3. Direct Peering & Carrier Peering (Layer 3 Public IP Access to Google Workspace, YouTube & Cloud APIs)

Direct Peering establishes BGP sessions directly at Google Edge Point of Presence (PoP) locations for access to public IP services (Google Workspace, APIs, YouTube). It does not directly route to RFC 1918 internal IPs inside a VPC unless combined with Cloud VPN.

```bash
# Verify active BGP peering sessions over Google Edge locations
gcloud compute routes list --filter="nextHopPeering:*"

# List active Interconnect Attachments and Peering Status
gcloud compute interconnects attachments list
```

---

### 4. Cross-Cloud Interconnect Provisioning (Dedicated Ports to AWS, Azure, OCI & Alibaba Cloud)

Cross-Cloud Interconnect provides 10 Gbps or 100 Gbps dedicated physical ports connecting Google Cloud directly to supported partner clouds (AWS, Azure, OCI, Alibaba Cloud). Google manages the link up to the target cloud provider border.

```bash
# Step 1: Create Cross-Cloud Interconnect ports targeting AWS
gcloud compute interconnects create cross-cloud-aws-01 \
    --interconnect-type=DEDICATED \
    --link-type=LINK_TYPE_ETHERNET_10G_LR \
    --requested-link-count=1 \
    --location=equinix-sv1 \
    --remote-location=aws-us-west-2 \
    --admin-enabled \
    --description="10G Cross-Cloud Interconnect Port to AWS us-west-2"

# Step 2: Create Interconnect Attachment for Cross-Cloud Link
gcloud compute interconnects attachments create cross-cloud-vlan-attachment-01 \
    --interconnect=cross-cloud-aws-01 \
    --router=interconnect-router-uscentral1 \
    --region=us-central1 \
    --vlan=200
```

---

### 5. Cloud HA VPN over Interconnect (High Bandwidth + Google-Managed IPsec Encryption)

Combines the high-throughput SLA of Interconnect (Dedicated or Partner) with Google-managed IPsec encryption for sensitive RFC 1918 traffic.

```bash
# Step 1: Create Regional Encrypted Interconnect Attachment
gcloud compute interconnects attachments create vpn-over-interconnect-attachment-01 \
    --interconnect=dedicated-interconnect-01 \
    --router=interconnect-router-uscentral1 \
    --region=us-central1 \
    --encryption=IPSEC \
    --ipsec-internal-addresses=169.254.20.1

# Step 2: Create HA VPN Gateway over Interconnect Attachment
gcloud compute vpn-gateways create ha-vpn-over-interconnect-gw \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --vpn-interfaces=0=attachment=vpn-over-interconnect-attachment-01

# Step 3: Create IPsec Tunnel running over Interconnect Attachment
gcloud compute vpn-tunnels create ha-vpn-over-interconnect-tunnel-0 \
    --vpn-gateway=ha-vpn-over-interconnect-gw \
    --interface=0 \
    --peer-address=169.254.20.2 \
    --shared-secret=InterconnectEncryptedKey123 \
    --router=interconnect-router-uscentral1 \
    --region=us-central1
```

---


## Category 14: Exhaustive Failure Diagnosis & Resolution Matrix

| Error Code / Status | Root Cause | Diagnosis Command | Immediate Resolution Command |
| :--- | :--- | :--- | :--- |
| **`NO_MORE_NETWORKS_AVAILABLE`** | Attempted to create VM in a project with no VPC networks (e.g. after deleting default VPC). | `gcloud compute networks list` | Create VPC network: `gcloud compute networks create VPC_NAME --subnet-mode=custom`. |
| **`CROSS_VPC_INTERNAL_IP_ISOLATION`** | Internal ping between VMs in different VPC networks failed despite being in the same zone. | `gcloud compute instances list --format="yaml(name, networkInterfaces)"` | VPCs are logically isolated private domains. Use External IPs or set up VPC Network Peering / Cloud VPN. |
| **`INTRAZONE_EXTERNAL_IP_COST_LEAK`** | Intra-zone VM traffic using External IPs is billed at Inter-Zone egress rate ($0.01/GB). | `gcloud compute instances list --format="yaml(name, networkInterfaces)"` | Reconfigure internal application configs to resolve internal DNS (`vm-2.zone.c.PROJECT.internal`) or internal IP (`10.x.x.x`). |
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
| **`VPN_TUNNEL_ESTABLISHMENT_FAILED`** | IKE pre-shared key mismatch, peer IP unreachable, or firewall blocking UDP 500/4500. | `gcloud compute vpn-tunnels describe TUNNEL --region=REGION` | Verify peer external IP, pre-shared key secret, and ensure on-premises firewall allows UDP 500/4500 and ESP. |
| **`BGP_PEER_DOWN_LINK_LOCAL`** | BGP session stuck in `CONNECT` or `ACTIVE` state over link-local address. | `gcloud compute routers get-status ROUTER --region=REGION` | Ensure link-local IP (`169.254.x.x/30`) matches on peer gateway and BGP ASNs are configured correctly. |
| **`HA_VPN_SLA_VIOLATION_SINGLE_INTERFACE`** | Only 1 interface configured on HA VPN gateway, forfeiting the 99.99% SLA. | `gcloud compute vpn-gateways describe HA_GW --region=REGION` | Create 2 or 4 tunnels connecting both `interface 0` and `interface 1` to peer gateway. |
| **`MTU_EXCEEDED_1460_VPN`** | Packet dropped or fragmented over Classic/HA VPN because peer MTU > 1460 bytes. | `gcloud compute ssh VM --command="ping -s 1432 -M do PEER_IP"` | Configure on-premises VPN gateway and host interface MTU to $\le 1460$ bytes due to IPsec encapsulation overhead. |
| **`INTERCONNECT_ATTACHMENT_PENDING`** | Partner Interconnect attachment created but pairing key not configured on partner portal. | `gcloud compute interconnects attachments describe ATTACHMENT --region=REGION` | Copy pairing key and submit to service provider partner (Equinix/Megaport) to activate VLAN circuit. |
| **`INTERCONNECT_SLA_METRO_SINGLE`** | Dedicated Interconnect circuits provisioned in single colocation facility, missing 99.99% SLA. | `gcloud compute interconnects list` | To achieve 99.99% SLA, provision at least 2 circuits in Metro 1 and 2 circuits in Metro 2 across distinct edge availability domains. |

---

## Category 15: Master End-to-End Sequential Deployment Commands

This section provides pure `gcloud` shell commands that provision an entire Google Cloud VPC network architecture in **strict logical dependency order**: Network → Subnet → Firewall Rules → Static IP → Virtual Router → Cloud NAT Gateway → Custom Routes → Private Cloud DNS → Workload VM.

### Execution Dependency Sequence
```text
1. Enable Required GCP APIs (Compute Engine, IAP, Cloud DNS, Network Management)
   ↓
2. Create Custom Mode VPC Network (Global BGP Routing, Custom MTU 1460)
   ↓
3. Create Regional Custom Subnets (Primary IPv4, Secondary GKE Ranges & Dual-Stack IPv6)
   ↓
4. Reserve Regional Static External IP (for Cloud NAT)
   ↓
5. Provision Baseline VPC Firewall Rules (IAP SSH Allow, Intra-Subnet Mesh, Health Checks)
   ↓
6. Provision Virtual Router & Cloud NAT Gateway (Secure Outbound Internet Access for Private VMs)
   ↓
7. Configure Custom VPC Static Routes (Default & Custom NVA Next-Hops)
   ↓
8. Provision Cloud DNS Private Managed Zone & A-Records
   ↓
9. Deploy Workload VM Instances (Private Internal IPs Only, Attached to Custom Subnet)
   ↓
10. Execute Automated Verification Suite
```

---

### 1. Step-by-Step Master Sequential Commands

Execute these shell commands in sequence:

```bash
# Step 1: Enable Required GCP Service APIs
gcloud services enable \
    compute.googleapis.com \
    iap.googleapis.com \
    dns.googleapis.com \
    networkmanagement.googleapis.com

# Step 2: Provision Custom Mode VPC Network
gcloud compute networks create gcd-prod-custom-vpc \
    --subnet-mode=custom \
    --bgp-routing-mode=global \
    --mtu=1460

# Step 3: Provision Primary & Dual-Stack Regional Subnets
gcloud compute networks subnets create prod-subnet-us-central1 \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --range=10.1.0.0/24 \
    --enable-private-ip-google-access \
    --secondary-range=pod-range=10.100.0.0/16,service-range=10.200.0.0/20

gcloud compute networks subnets create prod-dualstack-subnet \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --range=10.2.0.0/24 \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=EXTERNAL \
    --enable-private-ip-google-access

# Step 4: Reserve Static Regional External IP for Cloud NAT
gcloud compute addresses create gcd-nat-static-ip-uscentral1 \
    --region=us-central1 \
    --network-tier=PREMIUM

# Step 5: Provision VPC Ingress Firewall Rules
gcloud compute firewall-rules create gcd-prod-custom-vpc-allow-iap-ssh \
    --network=gcd-prod-custom-vpc \
    --direction=INGRESS \
    --priority=1000 \
    --action=ALLOW \
    --rules=tcp:22 \
    --source-ranges=35.235.240.0/20 \
    --target-tags=iap-enabled

gcloud compute firewall-rules create gcd-prod-custom-vpc-allow-internal-mesh \
    --network=gcd-prod-custom-vpc \
    --direction=INGRESS \
    --priority=1000 \
    --action=ALLOW \
    --rules=tcp,udp,icmp \
    --source-ranges=10.1.0.0/16

gcloud compute firewall-rules create gcd-prod-custom-vpc-allow-health-checks \
    --network=gcd-prod-custom-vpc \
    --direction=INGRESS \
    --priority=1000 \
    --action=ALLOW \
    --rules=tcp:80,tcp:443 \
    --source-ranges=35.191.0.0/16,130.211.0.0/22 \
    --target-tags=web-backend

# Step 6: Provision Cloud Router & Cloud NAT Gateway
gcloud compute routers create gcd-nat-router-uscentral1 \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --asn=65001

gcloud compute routers nats create gcd-nat-gateway-uscentral1 \
    --router=gcd-nat-router-uscentral1 \
    --region=us-central1 \
    --nat-custom-manual-ip-addresses=gcd-nat-static-ip-uscentral1 \
    --nat-all-subnet-ip-ranges \
    --enable-logging

# Step 7: Provision Custom VPC Routes
gcloud compute routes create gcd-prod-custom-vpc-route-to-nva \
    --network=gcd-prod-custom-vpc \
    --destination-range=172.16.0.0/12 \
    --next-hop-gateway=default-internet-gateway \
    --priority=800

# Step 8: Provision Private Cloud DNS Managed Zone & A-Records
gcloud dns managed-zones create gcd-private-dns-zone \
    --dns-name="gcd.internal." \
    --description="Private Internal DNS Managed Zone for VPC" \
    --visibility=private \
    --networks=gcd-prod-custom-vpc

gcloud dns record-sets create "db.gcd.internal." \
    --zone=gcd-private-dns-zone \
    --type=A \
    --ttl=300 \
    --rrdatas="10.1.0.10"

# Step 9: Launch Private Workload VM Instance (No External IP)
gcloud compute instances create private-app-server-01 \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --subnet=prod-subnet-us-central1 \
    --no-address \
    --tags=iap-enabled,web-backend \
    --private-network-ip=10.1.0.10
```

---

### 2. Single-Line Master Chained Command

Run all commands in one single terminal line chained with `&&`:

```bash
set -e && gcloud services enable compute.googleapis.com iap.googleapis.com dns.googleapis.com networkmanagement.googleapis.com && gcloud compute networks create gcd-prod-custom-vpc --subnet-mode=custom --bgp-routing-mode=global --mtu=1460 && gcloud compute networks subnets create prod-subnet-us-central1 --network=gcd-prod-custom-vpc --region=us-central1 --range=10.1.0.0/24 --enable-private-ip-google-access --secondary-range=pod-range=10.100.0.0/16,service-range=10.200.0.0/20 && gcloud compute networks subnets create prod-dualstack-subnet --network=gcd-prod-custom-vpc --region=us-central1 --range=10.2.0.0/24 --stack-type=IPV4_IPV6 --ipv6-access-type=EXTERNAL --enable-private-ip-google-access && gcloud compute addresses create gcd-nat-static-ip-uscentral1 --region=us-central1 --network-tier=PREMIUM && gcloud compute firewall-rules create gcd-prod-custom-vpc-allow-iap-ssh --network=gcd-prod-custom-vpc --direction=INGRESS --priority=1000 --action=ALLOW --rules=tcp:22 --source-ranges=35.235.240.0/20 --target-tags=iap-enabled && gcloud compute firewall-rules create gcd-prod-custom-vpc-allow-internal-mesh --network=gcd-prod-custom-vpc --direction=INGRESS --priority=1000 --action=ALLOW --rules=tcp,udp,icmp --source-ranges=10.1.0.0/16 && gcloud compute firewall-rules create gcd-prod-custom-vpc-allow-health-checks --network=gcd-prod-custom-vpc --direction=INGRESS --priority=1000 --action=ALLOW --rules=tcp:80,tcp:443 --source-ranges=35.191.0.0/16,130.211.0.0/22 --target-tags=web-backend && gcloud compute routers create gcd-nat-router-uscentral1 --network=gcd-prod-custom-vpc --region=us-central1 --asn=65001 && gcloud compute routers nats create gcd-nat-gateway-uscentral1 --router=gcd-nat-router-uscentral1 --region=us-central1 --nat-custom-manual-ip-addresses=gcd-nat-static-ip-uscentral1 --nat-all-subnet-ip-ranges --enable-logging && gcloud compute routes create gcd-prod-custom-vpc-route-to-nva --network=gcd-prod-custom-vpc --destination-range=172.16.0.0/12 --next-hop-gateway=default-internet-gateway --priority=800 && gcloud dns managed-zones create gcd-private-dns-zone --dns-name="gcd.internal." --description="Private Internal DNS Managed Zone" --visibility=private --networks=gcd-prod-custom-vpc && gcloud dns record-sets create "db.gcd.internal." --zone=gcd-private-dns-zone --type=A --ttl=300 --rrdatas="10.1.0.10" && gcloud compute instances create private-app-server-01 --zone=us-central1-a --machine-type=e2-medium --subnet=prod-subnet-us-central1 --no-address --tags=iap-enabled,web-backend --private-network-ip=10.1.0.10
```

#### Expected Terminal Output:
```text
Operation "operations/acf.123456789" finished successfully.
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/global/networks/gcd-prod-custom-vpc].
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/subnetworks/prod-subnet-us-central1].
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/subnetworks/prod-dualstack-subnet].
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/addresses/gcd-nat-static-ip-uscentral1].
Creating firewall rule...done.
Creating firewall rule...done.
Creating firewall rule...done.
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/routers/gcd-nat-router-uscentral1].
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/routers/gcd-nat-router-uscentral1/nats/gcd-nat-gateway-uscentral1].
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/global/routes/gcd-prod-custom-vpc-route-to-nva].
Created [https://dns.googleapis.com/dns/v1/projects/YOUR_PROJECT/managedZones/gcd-private-dns-zone].
Created [https://dns.googleapis.com/dns/v1/projects/YOUR_PROJECT/managedZones/gcd-private-dns-zone/rrsets/db.gcd.internal./A].
Created [https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/zones/us-central1-a/instances/private-app-server-01].
```

#### How to Verify Environment Correctness:
```bash
# Verify Subnet Status
gcloud compute networks subnets list --network=gcd-prod-custom-vpc --format="table(name, region, ipCidrRange, privateIpGoogleAccess)"

# Verify Firewall Rules
gcloud compute firewall-rules list --filter="network=gcd-prod-custom-vpc" --format="table(name, direction, priority, allow)"

# Verify NAT Configuration
gcloud compute routers nats describe gcd-nat-gateway-uscentral1 --router=gcd-nat-router-uscentral1 --region=us-central1 --format="yaml(name, natIpAllocateOption, userAllocatedNatIps)"

# Verify Private VM Instance
gcloud compute instances list --filter="name=private-app-server-01" --format="table(name, zone, status, internalIp)"

# Test Connectivity inside VM via IAP
gcloud compute ssh private-app-server-01 --zone=us-central1-a --tunnel-through-iap --command="curl -s https://ifconfig.me && dig +short db.gcd.internal."
```

#### Expected Verification Output:
```text
NAME                     REGION       RANGE        PRIVATE_IP_GOOGLE_ACCESS
prod-subnet-us-central1  us-central1  10.1.0.0/24  True
prod-dualstack-subnet    us-central1  10.2.0.0/24  True

NAME                                DIRECTION  PRIORITY  ALLOW
gcd-prod-custom-vpc-allow-health    INGRESS    1000      tcp:80,tcp:443
gcd-prod-custom-vpc-allow-iap-ssh   INGRESS    1000      tcp:22
gcd-prod-custom-vpc-allow-internal  INGRESS    1000      tcp,udp,icmp

name: gcd-nat-gateway-uscentral1
natIpAllocateOption: MANUAL_ONLY
userAllocatedNatIps:
- https://www.googleapis.com/compute/v1/projects/YOUR_PROJECT/regions/us-central1/addresses/gcd-nat-static-ip-uscentral1

NAME                  ZONE           STATUS   INTERNAL_IP
private-app-server-01  us-central1-a  RUNNING  10.1.0.10

34.122.10.55
10.1.0.10
```


