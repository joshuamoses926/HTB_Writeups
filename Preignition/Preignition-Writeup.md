# HTB: Preignition — Writeup

**Difficulty:** Very Easy
**OS:** Linux
**Category:** Web / Default Credentials

---

## Introduction

In most environments, web servers play a big part of the infrastructure and the daily processes of many departments. They can sometimes be used strictly internally by employees, but most of the time are public-facing — meaning anyone on the Internet can reach them to retrieve information and files from the hosted web pages. For the most part, the pages hosted on these servers are managed through administrative panels, locked behind a log-in screen.

Think of a common example: standing up a blog with WordPress. Once installed, the site has a public-facing side and a private-facing one — the latter being the admin panel, typically hosted at `www.yourwebsite.com/wp-admin` and locked behind a login screen.

This box walks through discovering exactly that kind of hidden admin panel, and shows what can go wrong when it's left with default credentials.

---

## Enumeration

### 1. Confirm connectivity and get the target IP

As with any engagement, start by verifying connectivity to the target. Grab the IP from the Starting Point lab page:

![Target IP](images-preignition/Target_IP.png)

A couple of successful `ping` replies is enough to confirm the connection — no need to run it for a long time; a quick snippet of the result is often more efficient than waiting on a full report.

### 2. Port scan with Nmap

```
ping 10.129.1.218
sudo nmap -sV 10.129.1.218
```

![Ping and Nmap scan](images-preignition/Nmap_scan.png)

**Results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 80/tcp | open | http | nginx 1.14.2 |

Only a web server is exposed. Time to take a look at what's being served.

### 3. Browsing to the web server

Navigating to `http://10.129.1.218` in the browser:

![Port 80 open — default nginx page](images-preignition/Port_80_open.png)

We're greeted with the default **"Welcome to nginx!"** landing page — meaning the actual application content hasn't been linked from the homepage, or hasn't been configured yet. This is a strong signal to start looking for hidden directories and files rather than relying on links from the page itself.

---

## Directory Enumeration

### Installing Gobuster

Gobuster is a fast directory/file brute-forcing tool. It's written in Go, so Go needs to be installed first:

```
sudo apt install golang-go
```

![Installing Go dependencies (1/2)](images-preignition/Gobuster_Installation_1.png)
![Installing Go dependencies (2/2)](images-preignition/GoBuster_Intallation_2.png)

Then install Gobuster itself:

```
sudo apt install gobuster
```

Confirm it's working and see the available modes:

```
gobuster --help
```

![Gobuster help output](images-preignition/GoBusterHelp.png)

We'll be using `dir` mode — directory/file enumeration.

### Getting a wordlist

We need a wordlist of common directory and file names to brute-force with. We can grab `common.txt` from SecLists:

```
sudo wget https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/common.txt -O /usr/share/wordlists/common.txt
```

### Running Gobuster

Key flags used:
- `dir` — directory busting mode
- `-w` — wordlist to brute-force with
- `-u` — target URL

```
sudo gobuster dir -w /usr/share/wordlists/common.txt -u 10.129.1.218
```

![Running Gobuster against the target](images-preignition/usingGoBuster.png)

**Result:**

```
admin.php   (Status: 200) [Size: 999]
```

A hidden `admin.php` page turns up — status 200, meaning it exists and is directly accessible.

---

## Foothold

### Discovering the admin panel

Navigating to the discovered path:

```
http://10.129.1.218/admin.php
```

![Admin Console Login page](images-preignition/AdminPage.png)

We're presented with an **Admin Console Login** form asking for a username and password.

### Trying default credentials

Normally, a login form like this would call for a proper brute-force attack with a credential list run over an extended period, since we have no context on valid usernames or passwords. But since this is a freshly-installed nginx instance, it's worth first trying the obvious default before reaching for heavier tooling — there's a good chance it was never reconfigured after install:

```
Username: admin
Password: admin
```

![Successfully logged in](images-preignition/loggedin.png)

The login succeeds immediately — the admin panel was left with default credentials.

---

## Obtaining the Flag

Once logged in, the page displays the flag directly:

```
Congratulations! Your flag is: 6483bee07c1c1d57f14e5b0717503c73
```

**Flag:**
```
6483bee07c1c1d57f14e5b0717503c73
```

---

## Summary

| Step | Action | Finding |
|------|--------|---------|
| 1 | `ping` / `nmap -sV` | Port 80/tcp open — nginx 1.14.2 |
| 2 | Browse to root | Default nginx landing page — no visible links |
| 3 | `gobuster dir` | Discovered hidden `admin.php` |
| 4 | Visit `/admin.php` | Found an admin login form |
| 5 | Try `admin:admin` | Default credentials worked — logged in |
| 6 | Read the response | Flag obtained |

**Root cause:** The nginx web server was left running with the default landing page, and a custom admin panel was deployed on top without being reconfigured — critically, still using default `admin:admin` credentials. Combined with the lack of any rate limiting or account lockout, this made the panel trivially accessible.

**Remediation:**
- Change all default credentials immediately after installing any application or admin panel
- Remove or replace default web server landing pages so they don't advertise an unconfigured install
- Enforce strong, unique passwords and consider multi-factor authentication on administrative interfaces
- Rate-limit or lock out repeated failed login attempts
- Avoid exposing admin panels on predictable, discoverable paths without additional access controls (VPN, IP allowlisting, etc.)
