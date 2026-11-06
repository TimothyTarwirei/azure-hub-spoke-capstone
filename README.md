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
