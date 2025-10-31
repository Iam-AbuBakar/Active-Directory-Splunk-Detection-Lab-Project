Active Directory & Splunk Detection Lab: Troubleshooting Report
This document outlines the specific errors encountered and solutions applied during the construction of the Active Directory & Splunk detection lab.
1. VirtualBox UI: Cannot Find "NAT Network" Settings
•	Problem: Following the video guide, the "Network" option was missing from the File > Preferences menu in VirtualBox 7.
•	Symptom: Unable to create the 192.168.10.0/24 NAT network.
•	Diagnosis: The video guide was created with an older version of VirtualBox (v6). In VirtualBox 7, this setting was moved.
•	Solution:
1.	In the main VirtualBox window, click File > Tools....
2.	Select Network Manager from the new tools menu.
3.	Click the NAT Networks tab to create and configure the lab network.
2. Splunk Install Fails: "No space left on device"
•	Problem: The Splunk server (Ubuntu) installation failed during the sudo dpkg -i ... command.
•	Symptom: The terminal showed the error: dpkg-deb: error: paste subprocess was killed by signal (Broken pipe) followed by failed to write (No space left on device).
•	Diagnosis: The default hard drive size for the Ubuntu VM was too small. df -h confirmed the root partition was 83% full. The system was also on an LVM (Logical Volume Management) setup.
•	Solution (LVM Resize):
1.	Shut down the Splunk (Ubuntu) VM.
2.	In VirtualBox, went to File > Tools > Virtual Media Manager and increased the disk size to 40GB+.
3.	Restarted the VM and installed LVM tools: sudo apt-get install cloud-guest-utils.
4.	Expanded the physical partition (sda3): sudo growpart /dev/sda 3
5.	Resized the LVM Physical Volume: sudo pvresize /dev/sda3
6.	Expanded the Logical Volume: sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
7.	Resized the filesystem: sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
8.	Verification: df -h confirmed the root (/) filesystem was successfully expanded.
3. Kali Linux: IP Stuck on Wrong Network
•	Problem: ip a showed an IP of 192.168.56.10 (a "Host-Only" network), even though VirtualBox settings correctly showed "NAT Network: AD-Project". This prevented nmcli from applying the correct static IP.
•	Solution: This was a persistent VirtualBox bug. The VM was deleted and re-created, which resolved the issue.
4. Network: "No Internet Access" on Windows 10 After Setting DNS
•	Problem: After correctly setting the Windows 10 (target-pc) DNS server to the ADDC01 IP (192.168.10.7), the target-pc lost all internet access.
•	Symptom: ping google.com failed with "could not find host."
•	Diagnosis: The ADDC01 DNS server was correctly resolving local abu.local queries but did not know how to resolve external addresses (like google.com).
•	Solution (Configure DNS Forwarding on ADDC01):
1.	Check Forwarders: On ADDC01, confirmed in DNS Manager > Properties > Forwarders that 8.8.8.8 was listed.
2.	Force Forwarder Use: To ensure the server used the forwarder instead of "Root Hints," we navigated to the Root Hints tab and Removed all default entries.
3.	Identify Root Cause: The ping google.com still failed. ipconfig /all on the target-pc showed the DNS setting was correct, but the ADDC01 server was not responding. This was because the ADDC01 VM was powered off or its DNS service was in a bad state.
4.	Final Fix:
Ensured the ADDC01 (Windows Server) VM was powered on.
On ADDC01, opened services.msc and Restarted the "DNS Server" service to apply all changes.
On target-pc, ran ipconfig /flushdns to clear the bad cache.
5.	Result: The target-pc could now resolve both internal (abu.local) and external (google.com) addresses.
5. Splunk Pipeline: "No results found" for target-PC
•	Problem: No data was appearing from the target-pc (Windows 10) host.
•	Diagnosis: A "fishbucket" (checkpoint database) corruption on the Windows 10 forwarder was causing the service to get stuck in an error loop.
•	Solution (Resetting the Fishbucket):
1.	Log Inspection: Inspected the forwarder's log at C:\...SplunkUniversalForwarder\var\log\splunk\splunkd.log.
2.	Root Cause Identified: The log file showed a blocking ERROR: ERROR BTree ... Cannot create a file when that file already exists.
3.	Final Fix:
On the Windows 10 host, open services.msc and Stop the SplunkForwarderService.
Navigate to C:\Program Files\SplunkUniversalForwarder\var\lib\splunk\.
Delete the entire fishbucket folder.
In services.msc, Start the SplunkForwarderService.
4.	Result: The forwarder started fresh and began sending logs.
6. Splunk Pipeline: "No results found" for ADDC01
•	Problem: The ADDC01 (Windows Server) host was not appearing in Splunk, even after the target-pc was fixed.
•	Diagnosis: The ADDC01 log file (splunkd.log) showed the forwarder was successfully connecting to Splunk (Connected to idx=192.168.10.10:9997...) but was not reading any Event Logs.
•	Root Cause Identified: The inputs.conf file was missing from the correct folder, so the forwarder had no instructions on what data to send.
•	Solution (Missing Configuration):
1.	Stop the SplunkForwarderService.
2.	Create (or fix) the inputs.conf file in the correct location: C:\Program Files\SplunkUniversalForwarder\etc\apps\search\local\.
3.	Paste the correct configuration (listing all the WinEventLog stanzas).
4.	Start the SplunkForwarderService.
5.	Result: The forwarder started, read its new instructions, and began sending logs.
7. Splunk Pipeline: Logs Stopped Flowing (Time Sync Issue)
•	Problem: After working initially, logs stopped appearing in "Last 15 minutes" searches.
•	Diagnosis: The clocks on the Windows VMs were 4+ hours behind the host machine and Splunk server. Splunk was ingesting the logs but timestamping them in the past.
•	Root Cause Identified: The ADDC01 (Domain Controller), which acts as the time source for the domain, had an incorrect time zone and time setting. The GUI settings to change time were greyed out.
•	Solution (Fix Time on Domain Controller):
1.	On ADDC01, opened PowerShell as Administrator.
2.	Set the correct Time Zone: tzutil /s "India Standard Time"
3.	Manually set the correct Date and Time: Set-Date -Date "YYYY-MM-DD HH:MM:SS"
4.	On target-PC, manually synced the clock: Date & Time settings > Sync now.
5.	On both ADDC01 and target-PC, Restarted the SplunkForwarderService.
6.	Result: Clocks across all VMs were synchronized, and recent logs appeared in Splunk.
8. RDP Attack: crowbar Tool Fails
•	Problem: The crowbar tool failed instantly (No results found...).
•	Diagnosis: An nmap -p 3389 192.168.10.100 scan showed the port was open. This indicated the failure was due to Network Level Authentication (NLA), which crowbar does not support.
•	Solution: Switched to a more modern tool, Hydra, which successfully completed the attack.
9. Atomic Red Team: Installation Fails
•	Problem: The Install-AtomicRedTeam script failed, erroring on No match was found for... 'powershell-yaml'.
•	Diagnosis: A required dependency (powershell-yaml) could not be found or installed by the main script.
•	Solution:
1.	Manually registered the PowerShell repository: Register-PSRepository -Default -InstallationPolicy Trusted
2.	Manually installed the dependency: Install-Module powershell-yaml -Force
3.	Re-ran Install-AtomicRedTeam -getAtomics, which then completed successfully.

