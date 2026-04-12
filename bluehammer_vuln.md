# BlueHammer Vulnerability Utilization and Analysis

Welcome! I am going to be using and analyzing the BlueHammer LPE (Local Privilege Exploit) vulnerability released on April 3, 2026.
This exploit was found by a cybersecurity researched that goes by "Nightmade Ecplise" who found an exploit that abuses Defneders update process with shadow vloume abuse, this allows a non-privileged user to escalate their privilege to NT AUTHORITY\SYSTEM.

The exploit code is a PoC code made public on his github:
 https://github.com/Nightmare-Eclipse/BlueHammer

My first step was to spin up a Win11 VM in my proxmox Host server:
![alt text](Pictures/bluehammer_VM.png)

My next step was to download download the vulnerability on my device, and load the project into Visual Studio so that i could package it ####as a singular .exe that would pass the runtime enviroment on the device. I will move this .exe over to my kali linux machine.
![alt text](Pictures/bluehammer_files.png)

Setup a verified non-admin user on the test VM:
![alt text](Pictures/bluehammer_nonadmin_user.png)
![alt text](Pictures/bluehammer_nonadmin_desktop.png)

Once inside of the host device, I will push this payload and execute it as the user to escalate my privileges to NT AUTHORITY\SYSTEM.

Here is the exploit running in action:

<div style="left: 0; width: 100%; height: 0; position: relative; padding-bottom: 56.25%;"><iframe src="https://www.youtube.com/embed/8b7tB3D6kAU?rel=0" style="top: 0; left: 0; width: 100%; height: 100%; position: absolute; border: 0;" allowfullscreen scrolling="no" allow="accelerometer *; clipboard-write *; encrypted-media *; gyroscope *; picture-in-picture *; web-share *;" referrerpolicy="strict-origin"></iframe></div>

This exploit utilizes 3 Windows system operations with perfect timing to be able to enumerate User NTLM hashes. This allows attackers to escalate their privilege as shown in the video above.

During certain Defender updates/remediations defender will create a Volume Shadow Copy as a privileged side-door into the filesystem. By utilizing the Cloud File Callback system, attackers can paushe defender at the exact precise moments when the volume shadow copy is still mounted and accessible.
This exploit, purely abuses the systems by the way they are operationally and system-timed designed. By placing two seperate Operational Locks (Oplocks), pausing defender at the precise moment allows attackers to gain access to NTLM Hashes so that they can crack them and escalate their privileges.

The program searches for a defender update, pulls the update in and packages the .CAB files.

The attackers trick Windows Defender into installing a malicious update. They do this by sending a fake command to one of Defender's internal functions, while using Windows folder shortcuts (NTFS junctions) to hijack the update path and force the system to pull files from the attacker's directory.

The program then triggers a Defender scan by creating a temporary directory with an EICAR test malware file. This malware file also opens RstrtMgr.dll with a batch oplock. This is the first tripwire to tell the program that defender is actively creating a Volume Shadow Copy.

