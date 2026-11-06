# Azure Hub-and-Spoke Secure Network Architecture (Capstone Project)

## Project Overview
This project implements a secure, production-style **Hub-and-Spoke network topology** in Microsoft Azure. It isolates backend Linux workloads from the public internet while enabling secure administrative access via **Azure Bastion**, strict Layer-4 traffic filtering via **Network Security Groups (NSGs)**, passwordless secret management with **Azure Key Vault**, and private PaaS connectivity using **Azure Private Endpoints**.

### Architecture Diagram
![Architecture Diagram](architecture.png)

---

## Step-by-Step Implementation

### Phase 1: Networking & VM Deployment

#### Step 1: Create the Hub Virtual Network (`nimbus-hub-vnet`)
* **Objective:** Provision the central Hub VNet (`10.0.0.0/16`) and carve out the primary workload subnet (`nimbus-hub-subnet` - `10.0.1.0/24`) inside the `nimbus-labs` resource group.
* **Why This Matters:** The Hub VNet acts as the central point of connectivity and security inspection. Using a `/16` CIDR block provides 65,536 IP addresses, leaving dedicated address space outside the `/24` workload subnet for shared platform services like `AzureBastionSubnet`.
* **Verification Screenshot:**
![Hub VNet Deployment](01-hub-vnet.png)

#### Step 2: Create the Spoke Virtual Network (`nimbus-spoke-vnet`) & Configure VNet Peering
* **Objective:** Provision the isolated Spoke VNet (`10.1.0.0/16`) with its workload subnet (`nimbus-spoke-subnet` - `10.1.1.0/24`), and establish bidirectional VNet Peering (`hub-spoke`) between the Hub and Spoke networks.
* **Why This Matters:** Separating workloads into a Spoke VNet enforces network segmentation. Using a non-overlapping `/16` CIDR block (`10.1.0.0/16` vs. `10.0.0.0/16`) prevents routing conflicts and allows seamless Layer-3 private connectivity across Microsoft's backbone via VNet Peering without traversing the public internet.
* **Peering Configuration:** Enabled bidirectional traffic access between `hub-vnet` and `nimbus-spoke-vnet`:
![Spoke VNet Peering Setup](02-1-spoke-vnet-peering-setup.png)
* **Verification (Peering Connected & Fully Synchronized):**
![Spoke VNet Peering Connected](02-spoke-vnet-peering.png)

#### Step 3: Deploy Three Linux Virtual Machines (`vm-hub-1`, `vm-hub-2`, `vm-spoke-1`)
* **Objective:** Provision three Ubuntu Server Linux VMs across the Hub and Spoke subnets (`vm-hub-1` and `vm-hub-2` in `nimbus-hub-subnet`; `vm-spoke-1` in `nimbus-spoke-subnet`) with all Public IP addresses disassociated.
* **Why This Matters:** Deploying compute instances without Public IP addresses eliminates direct internet exposure, preventing automated port-scanning and brute-force SSH attacks on port 22. All three VMs rely strictly on RFC 1918 private IP allocation (`10.0.1.x` in the Hub and `10.1.1.x` in the Spoke) and will be accessed exclusively through Azure Bastion.
* **Troubleshooting & Hardening Note:** During initial provisioning, Azure attached default Public IP resources to the VM Network Interface Cards (NICs). To enforce strict private-only access, the Public IPs were disassociated directly at the NIC `ipconfig1` level and removed from the resource group.
* **Verification Screenshot:**
![Three Linux VMs Deployed Privately](03-vms-deployed.png)

### Phase 2: Secure VM Access via Azure Bastion

#### Step 4: Deploy Azure Bastion & Establish Private SSH Sessions
* **Objective:** Provision `nimbus-bastion` inside `AzureBastionSubnet` (`10.0.2.0/26`) in `hub-vnet` to enable browser-based SSH access to internal VMs without exposing Public IP addresses or port 22 to the internet.
* **Why This Matters:** Azure Bastion acts as a fully managed PaaS jump-host. Administrators authenticate to the Azure Portal over TLS (port 443), and Bastion proxies the SSH connection (port 22) over the private VNet backbone directly to `vm-hub-1` (`10.0.1.4`).
* **Quota Troubleshooting Note:** Encountered a regional Public IP quota limit (`3 of 3` used on the Azure for Students subscription) caused by orphaned Public IP resources left behind after NIC disassociation. Resolved by purging the unattached Public IP objects from the `nimbus-labs` resource group before provisioning the dedicated Standard SKU Public IP required by the Bastion gateway.
* **Dedicated Bastion Subnet (`AzureBastionSubnet` - `10.0.2.0/26`):**
![Azure Bastion Subnet](04-Bastion-subnet.png)
* **Verification (Browser-Based SSH Session into `vm-hub-1` at `10.0.1.4`):**
![Bastion SSH Session into vm-hub-1](04-secured-bastion-access-on-vm-hub-1.png)

---

### Phase 3: Network Traffic Filtering with NSGs


### Phase 4: Secret Management with Azure Key Vault & RBAC
* Deployed `nimbus-keyvault-03` utilizing the Azure RBAC permission model, assigned the `Key Vault Secrets Officer` role, and securely stored `db-password`.

#### Step 5: Configure & Associate Subnet Network Security Groups (`nsg-hub-snet` & `nsg-spoke-snet`)
* **Objective:** Remove default NIC-level NSGs, provision dedicated Subnet-level NSGs (`nsg-hub-snet` and `nsg-spoke-snet`), configure least-privilege inbound rules for Bastion SSH and internal ICMP diagnostics, and verify connectivity across the VNet peering link.
* **Why This Matters:** Associating NSGs at the subnet boundary enforces uniform Layer-4 stateful packet filtering across all VMs in the subnet. Scoping port 22 strictly to `10.0.2.0/26` (`AzureBastionSubnet`) ensures SSH connections are only accepted from the Bastion host, while the `VirtualNetwork` Service Tag permits ICMP echo requests across both `10.0.0.0/16` and the peered `10.1.0.0/16` network.
* **Provisioned Subnet NSGs:**
![Deployed Subnet NSGs](04-deployed-nsgs.png)
* **Inbound Security Rules (`Allow-Bastion-SSH` & `Allow-vnet-ICMP`):**
![NSG Inbound Rules](05-nsg-rules.png)
* **Connectivity Verification (Intra-VNet & Cross-VNet Peering Ping Test):**
![VM Ping Connectivity Test](05-1-ping-test.png)

### Phase 5: Secure Blob Storage with Private Endpoint & Private DNS

#### Step 6: Provision Storage Account, Private Endpoint, and Private DNS Zones
* **Objective:** Deploy `nimbusstorage03` with public network access disabled entirely, exposing data access exclusively via a **Private Endpoint** (`10.0.1.6`) inside `nimbus-hub-subnet`. Integrated with a **Private DNS Zone** (`privatelink.blob.core.windows.net`) linked to both Hub and Spoke VNets for private name resolution.
* **Why This Matters:** Disabling public network access ensures the storage account is immune to internet-based data exfiltration. The Private Endpoint routes storage traffic entirely across Microsoft’s private backbone, while Private DNS zones ensure internal workloads resolve storage FQDNs to private IPs automatically.
* **Verification Screenshot (Secure Blob Container & Uploaded File):**
![Blob Storage Container](07-blob-storage.png)
