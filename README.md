# SoC-Wazuh-Homelab
A virtual Wazuh SIEM homelab with Ubuntu OS as the manager, and a LinuxMint machine and a Windows machine as the agents. Integrated with VirusTotal API to automatically detect malicious files and deployed a script to delete said files.

This is all setup and run locally on my machine using VMs.

The goal is to have an environment where I can test different types of cyberattacks and analyze what is logged by the SIEM.

## Infrastructure
- NAT Network: 192.168.0.X/24 with Static IP allocation to ensure consistent connectivity.
<img width="2062" height="1509" alt="image" src="https://github.com/user-attachments/assets/78d82eb7-f87c-433b-bee4-0f4ae1c7f054" />

- Hypervisor: VirtualBox
- Wazuh Manager OS: Ubuntu 24.04.3 (192.168.0.8)
- Wazuh Agents OS:
	- Windows 10 (192.168.0.10)
 	- LinuxMint 22.2 (192.168.0.9)
- Attacker Machine: Kali Linux (192.168.0.13)
<img width="824" height="814" alt="image" src="https://github.com/user-attachments/assets/16c7c7df-5f27-4084-a594-67f0fdb32b6a" />

## Implementations
1. Log Aggregation and File Integrity Monitoring (FIM)
- The Wazuh Agents are deployed on the Windows and LinuxMint endpoints to collect system events.
- The 'ossec.conf' files were configured manually to collect real-time telemetry in a directory and forward it to the Wazuh Manager, this is File Integrity Monitoring:
  
In the Windows machine, I configured it to look in the Public directory:
 <img width="724" height="339" alt="image" src="https://github.com/user-attachments/assets/e5ea19d2-ee8b-4864-a676-0746b13388bb" />
 
In the LinuxMint machine, I configured it to look in a directory I created in tmp/wazuhTest:
<img width="822" height="254" alt="image" src="https://github.com/user-attachments/assets/50d2d0ed-9afb-4ea3-b6bb-58c37b9b1281" />

2. Malware Detection with VirusTotal API
- Just having Wazuh provide logs about changes in a directory isn't enough, so I wanted a way for it to detect when malware was added to the system.
- Integrated the FIM module with the VirusTotal API. File hashes are cross-checked with VirusTotal's database and alerts when a known malware signature is detected.

This snippet is located in the 'ossec.conf' file of the Wazuh Manager (Ubuntu Machine):

3. Active Response
- Now the Wazuh manager raises alerts when malware is detected in a directory, there is no reason to keep them in the system for longer than they should be.
- This process automatically deletes any known malware detected.



