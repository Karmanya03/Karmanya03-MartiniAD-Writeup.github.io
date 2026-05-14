---
title: "Pwning MartiniAD"
date: 2026-05-14 12:00:00 +0530
categories: [Writeups, Active Directory]
tags: [kerberoasting, winrm, dcsync, pentesting, hack-smarter]
---

Let's talk about MartiniAD. I recently tackled this Active Directory box, and honestly, the attack path was a beautiful disaster of stacked misconfigurations. It started with credentials left out in the open, escalated through a classic Kerberoast, and ended because a Domain Admin basically left their password on a sticky note in the terminal history.

Here is the full breakdown of how I grabbed the KRBTGT hash and took over the `DRY.MARTINI.BARS` domain.

## 1. Recon and the "Grocery List" Intel

**Target IP:** `10.1.194.66`

I kicked things off with standard unauthenticated network enumeration. The usual AD suspects were all hanging out: DNS (53), Kerberos (88), RPC (135), SMB (445), LDAP (389), and WinRM (5985).

Poking around the anonymous SMB shares, I stumbled onto a world-readable text file named `notes.txt`. Someone literally left their to-do list right above a set of cleartext credentials. Priorities, right?

```bash
cat notes.txt
Plaintext
* Order more gin for lakeside
* Look for an engagement ring
* Check that notes works from Linux Mint

creds
mprice:*martini*%
The Fix: If you are playing defense, this is your friendly reminder to lock down file shares with strict access controls. Also, audit your internal shares to make sure domain passwords aren't saved next to reminders to buy gin.
{: .prompt-tip }

2. The Linux Shell Trolls Me
At first, throwing *martini*% at NetExec and LDAP resulted in timeouts and authentication failures. I was scratching my head until I realized the % was just a Linux shell artifact. Because the notes.txt file didn't have a trailing newline, my terminal just slapped a % at the end of the output.

The actual password was simply *martini*. Lesson learned: always check your formatting before blaming the network.

3. Going Old-School with RPC
Automated SMB enumeration was throwing persistent NetBIOS timeouts at me, so I went old-school and pivoted to manual enumeration over RPC. Using rpcclient with the newly validated mprice credentials, I got a stable connection to the SAM database.

Bash
rpcclient -U 'mprice%*martini*' 10.1.194.66
Inside the session, running enumdomusers gave me the domain user list. Two accounts immediately lit up like a Christmas tree:

athena.t0: The ".t0" naming convention is basically a neon sign for a Tier 0 administrative account.

ATHENA_SVC: The "_SVC" suffix screams "service account", making it my primary target for offline password cracking.

4. Kerberoasting a Dirty Martini
With a juicy service account in my sights, I fired off a Kerberoasting attack to request the Kerberos Ticket Granting Service (TGS) ticket for ATHENA_SVC.

Bash
impacket-GetUserSPNs dry.martini.bars/mprice:'*martini*' -dc-ip 10.1.194.66 -request
The Domain Controller happily handed over an RC4 encrypted ticket ($krb5tgs$23$). I saved it locally, tossed it into Hashcat against the rockyou.txt wordlist, and it cracked in under a second. Gotta love RC4.

Cracked Credential: ATHENA_SVC : 1dirtymartini

The Fix: The fact that this cracked instantly highlights a massive flaw. Enforce a minimum password length of 25+ characters for all Service Principal Name (SPN) accounts. Better yet, implement Group Managed Service Accounts (gMSA) so a human doesn't even have to remember (or choose) the password.
{: .prompt-warning }

5. WinRM and the Lazy Admin
Our initial port scan showed WinRM (Port 5985) was wide open. Armed with my shiny new service account credentials, I snagged an interactive PowerShell session right on the Domain Controller.

Bash
evil-winrm -i 10.1.194.66 -u 'ATHENA_SVC' -p '1dirtymartini'
Running whoami /all confirmed I was in. Sadly, ATHENA_SVC didn't have SeBackupPrivilege or Domain Admin rights yet, so a direct DCSync was off the table for the moment.

Looking for a quick local win, I checked the file system for any sensitive leftovers. The PowerShell PSReadLine history file is an absolute goldmine for lazy admin mistakes, and it did not disappoint.

PowerShell
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
A higher-privileged user had previously used the ATHENA_SVC session to forcefully reset the built-in Domain Administrator password and left the exact command in the history file. Ouch.

Plaintext
net user administrator "ebz0yxy3txh9BDE*yeh"
The Fix: Train your IT team on secure admin habits. Passwords absolutely do not belong in CLI arguments. If you want to be safe, tweak your Group Policy to disable PSReadLine history logging on Tier 0 assets to save admins from themselves.
{: .prompt-info }

6. DCSync and Profit
Now rocking valid Domain Administrator credentials (administrator : ebz0yxy3txh9BDE*yeh), I finally had the DCSync privileges needed to replicate directory data.

To finish the box, I used Impacket's secretsdump remotely to specifically extract the NT Hash for the krbtgt account. Checkmate.

Bash
impacket-secretsdump dry.martini.bars/administrator:'ebz0yxy3txh9BDE*yeh'@10.1.194.66 -just-dc-user krbtgt
Final Extracted Asset (KRBTGT NT Hash): 22ebc290e67668629c8d0812662a9c51

And that is how you completely compromise a domain without dropping a single exploit. Just good old-fashioned misconfiguration hunting.
