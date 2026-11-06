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
