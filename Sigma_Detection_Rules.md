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

```bash
sudo -l
