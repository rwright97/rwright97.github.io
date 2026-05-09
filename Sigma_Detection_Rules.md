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


<p>One of the first commands they may run is:</p>

<pre><code>sudo -l</code></pre>

This command shows which programs the current user is allowed to run with elevated privileges.

In this scenario, the attacker discovers that the user can run /usr/bin/find as root. This is dangerous because find can execute commands through the -exec option.

A potential privilege escalation command may look like this:

<pre><code>sudo find . -exec /bin/sh \; -quit</code></pre>

If successful, this command could spawn a shell running with elevated privileges.

## Lab Setup & Execution

To start, I created a Victim VM running Ubuntu 24.04 on my proxmox server.

I then create a non-privileged local user account on our victim VM, and set a rule to allow the test account to run `sudo` with no password to execute /usr/bin/find.

![alt text](Pictures/TestUser_Rule_Creation.png)

Once that was all setup, I moved over to my kali machine and did some reconnaissance. To save time, I directly scanned the host IP.

![alttext](Pictures/sigma_nmap_scan.png)

I noticed an open ssh port on the victim host, so i decided to compile a username and password wordlist and perform a brute force on the ssh login using hydra.

![alttext](Pictures/sigma_hydra.png)

We have identified credentials that we can use to login to the victim with via ssh.
I will now perform the commands below and look at the accounts sudo permissions,
then execute the second command below to gain access to a privileged shell:

<pre><code>sudo find . -exec /bin/sh \; -quit</code></pre>

![alttxt](Pictures/sigma_privesc.png)

We now have an escalated permissions shell session! So, this attack path is what we will be writing a sigma rule for.

Lastly, I will verify this showed up on logs, I checked inside of journalctl and here it is:

![alttxt](Pictures/sigma_log_validation.png)

## Detection Engineering


<p>
  After completing the attack path, the next step was to think through how this behavior could be detected from a defensive perspective.
  The goal of the detection was to identify suspicious sudo activity where the <code>find</code> binary is used to execute a shell.
</p>

<p>
  This behavior is suspicious because <code>find</code> is a legitimate Linux utility, but when it is executed with elevated privileges and combined with the <code>-exec</code> option, it can be abused to spawn a root shell.
</p>

<h3>Detection Goal</h3>

<p>
  The detection goal is to alert when Linux authentication logs show a user executing <code>/usr/bin/find</code> through <code>sudo</code> with command execution behavior that may indicate privilege escalation.
</p>

<h3>Log Source</h3>

<p>
  The primary log source for this detection is Linux authentication logging. On Ubuntu systems, this activity may appear in <code>/var/log/auth.log</code> or through <code>journalctl</code>.
</p>

<h3>Suspicious Indicators</h3>

<ul>
  <li>Successful SSH login as a low-privileged user</li>
  <li>Use of <code>sudo -l</code> to enumerate privileges</li>
  <li>Execution of <code>/usr/bin/find</code> through <code>sudo</code></li>
  <li>Use of the <code>-exec</code> option</li>
  <li>Execution of a shell such as <code>/bin/sh</code></li>
</ul>


<h3>Sigma Rule Development</h3>

<p>
  Based on the observed attack behavior, I created a Sigma rule that looks for sudo log messages containing the key command elements associated with this privilege escalation technique.
</p>

<pre><code>title: Linux Privilege Escalation Attempt Via Sudo Find Exec
status: experimental
description: Detects possible Linux privilege escalation using sudo find with -exec to spawn a shell.
author: Ridge Wright
logsource:
  product: linux
  service: auth
detection:
  selection:
    message|contains|all:
      - 'sudo'
      - 'find'
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




