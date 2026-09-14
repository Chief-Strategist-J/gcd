# VPC Networks & Subnets: Complete Operations & Verification Manual

This document is an operational reference manual for Google Cloud Virtual Private Cloud (VPC) Networks, Subnets, Firewall Rules, IP Address Management, Bring Your Own IP (BYOIP), Virtual Routers, and Cloud DNS.

Every command snippet includes:
1. **Command to Execute**3. **How to Verify Configuration Correctness**

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
15. [Category 15: 100% Complete Master Enterprise Deployment Sequence](#category-15-100-complete-master-enterprise-deployment-sequence)

---

## Category 1: VPC Network Creation & Mode Management

### 1. Provision Custom Mode VPC Network (Production Standard)

```bash
gcloud compute networks create gcd-prod-custom-vpc \
    --subnet-mode=custom \
    --bgp-routing-mode=global \
    --mtu=1460
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks describe gcd-prod-custom-vpc --format="yaml(name, autoCreateSubnetworks, routingConfig, mtu)"
```

---

### 2. Convert Existing Auto Mode VPC Network to Custom Mode (Irreversible)

```bash
gcloud compute networks switch-mode gcd-dev-auto-vpc --mode=custom
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks describe gcd-dev-auto-vpc --format="value(autoCreateSubnetworks)"
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

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks list
```

---

### 4. Enable Required Network Management & IAP APIs

```bash
gcloud services enable iap.googleapis.com networkmanagement.googleapis.com
```

#### How to Verify Configuration Correctness:
```bash
gcloud services list --enabled --filter="name:(iap.googleapis.com OR networkmanagement.googleapis.com)"
```

---

### 5. Create Auto Mode VPC Network for Prototyping

```bash
gcloud compute networks create mynetwork --subnet-mode=auto
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks subnets list --network=mynetwork --format="table(name, region, ipCidrRange)"
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

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks subnets list --sort-by=NETWORK --format="table(network, name, region, ipCidrRange)"
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

#### How to Verify Configuration Correctness:
```bash
# Confirm that isolated VPCs require VPC Peering or Cloud VPN for internal IP communication
gcloud compute network-peerings list
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

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks subnets describe prod-subnet-us-central1 \
    --region=us-central1 \
    --format="yaml(name, ipCidrRange, gatewayAddress, privateIpGoogleAccess)"
```

---

## Category 3: Zero-Downtime Subnet IP Range Expansion

### 1. Standard Subnet Range Expansion (e.g. /24 to /20)

```bash
gcloud compute networks subnets expand-ip-range prod-subnet-us-central1 \
    --region=us-central1 \
    --prefix-length=20
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks subnets describe prod-subnet-us-central1 \
    --region=us-central1 \
    --format="value(ipCidrRange)"
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

#### How to Verify Configuration Correctness:
```bash
gcloud compute firewall-rules describe allow-iap-ssh --format="yaml(name, sourceRanges, allowed, targetTags)"
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

#### How to Verify Configuration Correctness:
```bash
gcloud compute firewall-rules list --sort-by=NETWORK --format="table(network, name, priority, allow)"
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

#### Step 4: Restart VM & Verify Mutated Ephemeral External IP
```bash
gcloud compute instances start lifecycle-demo-vm --zone=us-central1-a
gcloud compute instances describe lifecycle-demo-vm --zone=us-central1-a \
    --format="yaml(status, networkInterfaces[0].networkIP, networkInterfaces[0].accessConfigs[0].natIP)"
```

---

### 3. Promote Existing Ephemeral External IP Address to Static External IP

```bash
gcloud compute addresses create promoted-static-ip \
    --region=us-central1 \
    --addresses=35.202.88.19
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute addresses describe promoted-static-ip --region=us-central1 --format="yaml(name, address, status)"
```

---

### 4. Audit & Release Unassigned Static External IPs to Eliminate Surcharge Billing

```bash
gcloud compute addresses list \
    --filter="status=RESERVED AND users:*" \
    --format="table(name, region, address, status)"
```

#### How to Verify Configuration Correctness & Release Unassigned IPs:
```bash
gcloud compute addresses delete abandoned-legacy-ip unused-test-ip --region=us-central1 --quiet
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

#### How to Verify Configuration Correctness:
```bash
gcloud compute instances describe private-backend-db --zone=us-central1-a \
    --format="yaml(name, networkInterfaces[0].accessConfigs)"
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

#### How to Verify Configuration Correctness:
```bash
gcloud compute instances describe custom-ip-vm --zone=us-central1-a \
    --format="value(networkInterfaces[0].networkIP)"
```

---

## Category 7: Bring Your Own IP (BYOIP) & Internal DNS Verification

### 1. Provision BYOIP Public Advertised Prefix (PAP) (/24 Minimum Block)

```bash
gcloud compute public-advertised-prefixes create my-company-byoip-pap \
    --dns-verification-ip=198.51.100.1 \
    --range=198.51.100.0/24
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute public-advertised-prefixes describe my-company-byoip-pap --format="yaml(name, ipCidrRange, status)"
```

---

### 2. Verify Network-Scoped Internal DNS Resolution Inside VM

```bash
gcloud compute ssh vm-1 --zone=us-central1-a --command="dig +short vm-2.us-central1-a.c.YOUR_PROJECT.internal"
```

---

## Category 8: Alias IP Ranges & OS NAT Transparency Inspection

### 1. Inspect OS Network Interfaces Inside VM (Demonstrating NAT Transparency)

```bash
# Execute ip addr show inside VM OS
gcloud compute ssh lifecycle-demo-vm --zone=us-central1-a --command="ip addr show eth0"
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

#### How to Verify Configuration Correctness:
```bash
gcloud compute instances describe container-host-vm --zone=us-central1-a \
    --format="yaml(networkInterfaces[0].aliasIpRanges)"
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

---

### 2. Add DNS A-Record Pointing to VM External IP

```bash
gcloud dns record-sets create "app.example.com." \
    --zone=company-public-zone \
    --type=A \
    --ttl=300 \
    --rrdatas="35.202.88.19"
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

#### How to Verify Configuration Correctness:
```bash
gcloud compute routes describe route-to-internal-appliance --format="yaml(name, destRange, priority, nextHopInstance)"
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

---

## Category 12: Private Google Access & Egress Cost Optimization

### 1. Enable Private Google Access on Subnet to Eliminate External Egress Charges

When VMs without external IPs connect to Google APIs (Cloud Storage, BigQuery, Maps), Private Google Access routes traffic internally across Google's private backbone for $0.00/GB egress cost.

```bash
gcloud compute networks subnets update prod-subnet-us-central1 \
    --region=us-central1 \
    --enable-private-ip-google-access
```

#### How to Verify Configuration Correctness:
```bash
gcloud compute networks subnets describe prod-subnet-us-central1 \
    --region=us-central1 \
    --format="value(privateIpGoogleAccess)"
```

---

### 2. Audit Intra-Zone External IP Communication Leaks

```bash
gcloud compute instances list --format="table(name, zone, networkInterfaces[0].networkIP, networkInterfaces[0].accessConfigs[0].natIP)"
```

#### How to Verify & Remediate:
```bash
# Test internal DNS connectivity to ensure internal communication is used instead of external IPs
gcloud compute ssh web-frontend-vm-1 --zone=us-central1-a --command="curl -I http://backend-api-vm-2.us-central1-a.c.YOUR_PROJECT.internal"
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

#### How to Verify Configuration Correctness:
```bash
gcloud compute vpn-tunnels describe classic-tunnel-1 --region=us-central1 --format="yaml(status, detailedStatus, peerIp)"
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

#### How to Verify Configuration Correctness:
```bash
# Verify BGP session status across both tunnels
gcloud compute routers get-status vpn-cloud-router --region=us-central1 --format="yaml(bgpPeerStatus)"
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

#### How to Verify Configuration Correctness:
```bash
gcloud compute interconnects attachments describe dedicated-vlan-attachment-01 --region=us-central1 --format="yaml(state, operationalStatus)"
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



## Category 15: 100% Complete Master Enterprise Deployment Sequence

This section provides pure `gcloud` shell commands that provision an entire enterprise-grade Google Cloud VPC network architecture covering **100% of GCP networking capabilities** in **strict logical dependency order**: APIs → Custom VPC → Subnets & Secondary CIDRs → Static IPs → Firewall Mesh → Cloud NAT → Private Service Access (PSA) → Private Service Connect (PSC) → Shared VPC → VPC Peering → HA VPN & BGP → Routes → Cloud DNS → Application Load Balancer → Workload VM → Connectivity Diagnostic Audit.

### 100% Complete Execution Dependency Flow
```text
 1. Enable Service APIs (Compute, IAP, DNS, Network Management, Service Networking, Container)
    ↓
 2. Provision Custom Mode VPC Network (Global BGP Routing, Custom MTU 1460)
    ↓
 3. Provision Regional Subnets (Primary IPv4, Secondary GKE Ranges, Dual-Stack IPv6 & Proxy-Only Subnet)
    ↓
 4. Reserve Static External & Internal IP Addresses (Cloud NAT, Load Balancer & PSC VIPs)
    ↓
 5. Apply Zero-Trust VPC Firewall Rules (IAP SSH/RDP, Intra-Subnet Mesh, Health Checks, Egress Web, Logged Deny)
    ↓
 6. Provision Cloud Router & Exhaustive Cloud NAT Gateway (Port Limits, Dynamic Scaling, Connection Timeouts, Logging)
    ↓
 7. Configure Private Service Access (PSA) for Managed Services (Cloud SQL / MemoryStore IP Peering)
    ↓
 8. Provision Private Service Connect (PSC) Endpoint (Google APIs Private Forwarding Rule)
    ↓
 9. Enable Shared VPC Host Project & Grant Service Project Subnet IAM Rights (roles/compute.networkUser)
    ↓
10. Establish Bi-Directional VPC Network Peering (Custom Route Export/Import)
    ↓
11. Provision High Availability (HA) Cloud VPN & BGP IPsec Tunnels (99.99% SLA Dual-Interface Link-Local)
    ↓
12. Provision Custom VPC Static Routes (Virtual Appliance Next-Hop Routing)
    ↓
13. Provision Private Cloud DNS Managed Zone & A-Records
    ↓
14. Deploy Global Application Load Balancer (Health Check, Backend Service, URL Map, Target Proxy, Forwarding Rule)
    ↓
15. Launch Workload VM Instances (Private Internal IPs Only, Attached to Custom Subnet & Managed Backend)
    ↓
16. Run Network Intelligence Center Connectivity Tests & Comprehensive Audit Suite
```

---

### Step-by-Step 100% Master Enterprise Commands

Execute these shell commands in sequence:

```bash
# Step 1: Enable All Required GCP Service APIs
gcloud services enable \
    compute.googleapis.com \
    iap.googleapis.com \
    dns.googleapis.com \
    networkmanagement.googleapis.com \
    servicenetworking.googleapis.com \
    container.googleapis.com

# Step 2: Provision Custom Mode VPC Network
gcloud compute networks create gcd-prod-custom-vpc \
    --subnet-mode=custom \
    --bgp-routing-mode=global \
    --mtu=1460

# Step 3: Provision Regional Subnets (Primary IPv4, GKE Secondary Ranges, Dual-Stack IPv6 & Proxy-Only Subnet)
# Subnet 3.1: Primary Workload Subnet with GKE Pod/Service Secondary CIDRs & VPC Flow Logs
gcloud compute networks subnets create prod-subnet-us-central1 \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --range=10.1.0.0/24 \
    --enable-private-ip-google-access \
    --secondary-range=pod-range=10.100.0.0/16,service-range=10.200.0.0/20 \
    --enable-flow-logs \
    --logging-aggregation-interval=INTERVAL_5_SEC \
    --logging-flow-sampling=0.5 \
    --logging-metadata=INCLUDE_ALL_METADATA

# Subnet 3.2: Dual-Stack IPv4/IPv6 Subnet
gcloud compute networks subnets create prod-dualstack-subnet \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --range=10.2.0.0/24 \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=EXTERNAL \
    --enable-private-ip-google-access

# Subnet 3.3: Envoy Proxy-Only Subnet (Required for Regional/Global Application Load Balancers)
gcloud compute networks subnets create proxy-only-subnet-uscentral1 \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --range=10.254.0.0/24 \
    --purpose=REGIONAL_MANAGED_PROXY \
    --role=ACTIVE

# Step 4: Reserve Static External & Internal IP Addresses
# Reserving Static NAT IP
gcloud compute addresses create gcd-nat-static-ip-uscentral1 \
    --region=us-central1 \
    --network-tier=PREMIUM

# Reserving Global Static External IP for Load Balancer
gcloud compute addresses create gcd-alb-global-ip \
    --global \
    --network-tier=PREMIUM

# Reserving Internal IP for Private Service Connect (PSC)
gcloud compute addresses create gcd-psc-google-apis-ip \
    --region=us-central1 \
    --subnet=prod-subnet-us-central1 \
    --addresses=10.1.0.99

# Step 5: Apply Zero-Trust VPC Firewall Policy Mesh
# Rule 5.1: Ingress SSH (22) & RDP (3389) via Identity-Aware Proxy (IAP)
gcloud compute firewall-rules create gcd-prod-custom-vpc-allow-iap-ssh-rdp \
    --network=gcd-prod-custom-vpc \
    --direction=INGRESS \
    --priority=1000 \
    --action=ALLOW \
    --rules=tcp:22,tcp:3389 \
    --source-ranges=35.235.240.0/20 \
    --target-tags=iap-enabled

# Rule 5.2: Ingress Intra-Subnet Mesh
gcloud compute firewall-rules create gcd-prod-custom-vpc-allow-internal-mesh \
    --network=gcd-prod-custom-vpc \
    --direction=INGRESS \
    --priority=1000 \
    --action=ALLOW \
    --rules=tcp,udp,icmp \
    --source-ranges=10.1.0.0/16

# Rule 5.3: Ingress Load Balancer & Envoy Proxy Probes
gcloud compute firewall-rules create gcd-prod-custom-vpc-allow-health-checks \
    --network=gcd-prod-custom-vpc \
    --direction=INGRESS \
    --priority=1000 \
    --action=ALLOW \
    --rules=tcp:80,tcp:443,tcp:8080 \
    --source-ranges=35.191.0.0/16,130.211.0.0/22,10.254.0.0/24 \
    --target-tags=web-backend

# Rule 5.4: Explicit Egress Allow for Web Traffic
gcloud compute firewall-rules create gcd-prod-custom-vpc-allow-egress-web \
    --network=gcd-prod-custom-vpc \
    --direction=EGRESS \
    --priority=1000 \
    --action=ALLOW \
    --rules=tcp:80,tcp:443 \
    --destination-ranges=0.0.0.0/0

# Rule 5.5: Logged Ingress Deny-All Baseline Policy
gcloud compute firewall-rules create gcd-prod-custom-vpc-deny-all-ingress-log \
    --network=gcd-prod-custom-vpc \
    --direction=INGRESS \
    --priority=65000 \
    --action=DENY \
    --rules=all \
    --source-ranges=0.0.0.0/0 \
    --enable-logging

# Rule 5.6: Logged Egress Deny-All Baseline Policy
gcloud compute firewall-rules create gcd-prod-custom-vpc-deny-all-egress-log \
    --network=gcd-prod-custom-vpc \
    --direction=EGRESS \
    --priority=65000 \
    --action=DENY \
    --rules=all \
    --destination-ranges=0.0.0.0/0 \
    --enable-logging

# Step 6: Provision Cloud Router & Exhaustive Cloud NAT Gateways (Public NAT & Private NAT)
# Step 6.1: Provision Public Cloud NAT Gateway (Outbound Internet Access for Private VMs)
gcloud compute routers create gcd-nat-router-uscentral1 \
    --network=gcd-prod-custom-vpc \
    --region=us-central1 \
    --asn=65001

gcloud compute routers nats create gcd-nat-gateway-uscentral1 \
    --router=gcd-nat-router-uscentral1 \
    --region=us-central1 \
    --nat-custom-manual-ip-addresses=gcd-nat-static-ip-uscentral1 \
    --nat-all-subnet-ip-ranges \
    --min-ports-per-vm=64 \
    --max-ports-per-vm=1024 \
    --enable-dynamic-port-allocation \
    --udp-idle-timeout=30s \
    --tcp-established-idle-timeout=1200s \
    --tcp-transitory-idle-timeout=30s \
    --icmp-idle-timeout=30s \
    --endpoint-types=ENDPOINT_TYPE_VM \
    --enable-logging \
    --log-filter=ALL

# Step 6.2: Provision Private NAT Gateway (Hybrid Interconnect Overlapping Subnet Translation)
gcloud compute routers nats create gcd-private-nat-uscentral1 \
    --router=gcd-nat-router-uscentral1 \
    --region=us-central1 \
    --type=PRIVATE \
    --nat-all-subnet-ip-ranges \
    --rules='type=NATP,subnet=prod-subnet-us-central1'

# Step 7: Configure Private Service Access (PSA) for Cloud SQL & MemoryStore
# Reserve IP Range for Service Networking Peering
gcloud compute addresses create google-managed-services-gcd \
    --global \
    --purpose=VPC_PEERING \
    --prefix-length=16 \
    --network=gcd-prod-custom-vpc

# Establish Private Service Access Connection to Service Networking Provider
gcloud services vpc-peerings connect \
    --service=servicenetworking.googleapis.com \
    --ranges=google-managed-services-gcd \
    --network=gcd-prod-custom-vpc

# Step 8: Provision Private Service Connect (PSC) Endpoint for Google APIs
gcloud compute forwarding-rules create psc-endpoint-google-apis \
    --global \
    --network=gcd-prod-custom-vpc \
    --address=gcd-psc-google-apis-ip \
    --target-google-apis-bundle=all-apis

# Step 9: Configure Shared VPC Infrastructure (Multi-Project Governance)
# Enable Current Project as Shared VPC Host Project
gcloud compute shared-vpc enable "$(gcloud config get-value project)"

# Attach Service Project to Host Project
gcloud compute shared-vpc associated-projects add SERVICE_PROJECT_ID \
    --host-project="$(gcloud config get-value project)"

# Grant Service Project Service Account IAM access to Subnet
gcloud compute networks subnets add-iam-policy-binding prod-subnet-us-central1 \
    --region=us-central1 \
    --member="serviceAccount:SERVICE_PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
    --role="roles/compute.networkUser"

# Step 10: Establish Bi-Directional VPC Network Peering
# Peering 10.1: Local to Remote VPC Peering
gcloud compute networks peerings create peer-prod-to-dev \
    --network=gcd-prod-custom-vpc \
    --peer-network=gcd-dev-auto-vpc \
    --export-custom-routes \
    --import-custom-routes

# Peering 10.2: Remote to Local VPC Peering Handshake
gcloud compute networks peerings create peer-dev-to-prod \
    --network=gcd-dev-auto-vpc \
    --peer-network=gcd-prod-custom-vpc \
    --export-custom-routes \
    --import-custom-routes

# Step 11: Provision High Availability (HA) Cloud VPN & Dynamic BGP IPsec Tunnels
# Provision HA VPN Gateway (2 Regional Interfaces Allocated Automatically)
gcloud compute vpn-gateways create ha-vpn-gw-01 \
    --network=gcd-prod-custom-vpc \
    --region=us-central1

# Provision External Peer VPN Gateway (Representing On-Premises Router)
gcloud compute external-vpn-gateways create onprem-peer-gw \
    --interfaces=0=203.0.113.10,1=203.0.113.11

# Provision IPsec Tunnel 0 over Interface 0
gcloud compute vpn-tunnels create ha-tunnel-0 \
    --vpn-gateway=ha-vpn-gw-01 \
    --interface=0 \
    --peer-external-gateway=onprem-peer-gw \
    --peer-external-gateway-interface=0 \
    --shared-secret="ComplexPresharedKey991823" \
    --router=gcd-nat-router-uscentral1 \
    --region=us-central1

# Provision IPsec Tunnel 1 over Interface 1 (99.99% SLA Redundancy)
gcloud compute vpn-tunnels create ha-tunnel-1 \
    --vpn-gateway=ha-vpn-gw-01 \
    --interface=1 \
    --peer-external-gateway=onprem-peer-gw \
    --peer-external-gateway-interface=1 \
    --shared-secret="ComplexPresharedKey991823" \
    --router=gcd-nat-router-uscentral1 \
    --region=us-central1

# Configure Link-Local BGP Interfaces & BGP Peers on Cloud Router
gcloud compute routers add-interface gcd-nat-router-uscentral1 \
    --interface-name=bgp-if-0 \
    --ip-address=169.254.0.1 \
    --mask-length=30 \
    --vpn-tunnel=ha-tunnel-0 \
    --region=us-central1

gcloud compute routers add-bgp-peer gcd-nat-router-uscentral1 \
    --peer-name=bgp-peer-0 \
    --interface=bgp-if-0 \
    --peer-ip-address=169.254.0.2 \
    --peer-asn=65002 \
    --region=us-central1

gcloud compute routers add-interface gcd-nat-router-uscentral1 \
    --interface-name=bgp-if-1 \
    --ip-address=169.254.1.1 \
    --mask-length=30 \
    --vpn-tunnel=ha-tunnel-1 \
    --region=us-central1

gcloud compute routers add-bgp-peer gcd-nat-router-uscentral1 \
    --peer-name=bgp-peer-1 \
    --interface=bgp-if-1 \
    --peer-ip-address=169.254.1.2 \
    --peer-asn=65002 \
    --region=us-central1

# Step 12: Provision Custom VPC Routes
gcloud compute routes create gcd-prod-custom-vpc-route-to-nva \
    --network=gcd-prod-custom-vpc \
    --destination-range=172.16.0.0/12 \
    --next-hop-gateway=default-internet-gateway \
    --priority=800

# Step 13: Provision Private Cloud DNS Managed Zone & Record Sets
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

# Step 14: Provision Global Application Load Balancer (HTTP ALB)
# Step 14.1: Create Regional HTTP Health Check
gcloud compute health-checks create http alb-web-health-check \
    --port=80 \
    --request-path="/healthz"

# Step 14.2: Create Instance Group / Backend Group
gcloud compute instance-groups unmanaged create web-backend-ig \
    --zone=us-central1-a

# Step 14.3: Create Global Backend Service
gcloud compute backend-services create alb-backend-service \
    --global \
    --protocol=HTTP \
    --health-checks=alb-web-health-check

gcloud compute backend-services add-backend alb-backend-service \
    --global \
    --instance-group=web-backend-ig \
    --instance-group-zone=us-central1-a

# Step 14.4: Create URL Map & Target HTTP Proxy
gcloud compute url-maps create alb-url-map \
    --default-service=alb-backend-service

gcloud compute target-http-proxies create alb-target-proxy \
    --url-map=alb-url-map

# Step 14.5: Create Global Forwarding Rule using Reserved External IP
gcloud compute forwarding-rules create alb-global-forwarding-rule \
    --global \
    --target-http-proxy=alb-target-proxy \
    --ports=80 \
    --address=gcd-alb-global-ip

# Step 15: Launch Private Workload VM Instance & Attach to Backend Group
gcloud compute instances create private-app-server-01 \
    --zone=us-central1-a \
    --machine-type=e2-medium \
    --subnet=prod-subnet-us-central1 \
    --no-address \
    --tags=iap-enabled,web-backend \
    --private-network-ip=10.1.0.10

gcloud compute instance-groups unmanaged add-instances web-backend-ig \
    --zone=us-central1-a \
    --instances=private-app-server-01

# Step 16: Execute Network Intelligence Center Connectivity Test & Diagnostic Audit
gcloud network-management connectivity-tests create test-vm-to-internet \
    --source-instance=projects/"$(gcloud config get-value project)"/zones/us-central1-a/instances/private-app-server-01 \
    --destination-ip-address=8.8.8.8 \
    --destination-port=53 \
    --protocol=UDP
```

---

#### How to Verify 100% Complete Infrastructure:

```bash
# 1. Verify Subnets, Secondary Ranges & Proxy-Only Subnet
gcloud compute networks subnets list --network=gcd-prod-custom-vpc --format="table(name, region, ipCidrRange, purpose, role)"

# 2. Verify Private Service Access (PSA) Peering Connection
gcloud services vpc-peerings list --network=gcd-prod-custom-vpc

# 3. Verify Private Service Connect (PSC) Endpoint Forwarding Rule
gcloud compute forwarding-rules list --filter="name=psc-endpoint-google-apis"

# 4. Verify Shared VPC & Attached Projects
gcloud compute shared-vpc get-host-project

# 5. Verify VPC Network Peerings
gcloud compute networks peerings list --network=gcd-prod-custom-vpc

# 6. Verify HA VPN Gateway & Dynamic BGP Tunnels Status
gcloud compute vpn-tunnels list --filter="region=us-central1"
gcloud compute routers get-status gcd-nat-router-uscentral1 --region=us-central1

# 7. Verify Global Load Balancer & Health Status
gcloud compute backend-services get-health alb-backend-service --global

# 8. Run Diagnostic Connectivity Test Evaluation
gcloud network-management connectivity-tests describe test-vm-to-internet
```




