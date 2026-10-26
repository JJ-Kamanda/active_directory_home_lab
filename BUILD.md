# Build Guide — Active Directory Home Lab
### Windows Server 2025 · Windows 11 Pro · Microsoft Hyper-V

[← Back to project overview](README.md)

This guide documents how I built and validated a small, virtualized Active Directory environment on a Windows 11 Pro computer. The initial infrastructure was configured primarily through the **Hyper-V and Windows Server graphical interfaces**, not through an automated VM-deployment script. Commands below are either validation commands used during the build or explicitly marked as **optional checks**.

The build is complete through **domain joining and domain authentication**. Additional user management, Group Policy, and automation exercises are planned, not part of this initial deployment.

---

## 1. Lab specifications

| Component | Configuration |
|:---|:---|
| Host operating system | Windows 11 Pro |
| Hypervisor | Microsoft Hyper-V |
| Domain controller VM (Hyper-V name) | `DomainController` |
| Domain controller guest hostname | `DC01` |
| Server operating system | Windows Server 2025 Standard Evaluation |
| Workstation VM / hostname | `Client1` / `CLIENT1` |
| Workstation operating system | Windows 11 Pro |
| Active Directory domain | `mydomain.com` |
| NetBIOS domain name | `MYDOMAIN` |
| Internal Hyper-V switch | `AD-LAB-internal` |
| External switch (DC01 only) | Hyper-V `Default Switch` |
| DC01 internal IPv4 address | `172.16.0.1/24` |
| DC01 internal NIC DNS setting | `127.0.0.1` (local DNS service) |
| DHCP address pool | `172.16.0.100`–`172.16.0.200` |
| DHCP subnet mask | `255.255.255.0` |
| DHCP default gateway | `172.16.0.1` |
| DHCP DNS server | `172.16.0.1` |
| DHCP DNS domain | `mydomain.com` |
| Observed Client1 DHCP address | `172.16.0.101` |
| Working external DNS forwarder | `1.1.1.1` |

> **Scope and security:** This is an isolated training environment, not a production domain. No passwords, recovery keys, or reusable credentials are included in this repository. VM sizes were adjusted for the available host resources and are not presented as production sizing recommendations.

### Network topology

```text
Windows 11 Pro host — Microsoft Hyper-V
│
├── Hyper-V Default Switch (external access)
│     │
│     └── DC01 external NIC (address allocated by Default Switch)
│           │
│           └── DC01 — Windows Server 2025
│                 AD DS | DNS | DHCP | RRAS/NAT
│                 Internal NIC: 172.16.0.1/24
│                       │
└── AD-LAB-internal -----┘
          │
          └── Client1 — Windows 11 Pro
                DHCP lease: 172.16.0.101/24
                Gateway + DNS: 172.16.0.1
                Domain: mydomain.com
```

DC01 has **two network adapters**. The internal adapter connects to Client1 and provides domain services. The external adapter uses the Hyper-V Default Switch. Client1 has **only the internal lab connection**, and its outbound traffic passes through DC01's routing/NAT configuration.

---

## 2. Prepare Hyper-V and create the virtual machines

1. Enabled **Hyper-V** on the Windows 11 Pro host and opened **Hyper-V Manager**.
2. Created the virtual switch named `AD-LAB-internal` for communication between the virtual machines.
3. Created the `DomainController` VM, attached Windows Server 2025 installation media, and installed **Windows Server 2025 Standard Evaluation** with the graphical interface.
4. Configured two network adapters for the domain controller: one connected to `AD-LAB-internal` and one to the Hyper-V **Default Switch**.
5. Created a separate Generation 2 VM for `Client1`, attached Windows 11 installation media, and connected it to `AD-LAB-internal`.
6. Enabled the client VM's **Secure Boot** and **virtual TPM** to meet Windows 11's virtual hardware requirements.
7. Used Hyper-V checkpoints at meaningful configuration milestones to provide rollback points during setup.

**Design choice:** The virtual machines were created and configured manually. This build does **not** use Ken Mwara's `create-lab-vms.ps1` script, and it should not be listed as work performed here.

---

## 3. Configure DC01 networking and server identity

On the server VM:

1. Renamed the Windows Server guest to **`DC01`**, then restarted when required.
2. Identified the adapter connected to the lab switch and assigned its static network settings:

   | Setting | Value |
   |:---|:---|
   | Internal IPv4 | `172.16.0.1` |
   | Subnet prefix | `/24` (`255.255.255.0`) |
   | Internal DNS | `127.0.0.1` |
   | Internal adapter default gateway | None; this machine acts as the gateway for the lab |

3. Left the second adapter connected to the Hyper-V Default Switch for outbound network access. Its dynamically allocated address may change as Hyper-V reconfigures the Default Switch.
4. Checked that the internal and external adapters served distinct purposes before adding domain services.

**Why this matters:** A stable internal IPv4 address gives the domain clients a predictable DNS server and gateway. Windows clients joining an AD domain must be able to discover the domain controller through DNS.

---

## 4. Install Active Directory Domain Services and create the forest

On **DC01**:

1. Opened **Server Manager → Manage → Add Roles and Features**.
2. Installed **Active Directory Domain Services (AD DS)** and the associated management tools.
3. Selected **Promote this server to a domain controller**.
4. Chose **Add a new forest** and entered `mydomain.com` as the root domain name.
5. Included **DNS Server** during the domain controller configuration, completed the prerequisite checks, and promoted the server.
6. Restarted and signed in using domain administrator credentials.
7. Opened **Server Manager → Tools → Active Directory Domains and Trusts** and confirmed that `mydomain.com` appeared.

**Result:** `DC01` provides Active Directory domain services for the new `MYDOMAIN` domain.

---

## 5. Create organizational units and administrative access

Opened **Server Manager → Tools → Active Directory Users and Computers** (`dsa.msc`) and selected the `mydomain.com` domain.

1. Created the `_ADMINS` organizational unit (OU) for administrative accounts.
2. Created the `_EMPLOYEES` OU as the starting point for future standard user accounts.
3. Created a separate administrative user account and added it to **Domain Admins**.
4. Verified the OU structure and the administrative account in Active Directory Users and Computers.

| Directory object | Purpose |
|:---|:---|
| `_ADMINS` | Separate administrative identities from standard user accounts |
| `_EMPLOYEES` | Organize standard domain users in later exercises |
| Domain Admins membership | Permit domain-wide administrative actions in the lab |

**Limit of this phase:** A bulk user-provisioning script and large employee account population are **not** claimed as completed in this guide. Those belong to a future exercise if implemented and verified.

---

## 6. Configure Routing and Remote Access with NAT

On **DC01**:

1. Installed the **Remote Access / Routing** role through Server Manager.
2. Opened **Tools → Routing and Remote Access**.
3. Configured and enabled **Network Address Translation (NAT)**.
4. Designated the adapter connected to the Hyper-V Default Switch as the **public-facing** interface.
5. Used the adapter connected to `AD-LAB-internal` as the **private-facing** interface.
6. Verified that the routing service was enabled, then later tested outbound connectivity from Client1.

**Traffic path:** `Client1 → 172.16.0.1 (DC01) → NAT → Hyper-V Default Switch → external network`.

This allowed the workstation to remain on the internal lab network rather than being directly attached to the Default Switch.

---

## 7. Install and configure DHCP

On **DC01**:

1. Installed the **DHCP Server** role and completed its post-installation configuration.
2. Opened **Server Manager → Tools → DHCP**.
3. Configured an IPv4 scope for the `172.16.0.0/24` lab subnet.
4. Set the lease range and scope options as follows:

   | Scope option | Value |
   |:---|:---|
   | Starting address | `172.16.0.100` |
   | Ending address | `172.16.0.200` |
   | Subnet mask | `255.255.255.0` |
   | Router (default gateway) | `172.16.0.1` |
   | DNS server | `172.16.0.1` |
   | DNS domain | `mydomain.com` |

5. Started/activated the DHCP scope and checked the settings from the Windows client.

On **Client1**, Windows received the following settings automatically:

```text
IPv4 address : 172.16.0.101
Subnet mask  : 255.255.255.0
Gateway      : 172.16.0.1
DNS server   : 172.16.0.1
```

This successful lease provided functional evidence that the client's virtual NIC could reach the domain controller's DHCP service.

---

## 8. Install Windows 11 Pro and join the domain

On the **Client1** virtual machine:

1. Installed **Windows 11 Pro** with Secure Boot and virtual TPM enabled.
2. Named the device **`Client1`** and restarted as needed.
3. Confirmed the VM was connected to the `AD-LAB-internal` switch.
4. Checked client networking and domain discovery **before joining**:

   ```cmd
   ipconfig /all
   ping 172.16.0.1
   nslookup mydomain.com
   nslookup -type=SRV _ldap._tcp.dc._msdcs.mydomain.com
   ```

5. Joined the workstation to **`mydomain.com`** using domain administrator credentials.
6. Received the successful **“Welcome to the mydomain.com domain”** message and restarted.
7. On **DC01**, opened **Active Directory Users and Computers → mydomain.com → Computers** and confirmed that the **`CLIENT1`** computer object had been created.
8. Returned to Client1 and selected **Other user** at the Windows sign-in screen to log in with the domain administrator account.

To verify that Windows authenticated the domain identity rather than a local identity, I opened Command Prompt on Client1:

```cmd
whoami
```

Output observed:

```text
mydomain\administrator
```

**Result:** The machine was joined to the domain, registered in Active Directory, and successfully accepted a domain account at sign-in.

---

## 9. Validation checklist

These checks were completed during the lab build. Commands are shown with the machine on which they were used.

| Check | Where | Command or method | Observed result |
|:---|:---|:---|:---|
| DHCP assignment | Client1 | `ipconfig /all` | `172.16.0.101`; gateway/DNS `172.16.0.1` |
| DC connectivity | Client1 | `ping 172.16.0.1` | Successful; 0% packet loss |
| Domain DNS | Client1 | `nslookup mydomain.com` | Domain resolved |
| AD discovery | Client1 | `nslookup -type=SRV _ldap._tcp.dc._msdcs.mydomain.com` | LDAP SRV record pointing to `dc01.mydomain.com` on port `389` |
| Internet access | Client1 | `ping www.google.com` | Hostname resolved and ping succeeded |
| AD computer object | DC01 | Active Directory Users and Computers | `CLIENT1` appeared in `Computers` |
| Domain authentication | Client1 | `whoami` | `mydomain\administrator` |

**Additional optional revalidation commands** (useful after configuration changes, not claimed as commands executed in the initial build):

```powershell
# On DC01: display configured DHCP scopes
Get-DhcpServerv4Scope

# On DC01: inspect DNS forwarders
Get-DnsServerForwarder

# On Client1: identify the authentication server
$env:LOGONSERVER
```

---

## 10. Troubleshooting: external DNS resolution failure

### Symptom

DC01 initially had working outbound connectivity, but it could not consistently resolve external hostnames. The issue emerged after the Hyper-V Default Switch network changed, leaving the existing DNS forwarder pointing at an outdated address.

### Diagnostic approach

1. Tested outbound TCP connectivity to distinguish loss of network access from name-resolution failure. **Outbound TCP port 443 was reachable**, so the connection was not entirely down.
2. Examined the DNS forwarding configuration and identified the outdated forwarder **`172.22.80.1`**.
3. Recognized that the Default Switch can be reassigned to a different network, so a previously usable forwarding address might stop working.

### Fix and validation

Updated DC01's DNS forwarder to **`1.1.1.1`** and tested resolution:

```powershell
Resolve-DnsName google.com
```

The name resolved successfully, and Microsoft Edge on DC01 loaded Google afterward.

**Lesson:** Internet reachability and DNS resolution are different tests. Checking connectivity first, then DNS, helped isolate the failure to a stale forwarder rather than the NAT path.

### Other setup issues encountered

| Issue | Action / lesson |
|:---|:---|
| Windows 11 VM did not initially meet installation requirements | Enabled the appropriate Hyper-V security settings, including virtual TPM and Secure Boot. |
| Client could not use domain discovery without the right DNS settings | Pointed the client at DC01 (`172.16.0.1`) and verified the LDAP SRV record. |
| Hyper-V resource pressure on the host | Adjusted VM resources and reduced unnecessary host applications while working with both VMs. |

> **Configuration note:** A domain controller with internal and external NICs should avoid registering an inappropriate external address in its AD DNS records. The external adapter's DNS registration settings are worth reviewing as a later hardening/cleanup task; this guide does **not** claim that cleanup has already been completed.

---

## 11. Evidence, documentation boundaries, and next steps

Proof of the completed milestones is stored in the repository's [`screenshots/`](screenshots/) directory, and the **README's [Implementation Milestones](README.md#implementation-milestones)** table links to the relevant numbered evidence. This build guide remains primarily text-based so it can function as a usable technical runbook.

The screenshots cover the network and server setup, domain configuration, installed server roles, DHCP scope, Active Directory objects, the joined client, and a verified domain sign-in. Screenshot filenames will be reconciled with the actual files before publication.

### Planned extensions — not yet completed

- Create and sign in as **standard** domain users (rather than using an administrator account for validation).
- Practice user creation, password resets, unlocks, access changes, and account disablement.
- Test PowerShell automation for employee account provisioning and administration.
- Configure and validate Group Policy on the client workstation.
- Diagnose intentional DNS, authentication, and domain connectivity faults.
- Rebuild the lab independently as a repeatability exercise.

---

*This guide describes a personal practice environment and reflects the work verified during the initial infrastructure build. It is not a production hardening guide. Credentials and other secrets have been excluded.*
