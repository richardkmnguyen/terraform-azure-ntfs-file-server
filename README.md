Watch me explain this lab: [LoomLink](PASTE-VIDEO-LINK-HERE)
 
# terraform-azure-ntfs-file-server
 
Used Terraform and PowerShell to build a three-VM Windows environment in Azure (a domain controller, a file server, and a Windows 11 workstation) with Active Directory groups, NTFS permissions, and SMB shares, then tested that each user could reach only the data their job requires. Getting there meant working through multiple real problems (shell syntax, my home network's DNS, VM image, quota and capacity limits, an IP address race, a Terraform provider quirk, and a bug in my own domain join script) until every access test came out the way the permission model says it should.
 
# Cloud Systems Administration, Lab 1: NTFS File Server
 
**Terraform · Azure · Active Directory · NTFS Permissions · SMB · Group Policy · Azure Key Vault · PowerShell**
 
Controlling who can open which files is one of the oldest jobs in systems administration, and it is still how thousands of companies run their file servers, including in hybrid cloud setups. In this lab I built that setup from scratch on Azure using code: Terraform creates the network and the three VMs, and one PowerShell script then configures Active Directory, the file shares, and the permissions by sending scripts to each VM through the Azure agent, with no manual clicking inside the servers. The admin password lives in Azure Key Vault and is never written to a file. Below is what I built, what broke along the way, and what I took from it.
 
This is the first lab of a three-lab Cloud Systems Administration series. Lab 2 (Azure RBAC) builds directly on this environment, and the third lab (Azure Update Manager) is standalone. It also builds on my earlier labs: the manual Active Directory lab, and the Terraform domain controller lab, which automated one VM. This one automates a whole working environment and tests it against a realistic org chart.
 
| | |
|---|---|
| **Platform** | Terraform, Azure CLI, and PowerShell on a local Windows PC, deploying three Azure VMs (Windows Server 2022 x2, Windows 11 Pro) in Central US |
| **Time** | 6 hours |
 
---
 
## What this lab is for
 
The business problem is simple to state. Finance data should only be readable by Finance staff. HR files should be invisible to Sales. IT needs administrative access everywhere to do their job. The classic Windows answer is a file server backed by Active Directory groups, with permissions set on the folders instead of on individual people.
 
Three ideas make that work:
 
- **Groups, not people.** I put users into security groups (`GRP_Finance`, `GRP_HR`, `GRP_Sales`, `GRP_IT`) and give the groups access. When someone changes teams, I change one group membership instead of editing every folder.
- **Two layers of permissions.** SMB share permissions control access to the share over the network, and NTFS permissions control access to the files on disk. When both apply, the more restrictive one wins. I left the share layer open (`Everyone: Full Access`) and let NTFS do the real enforcement, which is a common pattern because NTFS permissions apply no matter how someone reaches the files.
- **Infrastructure as code.** Terraform builds the same environment every time, so it can be reviewed, repeated, and torn down. A sysadmin who can do this is more useful than one who only knows the portal.
The five test users represent a realistic org chart:
 
| User | Department | What they should be able to do |
|---|---|---|
| `sarah.jones`, `mike.brown` | Finance | Read and write Finance data |
| `lisa.white` | HR | Read and write HR data, read-only on Finance (for cross-department reporting) |
| `john.smith` | IT | Full control everywhere |
| `tom.davis` | Sales | Read and write Sales data, nothing on Finance or HR |
 
---
 
## Architecture
 
```
My local Windows PC (PowerShell, Azure CLI, Terraform)
        │
        │  az login authenticates; Terraform calls the Azure management API
        │  Terraform state is stored remotely in Azure Blob Storage
        ▼
Azure resource group: RG-FileServerLab (Central US)
        │
        ├── VNet 10.0.0.0/16  →  Subnet 10.0.1.0/24
        ├── NSG: inbound RDP (3389) from my IP only, attached to every NIC
        ├── Azure Key Vault (RBAC model) holding the VM admin password
        │
        ├── DC01      Windows Server 2022   static IP 10.0.1.4
        │             Active Directory + DNS for lab.local, Group Policy
        ├── FS01      Windows Server 2022   member server
        │             4 SMB shares, NTFS permissions per security group
        └── CLIENT01  Windows 11 Pro        member workstation
                      where the test users sign in
 
configure-lab.ps1 (run from my PC)
        │  reads the admin password from Key Vault at runtime
        │  az vm run-command: pushes each script to a VM through the Azure agent
        ▼
Promote DC01 → create OUs, groups, users → join FS01 → build shares and NTFS ACLs
→ join CLIENT01 → grant Domain Users RDP → create the RDP GPO → automated PASS/FAIL checks
```
 
| Component | Role |
|---|---|
| **Terraform** | Builds the network, NSG, public IPs, NICs, Key Vault, and the three VMs from code |
| **Remote state (Azure Blob Storage)** | Holds Terraform's record of what it built, so it survives my PC and can be shared with Lab 2 under a different state key |
| **DC01** | Runs Active Directory, DNS, and Group Policy for `lab.local` |
| **FS01** | Hosts the four shares (`Finance`, `HR`, `Sales`, `IT`) with NTFS permissions per group |
| **CLIENT01** | The Windows 11 workstation test users sign in on to exercise the whole permission chain |
| **Azure Key Vault** | Stores the VM admin password so it never appears in a file, command, or terminal history |
| **`az vm run-command`** | Runs scripts on the VMs through the Azure agent, so no WinRM and no extra open ports are needed |
| **Custom Script Extension (CLIENT01)** | Turns RDP on inside Windows 11, which ships with it disabled |
 
Key concepts I needed:
 
- **Static private IP on DC01 (`10.0.1.4`):** FS01 and CLIENT01 use DC01 as their DNS server to find `lab.local`. If that address ever changed, domain joins and logins would break.
- **Key Vault RBAC (`enable_rbac_authorization = true`):** without it, role assignments on the vault are ignored and every secret read returns a 403.
- **`random_id` suffix:** Key Vault names are unique across all of Azure, so the name gets a random 8-character hex suffix.
- **`time_sleep`:** short pauses after the VNet and NSG are created, because Azure's control plane can take a moment to catch up and dependent resources otherwise fail with transient "not found" errors.
- **Quota vs. capacity:** a quota is how much of a VM family my subscription may use in a region. Capacity is whether Azure has that hardware free right now. They fail with different errors.
- **Terraform state:** Terraform's record of what it built. When state and reality disagree, the fix is usually to repair the state, not to rebuild the resource.
---
 
## What I built
 
### 1. Prepared the project and remote state
I created a project folder with Terraform files in the root and the PowerShell scripts in a `scripts` folder (the orchestration script finds them by relative path). Because Terraform state can contain secrets and Lab 2 needs to share this environment, I stored it in Azure Blob Storage instead of on my disk. That meant creating a resource group, a storage account (the name must be globally unique), and a container first, then pointing `backend.tf` at them:
 
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "RG-TerraformState"
    storage_account_name = "<your-unique-storage-account-name>"
    container_name       = "tfstate"
    key                  = "ntfs-lab.terraform.tfstate"
  }
}
```
 
<img width="631" height="46" alt="NTFSRemoteStateAndConfig" src="https://github.com/user-attachments/assets/4b6c7899-d8f5-4015-a319-2c086ae72fd4" />
 
### 2. Wrote the Terraform configuration
Six files make up the infrastructure:
 
- **`versions.tf`:** the `azurerm`, `random`, and `time` providers.
- **`variables.tf`:** inputs for region, names, VM sizes, my IP for RDP, and the admin password (marked `sensitive`).
- **`main.tf`:** the resource group, VNet, subnet, NSG, public IPs, NICs, the three VMs, and the CLIENT01 RDP extension.
- **`keyvault.tf`:** the Key Vault, a role assignment for my own login, and the secret holding the admin password.
- **`outputs.tf`:** the IP addresses and the Key Vault name.
- **`terraform.tfvars`:** my values, kept out of source control. The admin password comes from an environment variable instead of any file:
```powershell
$env:TF_VAR_admin_password = '<a strong password>'
```
 
The NSG is the key security control. It only allows RDP from my own public IP:
 
```hcl
security_rule {
  name                       = "Allow-RDP-3389"
  priority                   = 100
  direction                  = "Inbound"
  access                     = "Allow"
  protocol                   = "Tcp"
  source_address_prefix      = var.rdp_source   # my IP with /32
  source_port_range          = "*"
  destination_port_range     = "3389"
  destination_address_prefix = "*"
}
```
 
I ran `terraform init`, `terraform validate`, and `terraform plan`. The plan showed 23 resources to create, and reading it carefully caught a mistake (below).
 
<img width="290" height="45" alt="NTFSTerraformValidate" src="https://github.com/user-attachments/assets/218d1018-b5b5-4f46-bdfd-114b06f2ec25" />
<img width="1854" height="425" alt="NTFSTerraformPlan" src="https://github.com/user-attachments/assets/d279e431-8dee-4395-a446-cb7df6a97a8c" />

> I forgot to take a screenshot during this process, but both should showcase the configuration was valid and worked.

 
### 3. Wrote the configuration scripts
Eight PowerShell scripts do the work inside the VMs, and `configure-lab.ps1` runs them in order:
 
| Script | Runs on | What it does |
|---|---|---|
| `00-promote-dc.ps1` | DC01 | Installs AD DS and creates the `lab.local` forest with DNS |
| `01-create-ad-users-groups.ps1` | DC01 | Creates three OUs, four groups, and five users with group memberships |
| `04-domain-join.ps1` | FS01, CLIENT01 | Points DNS at DC01, waits for `lab.local` to resolve, joins the domain, restarts |
| `02-configure-shares-and-permissions.ps1` | FS01 | Creates the four SMB shares and sets NTFS permissions per group |
| `06-add-rdp-users.ps1` | CLIENT01 | Adds `LAB\Domain Users` to the local Remote Desktop Users group |
| `03-configure-rdp-gpo.ps1` | DC01 | Creates and links a GPO that allows RDP, and moves CLIENT01 into the Lab Computers OU |
| `05-verify-ad.ps1`, `05-verify-shares.ps1` | DC01, FS01 | Automated PASS/FAIL checks of the AD objects and the permissions |
 
The orchestration script pulls the admin password from Key Vault at runtime and sends each script to the right VM with `az vm run-command`, which goes through the Azure agent and works even though the NSG only allows port 3389.
 
### 4. Deployed, and worked through the errors
`terraform apply` took several attempts, and `configure-lab.ps1` needed one re-run from the middle. Fourteen separate problems came up before everything worked. Each one is covered in the next section.
 
### 5. Verified the permission model as each user
I connected to CLIENT01 over RDP and signed in as `LAB\` users (using the domain prefix, because a plain username would try a local account), then opened each share by typing its path into File Explorer:
 
| Log in as | Share | Expected | Result |
|---|---|---|---|
| `LAB\sarah.jones` | `\\FS01\Finance` | Read and write | As expected |
| `LAB\sarah.jones` | `\\FS01\HR` | Access Denied | As expected |
| `LAB\lisa.white` | `\\FS01\Finance` | Read only | As expected |
| `LAB\lisa.white` | `\\FS01\HR` | Read and write | As expected |
| `LAB\john.smith` | `\\FS01\IT` | Full control | As expected |
| `LAB\tom.davis` | `\\FS01\Finance` | Access Denied | As expected |

<img width="1909" height="1022" alt="NTFSSarahJonesLogin" src="https://github.com/user-attachments/assets/ef008ec6-d263-440e-a461-6e8bad6feee3" />
<img width="845" height="476" alt="NTFSSarahJonesFinanceShareTrueFinal" src="https://github.com/user-attachments/assets/a5e5a3e9-e673-4b65-9ec4-5d068507cea3" />
<img width="845" height="478" alt="NTFSSarahJonesHRShare" src="https://github.com/user-attachments/assets/f1189f39-622d-4463-a873-3e593781f028" />
<img width="1120" height="254" alt="NTFSSarahJonesGroup" src="https://github.com/user-attachments/assets/58196747-2afc-49b2-a61e-385bacc50fec" />

> Sarah as part of finance can read and write in the Finance share, but has no access to the HR share.

<img width="1907" height="1021" alt="NTFSLisaWhiteLogin" src="https://github.com/user-attachments/assets/8d6bc3a8-ebc1-4cc4-99ac-7c241dc2d3cf" />
<img width="845" height="475" alt="NTFSLisaWhiteHRShare" src="https://github.com/user-attachments/assets/0ad01a4b-d96a-41ea-8998-12928727be0e" />
<img width="844" height="476" alt="NTFSLisaWhiteFinanceShare" src="https://github.com/user-attachments/assets/4d5956ed-5942-4ded-9eef-a5cb8a38d2c9" />
<img width="1123" height="254" alt="NTFSLisaWhiteGroup" src="https://github.com/user-attachments/assets/609e9051-dda1-4aae-a944-f7f1e32a447d" />

> Lisa as part of HR can read and write in the HR share, but only read the Finance share.

<img width="1907" height="1024" alt="NTFSJohnSmithLogin" src="https://github.com/user-attachments/assets/f9f0ac51-d8a0-4ead-9574-c8afa52a47db" />
<img width="843" height="476" alt="NTFSJohnSmithITShare" src="https://github.com/user-attachments/assets/126d8520-e78e-43a7-833f-e91e7c41957e" />
<img width="844" height="476" alt="NTFSJohnSmithFinanceShare" src="https://github.com/user-attachments/assets/a92eeb3a-9e6c-451d-9bda-ecc473b4dbc3" />
<img width="844" height="476" alt="NTFSJohnSmithHRShare" src="https://github.com/user-attachments/assets/8b772273-104d-4b43-9e68-fd356ecc165d" />
<img width="1115" height="256" alt="NTFSJohnSmithGroup" src="https://github.com/user-attachments/assets/da6e2596-7718-41d7-ab93-74650f78c324" />

> John as part of IT can read and write in all file shares.

<img width="1909" height="1023" alt="NTFSTomDavisLogin" src="https://github.com/user-attachments/assets/56d29109-6470-4c78-aba8-57ae11649fa2" />
<img width="844" height="476" alt="NTFSTomDavisFinanceShare" src="https://github.com/user-attachments/assets/a30b66eb-a4c3-4a57-b027-cd5814bc154a" />
<img width="1122" height="256" alt="NTFSTomDavisGroup" src="https://github.com/user-attachments/assets/c5f3764c-bf35-4e70-a904-b0213c3553c5" />

> Tom as part of sales cannot read the Finance share as he is part of sales.


<!-- SCREENSHOT: sarah.jones writing to Finance, then Access Denied on HR -->
<!-- SCREENSHOT: lisa.white failing to create a file on Finance -->
<!-- SCREENSHOT: tom.davis Access Denied on Finance -->
 
---
 
## The errors, one at a time
 
### PowerShell rejected the bash-style commands
```
the following arguments are required: --resource-group/-g, --name/-n
--name : The term '--name' is not recognized ...
```
The guide wrote multi-line `az` commands with `\` at the end of each line, which is bash syntax. In PowerShell, `\` is not a line continuation, so the first line ran with no arguments and every following line ran as its own broken command. Joining the command into one line (or using a backtick at the end of each line) fixed it.
 
### HCL does not allow several arguments on one line in a block
The guide's `variables.tf`, `main.tf`, and `outputs.tf` had short blocks written like `variable "location" { type=string default="Central US" description="..." }`. Terraform only accepts one argument in a single-line block, and long lines in the PDF were also wrapping mid-word when copied. I rewrote the files with one argument per line, and `terraform validate` passed.
 
### `no such host`, three times
```
lookup kv-fslab-<suffix>.vault.azure.net: no such host
lookup management.azure.com: no such host
lookup <storage-account>.blob.core.windows.net: no such host
```
<img width="1029" height="296" alt="NTFSKeyVaultTainted" src="https://github.com/user-attachments/assets/1e31b853-56db-49a4-a2ac-f5e1925ac9d4" />

Terraform could not resolve Azure hostnames from my PC, first for the Key Vault, then for the Azure management API, and finally when releasing the state lock. I tested name resolution against my router and against `1.1.1.1`, and Azure itself showed the vault as `Succeeded`, so the problem was my router's DNS, which seemed to cache the "does not exist yet" answer for brand-new names. Because the first failure left the vault marked **tainted** in state (Terraform would have deleted and recreated a healthy vault), I ran `terraform untaint`. Switching my PC's DNS to public resolvers (`1.1.1.1` and `8.8.8.8`) stopped the failures.

<img width="416" height="62" alt="NTFSUntaint" src="https://github.com/user-attachments/assets/3273757f-cff1-4439-97bf-5cea937c1e62" />
<img width="623" height="160" alt="NTFSDNSFix" src="https://github.com/user-attachments/assets/4930e153-e20f-4ab0-ba96-298760478948" />

### `PrivateIPAddressIsAllocated`: a race for 10.0.1.4
```
IP configuration ... is using the private IP address 10.0.1.4 which is already allocated to resource .../client01-nic
```
Terraform creates the three NICs at the same time. CLIENT01's NIC uses a dynamic IP and finished first, so Azure handed it the lowest free address, `10.0.1.4`, which DC01's NIC needs as a static IP. The fix was making the FS01 and CLIENT01 NICs wait for DC01's NIC:
 
```hcl
depends_on = [time_sleep.wait_after_nsg, azurerm_network_interface.dc01]
```
 
I then destroyed only CLIENT01's NIC (`terraform destroy -target`) to free the address, and re-applied.
 
### `PlatformImageNotFound`
```
The platform image 'MicrosoftWindowsDesktop:windows-11:win11-23h2-pro:latest' is not available.
```
The Windows 11 version in the guide had been retired from the marketplace. I listed the SKUs available in my region with `az vm image list-skus` and switched to `win11-25h2-pro`. I chose the plain Pro SKU because the lab is built on Windows 11 Pro, and skipped the Enterprise, multi-session, and "N" variants.
 
### Quota, then capacity, then a newer disk interface
```
exceeding approved standardBasv2Family Cores quota ... Current Limit: 0
SkuNotAvailable: ... Capacity Restrictions: Standard_D2s_v4
```
<img width="1920" height="447" alt="NTFSQuotaReal" src="https://github.com/user-attachments/assets/40e87114-a6cd-49dc-ba74-1d9141a20e59" />
<img width="1920" height="368" alt="NTFSQuota" src="https://github.com/user-attachments/assets/99d2a6d4-ba38-497e-8fae-660c2ec046ce" />

This was the same quota-versus-capacity pair as my last Terraform lab, in two stages. First, my subscription had a quota of zero for the VM family in the guide, so all three VMs were refused. I read my limits with `az vm list-usage` and picked `Standard_D2s_v4`, a family with quota. Then Azure said it had no *capacity* for that size in Central US. A quota table cannot show that, so I asked Azure which sizes were unrestricted for my subscription with `az vm list-skus`, and picked `Standard_D2as_v7` (2 vCPUs, 8 GiB, with quota in my table). Sizes in the v6 and v7 generations use NVMe disks, and the controller type is fixed when a VM is created, so I set it explicitly on each VM block:
 
```hcl
size                 = var.server_vm_size    # Standard_D2as_v7
disk_controller_type = "NVMe"
```
<img width="788" height="1044" alt="NTFSQuotaFix" src="https://github.com/user-attachments/assets/597d6c0c-46c0-4f7d-ad3a-b7a53919a478" />
<img width="775" height="564" alt="NTFSQuotaFix2" src="https://github.com/user-attachments/assets/3750f4dd-8e10-4167-b34c-45707130ec50" />

All three VMs were created in under two minutes each.
 
### Key Vault certificate-contacts timeout
```
retrieving `contact` for KeyVault: ... context deadline exceeded
```
<img width="1691" height="341" alt="NTFSKeyVaultTimeoutAndFix" src="https://github.com/user-attachments/assets/3c8df2c0-9c09-4bca-97f3-285799470301" />

During its refresh step, Terraform asks the vault for its certificate contacts and that request timed out, which was the same flaky name resolution again. The remaining apply did not touch the vault, so I ran `terraform apply -refresh=false`, which trusts the saved state instead of re-reading every resource. That is safe only when nothing has changed outside Terraform since the last successful refresh.
 
### Extension error: `unimplemented polling status "Unknown"`, and a stuck state lock
```
polling after CreateOrUpdate: internal-error: unimplemented polling status "Unknown"
Error releasing the state lock ... no such host
```
<img width="596" height="299" alt="NTFSPollingStatusAndStateLock" src="https://github.com/user-attachments/assets/38f42ad7-3e9b-4b64-b999-3c5fe3a15b7d" />

Terraform lost track of the CLIENT01 RDP extension while it ran, and the DNS failure then stopped it from releasing the state lock. Before re-running anything I checked what Azure really had:
 
```powershell
az vm extension list -g RG-FileServerLab --vm-name CLIENT01 --query "[].{name:name, state:provisioningState}" -o table
```
 
The extension was `Succeeded`, and `terraform state list` showed the three VMs were saved. I released the leftover lock with `terraform force-unlock <ID>` (safe because I was the only user and no other Terraform process was running), then imported the extension into state:
 
```powershell
terraform import azurerm_virtual_machine_extension.client01_enable_rdp "<extension resource ID>"
```
<img width="1685" height="572" alt="NTFSExtensionErrorFix" src="https://github.com/user-attachments/assets/6b2d71ed-7adb-4a53-8af0-37154da7d403" />

 
### A leftover diff after the import
After the import, `terraform plan` showed three in-place changes: `vm_agent_platform_updates_enabled = true -> false` on each VM. Azure had turned the setting on by default and my files did not mention it. I added `vm_agent_platform_updates_enabled = true` to each VM block, and the next plan reported `No changes`.
 
### The CLIENT01 domain join failed in my own script
```
Set-DnsClientServerAddress : No MSFT_DNSClientServerAddress objects found with property 'InterfaceIndex' equal to '10'.
lab.local did not resolve after 3 minutes.
```
<img width="691" height="517" alt="NTFSPart5Fail" src="https://github.com/user-attachments/assets/9ae22d66-2edd-4709-b13e-b363e39ef16d" />

`configure-lab.ps1` ran for 16 minutes and passed the checks for everything on DC01 and FS01, but the CLIENT01 join failed, and the script still printed "LAB FULLY CONFIGURED". My domain join script chose the first network adapter that was "Up" and set DNS on it. On CLIENT01 that adapter was not the one holding the VM's IP address, so DNS never pointed at DC01 and `lab.local` never resolved. It worked on FS01, so the failure only showed up on the third VM. Two later messages ("Added LAB\Domain Users" and "CLIENT01 not yet in AD") were fallout from the same failure. The fix was to find the interface by its `10.0.1.x` address:
 
```powershell
$nic = Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.IPAddress -like "10.0.1.*" } | Select-Object -First 1
if (-not $nic) { throw "No 10.0.1.x IPv4 address found on this VM." }
Set-DnsClientServerAddress -InterfaceIndex $nic.InterfaceIndex -ServerAddresses "10.0.1.4"
```
 
I also made the RDP group script stop on error instead of printing success, and made the GPO script safe to re-run. Then I re-ran only the second half of the build with a `-StartAtStep` option I had added to the orchestration script:
 
```powershell
.\configure-lab.ps1 -KeyVaultName "kv-fslab-<suffix>" -StartAtStep 5
```
 
This time CLIENT01 resolved `lab.local` on the first attempt, joined, and landed in the Lab Computers OU. All AD and share checks passed again.
 
<img width="544" height="275" alt="NTFSPart5Fix" src="https://github.com/user-attachments/assets/ac134887-c958-4a2e-89ee-45db8b88a3b2" />
 
### Fixes I made before they could bite
Reading the scripts before running them also turned up a few changes I made up front:
 
- **Password injection.** The original script swapped the password into each script with `-replace` and `[regex]::Escape()`. Escaping adds backslashes that a replacement string does not treat as escapes, so a password with a character like `$` or `^` would have been altered silently. I switched to `$script.Replace($key, $value)`, which is plain text substitution.
- **Resource group default.** The script's default had a typo (`RGFileServerLab` without the hyphen).
- **`Wait-VMOnline`.** A VM mid-reboot still reports "VM running", so it now also checks that the Azure agent answers a trivial command.
- **Cost control.** The guide's pause command, `az vm stop`, leaves VMs allocated and billing. `az vm deallocate` is the command that stops compute charges.
 
---
 
## Problems and fixes
 
| What broke | Cause | Fix |
|---|---|---|
| `az storage account create \` failed in PowerShell | `\` line continuation is bash syntax | Joined into one line (or used backticks) |
| Invalid single-argument block errors | Several arguments on one line in the guide's HCL | One argument per line |
| `no such host` (Key Vault, management API, state lock) | My router's DNS failed on new Azure hostnames | Public DNS resolvers, `terraform untaint` for the vault |
| `PrivateIPAddressIsAllocated` for `10.0.1.4` | CLIENT01's dynamic NIC won the race for DC01's static address | `depends_on` the DC01 NIC, destroyed CLIENT01's NIC, re-applied |
| `PlatformImageNotFound` | The Windows 11 SKU was retired | Listed SKUs, switched to `win11-25h2-pro` |
| `standardBasv2Family` quota of 0 | New subscription, no quota for that family | Read `az vm list-usage`, chose a family with quota |
| `SkuNotAvailable` for `Standard_D2s_v4` | No capacity in Central US (separate from quota) | `az vm list-skus` for unrestricted sizes, used `Standard_D2as_v7` with `disk_controller_type = "NVMe"` |
| `context deadline exceeded` on the Key Vault | Same flaky name resolution during refresh | `terraform apply -refresh=false` |
| `unimplemented polling status "Unknown"` and a stuck lock | Provider quirk plus the DNS failure | Checked Azure with `az vm extension list`, `force-unlock`, `terraform import` |
| `plan` showed `vm_agent_platform_updates_enabled` changes | Azure default not in my config | Added the setting to each VM |
| CLIENT01 domain join failed | Script set DNS on an adapter that did not hold the VM's IP | Selected the interface by its `10.0.1.x` address, re-ran with `-StartAtStep 5` |
 
---
 
## Key takeaways
 
1. **Read the plan before you apply.** The placeholder IP showed up in the plan and would have caused a failure or a lockout. The plan is cheap and the apply is not.
2. **A success message is not a result.** The orchestration script announced a fully configured lab while one VM had failed to join. I now check the verification output, not the banner, and the scripts stop on error instead of printing success.
3. **Quota, capacity, and image support are three separate checks.** `az vm list-usage` answers "am I allowed", `az vm list-skus` answers "can I actually deploy it", and `az vm image list-skus` answers "does this image exist". I used to rely on the first one alone.
4. **Network problems on my own side can look like cloud problems.** Three different Terraform errors came from one flaky DNS resolver. Testing the same name against two resolvers separated my network from Azure.
5. **When an apply fails partway, check reality before re-running.** `az vm extension list` and `terraform state list` showed what existed, and that decided between re-applying, importing, and unlocking.
6. **Parallel resource creation can create races.** Terraform builds independent resources at once, and a static IP that matters needs an explicit dependency.
7. **Pasted code needs a read-through.** Line-wrapped PDF text, bash syntax in a PowerShell session, and regex escaping in a replacement string all looked fine at a glance.
8. **Stopped and deallocated are different.** Only deallocating a VM stops compute billing.
If I did this again, I would run `az vm list-skus` and `az vm image list-skus` before the first `apply`, set `depends_on` on the NICs and the Windows 11 SKU from the start, and put the public DNS resolvers in place before running Terraform at all.
 
---
 
## Security notes
 
This is a lab configuration. Some of it is solid and some of it would not be acceptable in production:
 
**What the lab does well**
- **RDP is restricted to my own IP** (`/32`) instead of the whole internet.
- **The admin password is never stored in a file.** It is passed in as an environment variable, stored in Key Vault, and read back at runtime with my own login.
- **No extra management ports are open.** Scripts reach the VMs through the Azure agent, not through WinRM.
- **Credentials are kept out of source control.** `terraform.tfvars` is excluded with `.gitignore`, and the state lives in remote storage instead of the repo.
**What I would change for production**
- **Every VM has a public IP with RDP exposed** (to my IP only). Production would use Azure Bastion or a VPN and no public IPs.
- **The test users share a known password** (`P@ssw0rd123!`) and it never expires. That is fine for a lab and wrong for real accounts.
- **Terraform state contains secrets.** It sits in a Blob Storage container I control, and it still holds the admin password in plain text.
- **The orchestration script writes a temporary file containing the password** while it runs and deletes it afterward. It is short-lived, but it does touch disk.
- **`lab.local` is a lab-only domain name.** Production should use a subdomain of a domain I own, because `.local` collides with mDNS.
- **`enable_rbac_authorization` is deprecated** in newer provider versions (renamed `rbac_authorization_enabled`). I pinned the provider to 3.x, where it still works.
---
 
## Verification
 
| Check | How I verified it |
|---|---|
| Remote state works | The container list command returned `tfstate` and `terraform init` succeeded |
| Configuration valid | `terraform validate` returned `Success! The configuration is valid.` |
| Terraform matches Azure | After the import and the VM setting, `terraform plan` reported `No changes` |
| All three VMs running | `az vm list -d` showed `VM running` on `Standard_D2as_v7` for DC01, FS01, and CLIENT01 |
| Domain controller promoted | `configure-lab.ps1` step 1 finished with `Operation completed successfully` |
| AD objects exist | `05-verify-ad.ps1` passed 12 of 12 checks (3 OUs, 4 groups, 5 users with memberships) |
| Shares and NTFS permissions correct | `05-verify-shares.ps1` passed every check on all four shares |
| CLIENT01 is in the domain | The re-run joined it and moved it into the Lab Computers OU |
| Access works as designed | Signed in as each test user and confirmed all six scenarios behaved as expected |
 
---
 
## Teardown
 
Running VMs bill by the hour. Because Lab 2 needs this environment to still exist, I deallocate the VMs instead of destroying them:
 
```powershell
# Pause: stops compute charges (deallocate, not just stop)
az vm deallocate --ids (az vm list -g RG-FileServerLab --query "[].id" -o tsv) --no-wait
 
# Resume (start DC01 first and give it a couple of minutes)
az vm start -g RG-FileServerLab -n DC01
az vm start -g RG-FileServerLab -n FS01 --no-wait
az vm start -g RG-FileServerLab -n CLIENT01 --no-wait
 
# Full teardown, only when I am done with the series
terraform destroy
```
<img width="643" height="151" alt="NTFSDeallocating" src="https://github.com/user-attachments/assets/dcb4ae04-56bc-4a83-afaa-0be4e5271a19" />

Deallocated VMs still cost a little for their disks and static public IPs. The remote state resource group is not managed by this Terraform project, so `terraform destroy` does not remove it.
 
---
 
## Next steps
 
- **Lab 2:** Azure RBAC role assignments on FS01, building on this environment and sharing the same state storage under a different key
- Set DNS at the VNet level (`dns_servers` on the VNet) instead of inside each VM, so it survives NIC changes
- Use domain local groups for resource permissions (AGDLP) instead of putting global groups directly on folders
- Replace public IPs and RDP with Azure Bastion
- Upgrade to `azurerm` 4.x and rename the deprecated Key Vault argument
- Add the Splunk forwarder and Nessus scan from my other labs on top of this environment
---
 
**Status: complete.** Terraform deployed the network, Key Vault, and three VMs → fourteen problems diagnosed and fixed → AD, shares, and NTFS permissions configured by script → all six access scenarios verified as the test users.
