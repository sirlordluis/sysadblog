---
title: 'Linux Hardening Tips'
description: 'Hardening linux root user'
pubDate: 'Feb 29 2024'
heroImage: '/blog-placeholder-1.jpg'
---
## Introduction

Learn how systems hardening methods help to protect your networks, hardware and valuable data by reducing your overall threat profile.

Systems hardening is an essential function for both security and compliance and a crucial part of a broader informarion security strategy.
In this guide, i will share some tips to increase the security on a server.

### BTRFS snapshots and system rollbacks

Set up snapshots of a BTRFS file system, add these snapshots to the GRUB boot menu, and gain the ability to rollback an Linux system to an earlier state in case of being attacked by ransomware.


### Disk encryption 

Encrypting the hard drive of a device can save your data in case the device is
stolen. Despite having security measures such as firewalls and antiviru
software, encrypting the disk is a necessary step to safeguard the informatio
stored on our devices

In this tutorial i show how to set up and encrypt volumes in btrfs [Link](https://www.youtube.com/watch?v=xXJ5QNe67hk).

### Remove sudo

There are many ways to escalate privileges by exploiting **sudo**, either by profiting from insecure configuration, or by exploiting the program's vulnerabilities. Instead of using sudo maybe you should use **su** command to switch between users.

Below I list recent vulnerabilities of the sudo command.
<ol>
  <li><a href="https://nvd.nist.gov/vuln/detail/CVE-2021-3156">CVE-2021-3156</a></li>
  <li><a href="https://nvd.nist.gov/vuln/detail/CVE-2019-14287">CVE-2019-14287</a></li>
  <li><a href="https://nvd.nist.gov/vuln/detail/cve-2023-22809">CVE-2023-22809</a></li>
</ol>

### Add another superuser
**Root** is the superuser account in Unix and Linux. It is a user account for administrative purposes, and typically has the highest access rights on the system, but you can add another super user and desactivate **root** in passwd file.

<script async id="asciicast-OJ0FwRV7d3Kgc3PkykPzzdLM8" src="https://asciinema.org/a/OJ0FwRV7d3Kgc3PkykPzzdLM8.js"></script>

This action doesn't break the system because **root** is still there, but you can't access it. The processes that depend on **root** will continue working.

### Desactivate efi partition

LogoFAIL is a set of firmware vulnerabilities in image parsing libraries used to load logos during the device boot process. Exploiting LogoFAIL requires an attacker to have access to the EFI System Partition (ESP) where the logo image is stored.

<script async id="asciicast-Q92Gd9KyrflFptMm3mYt4rp6u" src="https://asciinema.org/a/Q92Gd9KyrflFptMm3mYt4rp6u.js"></script>

### IP deny 

Blocks access to your website from specific IP addresses. This is useful to eliminate annoying internet users who consume a large amount of bandwidth or attempt to hack your page.

I will block IP addresses that are not in Mexico 🇲🇽 using the iptables command and a list of IPs downloaded from [here](https://www.ipdeny.com/ipblocks/).

```console
root@prueba:~$ wget https://www.ipdeny.com/ipblocks/data/countries/mx.zone
root@prueba:~$ (echo "iptables -P INPUT DROP"; cat mx.zone | awk '{print "iptables -t filter -A INPUT -s " $1 " -j ACCEPT"}') > fw.sh
root@prueba:~$ chmod +x fw.sh
root@prueba:~$ ./fw.sh

```