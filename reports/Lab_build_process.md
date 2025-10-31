Active Directory & Splunk Detection Lab: Full Process Report
This document details the end-to-end process for building a fully functional cybersecurity detection lab. The lab is designed to simulate a small corporate network, ingest security logs into a SIEM, and then run simulated attacks to test detection capabilities.
The final architecture consists of four virtual machines:
•	ADDC01: A Windows Server 2022 Domain Controller (abu.local).
•	abu@ubuntuserver: An Ubuntu Server running Splunk Enterprise.
•	target-PC: A Windows 10 victim workstation, joined to the domain.
•	Kali-Attacker: A Kali Linux machine for Red Team operations.
Phase 1: Virtual Network & VM Setup
1.1. Create Virtual Network
The first step was to create an isolated network for the VMs to communicate.
1.	Tool: VirtualBox
2.	Action: Navigated to File > Tools > Network Manager.
3.	Configuration:
o	Created a new NAT Network named AD-Project.
o	Set the Network CIDR to 192.168.10.0/24.
o	Ensured Enable DHCP was checked.
1.2. Provision Virtual Machines
The following four VMs were created and downloaded:
•	ADDC01: Windows Server 2022 (Standard w/ Desktop Experience)
•	abu@ubuntuserver: Ubuntu Server 22.04
•	target-PC: Windows 10 Enterprise
•	Kali-Attacker: Kali Linux (Pre-built VM image)
All four VMs were configured in VirtualBox settings to have their Network Adapter 1 "Attached to: NAT Network" with the Name: "AD-Project".
Phase 2: Splunk SIEM Server Configuration (abu@ubuntuserver)
2.1. Set Static IP
The Ubuntu server was configured with a static IP to be a reliable target for logs.
1.	File Edited: /etc/netplan/00-installer-config.yaml
2.	Configuration:
3.	network:
4.	  ethernets:
5.	    enp0s3:
6.	      dhcp4: no
7.	      addresses: [192.168.10.10/24]
8.	      routes:
9.	        - to: default
10.	          via: 192.168.10.1
11.	      nameservers:
12.	        addresses: [8.8.8.8]
13.	  version: 2
14.	Command: sudo netplan apply was run to apply the changes.
2.2. Install Splunk Enterprise
1.	File Transfer: The Splunk Enterprise .deb package was downloaded on the host machine. A VirtualBox Shared Folder was used to transfer the file to the Ubuntu VM.
2.	Disk Resizing: The initial install failed due to No space left on device. The VM's disk was resized using the LVM command-line tools (growpart, pvresize, lvextend, resize2fs). (See Troubleshooting Report for details).
3.	Installation: sudo dpkg -i splunk-enterprise-package.deb
4.	First-Time Setup:
o	Started Splunk: /opt/splunk/bin/splunk start --accept-license
o	Created admin user and password.
o	Enabled Splunk to start on boot: /opt/splunk/bin/splunk enable boot-start
5.	Splunk Web UI Configuration:
o	Logged into Splunk at http://192.168.10.10:8000.
o	Enabled Log Receiving: Navigated to Settings > Forwarding and receiving > Configure receiving. Created a new "Receive data" port on 9997.
o	Created Index: Navigated to Settings > Indexes. Created a new index named endpoint.
Phase 3: Endpoint & AD Server Configuration
3.1. Install Sysmon
On both target-PC (Windows 10) and ADDC01 (Windows Server), Sysmon was installed for advanced logging.
1.	Configuration File: The popular Olaf Hartong Sysmon configuration was downloaded.
2.	Command: sysmon.exe -accepteula -i sysmonconfig.xml
3.2. Install Splunk Universal Forwarder
On both target-PC and ADDC01, the Splunk Universal Forwarder was installed.
1.	During the graphical installer, the Receiving Indexer was set to 192.168.10.10:9997.
2.	The Deployment Server was left blank.
3.3. Configure inputs.conf
On both target-PC and ADDC01, the file C:\Program Files\SplunkUniversalForwarder\etc\apps\search\local\inputs.conf was created to tell the forwarder which logs to send.
[WinEventLog://Application]
disabled = 0
index = endpoint

[WinEventLog://Security]
disabled = 0
index = endpoint

[WinEventLog://System]
disabled = 0
index = endpoint

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
index = endpoint

[WinEventLog://Microsoft-Windows-PowerShell/Operational]
disabled = 0
index = endpoint
3.	The SplunkForwarderService was Restarted on both machines via services.msc to apply the new configuration. After troubleshooting, logs from both hosts appeared in Splunk under index=endpoint.
3.4. Configure Active Directory (ADDC01)
1.	Static IP: Set the server's static IP to 192.168.10.7 and its DNS server to 127.0.0.1.
2.	Install AD DS: Opened Server Manager and used the "Add Roles and Features" wizard to install the Active Directory Domain Services role.
3.	Promote to Domain Controller:
o	Promoted the server to a new forest.
o	Set the Root domain name: abu.local.
o	Set a DSRM password.
4.	Configure DNS Forwarder:
o	Opened Server Manager > Tools > DNS.
o	Right-clicked the server (ADDC01) > Properties > Forwarders tab.
o	Added 8.8.8.8 as a forwarder to allow the domain to resolve external internet addresses.
5.	Create Users & OUs:
o	Opened Active Directory Users and Computers (ADUC).
o	Created new Organizational Units (OUs) for IT and HR.
o	Created a user Aryan Singh (asingh) in the IT OU.
o	Created a user Akansha Priya (apriya) in the HR OU.
o	Set complex passwords for both (e.g., PassWord@123!).
3.5. Join Workstation to Domain (target-PC)
1.	Set Static IP: Set the target-PC IP to 192.168.10.100.
2.	Set DNS: Critically, set the target-PC's Preferred DNS server to 192.168.10.7 (the Domain Controller).
3.	Join Domain:
o	Navigated to System Properties > Change...
o	Changed from "Workgroup" to "Domain:" and entered abu.local.
o	Authenticated with the domain admin credentials to join the computer to the domain.
o	Rebooted the machine and logged in as the domain user ABU\asingh.
Phase 4: Attacker Machine Configuration (Kali-Attacker)
1.	Set Static IP: The VM was re-installed to fix a VirtualBox networking bug. The static IP was then set using nmcli to ensure it was permanent.
2.	# Create a new, clean connection profile tied to the correct interface
3.	sudo nmcli connection add type ethernet con-name "LabNetwork" ifname eth0 ipv4.method manual ipv4.addresses 192.168.10.250/24 ipv4.gateway 192.168.10.1 ipv4.dns "192.168.10.7"
4.	
5.	# Bring the new connection up
6.	sudo nmcli connection up "LabNetwork"
7.	Install Tools: Installed hydra and crowbar (sudo apt-get install hydra crowbar).
Phase 5: Attack, Detection, & Analysis
Attack 1: RDP Brute Force (MITRE T1110.001)
1.	Preparation (on target-PC):
o	Enabled "Remote Desktop" in System Properties.
o	Added ABU\asingh to the "Remote Desktop Users" group.
o	Disabled Network Level Authentication (NLA) to allow older tools (like hydra) to connect.
2.	Attack (on Kali-Attacker):
o	A password list passwords.txt was created, including the known password PassWord@123.
o	crowbar failed, so Hydra was used:
3.	hydra -V -t 4 -l asingh -P passwords.txt rdp://192.168.10.100
o	The attack succeeded and found the password.
4.	Detection (in Splunk):
o	Query: index=endpoint host=target-PC EventCode=4625
o	Result: This query returned 21 events for "An account failed to log on," all for the user asingh and originating from the Kali IP (192.168.10.250), confirming the brute-force attack.
Attack 2: Local Account Creation (MITRE T1136.001)
1.	Preparation (on target-PC):
o	Atomic Red Team was installed.
o	The missing powershell-yaml dependency was installed via Install-Module powershell-yaml -Force.
o	The main framework was installed: Install-AtomicRedTeam -getAtomics.
2.	Attack (on target-PC):
o	The module was imported: Import-Module AtomicRedTeam
o	The test was run: Invoke-AtomicTest T1136.001
o	This test successfully created a new local user (Newlocaluser), added it to the Administrators group, and then deleted it.
3.	Detection (in Splunk):
o	Query 1 (Creation): index=endpoint host=target-PC EventCode=4720 TargetUserName=Newlocaluser
o	Query 2 (Privilege Escalation): index=endpoint host=target-PC EventCode=4732 Group_Name=Administrators MemberName=Newlocaluser
o	Result: All events were successfully found, showing the full lifecycle of the attacker's persistence attempt.
Attack 3: System Information Discovery (MITRE T1082)
1.	Preparation (on target-PC):
o	All Windows Defender protections (Real-time, Cloud-delivered, etc.) were disabled to allow the test script to run unblocked.
2.	Attack (on target-PC):
o	Invoke-AtomicTest T1082
o	This test ran a series of commands (whoami.exe, systeminfo.exe, netstat.exe, etc.) to gather information about the host.
3.	Detection (in Splunk):
o	Query 1 (Sysmon): index=endpoint host=target-PC sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1
o	Query 2 (PowerShell): index=endpoint host=target-PC EventCode=4104
o	Result: Both queries successfully detected the attack. The EventCode=1 query showed the suspicious cluster of process creations, and EventCode=4104 captured the exact PowerShell script block content.
Attack 4: Process Injection (MITRE T1055.001)
1.	Preparation (on target-PC):
o	All Windows Defender protections were confirmed to be disabled.
2.	Attack (on target-PC):
o	Invoke-AtomicTest T1055.001 -TestNumbers 1
o	This test simulated injecting a malicious DLL into a PowerShell process.
3.	Detection (in Splunk):
o	Query: index=endpoint host=target-PC sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=8 SourceImage="*\\powershell.exe"
o	Result: The query successfully identified the EventCode=8 (CreateRemoteThread) event where powershell.exe was the source image, confirming detection of the injection.

