Active Directory & Splunk Detection Lab: Full Process Report
This report details the end-to-end process for building the lab, based on the MyDFIR video series and tailored to this specific project's configuration.
1. Project Overview
Objective
To build a functional cyber-security lab to simulate, detect, and analyze adversary techniques using Active Directory, Splunk, Sysmon, Kali Linux, and Atomic Red Team.
Lab Architecture
VM Name		Operating System		IP Address	Purpose
ADDC01		Windows Server 2022		192.168.10.7	Domain Controller (abu.local), DNS Server
SplunkSrv		Ubuntu Server 22.04		192.168.10.10	SIEM / Log Aggregator
target-PC		Windows 10		192.168.10.100	Victim Workstation, joined to domain
Kali		Kali Linux		192.168.10.250	Attacker Machine

2. Phase 1: VM Creation & Network Setup
1.	Download OS Images:
o	Windows Server 2022 (Evaluation ISO)
o	Windows 10 Enterprise (Evaluation ISO)
o	Ubuntu Server 22.04 LTS (ISO)
o	Kali Linux (VirtualBox 64-bit Image)
2.	Install VirtualBox:
o	Installed VirtualBox 7.x and the accompanying Extension Pack.
3.	Create Virtual Network:
o	In VirtualBox, navigated to File > Tools... > Network Manager.
o	Created a new NAT Network with the following settings:
Network Name: AD-Project
Network CIDR: 192.168.10.0/24
DHCP: Enabled
4.	Install Virtual Machines:
o	Created four new VMs (ADDC01, SplunkSrv, target-PC, Kali) with appropriate RAM (4GB+) and Disk Size (40GB+).
o	For the three Windows/Ubuntu VMs, installed the OS from the downloaded ISOs.
o	For Kali, imported the downloaded .ova file.
o	Attached all four VMs' network adapters to the AD-Project NAT Network.
3. Phase 2: Splunk Server Configuration (SplunkSrv)
1.	Set Static IP:
o	Logged into the Ubuntu server (abu@ubuntuserver).
o	Edited the netplan config file: sudo nano /etc/netplan/00-installer-config.yaml.
o	Set the configuration for a static IP:
o	network:
o	  ethernets:
o	    enp0s3:
o	      dhcp4: no
o	      addresses: [192.168.10.10/24]
o	      routes:
o	        - to: default
o	          via: 192.168.10.1
o	      nameservers:
o	        addresses: [8.8.8.8]
o	  version: 2
o	Applied the settings: sudo netplan apply.
2.	Install Splunk Enterprise:
o	Downloaded the Splunk Enterprise .deb file from splunk.com.
o	To get the file into the VM, a VirtualBox Shared Folder was set up:
Installed guest utilities: sudo apt-get install virtualbox-guest-utils
Added user to the group: sudo adduser abu vboxsf (requires reboot)
Mounted the share: sudo mount -t vboxsf [Share_Name] [mount_point]
o	Installed Splunk from the shared file: sudo dpkg -i splunk-enterprise-....deb
o	Started Splunk and accepted the license: sudo /opt/splunk/bin/splunk start --accept-license
o	Enabled Splunk to start on boot: sudo /opt/splunk/bin/splunk enable boot-start -user splunk
3.	Configure Splunk Server:
o	Logged into the web UI at http://192.168.10.10:8000.
o	Enable Receiving: Navigated to Settings > Forwarding & Receiving > Configure Receiving and enabled port 9997.
o	Create Index: Navigated to Settings > Indexes and created a new index named endpoint.
4. Phase 3: Domain Controller Setup (ADDC01)
1.	Rename Server:
o	In Server Manager or PowerShell, renamed the server to ADDC01 and rebooted.
2.	Set Static IP & DNS:
o	Configured the server's network adapter with its static IP:
IP: 192.168.10.7
Subnet: 255.255.255.0
Gateway: 192.168.10.1
DNS: 8.8.8.8 (This is temporary, before AD is installed).
3.	Install Active Directory Role:
o	In Server Manager, selected Manage > Add Roles and Features.
o	Installed the Active Directory Domain Services (AD DS) role.
4.	Promote to Domain Controller:
o	After installation, clicked the notification flag and selected "Promote this server to a domain controller."
o	Selected "Add a new forest" and set the Root domain name: abu.local.
o	After the promotion and reboot, the server's DNS was automatically set to 127.0.0.1.
o	Configured DNS Forwarders: In the DNS Manager tool, went to server Properties > Forwarders, and added 8.8.8.8 to allow the domain to resolve external addresses.
5.	Create Users & OUs:
o	Opened Active Directory Users and Computers (ADUC).
o	Created two new Organizational Units (OUs): IT and HR.
o	Created two new users:
Aryan Singh (asingh) in the IT OU.
Akansha Priya (apriya) in the HR OU.
o	Set a complex password for both users (e.g., PassWord@123).
5. Phase 4: Endpoint Configuration (target-PC)
1.	Rename PC:
o	Renamed the Windows 10 VM to target-PC and rebooted.
2.	Install Sysmon:
o	Downloaded Sysmon from Microsoft.
o	Downloaded a community configuration file (e.g., Olaf Hartong's) and named it sysmonconfig.xml.
o	Installed from an administrative command prompt: sysmon64.exe -accepteula -i sysmonconfig.xml
3.	Install Splunk Universal Forwarder:
o	Installed the .msi package.
o	During setup, for "Receiving Indexer," entered 192.168.10.10:9997.
o	Created the inputs.conf file at C:\Program Files\SplunkUniversalForwarder\etc\apps\search\local\inputs.conf with the logging stanzas for Application, Security, System, Microsoft-Windows-Sysmon/Operational, and Microsoft-Windows-PowerShell/Operational, all pointing to index = endpoint.
o	Critical: Opened services.msc, right-clicked SplunkForwarderService, went to Properties > Log On, and changed the account to "Local System account". This is required for the forwarder to have permission to read all event logs.
o	Restarted the SplunkForwarderService.
4.	Join PC to Domain:
o	Set the target-PC's network adapter Preferred DNS to 192.168.10.7 (the AD server).
o	Went to System Properties > Change... and joined the domain abu.local.
o	Rebooted and logged in as the domain user abu\asingh.
6. Phase 5: Attack Simulation & Detection
Attack 1: RDP Brute Force (T1110.001)
1.	Kali Setup:
o	Configured the Kali VM with a static IP (192.168.10.250) and DNS (192.168.10.7) using nmcli.
o	Prepared a small password list, passwords.txt, containing PassWord@123.
2.	Target Setup (target-PC):
o	As an administrator, enabled Remote Desktop.
o	In SystemPropertiesAdvanced > Remote, unchecked the box for "Allow connections only from... with Network Level Authentication (NLA)" to allow older tools to connect.
o	In Windows Defender Firewall, allowed the "Remote Desktop" app through for the "Domain" profile.
3.	Attack (Kali):
o	Used Hydra to simulate the attack (as crowbar failed due to protocol issues).
o	Command: hydra -V -t 4 -l asingh -P passwords.txt rdp://192.168.10.100
o	Result: [3389][rdp] host: 192.168.10.100 login: asingh password: PassWord@123
4.	Detection (Splunk):
o	Hunted for the behavior of a brute-force attack (a high volume of failed logins).
o	Splunk Query:
o	index=endpoint EventCode=4625
o	| stats count by "Source Network Address", "Account Name" 
o	| sort - count
o	Result: The query returned 21 failed login attempts (EventCode 4625) from the Kali IP (192.168.10.250) against the asingh account, successfully identifying the attack.
Attack 2: Atomic Red Team (T1136.001 - Local Account)
1.	Target Setup (target-PC):
o	Defender Exclusion: Add-MpPreference -ExclusionPath "C:\"
o	PowerShell Policy: Set-ExecutionPolicy Bypass -Force
o	Install Atomic Red Team:
o	IEX (IWR [https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1](https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1) -UseBasicParsing);
o	Install-AtomicRedTeam -getAtomics
o	(Note: This required manually installing the powershell-yaml module via Install-Module powershell-yaml -Force).
2.	Attack (target-PC):
o	Loaded the module: Import-Module AtomicRedTeam
o	Ran the test battery for T1136.001: Invoke-AtomicTest T1136.001
o	Result: The test output showed the successful creation, escalation (added to Administrators group), and deletion of a user named Newlocaluser.
3.	Detection (Splunk):
o	Hunted for the specific Event Codes related to this activity.
o	Query 1 (Creation): index=endpoint host=target-PC EventCode=4720 TargetUserName=Newlocaluser
o	Query 2 (Added to Admin): index=endpoint host=target-PC EventCode=4732 Group_Name=Administrators MemberName=Newlocaluser
o	Query 3 (Deletion): index=endpoint host=target-PC EventCode=4726 TargetUserName=Newlocaluser
o	Result: All three queries returned events, successfully showing the entire lifecycle of the attacker's persistence attempt.
7. Conclusion
The lab environment was successfully built, configured, and hardened. All setup steps were completed, resulting in a stable platform. We successfully simulated two distinct adversary techniques (RDP brute-force and local account creation) and subsequently detected the full event chain for both attacks using targeted Splunk queries. The lab is now fully operational for further detection engineering and threat hunting exercises.

