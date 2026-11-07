# Azure Hub-and-Spoke Network Lab (Capstone Project)

## 1. Project Overview
For my Azure Fundamentals capstone project, I built a secure **Hub-and-Spoke network** from scratch using the Azure Portal. 

The main goal of this lab was to learn how to deploy Linux virtual machines and cloud storage without exposing them to the public internet. Instead of giving my VMs public IP addresses, I used **Azure Bastion** to SSH into them safely from my browser, **Network Security Groups (NSGs)** to control what traffic is allowed in, **Managed Identities** so the VMs can grab secrets from **Azure Key Vault** without needing hardcoded passwords, and a **Private Endpoint** to keep storage traffic inside my private network.

### Architecture Diagram
![Architecture Diagram](architecture.png)

---

## 2. Resources Used (IaaS vs. PaaS)

| Resource | Type | What It Does in My Lab |
| :--- | :--- | :--- |
| **Virtual Networks (`nimbus-hub-vnet`, `nimbus-spoke-vnet`)** | IaaS | Two separate private networks (`10.0.0.0/16` and `10.1.0.0/16`) connected together using VNet Peering. |
| **Linux VMs (`vm-hub-1`, `vm-hub-2`, `vm-spoke-1`)** | IaaS | Three Ubuntu servers with private IPs only (no public IPs attached). |
| **Network Security Groups (`nsg-hub`, `nsg-spoke`)** | IaaS | Virtual firewalls attached to the subnets to only allow SSH from Bastion and ping tests between VMs. |
| **Azure Bastion (`nimbus-bastion`)** | PaaS | Let me SSH into my private VMs directly through the Azure Portal without opening port 22 to the internet. |
| **Managed Identities & RBAC** | PaaS | Gives each VM its own built-in Azure identity so it has permission to read secrets from the Key Vault. |
| **Azure Key Vault (`nimbus-kv`)** | PaaS | A secure digital safe used to store my database password (`db-password` / `app-secret`). |
| **Storage Account (`nimbustorage`)** | PaaS | Holds my blob container (`nimbus-blob`) and connects privately to the VNet using a Private Endpoint. |

---

## 3. Step-by-Step Build

### Phase 1: Networking & Virtual Machines

#### Step 1: Create the Hub Virtual Network (`nimbus-hub-vnet`)
* **What I Did:** Created my main "Hub" network with the address space `10.0.0.0/16` and added a subnet called `nimbus-hub-subnet` (`10.0.1.0/24`) inside the `nimbus-labs` resource group.
* **Why:** VMs need a private network to live in. Making the main VNet `/16` and the VM subnet `/24` leaves plenty of free IP addresses for extra subnets later (like the one Azure Bastion needs).
* **Screenshot:**
![Hub VNet Deployment](01-hub-vnet.png)

#### Step 2: Create the Spoke Virtual Network (`nimbus-spoke-vnet`) & VNet Peering
* **What I Did:** Created a second network (`10.1.0.0/16`) with its own subnet (`nimbus-spoke-subnet` - `10.1.1.0/24`), then linked it to the Hub VNet using **VNet Peering**.
* **Why:** In real environments, workloads are split into separate networks for better security and organization. Giving the Spoke network a different IP range (`10.1.x.x` vs `10.0.x.x`) ensures the two networks don't clash when peered together.
* **Peering Setup:**
![Spoke VNet Peering Setup](02-1-spoke-vnet-peering-setup.png)
* **Peering Connected:**
![Spoke VNet Peering Connected](02-spoke-vnet-peering.png)

#### Step 3: Deploy Three Linux VMs (`vm-hub-1`, `vm-hub-2`, `vm-spoke-1`)
* **What I Did:** Deployed two Ubuntu VMs in the Hub subnet and one Ubuntu VM in the Spoke subnet, and made sure none of them had a Public IP address.
* **Why:** Leaving Public IPs off the VMs means strangers on the internet can't scan them or try to brute-force the SSH login. They can only be reached from inside the private network.
* **Screenshot:**
![Three Linux VMs Deployed Privately](03-vms-deployed.png)

---

### Phase 2: Connecting to the VMs with Azure Bastion

#### Step 4: Deploy Azure Bastion & SSH into `vm-hub-1`
* **What I Did:** Created an `AzureBastionSubnet` (`10.0.2.0/26`) inside the Hub VNet, deployed `nimbus-bastion`, and used it to open an SSH terminal to `vm-hub-1` (`10.0.1.4`).
* **Why:** Since my VMs don't have public IPs, I needed a safe way to log into them. Azure Bastion acts as a secure jump-box that lets me log into the VMs right inside my browser over HTTPS.
* **Bastion Subnet:**
![Azure Bastion Subnet](04-Bastion-subnet.png)
* **Logged into `vm-hub-1` via Bastion:**
![Bastion SSH Session into vm-hub-1](04-secured-bastion-access-on-vm-hub-1.png)

---

### Phase 3: Locking Down Traffic with NSGs

#### Step 5: Set Up Network Security Groups (`nsg-hub-snet` & `nsg-spoke-snet`)
* **What I Did:** Created two NSGs, attached them to my Hub and Spoke subnets, and added two inbound rules:
  1. `Allow-Bastion-SSH`: Only allows SSH (port 22) coming from the Bastion subnet (`10.0.2.0/26`).
  2. `Allow-vnet-ICMP`: Allows VMs inside the virtual networks to ping each other.
* **Why:** NSGs act like a firewall rule list for the subnet. Even inside a private network, it's good practice to restrict traffic so SSH only works when coming through Bastion. Afterward, I ran `ping` from `vm-hub-1` to test that both internal and peered VMs could talk to each other.
* **Deployed NSGs:**
![Deployed Subnet NSGs](04-deployed-nsgs.png)
* **Inbound Rules:**
![NSG Inbound Rules](05-nsg-rules.png)
* **Ping Test Results:**
![VM Ping Connectivity Test](05-1-ping-test.png)

---

### Phase 4: Storing Secrets in Azure Key Vault

#### Step 6: Set Up Key Vault, RBAC Permissions, and VM Managed Identities
* **What I Did:** Created an Azure Key Vault (`nimbus-kv`), gave my account the `Key Vault Secrets Officer` role to add a test secret (`db-password`), and turned on **System-Assigned Managed Identities** for all three VMs with the `Key Vault Secrets User` role.
* **Why:** Storing passwords in plain text or inside code is risky. Putting them in Key Vault keeps them safe, and turning on Managed Identities gives each VM an automatic "ID badge" in Azure so it can request secrets without needing a password of its own.
* **How the VMs Get the Secret:**
  1. The VM uses its Managed Identity to ask Azure for a temporary access token.
  2. The VM sends that token to the Key Vault to ask for the secret.
  3. Key Vault checks that the VM has the `Key Vault Secrets User` role and sends the secret back over HTTPS.
* **Screenshot:**
![Azure Key Vault Secret](06-keyvault-secret.png)

---

### Phase 5: Securing Blob Storage with a Private Endpoint

#### Step 7: Create Storage Account, Private Endpoint, and Private DNS Zone
* **What I Did:** Created a storage account (`nimbustorage`), turned off public access, and connected it to `nimbus-hub-subnet` using a **Private Endpoint** and a **Private DNS Zone** (`privatelink.blob.core.windows.net`). Then I created a blob container (`nimbus-blob`) and uploaded a test file (`test.txt`).
* **Why:** Normally, Azure Storage has a public web link. Adding a Private Endpoint gives the storage account a private IP inside my Hub VNet (`10.0.1.6`), and the Private DNS Zone makes sure my VMs use that private IP whenever they connect to the storage account.
* **Screenshot:**
![Blob Storage Container](07-blob-storage.png)

---

## 4. Issues I Ran Into & How I Fixed Them

While building this lab, I ran into a few errors that taught me a lot about how Azure works behind the scenes:

1. **Hitting the Student Subscription Public IP Limit (`3 of 3`):**
   * **What happened:** When I first created the three VMs, Azure automatically created and attached 3 Public IPs to them. Later, when I tried to deploy Azure Bastion, it failed because my *Azure for Students* subscription only allows 3 Public IPs per region.
   * **How I fixed it:** I went into each VM's network interface (`ipconfig1`), unchecked the Public IP to detach it, and deleted the 3 unused Public IP resources from my resource group. That freed up my quota for Bastion and kept my VMs completely private.
2. **Extra Default NSGs on the VMs:**
   * **What happened:** Azure also auto-created separate NSGs for each VM's network card (`vm-hub-1-nsg`, etc.), which got messy when I wanted to manage rules at the subnet level.
   * **How I fixed it:** I detached and deleted the default NIC NSGs so I only had to manage two clean subnet-level NSGs (`nsg-hub-snet` and `nsg-spoke-snet`).
3. **`403 Not Authorized` Error When Opening the Blob Container:**
   * **What happened:** After creating my storage account with public access turned off, Azure blocked me from opening the container in my browser with a `403` error—even though I own the subscription.
   * **How I fixed it:** I learned two things here: first, being Subscription Owner doesn't automatically let you view files inside storage, so I had to assign myself the **Storage Blob Data Contributor** role in IAM. Second, because public network access was disabled, the Azure Portal in my browser was being blocked by the storage firewall, so I had to whitelist my home IP under the Networking tab to upload `test.txt`.
