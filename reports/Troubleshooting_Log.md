Active Directory & Splunk Detection Lab: Troubleshooting Log
This document logs all errors encountered, diagnostic steps, and solutions implemented during the lab build.
1. VirtualBox: Network Manager Not Found
•	Problem: The "Network" option was missing from File > Preferences, as shown in older tutorials.
•	Diagnosis: The VirtualBox 7.x UI has changed.
•	Solution: The Network Manager was located in the main window under File > Tools > Network Manager.
2. Netplan: netplan apply Warnings
•	Problem: After applying the netplan static IP configuration on the Splunk (Ubuntu) server, warnings appeared: Permissions ... are too open and Cannot call Open vSwitch....
•	Diagnosis: These are non-blocking warnings. The first is a permissions suggestion, and the second is for a service we are not using.
•	Solution: Ignored the warnings. Verified the static IP was set correctly using ip a.
3. VirtualBox: Shared Folder Mount Failed
•	Problem: When trying to mount the VirtualBox Shared Folder, the command sudo mount -t vboxsf ... failed with No such file or directory.
•	Diagnosis: The vboxsf filesystem driver was not loaded, likely because the user was not in the correct group or Guest Additions were incomplete.
•	Solution:
1.	Installed Guest Additions utilities: sudo apt-get install virtualbox-guest-utils
2.	Added the user to the vboxsf group: sudo adduser $USER vboxsf
3.	Rebooted the VM to apply the group changes. The mount command succeeded after the reboot.
4. Splunk Install: No space left on device (LVM Resize)
•	Problem: sudo dpkg -i splunk... failed with dpkg-deb: error: paste subprocess was killed by signal (Broken pipe) and failed to write (No space left on device).
•	Diagnosis: The default disk size for the Ubuntu VM was full. df -h confirmed the root partition was 100% used.
•	Solution: This was a complex, multi-stage fix for an LVM-based filesystem.
1.	Increase Disk Size: Shut down the VM. In VirtualBox, went to File > Tools > Virtual Media Manager, selected the VM's .vdi file, and increased its size (e.g., from 12G to 90G).
2.	Boot VM: Started the VM. The new space was "unallocated."
3.	Find Partition: Ran lsblk to identify the disk (/dev/sda) and the LVM partition (/dev/sda3).
4.	Install Tools: sudo apt-get install cloud-guest-utils
5.	Expand Partition: sudo growpart /dev/sda 3
6.	Resize LVM Physical Volume: sudo pvresize /dev/sda3
7.	Expand LVM Logical Volume: sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
8.	Resize Filesystem: sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
9.	Verification: df -h confirmed the root partition was resized and had free space. The Splunk installation then completed successfully.
5. Splunk Data: Logs Not Showing for target-PC
•	Problem: After configuring the Splunk Forwarder, no logs from target-PC appeared in Splunk.
•	Diagnosis: Checked the forwarder logs at C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log. Found the error: ERROR BTree ... reading one headers failed: Cannot create a file when that file already exists.
•	Solution: This error indicates a corrupted fishbucket (the forwarder's checkpoint database).
1.	On target-PC, opened services.msc and Stopped the SplunkForwarderService.
2.	Navigated to C:\Program Files\SplunkUniversalForwarder\var\lib\splunk\.
3.	Deleted the entire fishbucket folder.
4.	Started the SplunkForwarderService. Logs began to flow immediately.
6. Splunk Data: Logs Not Showing for ADDC01
•	Problem: Logs from target-PC were working, but logs from ADDC01 were still missing.
•	Diagnosis: Checked the ADDC01 forwarder log. It showed successful connections (Connected to idx=192.168.10.10:9997...) but was not reading any event logs.
•	Solution: The inputs.conf file was missing. The Splunk Forwarder was installed but had no instructions.
1.	Created the inputs.conf file at C:\Program Files\SplunkUniversalForwarder\etc\apps\search\local\inputs.conf.
2.	Pasted the correct configuration (monitoring Security, System, Sysmon, etc.).
3.	Restarted the SplunkForwarderService. Logs began to flow immediately.
7. Active Directory: Password Complexity Error
•	Problem: When creating new users (asingh) in ADUC, an error appeared: "Windows cannot set the password... The password does not meet the password policy requirements."
•	Diagnosis: The simple password being used (e.g., "password") was being rejected by the Default Domain Password Policy.
•	Solution: Used a complex password that met the requirements (e.g., PassWord@123!).
8. Active Directory: Domain Join Fails & No Internet
•	Problem: target-PC could not join abu.local ("domain controller could not be contacted"). After changing the PC's DNS to point to the DC (192.168.10.7), the domain join worked, but the PC lost all internet access.
•	Diagnosis: The ADDC01 server (acting as DNS) did not know how to resolve external addresses like google.com.
•	Solution:
1.	On ADDC01, opened Server Manager > Tools > DNS.
2.	Right-clicked the server > Properties > Forwarders tab.
3.	Added 8.8.8.8 as a forwarder.
4.	Troubleshooting: Internet was still not working.
5.	Fix: On the ADDC01 DNS Properties, went to the Root Hints tab and Removed all entries. This forced the server to use the 8.8.8.8 forwarder.
6.	Troubleshooting: Still not working.
7.	Final Fix: On ADDC01, opened services.msc and Restarted the "DNS Server" service.
8.	On target-PC, ran ipconfig /flushdns. Internet access was successfully restored.
9. Kali VM: Stuck on Wrong Network IP
•	Problem: After setting the Kali VM's network to "NAT Network," the ip a command still showed an old IP from a "Host-Only" network (192.168.56.10/24).
•	Diagnosis: This was a persistent VirtualBox bug where the VM's configuration was "stuck" and not reflecting the GUI settings, even after reboots.
•	Solution: The VM was too corrupted. Deleted the Kali VM and re-imported a fresh one, which immediately attached to the correct AD-Project NAT Network.
10. Attack 1: crowbar (RDP) Fails Instantly
•	Problem: The crowbar -b rdp ... command failed instantly. nmap confirmed port 3389 was open.
•	Diagnosis: The RDP service was protected by Network Level Authentication (NLA), which crowbar does not support.
•	Solution:
1.	On target-PC, went to System Properties > Remote and unchecked "Allow connections only from computers running Remote Desktop with Network Level Authentication."
2.	Troubleshooting: crowbar still failed.
3.	Final Fix: Switched to a modern tool that supports RDP. Hydra (hydra -V -l asingh -P ... rdp://...) worked perfectly.
11. Atomic Red Team: Install Fails (Missing Dependency)
•	Problem: Install-AtomicRedTeam failed with No match was found for... 'powershell-yaml' and Import-Module : The required module 'powershell-yaml' is not loaded.
•	Diagnosis: The script's primary dependency was missing.
•	Solution:
1.	Ensured the PSGallery repository was trusted: Register-PSRepository -Default -InstallationPolicy Trusted
2.	Manually installed the dependency: Install-Module powershell-yaml -Force
3.	Re-ran Install-AtomicRedTeam -getAtomics, which completed successfully.
12. Splunk Data: Logs Suddenly Stop (Time Sync)
•	Problem: After running for a while, logs from all hosts stopped appearing in Splunk in "Last 15 minutes."
•	Diagnosis:
1.	Checked Settings > Licensing: License was fine, 0.02GB of 500MB used.
2.	Checked splunkd.log on target-PC. Found a critical warning: System time went forwards by 109990.062 seconds.
3.	This confirmed a massive time synchronization issue. The host (Oct 30) and VMs (Oct 29) were on different days. Splunk was receiving logs with old timestamps and (correctly) not showing them in a "Last 15 minutes" search.
•	Solution:
1.	On ADDC01, the GUI for changing time was greyed out ("Managed by your organization").
2.	Opened Admin PowerShell on ADDC01 and forced the time:
	tzutil /s "India Standard Time"
	Set-Date -Date "2025-10-30 00:35:00" (or current time)
3.	On target-PC, went to "Date & time" settings and clicked "Sync now" to sync with the now-correct Domain Controller.
4.	Restarted the SplunkForwarderService on all Windows VMs. Logs immediately resumed flowing with correct timestamps.
13. Atomic Red Team: Attacks Fail (Windows Defender)
•	Problem: Invoke-AtomicTest commands (T1082, T1055, T1003) would fail silently, or with "Access is denied" errors. No corresponding EventCode=1, EventCode=8, or EventCode=10 logs were created in Splunk.
•	Diagnosis: A "Virus & threat protection" pop-up was seen. Windows Defender was blocking the behavior of the scripts (e.g., process injection, LSASS access) even though "Real-time protection" was off.
•	Solution:
1.	On target-PC, navigated to "Virus & threat protection" > "Manage settings".
2.	Turned OFF all layers of protection:
	Real-time protection
	Cloud-delivered protection
	Automatic sample submission
	Tamper Protection
3.	Re-ran the Invoke-AtomicTest commands. The attacks now executed successfully, and the corresponding detection logs (EventCode=1, EventCode=8, EventCode=4104) appeared in Splunk as expected.

