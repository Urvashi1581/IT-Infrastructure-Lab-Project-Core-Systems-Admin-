# IT-Infrastructure-Lab-Project-Core-Systems-Admin-

Goal: Build a realistic, two-site environment to secure site-to-site networking, Active Directory with Sites & Services, a branch RODC for resilient logons, software/patch management, a light M&A (staged) migration, and a Backup/Restore drill. All implemented and validated in VMware Workstation on a single laptop.

Why this matters (business impact)
Keep the branch productive: Staff can still sign in if the WAN/VPN drops (RODC caches allowed passwords).
Reduce risk: Branch gets a read-only domain controller and least-privilege file/share access.
Faster onboarding: Staged migration pulls users/computers from an acquired org with minimal disruption.
Recover from mistakes/malware: Daily backups and a tested file-level restore with measured RPO/RTO.

Topology

![WhatsApp Image 2025-09-01 at 18 05 43_50402205](https://github.com/user-attachments/assets/46ae1009-8d80-49dd-9b4c-ad4b7d3fad1c)


Domains: home.local (primary) and corp.local (tiny “acquired” forest used for the migration demo).
Sites: HQ (10.10.0.0/24), BRANCH1 (10.20.0.0/24).



What I built (step-by-step)

0) Prereqs & Lab Networks

VMware Workstation networks:
VMnet8 (NAT) → “Internet/WAN” for both pfSense WAN NICs.
VMnet2 (Host-Only) → HQ-LAN 10.10.0.0/24.
VMnet3 (Host-Only) → BRANCH1-LAN 10.20.0.0/24.
ISOs: Windows Server 2022/2019, Windows 11 Pro, pfSense CE, 7-Zip MSI.

1) Edge Routers + Site-to-Site VPN (pfSense)

Built pfSense-HQ (LAN 10.10.0.1) and pfSense-BR1 (LAN 10.20.0.1).
Configured OpenVPN site-to-site and firewall rules; verified routing both ways.
Test: Ping BR1 from HQ and vice-versa; confirm LAN-to-LAN connectivity.

2) HQ Core (DC1: AD DS, DNS, DHCP)

Promoted DC1 to new forest cccu.local; DNS running locally.
Created DHCP scope 10.10.0.100–200 (GW 10.10.0.1, DNS 10.10.0.10).
Designed base OU tree (_home\Users, _home\Computers, _HQ, _BRANCH1, Finance, IT) and created sample groups/users.

3) File Services (FS1)

Joined FS1 to domain; created Dept_Finance share with NTFS + Share permissions:
GG_Finance_RW = Modify; Authenticated Users = Read.
Optional: FSRM quotas/file screens and DFS namespace.
Test: Finance user can read/write; non-Finance user denied.

4) Group Policy & Security Baseline

Default Domain Policy: length ≥ 12, lockout after 5 attempts.
Base-Workstation-Security GPO: firewall on, disable SMBv1, disable guest.
USB-Block (Finance OU): deny removable storage.
Map-Drives-Finance: map U: → \\FS1\Dept_Finance.
BitLocker-Laptops: allow w/out TPM for demo; escrow keys to AD (optional).
Validate: gpupdate /force, gpresult /r on clients.

5) Software Deployment via GPO (7-Zip)

Placed MSI on read-only share (e.g., \\FS1\Software$).
Computer Config → Software installation → new package.
Linked to pilot OU; rebooted clients to confirm 7-Zip appears in Programs & Features.

6) WSUS Patching (optional but strong)

Built WSUS1, synced Microsoft updates, selected products/classifications.
GPO pointed clients to http://WSUS1:8530.
Validate: detection/scan and compliance snapshot.

7) Branch Site, Subnets & RODC

AD Sites & Services: Added BRANCH1 site; mapped 10.20.0.0/24 to it.
Built DC2 at branch; joined as Read-Only Domain Controller (RODC) in BRANCH1.
Password Replication Policy: allowed branch users/computers (pre-populated for test).
Branch DHCP:
Option A: Central DHCP at DC1 + DHCP relay on pfSense-BR1.
Option B: DHCP on DC2 (scope 10.20.0.100–200).
Outage test: stopped OpenVPN temporarily → branch users still signed in via RODC.

8) Helpdesk Scenarios (ticket-style)

Wrote 4–6 short KBs: mail flow fix (Autodiscover/DNS), share access (group membership), GPO-site mismatch (fix subnets), slow logons (RODC DNS), printer deploy via GPO.

9) Light M&A (staged migration)

Built small corp.local forest (corp-DC1 + ACME-CL1).
Exported corp users → transformed CSV → imported to home.local with PowerShell (New-ADUser).
Computer migration (simulated): disjoined ACME-CL1 from acme.local → joined to cccu.local (netdom or GUI).
GPOs auto-delivered software, drive maps, and baselines.
Re-ACL data with icacls: granted new CCCU groups; removed legacy ACME SIDs (documented).

10) Backup/Restore (BC/DR Drill)

Added a dedicated backup VMDK to FS1; installed Windows Server Backup.
Scheduled daily backup of shares; took an on-demand backup.
Simulated data loss (rename/delete), then restored files.
Captured RPO/RTO and success screenshots.



Tech used
pfSense, OpenVPN, Windows Server AD DS/DNS/DHCP, RODC, Group Policy/GPP, WSUS, Windows Server Backup, PowerShell, VMware Workstation.
