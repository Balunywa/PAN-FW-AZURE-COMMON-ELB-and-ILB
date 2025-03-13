# Palo Alto Networks VM-Series Firewall Deployment Template

This ARM template deploys a scalable Palo Alto Networks VM-Series firewall solution in Azure. It creates the following resources:

![image](https://github.com/user-attachments/assets/14db5c9f-36fc-4d81-90ac-2ad4ec133c53)


- **Availability Sets:** For both firewall and management deployments.
- **Network Security Groups (NSGs):** With inbound and outbound rules.
- **Public IP Addresses:** Standard SKU public IPs with static allocation (required by Standard SKU). These are used for load balancers and for VM management access.
- **Load Balancers:** 
  - An external (egress) load balancer with a frontend IP (private, dynamically allocated from the internal subnet) and backend pool for firewall traffic.
  - An internal (ingress) load balancer with a frontend configuration using a public IP (with a DNS label if provided) and a backend pool for management traffic.
- **Virtual Machines:** Two firewall VMs (by default) deployed with the Palo Alto Networks VM-Series firewall image from the Azure Marketplace.  
  - The OS profile for the VMs hardcodes the administrator username as **panuser**. **You must supply a secure password during deployment** via the parameter `adminPasswordOrKey`.
- **Network Interfaces (NICs):** Each VM is configured with three NICs (management, untrust, and trust) with dynamic private IP allocation. The management NIC is set as primary in the VM's network profile.

> **Important:**  
> - **Test in a Dev Environment First:** Always test this template in a development or staging environment before using it in production.  
> - **Set Administrator Credentials:** You must supply a valid administrator username and password. Although the template uses `panuser` as the default username, you can modify this in the parameters if needed.  
> - **Pass Resource IDs:** You must pass the resource IDs for your subnets (management, internal/trust, and external/untrust). These are not hardcoded in the template.
>
> **Example Resource ID:**  
> `/subscriptions/<your-subscription-id>/resourceGroups/<your-resource-group>/providers/Microsoft.Network/virtualNetworks/<your-vnet-name>/subnets/<your-subnet-name>`  
> You can find this in the Azure portal by navigating to your Virtual Network, selecting the **Subnets** blade, and copying the Resource ID for the desired subnet.

## How the Template Works

1. **Availability Sets:**  
   Two availability sets are created for the firewall VMs and for the ingress deployment.

2. **Network Security Groups:**  
   Two NSGs are deployed:
   - **Allow-All NSG:** Typically applied to untrust/public subnets.
   - **Management NSG:** Applied to management subnet or NIC to allow inbound management traffic.
   
3. **Public IP Addresses:**  
   All public IP resources use the Standard SKU with **static** allocation (as required by the Standard SKU).  
   The Ingress load balancer public IP uses a parameterized DNS label (e.g. `myingresslabel`).

4. **Load Balancers:**  
   - The **Egress LB** uses a dynamic private IP from the internal (private/trust) subnet for its frontend configuration.
   - The **Ingress LB** uses the public IP (with DNS label) and forwards traffic to the untrust NICs based on health probes and load balancing rules.

5. **Virtual Machines:**  
   Two Palo Alto Networks VM-Series firewall VMs are deployed. They use the marketplace image (plan: publisher=`paloaltonetworks`, product=`vmseries-flex`, SKU=`bundle1`).  
   The **OS Profile** for the VMs uses:
   - **Username:** `panuser` (default – change if needed)
   - **Password:** Must be provided during deployment via the secure parameter.
   
6. **Network Interfaces:**  
   Each VM attaches three NICs:
   - **Management NIC:** Marked as primary.
   - **Trust NIC:** For internal traffic.
   - **Untrust NIC:** For external traffic.
   
   The backend pools of the load balancers reference the NIC IP configurations so that Azure automatically routes traffic to the VMs.

## How to Deploy

1. **Customize the Parameters:**  
   Update the parameters file (or supply parameters at deployment) with:
   - Your preferred names for resources.
   - The resource IDs for:
     - **Management Subnet**
     - **Public/Untrust Subnet**
     - **Private/Trust Subnet**
   - The administrator username and secure password.
   - Any custom tags you want to apply.

2. **Deploy the Template:**  
   Use the Azure CLI, PowerShell, or the Azure Portal (easiest way simply upload in template deployment) to deploy the ARM template.

   Example: Hit edit and copy paste or simply upload the file.

   ![image](https://github.com/user-attachments/assets/8f958371-7a3b-46d6-a584-18dfa5c5f426)

   For example, using Azure CLI:
   ```bash
   az deployment group create \
     --resource-group <your-resource-group> \
     --template-file azuredeploy.json \
     --parameters @azuredeploy.parameters.json

   Verify the Deployment:
After deployment, verify that:

All VMs are running and have been assigned IP addresses from the correct subnets.
The load balancer health probes are healthy.
You can access the management interface using the public IP and credentials provided.
Testing:
Important: Test the deployment thoroughly in your development environment before moving to production. Confirm connectivity, health probe status, and proper routing of traffic.

Troubleshooting Tips
Connectivity Issues:
If you cannot access the management interface, verify the NSG rules on both the NICs and subnets.

Health Probe Failures:
Check that the firewall is configured to respond to TCP probes on port 443 (or the port specified in the probe configuration).

Credential Issues:
If you forget your credentials, you may need to reset the password using Azure’s VM password reset options or redeploy with known credentials.

Final Notes

Customization:
This template is provided as a starting point. Modify resource names, subnet IDs, and security rules as needed to fit your environment.

Feedback:
Feel free to submit issues or pull requests to improve the template further.

   
