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
