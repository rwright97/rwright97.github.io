# Sigma Detection Rule Write-Up: Linux Privilege Escalation via Sudo Find

## Overview

This write-up documents a Sigma detection concept for identifying suspicious Linux privilege escalation behavior involving the abuse of the `find` binary through `sudo`.

In Linux environments, privilege escalation often comes from misconfigured permissions rather than complex exploits. One common example is when a low-privileged user is allowed to run certain binaries as root using `sudo`. If those binaries can execute system commands, an attacker may be able to abuse them to spawn a root shell.

The `find` command is one example of a legitimate Linux utility that can become dangerous when granted elevated permissions. Because `find` supports the `-exec` option, it can be used to execute another command, including a shell such as `/bin/sh`.

This write-up walks through the attack concept, why the behavior is risky, and how a Sigma rule could be used to detect suspicious usage in Linux authentication logs.

## Lab Objective

The objective of this lab is to document a detection engineering use case for identifying possible privilege escalation attempts using `sudo find -exec`.

This lab focuses on:

- Understanding how `find` can be abused for privilege escalation
- Identifying suspicious command patterns in Linux logs
- Writing Sigma detection logic
- Mapping the activity to MITRE ATT&CK
- Recommending defensive controls to reduce risk

## Attack Scenario

An attacker gains access to a low-privileged Linux account and begins enumerating the system for privilege escalation opportunities. One of the first commands they may run is:

<pre><code>sudo -l</code></pre>

This command shows which programs the current user is allowed to run with elevated privileges.

In this scenario, the attacker discovers that the user can run /usr/bin/find as root. This is dangerous because find can execute commands through the -exec option.

A potential privilege escalation command may look like this:

<pre><code>sudo find . -exec /bin/sh \; -quit</code></pre>

If successful, this command could spawn a shell running with elevated privileges.

## Lab Setup & Execution

To start, I created a Victim VM running Ubuntu 24.04 on my proxmox server.

I then created a non-privileged local user account on our victim VM, and set a rule to allow the test account to run `sudo` with no password to execute /usr/bin/find.

![alt text](Pictures/TestUser_Rule_Creation.png)

Once that was all setup, I moved over to my kali machine and did some reconnaissance. To save time, I directly scanned the host IP.

![alttext](Pictures/sigma_nmap_scan.png)

I noticed an open ssh port on the victim host, so I decided to compile a username and password wordlist and perform a brute force on the ssh login using hydra.

![alttext](Pictures/sigma_hydra.png)

We have identified credentials that we can use to login to the victim with via ssh.
I will now perform the commands below and look at the accounts sudo permissions,
then execute the second command below to gain access to a privileged shell:

<pre><code>sudo -l</code></pre>
<pre><code>sudo find . -exec /bin/sh \; -quit</code></pre>

![alttxt](Pictures/sigma_privesc.png)

We now have an escalated permissions shell session! So, this attack path is what we will be writing a sigma rule for.

## Detection Engineering and Sigma Rule Validation

<p>
  After completing the attack path, I reviewed the victim machine's journalctl logs to identify what evidence was created by the activity. 
  The goal was to determine which log artifacts could be used to detect the privilege escalation behavior from a defensive perspective.
</p>

![alttxt](Pictures/sigma_log_validation.png)

<p>
  The most important log evidence was the sudo command showing <code>labuser</code> executing <code>/usr/bin/find</code> with elevated privileges. 
  Since the attack used the <code>-exec</code> option to launch <code>/bin/sh</code>, those command elements became the basis for the detection logic.
</p>

<h3>Detection Goal</h3>

<p>
  The goal of this detection is to identify suspicious sudo activity where the <code>find</code> binary is used to execute a shell. 
  While <code>find</code> is a legitimate Linux utility, it can become dangerous when a low-privileged user is allowed to run it as root.
</p>

<h3>Observed Indicators</h3>

<ul>
  <li>SSH authentication activity for <code>labuser</code></li>
  <li>Sudo privilege enumeration using <code>sudo -l</code></li>
  <li>Execution of <code>/usr/bin/find</code> through sudo</li>
  <li>Use of the <code>-exec</code> option</li>
  <li>Execution of <code>/bin/sh</code> from the elevated command</li>
</ul>

<h3>Sigma Rule Creation</h3>

<p>
  Based on the observed log evidence, I created a Sigma rule to detect this behavior in Linux authentication logs.
</p>

<pre><code>title: Linux Privilege Escalation Attempt Via Sudo Find Exec
id: 4d8f5c9a-5fd5-4a9c-9b51-2a73e2d12f42
status: experimental
description: Detects possible Linux privilege escalation using sudo find with -exec to spawn a shell.
author: Ridge Wright
date: 2026-05-09
logsource:
  product: linux
  service: auth
detection:
  selection:
    message|contains|all:
      - 'COMMAND=/usr/bin/find'
      - '-exec'
      - '/bin/sh'
  condition: selection
fields:
  - timestamp
  - hostname
  - user
  - message
falsepositives:
  - Administrative testing
  - Security labs
level: high
tags:
  - attack.privilege-escalation
  - attack.t1548</code></pre>

<h3>Rule Validation</h3>

<p>
  After creating the rule, I used Sigma CLI to validate the rule syntax and confirm that the rule was structured correctly.
</p>

<pre><code>~/.local/bin/sigma check linux_sudo_find_exec_privesc.yml</code></pre>

![alttxt](Pictures/sigma_rule_validation.png)

<p>
  The rule successfully passed validation, confirming that the Sigma rule was syntactically valid and ready to be converted for a supported SIEM backend.
</p>

<h2>MITRE ATT&amp;CK Mapping</h2>

<p>
  This lab maps to several MITRE ATT&amp;CK techniques across initial access, credential access, lateral movement-style remote access, execution, and privilege escalation.
</p>

<table>
  <thead>
    <tr>
      <th>Tactic</th>
      <th>Technique</th>
      <th>ID</th>
      <th>How It Applies to This Lab</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Credential Access</td>
      <td>Brute Force: Password Guessing</td>
      <td>T1110.001</td>
      <td>Hydra was used to guess the SSH password for the low-privileged <code>labuser</code> account.</td>
    </tr>
    <tr>
      <td>Lateral Movement / Remote Access</td>
      <td>Remote Services: SSH</td>
      <td>T1021.004</td>
      <td>The attacker authenticated to the Ubuntu victim machine over SSH using the discovered credentials.</td>
    </tr>
    <tr>
      <td>Execution</td>
      <td>Command and Scripting Interpreter: Unix Shell</td>
      <td>T1059.004</td>
      <td>The attack used <code>/bin/sh</code> to spawn an interactive shell on the Linux system.</td>
    </tr>
    <tr>
      <td>Privilege Escalation</td>
      <td>Abuse Elevation Control Mechanism: Sudo and Sudo Caching</td>
      <td>T1548.003</td>
      <td>The attacker abused a sudoers misconfiguration that allowed <code>labuser</code> to run <code>/usr/bin/find</code> as root.</td>
    </tr>
  </tbody>
</table>

<p>
  The primary technique for the Sigma detection rule is <strong>T1548.003 - Abuse Elevation Control Mechanism: Sudo and Sudo Caching</strong>, because the rule focuses on detecting sudo-based privilege escalation through the abuse of <code>/usr/bin/find</code>.
</p>

## SIEM Deployment Considerations

<p>
  This lab focused on writing and validating the Sigma rule rather than deploying it into a production SIEM. 
  In a real environment, this Sigma rule could be converted into a SIEM-specific query using Sigma CLI and then tuned against the organization's Linux authentication logs.
</p>

<h2>Defensive Recommendations</h2>

<p>
  This lab shows how a simple sudoers misconfiguration can create a serious privilege escalation path. 
  The main defensive takeaway is that administrative permissions should be reviewed carefully, especially when they allow users to run binaries that can execute other commands.
</p>

<ul>
  <li>
    Regularly review <code>/etc/sudoers</code> and files inside <code>/etc/sudoers.d/</code> for risky permissions.
  </li>
  <li>
    Avoid granting <code>NOPASSWD</code> permissions unless there is a clear operational need.
  </li>
  <li>
    Do not allow low-privileged users to run command-execution-capable binaries as root, such as <code>find</code>, <code>vim</code>, <code>less</code>, <code>awk</code>, <code>bash</code>, or <code>python</code>.
  </li>
  <li>
    Compare sudo permissions against known GTFOBins techniques to identify binaries that may be abused for privilege escalation.
  </li>
  <li>
    Monitor Linux authentication logs for suspicious sudo activity, especially commands containing <code>-exec</code>, <code>/bin/sh</code>, <code>/bin/bash</code>, or other shell-spawning behavior.
  </li>
  <li>
    Alert on unusual sudo usage by low-privileged users, especially after successful SSH logins from unfamiliar hosts.
  </li>
  <li>
    Use centralized logging or a SIEM to collect Linux authentication logs so suspicious privilege escalation behavior can be detected across multiple systems.
  </li>
  <li>
    Apply the principle of least privilege by only granting users the exact permissions needed for their role.
  </li>
</ul>

<h2>Conclusion</h2>

<p>
  This lab demonstrated a complete attack and detection workflow against an Ubuntu Linux victim machine. 
  The attack path began with SSH credential brute forcing against a low-privileged user account, followed by sudo privilege enumeration and privilege escalation through a misconfigured <code>/usr/bin/find</code> permission.
</p>

<p>
  After completing the attack, I reviewed the victim machine's authentication logs to identify the evidence created by the activity. 
  The sudo log entry showing <code>labuser</code> executing <code>/usr/bin/find</code> with the <code>-exec</code> option provided a clear detection opportunity.
</p>

<p>
  I then created a Sigma rule to detect this behavior and validated the rule using Sigma CLI. 
  This confirmed that the rule was syntactically valid and could be converted for use in a supported SIEM backend.
</p>

<p>
  Overall, this project demonstrates the connection between offensive security testing and defensive detection engineering. 
  By understanding how an attacker abuses a Linux misconfiguration, defenders can build more effective detections, improve logging coverage, and reduce the risk of privilege escalation in production environments.
</p>

