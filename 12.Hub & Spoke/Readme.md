# **Hub & Spoke Explanation with Real time Example:**


# 🚚 Real-Time Example: “ShopEZ” E‑commerce on Azure
Imagine ShopEZ, a retail company moving to Azure. They want:

* Centralized security & connectivity (to on‑prem, internet)
* Isolated workloads per function/team
* A structure that scales to many apps and environments

# They choose Hub-and-Spoke:

## 🏢 Hub VNet (Shared Services & Security)

* Azure Firewall Premium (central egress/ingress, DNAT/SNAT)
* Azure Bastion (secure SSH/RDP without public IPs)
* Private DNS Resolver + custom DNS forwarders to on‑prem
* VPN/ExpressRoute Gateway (hybrid connectivity)
* Jump box / management tooling
* Optionally a Virtual Appliance (NVA) like Palo Alto for deep inspection

## 🌐 Spokes (Isolated Workloads)

* Spoke‑Web: Frontend web apps, AKS Ingress, App Gateways
* Spoke‑App: Microservices, APIs, AKS nodes, Function Apps (with VNet injection)
* Spoke‑Data: Databases (SQL MI), Redis, Storage Private Endpoints
* Spoke‑Ops: DevOps tools (ADO agents), monitoring, logging
* Each spoke has its own subnets, NSGs, and UDRs pointing to the Hub firewall.

## 🔒 Traffic policy (high-level)

* Inbound (Internet → Web): via Azure Firewall DNAT → App Gateway → Web apps
* East-West (Web → App → Data): flows through Azure Firewall for visibility & policy
* Outbound (to Internet): forced through Azure Firewall SNAT
* Hybrid (to On‑Prem): routed via Hub Gateway, inspected centrally


## 🧭 How Traffic Flows (Concrete Scenarios)


### 1. Customer hits the website

* 🌍 Internet → Public IP (Azure Firewall DNAT) → App Gateway (Spoke‑Web) → Web App/AKS Ingress
* Firewall DNAT translates public IP → internal web endpoint.



### 2. Web talks to API

* Spoke‑Web → Hub Azure Firewall (via UDR) → Spoke‑App
* Firewall Application rules allow only https to App subnets.



### 3. API talks to Database

* Spoke‑App → Hub Firewall → Spoke‑Data (SQL MI Private Endpoint / subnet)
* Firewall Network rules allow SQL port + FQDN for private endpoints.



### 4. Outbound updates (e.g., NPM, OS patches)

* Any Spoke → Hub Firewall SNAT → Internet
* Firewall FQDN rules or TLS inspection restrict destinations.



### 5.On‑Prem ERP integration

* Spoke‑App → Hub Firewall → Hub ER/VPN Gateway → On‑Prem IPs
* Firewall rules enforce only required IPs/ports.



### 6. Admins RDP/SSH

* Admin → Azure Bastion (Hub) → Target VM (no public IPs anywhere).

# 📊 Architecture Diagram (Mermaid)

![image alt] (https://github.com/Hanu-Devops-Practice/a-to-z-of-networking/blob/main/Hub&%20Spoke.png?raw=true)

# 🏗️ Build It (Step-by-Step)


### 1. Create VNets

* Hub VNet (e.g., 10.0.0.0/16), subnets: AzureFirewallSubnet, AzureBastionSubnet, GatewaySubnet, Mgmt
* Spoke VNets (e.g., 10.1.0.0/16, 10.2.0.0/16, 10.3.0.0/16), each with workload subnets



### 2.Peerings

* Spokes ↔ Hub with Use Remote Gateways = True in Spokes, Allow Gateway Transit = True in Hub
* Enable “Allow forwarded traffic” on peerings



### 3. Deploy Hub Services

* Azure Firewall Premium (+ Public IP)
* Azure Bastion
* VPN/ExpressRoute Gateway (if hybrid)
* Private DNS Resolver + DNS rules (forwarders to on‑prem DNS if needed)



### 4. UDRs (User Defined Routes)

* In each Spoke subnet: 0.0.0.0/0 → Next Hop: Azure Firewall (Hub)
(forces all outbound/east‑west via the Firewall)
* For private endpoints: ensure no asymmetric routing (see pitfalls below)



### 5. Firewall Policies

* DNAT: Public IP:443 → App Gateway (Spoke‑Web)
* Application Rules: allow https from Web → App; egress FQDNs for updates
* Network Rules: allow App → SQL (e.g., 1433), App → On‑Prem subnets via Gateway
* Threat Intelligence: alert/deny known bad IPs/domains
* TLS inspection (Premium) if policy permits



### 6. Security Groups

* NSGs on Spoke subnets (least privilege; allow from Hub Firewall IPs/subnets as needed)
* Disable default outbound on subnets and rely on Firewall/NSGs



### 7. Bastion Access

* Use Bastion to RDP/SSH VMs in spokes; no public IPs on workload NICs


## ✅ Why This Works Well in Real Life

* Centralized security: One place to set allow/deny for all spokes
* Cost control: One firewall instead of one per app/spoke
* Operational simplicity: Shared services in hub; teams own their spoke
* Scale & isolation: Add/remove spokes per app/team/environment
* Hybrid ready: On‑prem connectivity terminates in hub
* Observability: Firewall logs → Sentinel/Log Analytics; per-spoke NSG flow logs

# ⚠️ Common Pitfalls & How to Avoid Them


## 1. Asymmetric Routing

* If you send traffic via the Firewall in one direction and it returns another path, sessions break.

        ✅ Fix: Ensure UDRs send both directions via the Firewall (or use Private Link for PaaS).



## 2. Private Endpoint DNS

* PaaS Private Endpoints require correct Private DNS zones linked to spokes.

        ✅ Fix: Centralize Private DNS in hub; link all spokes; forward on-prem as needed.



## 3. Peerings not set for forwarded traffic

* If you don’t enable “Allow forwarded traffic”, hub firewall can’t pass traffic.

        ✅ Fix: Enable on both peering directions.



## 4. Forgetting AzureFirewallSubnet name

* Firewall must sit in a subnet named AzureFirewallSubnet.

        ✅ Fix: Use the exact reserved name.



## 5. Bypassing Firewall with service tags

* Some PaaS with service endpoints may bypass Firewall unintentionally.

        ✅ Fix: Use Private Endpoints + UDRs to keep traffic observable.


# 🧪 Quick Test Plan

* Inbound: Hit the firewall public IP on 443 → confirm it DNATs to App Gateway
* East-West: From Web pod/VM, call API FQDN/IP → allowed by App rule
* DB access: API to SQL MI (1433) → allowed by Network rule
* Outbound: curl to external repositories → allowed per FQDN rules
* Hybrid: Ping/curl on‑prem internal hosts via ER/VPN gateway
* Admin: Bastion → SSH/RDP to any VM in spokes (no public IPs)

# 🧩 When to add an NVA (Virtual Appliance)
Add a Palo Alto / FortiGate / Check Point in the hub if you need:

* SSL decryption with enterprise policy
* Advanced IDS/IPS, URL filtering, threat intel feeds
* Complex routing (BGP/OSPF) beyond Azure Firewall’s scope

Use it with Azure Firewall or instead of it, depending on ops and cost.