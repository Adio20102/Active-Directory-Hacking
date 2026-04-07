# Active Directory Penetration Testing Lab

A hands-on home lab simulating real-world Active Directory attacks 
in a controlled environment, built as part of the TCM Security 
Practical Ethical Hacking (PEH) course.

## Lab Environment
- **Domain Controller:** Windows Server 2019 (HYDRA-DC, MARVEL.local)
- **Workstations:** Windows 10 Enterprise (THEPUNISHER, SPIDERMAN)
- **Attacker Machine:** Kali Linux

## Attacks Covered
- LLMNR Poisoning
- SMB Relay
- IPv6 DNS Takeover
- Pass-the-Hash
- Token Impersonation
- Kerberoasting
- GPP Password Attack
- Credential Dumping with Mimikatz
- Golden Ticket Attack

## Tools Used
Responder, Impacket, Metasploit, Mimikatz, BloodHound, 
PowerView, CrackMapExec, Hashcat

## ⚠️ Disclaimer
All credentials in this repo (e.g., Password1, P@$$w0rd!) are 
intentionally weak and used solely for lab/educational purposes 
in an isolated virtual environment. Do not use in production.
