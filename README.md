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
### 1. Log Aggregation and File Integrity Monitoring (FIM)
- The Wazuh Agents are deployed on the Windows and LinuxMint endpoints to collect system events.
- The 'ossec.conf' files were configured manually to collect real-time telemetry in a directory and forward it to the Wazuh Manager, this is File Integrity Monitoring:
  
In the Windows machine, I configured it to look in the Public directory:
 <img width="724" height="339" alt="image" src="https://github.com/user-attachments/assets/e5ea19d2-ee8b-4864-a676-0746b13388bb" />
 
In the LinuxMint machine, I configured it to look in a directory I created in tmp/wazuhTest:
<img width="822" height="254" alt="image" src="https://github.com/user-attachments/assets/50d2d0ed-9afb-4ea3-b6bb-58c37b9b1281" />

- In the 'tmp/wazuhTest/' directory in the LinuxMint machine, I created a text file that just says "hello":
<img width="515" height="39" alt="image" src="https://github.com/user-attachments/assets/e8d48636-1d91-4677-a6f3-869a1b713f52" />

- This is what's shown in the Wazuh dashboard in the Ubuntu machine, showing that the file I created was harmless:
<img width="1839" height="505" alt="image" src="https://github.com/user-attachments/assets/13d17e0b-6580-4dc0-9d9e-5ed1124f1b35" />

- I can also see what I wrote in the text file:
<img width="483" height="83" alt="image" src="https://github.com/user-attachments/assets/c3222b8f-c0e6-42fd-be29-eeea3e1af84a" />

### 2. Malware Detection with VirusTotal API
- Just having Wazuh provide logs about changes in a directory isn't enough, so I wanted a way for it to detect when malware was added to the system.
- Integrated the FIM module with the VirusTotal API. File hashes are cross-checked with VirusTotal's database and alerts when a known malware signature is detected.

This snippet is located in the 'ossec.conf' file of the Wazuh Manager (Ubuntu Machine):
<img width="853" height="191" alt="Screenshot 2026-01-18 011858" src="https://github.com/user-attachments/assets/28bf3479-69c7-4690-9b4a-a401d605ebc8" />

### 3. Active Response (on LinuxMint)
- Now the Wazuh manager raises alerts when malware is detected in a directory, there is no reason to keep them in the system for longer than they should be.
- This process automatically deletes any known malware detected.

A 'remove-threat.sh' script was created in the 'active-response' directory of the LinuxMint machine:
<img width="848" height="104" alt="image" src="https://github.com/user-attachments/assets/756d2dc9-1640-4f79-aba3-382f091f92cf" />

The script itself is provided in the [Official Wazuh Documentation](https://documentation.wazuh.com/current/proof-of-concept-guide/detect-remove-malware-virustotal.html):
<img width="950" height="670" alt="image" src="https://github.com/user-attachments/assets/458fa3f0-063a-4418-9262-c21cfad4d038" />

This snippet is found in the 'ossec.conf' file of the Wazuh Manager machine, inferring the 'remove-threat.sh' script when a file with rule_id=87105 is found, meaning that it's guaranteed malware:
<img width="633" height="374" alt="image" src="https://github.com/user-attachments/assets/30d7c61a-d2e1-493c-9f90-2b56bff072ed" />

To test with an actual malware, I will download a sample from EICAR into the wazuhTest directory in the LinuxMint machine, which isn't actually malware, but will trigger the VirusTotal detection and subsequently the auto-deletion:
<img width="796" height="566" alt="image" src="https://github.com/user-attachments/assets/bbf4e462-9e45-4a91-980b-0998c1177083" />
<img width="946" height="90" alt="image" src="https://github.com/user-attachments/assets/cdad2545-4dd8-4a9f-831d-54671dde38ba" />
<img width="803" height="567" alt="image" src="https://github.com/user-attachments/assets/17971163-f65d-4baa-8649-aae8f185df44" />

### 4. Custom Rule Detection (sudo commands)
Right after setting up Wazuh, I realized that a lot of events were classified as low severity, even when running a command like 'sudo bash' on my LinuxMint machine to perform privilege escalation.

<img width="598" height="380" alt="image" src="https://github.com/user-attachments/assets/8932974d-6374-4855-a7f3-2a79a572d610" />
<img width="595" height="676" alt="image" src="https://github.com/user-attachments/assets/4562a3fd-629d-4fe1-b573-3a67a6725425" />

So I wanted to create a custom rule that puts sudo commands high on the severity list. 

First things first, I examined the logs to find which one was for the sudo command I ran. 

<img width="695" height="763" alt="image" src="https://github.com/user-attachments/assets/c446a5eb-fd48-49f7-91bb-2e3ff9420966" />
<img width="599" height="539" alt="image" src="https://github.com/user-attachments/assets/f4ce9b0f-f6f1-40e0-94b2-5a10fbada008" />

A couple things to take note here:
- The current Rule Level, which is 3, putting it low on the severity scale
- The Rule ID, which is the unique identifier for this kind of event (running sudo command)

Within the Wazuh Manager machine, I went into the 'local_rules.xml' file and added a rule to put sudo command events on Rule Level 12, which high severity.
(i am aware of the typo, i meant to write 'custom' instead of 'custon', but I live with my sins)

<img width="600" height="425" alt="image" src="https://github.com/user-attachments/assets/c603ab51-cef5-4d9a-a3bc-245b6518ab16" />

After restarting the Wazuh Manager, I ran 'sudo bash' on my linux machine again, and what do you know, I immediately see 1 High Alert event recorded, and I can easily examine it. 

<img width="598" height="640" alt="image" src="https://github.com/user-attachments/assets/3501e176-dedb-4975-9ed8-65f39f804c17" />
<img width="598" height="695" alt="image" src="https://github.com/user-attachments/assets/65aa5376-3ebc-4a8f-a176-a4282ec19121" />

## Attacks
### 1. SSH Brute Force using Kali Linux and Hydra

Here I attempted an SSH brute force attack on my LinuxMint machine (192.168.0.9) using Kali Linux and Hydra. 

Scanning the services on the target machine I can see that port 22 is open. 
<img width="624" height="158" alt="image" src="https://github.com/user-attachments/assets/51e74a46-67f6-427b-8206-90f1d7a72598" />

Kali Linux comes with a .txt file called rockyou that contains a very extensive list of passwords
<img width="568" height="108" alt="image" src="https://github.com/user-attachments/assets/8871c61c-324a-4b9c-8a92-9bce9ef5d7c0" />

I ran this command in the Kali Linux terminal to perform an SSH brute force attack to my target LinuxMint machine to login
to the root user.
`hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.0.9 -t 4`

What this does is brute force login to the root user of the LinuxMint machine using every passwords stored in the .txt file. 

Looking at my Wazuh dashboard, I can see that events in the Medium and Low severity scale are increasing rapidly. 
<img width="625" height="382" alt="image" src="https://github.com/user-attachments/assets/d0661666-ff35-49f1-b69b-3036960ef55f" />
<img width="623" height="423" alt="image" src="https://github.com/user-attachments/assets/525ae247-2ca7-4fc4-aec8-4012ac37fcb9" />

A sudden spike of events have suddenly appeared in my Wazuh dashboard, with the source coming from the LinuxMint machine. A lot of these alerts (possibly hundreds) are Level 5 on the severity scale, and have a rule id of 5760, which only indicates that someone or something inputted an incorrect password. This is hard to scroll through.

<img width="610" height="713" alt="image" src="https://github.com/user-attachments/assets/ea2c29a4-d3cf-410f-bc6a-21d702b89cd7" />

From the perspective of an analyst, how do we actually know whether this is an actual brute force attack or just someone who cannot remember their password? The hundreds of alerts could signal bot-driven behavior, but we want confirmation. 

Doing some research (a google search), we can see that the Wazuh rule id for an SSH-brute force attempt is 5763. 

<img width="624" height="286" alt="image" src="https://github.com/user-attachments/assets/4b3f0a48-bb6b-43bf-af08-51d06d2546d5" />

Now I'm gonna filter my logs for rule 5763 specifically.

<img width="618" height="518" alt="image" src="https://github.com/user-attachments/assets/df17fe0a-f00a-446f-a24b-ec30ea701ab5" />

Now I can confirm that this is a brute force attack

<img width="616" height="419" alt="image" src="https://github.com/user-attachments/assets/5b885147-7c07-43b7-aa8f-c9f06da383c3" />

I can even see further details, such as the attacker trying to log into the root user in the LinuxMint machine.

<img width="623" height="266" alt="image" src="https://github.com/user-attachments/assets/78f07900-9e5b-460f-b349-26613b985535" />










